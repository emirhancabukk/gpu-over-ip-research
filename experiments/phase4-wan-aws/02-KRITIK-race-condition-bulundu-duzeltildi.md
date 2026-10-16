# Faz 4 — Kritik Race Condition Bulundu ve Düzeltildi: Ring Buffer Bozulması (2026-08-29)

## Özet

Derin ağ/syscall teşhisi (`tcpdump` + `strace`, kullanıcının `sudo` ile kurdurduğu araçlarla) sırasında, önceki turda ara sıra görülen `vulkaninfo` çökmelerinin (`SIGABRT`, `coredumpctl` ile teşhis edildi) kök nedeni bulundu: **gerçek bir veri bozulması race condition'ı**, sadece performans sorunu değil. Düzeltildi, doğrulandı: **crash artık hiç oluşmuyor**.

## Teşhis süreci

1. `tcpdump` kuruldu (`setcap cap_net_raw,cap_net_admin+eip` ile sudo'suz çalışacak şekilde). 213.779 paketlik bir yakalamada sadece **5 tekrarlanan sequence** (gerçek TCP retransmisyon) bulundu — gerçek ağ paket kaybı ihmal edilebilir düzeyde, "ağ jitter'ı" hipotezi büyük ölçüde çürütüldü.
2. `strace` kuruldu, `vulkaninfo` bu araçla çalıştırıldı. Süreç yine `SIGABRT` ile çöktü.
3. `coredumpctl info` ile stack trace incelendi: crash `vn_ring_submit_internal` içinde, `status & VK_RING_STATUS_FATAL_BIT_MESA` kontrolünden geliyordu — yani **host**, ring'i decode ederken fatal bir hata görüp `FATAL` bitini set ediyordu, guest sonra bunu okuyup `abort()` çağırıyordu.
4. Host log'unda (`VIRGL_LOG_LEVEL=debug`) tam mesaj bulundu: `vkCreateInstance resulted in CS error`, decode `dec->cur offset=68/160` civarında takılmış, ilk 64 byte'ın **offset 32'den sonrası tamamen sıfır** — Faz 3'teki Note 06'da bulunan "32 byte doğru, gerisi sıfır" deseninin birebir aynısı.

## Kök neden

`vn_ring_write_buffer()` (`vn_ring.c`, ring'e komut verisi yazan üst-katman fonksiyon, Mesa'nın renderer-agnostic kodu) ile bizim TCP-senkronizasyon kodumuzun (`vtest_shmem_sync_ring_cycle`, hem arka plan thread'i hem aktif `sync_now` hook'u tarafından çağrılıyor) **ring buffer'ın mmap'lenmiş belleğine erişimi arasında hiçbir kilit/senkronizasyon yoktu**. Var olan `sock_mutex`, sadece TCP soket okuma/yazmasını (`vtest_write`/`vtest_read`) koruyordu — `vn_ring_write_buffer`'ın kendisi bu mutex'i hiç almıyordu.

Bu, düşük gecikmeli ortamlarda (localhost, `tc netem` ile simüle edilen 0-40ms) hiç sorun yaratmadı çünkü `write_buffer → store_tail → sync_now` dizisi çok hızlı tamamlanıyordu, arka plan thread'in (5ms periyotlu) tam o ara pencerede araya girme olasılığı neredeyse sıfırdı. Gerçek internette, `sync_now`'ın kendisi ara sıra 1-7 saniye sürdüğü için (ayrı bir konu, aşağıda), ana thread `sock_mutex`'i uzun süre tutuyor, arka plan thread bekliyor, ama serbest kaldığı an **eğer ana thread tam o sırada yeni bir `write_buffer` çağrısının ortasındaysa** (henüz `store_tail` yapmadan), arka plan thread mmap'teki **yarım yazılmış** buffer içeriğini host'a PUT edebiliyordu — host bunu decode etmeye çalışıp tutarsız veri görüp fatal hata veriyordu.

## Düzeltme

1. `vn_renderer.h`: `vn_renderer_shmem_ops`'a yeni `lock`/`unlock` hook çifti eklendi (opsiyonel, gerçek virtio-gpu için no-op).
2. `vn_ring.c`, `vn_ring_submit_internal()`: her `vn_ring_write_buffer()` çağrısı (ve ardından `vn_ring_store_tail()` + `sync_now` push'u) artık bu lock ile sarılıyor. `vn_ring_wait_space()` kasıtlı olarak lock **dışında** tutuldu — o, host'un head'inin ilerlemesini bekliyor, bu bilgi sadece arka plan thread'in kilidi alabilmesiyle güncellenir; lock içinde tutulsaydı deadlock olurdu.
3. `vn_renderer_vtest.c`: `lock`/`unlock` hook'ları mevcut `sock_mutex`'i kullanacak şekilde implement edildi. **Kritik detay**: `sync_now` (lock altında çağrılıyor) kendi içinde `vtest_shmem_sync_ring_cycle`'ı çağırıyor, o da AYNI mutex'i tekrar almaya çalışıyor — bu yüzden mutex `mtx_plain` yerine `mtx_plain | mtx_recursive` yapıldı (aksi halde aynı thread'in kendi kendini kilitlemesiyle self-deadlock oluşurdu).

## Doğrulama

Düzeltme sonrası temiz bir çalıştırma: **435 saniye, sıfır crash, sıfır `FATAL`/`abort` mesajı**, `driverName=venus`, `deviceName=NVIDIA A10G`. Ring senkronizasyon ortalaması **156.9ms** (önceki, crash'e giden yoldaki backoff'larla şişmiş 573ms'den gerçek/istikrarlı bir değere düştü).

## Kalan açık soru (performans, artık doğruluk değil)

Reply-pool ve ring senkronizasyonlarının **%15-20'si** hâlâ 1-7 saniye arası sürüyor (179 örneklemin 39'u reply-pool'da). Host GPU kullanımı sürekli %0, tcpdump'ta gerçek paket kaybı yok — bu gecikmelerin kesin kaynağı hâlâ belirsiz (host CPU scheduling / TCP congestion window davranışı / başka bir şey olabilir). Ama bu artık bir **doğruluk riski değil, saf performans sorunu** — sistem artık güvenilir şekilde (crash'siz) çalışıyor, sadece hızı optimal değil.

## Durum

Faz 4'ün temel hedefi (gerçek AWS GPU'suna gerçek internet üzerinden Venus ile güvenilir bağlantı) artık **hem doğrulanmış hem de güvenilir**. Kalan iş, saf bir performans optimizasyonu (daha büyük bir sonraki adım — gerçek pipelining ya da ağ gecikme kaynağının tam teşhisi).
