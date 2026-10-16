# Faz 3 — Aktif Push/Pull Hook'ları Hızı Artırdı; Net Bir "Trailing Zero Bytes" Bulgusu (2026-08-28, otonom tur 11)

## Bu turda yapılan

05 numaralı notun iki somut görevinden ikisi de uygulandı:

1. **Host-taraflı debug dump** (`vendor/virglrenderer/src/venus/vkr_ring.c`, `vkr_ring_submit_cmd()`): decoder fatal olduğunda, decode edilmekte olan buffer'ın ilk 64 baytını + `dec->cur` offset'ini hex olarak logluyor. Bu, bu turun en değerli bulgusunu üretti (aşağıda).

2. **Ring için aktif push/pull** (`vn_ring.c`, `vn_renderer_vtest.c`, `vn_renderer.h`): önceki turda SADECE reply-pool için eklenen `sync_now` hook'u artık ring'in kendisi için de kullanılıyor:
   - **Push:** `vn_ring_submit_internal()`'da `vn_ring_store_tail()`'dan hemen sonra, yeni yazılan komut verisini arka plan thread'inin ~200µs'lik bir sonraki turunu beklemeden HEMEN host'a gönderiyor.
   - **Pull:** `vn_ring_wait_seqno()`'nun bekleme döngüsünün HER iterasyonunda (sadece arka plan thread'inin kendi temposuna güvenmek yerine) host'un ilerlemesini aktif olarak çekiyor.
   - Her ikisi de `vtest_shmem_sync_now()` üzerinden, ring'e özel `vtest_shmem_sync_ring_cycle()` fonksiyonunu (zaten alan-sahipliğine saygılı, doğru sıralı) çağırıyor — kör tam-buffer GET kullanmıyor.

## Sonuç: gözle görülür hız artışı, ama YENİ (ya da daha önce nadir görülen) bir veri sorunu ortaya çıktı

Bu değişiklikten sonra ilk çalıştırma **daha önce hiç ulaşamadığımız kadar ileri gitti** (ring head 2308'e kadar ilerledi — onlarca komut başarıyla işlendi) ama sonunda yine `vkCreateInstance resulted in CS error` ile çöktü. Yeni eklenen host-taraflı debug dump'ı sayesinde bu sefer **net bir kanıt** elde edildi:

```
ring_submit_cmd: fatal at dec->cur offset=32/36 size=36 ring_head=2308
first_36_bytes: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
                00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
                00 00 00 00
```

**Yorum:** Decoder 36 baytlık bir komutun **32 baytını başarıyla decode etti**, sadece **son 4 bayt** sorunlu — ve o son 4 bayt **tamamen sıfır**. Bu rastgele bozulma (bit-flip, yanlış offset) gibi görünmüyor; tam tersine "bu bellek konumu henüz hiç yazılmamış/host'a hiç ulaşmamış" imzası taşıyan temiz bir sıfır-dolgu deseni. Bu, ARTIK spesifik ve test edilebilir bir hipotez: **client'ın push'u, tail'i host'a duyurduğu anda, karşılık gelen buffer baytları henüz host'un belleğine TAM olarak yansımamış olabilir** — 03 numaralı nottaki "buffer önce, tail sonra" düzeltmesi PUT'ların KENDİ ARALARINDAKİ sırayı garanti ediyor, ama bu, host'un ring-tüketici thread'inin YARIŞ PENCERESİNİ tamamen kapatmıyor olabilir (ör. host'un kendi belleğe yazma/senkronizasyon davranışı, ya da bizim aktif push/pull hook'larının ARTIK çok daha sık/hızlı tetiklenmesi önceden nadir olan bir pencereyi daha sık açığa çıkarıyor olabilir).

## Sonraki Tur İçin Somut Görev

1. **Client tarafına da eşleşen bir debug print ekle**: her `vtest_shmem_sync_ring_cycle()` çağrısında, PUT edilen tail değerini ve buffer'ın push edilen bölgesinin bir kontrol-toplamını (örn. son yazılan birkaç yüz baytın hash'i veya sadece son N baytı) logla. Bunu host'un fatal anındaki dump'ıyla karşılaştır — client'ın gönderdiği ile host'un aldığı GERÇEKTEN farklı mı, yoksa client zaten YANLIŞ/eksik veri mi gönderiyor (yerel yazma tamamlanmadan push tetiklenmiş olabilir)?
2. **Hipotez A (client-side race):** `vn_ring_write_buffer()`'ın kendisi write'ı nasıl yapıyor kontrol et — eğer birden fazla `cs->buffers[i]` varsa (çok parçalı komut) ve HERHANGİ bir ara adımda (örn. `vn_ring_wait_space` içindeki bir bekleme sırasında) bizim YENİ eklenen aktif pull/push hook'larımız (özellikle `vn_ring_wait_seqno`'nun pull'u, EĞER `vn_ring_wait_space` de dolaylı olarak onu tetikliyorsa) ARADA bir push tetikleyip TAMAMLANMAMIŞ bir yazımı host'a gönderiyor olabilir mi -- kontrol et.
3. **Hipotez B (host-side visibility):** Host'un `vkr_ring_thread`'i, `ring->control.tail`'i `memory_order_acquire` ile okuyor (doğru), ama VERİ baytlarını (`buf->data`, ayrı bir bellek bölgesi) okurken herhangi bir senkronizasyon birimi YOK -- sadece "tail arttıysa demek ki veri de oradadır" varsayımına dayanıyor. Bizim TCP-sync modelimizde, PUT-buffer ve PUT-tail AYRI VTEST KOMUTLARI (ayrı `vtest_block_write` çağrıları, ayrı TCP segment'leri olabilir) -- **host'un server dispatch thread'i bunları sırayla işlese bile**, host'un `memcpy` yazdığı `res->iov.iov_base` belleğine, ring-tüketici thread'inin (FARKLI bir thread) okuması arasında GERÇEK bir memory barrier/atomic senkronizasyon yok (sadece `tail` atomic, ama `buffer` alanı DEĞİL). x86'da normal store'lar sırayla görünür olsa da, iki AYRI thread arası (dispatch thread'in memcpy'si vs ring-consumer thread'in okuması) hiçbir senkronizasyon primitive'i (mutex/atomic/fence) YOK -- teorik olarak veri görünürlüğü garantisi eksik olabilir. Bunu doğrulamak için: server tarafında PUT-buffer işlendikten hemen sonra bir `atomic_thread_fence` veya PUT-tail işlenmeden önce host'un ring-tüketici thread'iyle senkronize eden bir mekanizma eklenip eklenmediğini incele.
4. Eğer B doğruysa, düzeltme: server'ın PUT-buffer handler'ının sonunda (`vtest_transfer_put_internal`, memcpy'den hemen sonra) bir `atomic_thread_fence(memory_order_release)` eklemek, ring-tüketicinin tail okumasının `memory_order_acquire` ile zaten eşleştiğini varsayarsak yeterli olabilir (release-acquire pairing için server'ın YAZDIĞI taraf da release semantiği taşımalı).
5. Bulunursa düzelt, rebuild, 5/5 tutarlı `vulkaninfo --summary` testini tekrar dene.

## Not
Debug log'lar artık epey kalabalık (`server transfer_get on reply-pool res` gibi yüksek frekanslı print'ler) -- gerekirse ileride bir env-gate ile kısılabilir, şimdilik dokunulmadı.
