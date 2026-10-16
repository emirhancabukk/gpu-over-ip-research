# Faz 3 — Reply-Pool Tazelik Hook'u Eklendi (Çalışıyor); Ayrı, Aralıklı Bir "CS Error" Kalıyor (2026-08-28, otonom tur 10)

## Bu turda yapılan: senkron reply-pool "sync_now" hook'u

Önceki turun (04 numaralı not) bıraktığı somut görev uygulandı: kör periyodik poll yerine, `vn_ring_wait_seqno()`'nun (`vn_ring.c`) başarıyla döndüğü TAM noktada bekleyen reply-pool kaynağı için senkron bir GET tetiklendi.

**Neden doğru yer burası:** Venus protokolünün kendi sıralama garantisi — host, bir komutun cevabını yazmayı bitirmeden ring seqno'yu ilerletmez. Yani `vn_ring_get_seqno_status()` true dönüp `wait_seqno` başarıyla çıktığı an, host'un cevabı (varsa) KESİNLİKLE hazırdır. Arka plan poll thread'inin bunu YAKALAMIŞ olması garanti değildi (04 numaralı notun bulduğu asıl açık), bu turun düzeltmesi bu garantiyi kapatıyor.

**Uygulama (3 dosya):**
- `vn_renderer.h`: `struct vn_renderer_shmem_ops`'a opsiyonel `sync_now` alanı + `vn_renderer_shmem_sync_now()` inline yardımcı fonksiyonu eklendi (renderer desteklemiyorsa no-op).
- `vn_renderer_vtest.c`: `vtest_shmem_sync_now()` eklendi — sadece `sync_active` (TCP modunda arka plan thread'i olan) kaynaklar için senkron bir `vtest_vcmd_transfer_get2` yapar; `shmem_ops.sync_now`'a bağlandı.
- `vn_ring.c`: `vn_ring_wait_seqno()`'nun başarı dönüş noktasına `vn_renderer_shmem_sync_now(ring->instance->renderer, ring->instance->reply_shmem_pool.shmem)` çağrısı eklendi.

## Sonuç: kısmi ama gerçek iyileşme

Bu hook **ilk çağrının (vkEnumerateInstanceVersion) reply tazelik sorununu tamamen çözdü** — art arda 5 çalıştırmanın **hepsinde** `command_type=137` (beklenen değerle tam eşleşen) doğru şekilde decode edildi, önceki turun ~%50 başarı oranına kıyasla belirgin bir iyileşme.

Ancak süreç artık **daha sonraki bir noktada** (`vkEnumeratePhysicalDevices`, bir sonraki senkron Venus çağrısı) takılıyor/çok yavaşlıyor. `coredumpctl` ile canlı bir çalıştırmaya `SIGABRT` gönderip çekirdek döküm analizi yapıldı — bu, **kilitlenme (deadlock) değil**, hâlâ meşru şekilde `vn_ring_wait_seqno` → `os_time_sleep` içinde bekliyordu; yani ilerleme oluyor ama beklenenden çok yavaş.

## Yeni gözlemlenen ayrı sorun: "CS error" bazı çalıştırmalarda hâlâ oluyor

Bir sonraki test turunda (60 saniyelik bounded test), **önceki turda "tamamen düzeltildi" sanılan** `vkCreateInstance resulted in CS error` / `ring_submit_cmd: vn_dispatch_command failed` hatası **yeniden** gözlemlendi — bu sefer reply-pool hiç sıfırdan öteye geçemedi (host, komutu hiç başarıyla işleyemediği için cevap da hiç yazılmadı).

Bu, "buffer'ı tail'den önce PUT et" düzeltmesinin (03 numaralı not, Bug #6) sorunu **azalttığını ama tamamen ortadan kaldırmadığını** gösteriyor — hâlâ aralıklı (deterministik olmayan) bir veri-görünürlüğü veya zamanlama sorunu var host'un ring-tüketici thread'i ile ana dispatch thread'i arasında. Olası nedenler (doğrulanmadı, sonraki tur için hipotez):
- Host'un ring-tüketici thread'i (`vkr_ring.c`) tail değişikliğini periyodik olarak (cnd_wait/timeout ile) kontrol ediyor olabilir ve BİZİM ardışık PUT'larımız (buffer sonra tail) arasında host'un o SIRADA başka bir eski tail değeriyle zaten okumaya başlamış olması mümkün (yani sorun sıra değil, ÇAKIŞAN birden fazla senkronizasyon döngüsü/tekrar eden PUT'lar host'u karıştırıyor olabilir — biz HER döngüde TÜM 128KB buffer'ı yeniden gönderiyoruz, sadece yeni yazılan kısmı değil).
- Mümkün: ring sarmalama (wraparound) davranışı düzgün ele alınmıyor olabilir.

## Sonraki Tur İçin Somut Görev

1. `vkr_ring.c`'deki ring-tüketici thread'in tail'i nasıl/ne zaman okuduğunu, host'un decode hatası anında TAM OLARAK hangi baytları gördüğünü anlamak için server tarafına debug print ekle (decode hatası anında ring buffer'ının ilk birkaç yüz baytını dump et).
2. `vkEnumeratePhysicalDevices`'in neden bu kadar yavaş olduğunu araştır — muhtemelen sync_now hook'unun SADECE `wait_seqno`'nun BAŞARI durumunda çalışması yeterli değil; ring'in KENDİSİNİN (head/status) senkronizasyon hızı hâlâ arka plan thread'inin 200µs döngüsüne bağlı. Bunun için ring senkronizasyonuna da benzer bir "olay tetiklemeli" yaklaşım (örn. yeni komut gönderildiğinde hemen bir senkron ring-sync tetikle, sadece arka plan thread'ine güvenme) denenebilir.
3. Hâlâ ilerleme kaydedilemezse, projenin planındaki gerçekçi beklenti bölümünün öngördüğü gibi, bu "kör poll + düzeltici hook'lar" yaklaşımının doğal sınırına yaklaşılmış olabilir — bu durumda gerçek bir olay-bildirimli mekanizma (örn. host'un yeni veri olduğunda AKTİF OLARAK bir vtest komutu göndermesi, sadece pasif poll'e güvenmek yerine) düşünülmeli. Bu, vtest protokolüne YENİ bir komut eklemeyi gerektirir (host→guest yönünde asenkron bildirim) — daha büyük bir adım ama net bir hedef.

## Kullanılan/güncellenen dosyalar
- `vendor/mesa/src/virtio/vulkan/vn_renderer.h`
- `vendor/mesa/src/virtio/vulkan/vn_renderer_vtest.c`
- `vendor/mesa/src/virtio/vulkan/vn_ring.c`

## Not
`VIRGL_LOG_LEVEL=debug` kullanmaya devam et — olmadan host tarafı `vkr_log` mesajları (CS error dahil) sessizce bastırılıyor.
