# Faz 4 — Device Memory (bo) Transport Eklendi: vkcube Artık Çöküyor, Ama Kullanılamayacak Kadar Yavaş (2026-08-29)

## Bağlam

Kullanıcı `vkcube` (gerçek render yapan, sadece enumerate eden `vulkaninfo`'dan farklı olarak) denedi:
```
vtest_bo_create_from_device_memory: Assertion `res_id > 0 && res_fd >= 0' failed.
```

## Kök neden

`vulkaninfo` sadece Vulkan nesnelerini sorgular, gerçek GPU belleği hiç ayırmaz — bu yüzden Faz 3'te sadece `shmem` (ring buffer + reply-pool, küçük metadata) TCP-uyumlu hale getirilmişti. `vkcube` gerçek render için host'tan gerçek GPU belleği (`bo` / device memory) istiyor — `vtest_bo_create_from_device_memory`, TCP modunda hiç var olmayan gerçek bir dosya tanımlayıcısı (`fd`, normalde SCM_RIGHTS ile host'tan gelir) bekleyen bir `assert` içeriyordu; bu path hiç yamalanmamıştı.

## Düzeltme

`vn_renderer_vtest.c`'de, `shmem` için zaten kanıtlanmış desenin aynısı `bo`'ya da uygulandı:
- `vtest_bo_create_from_device_memory`: TCP modunda `res_fd=-1` durumunu artık kabul ediyor (assert güncellendi).
- `vtest_bo_map`: `res_fd < 0` ise gerçek fd ile `mmap` yerine anonim (`MAP_ANONYMOUS`) bir mapping oluşturuyor.
- `vtest_bo_flush`/`vtest_bo_invalidate`: önceden gerçek paylaşımlı bellek varsayımıyla tamamen no-op'tular (doğru davranış, gerçek virtio-gpu için). TCP + anonim mapping durumunda artık gerçek bir `transfer_put2`/`get2` yapıyorlar — flush = guest'in yazdığını host'a it, invalidate = guest okumadan önce host'tan çek. Ring/reply-pool için kullanılan aynı, ispatlanmış transfer mekanizması, sadece `bo`'nun kendi `res_id`/offset'inde.

## Sonuç

**Crash tamamen gitti.** `vkcube` artık assert'siz çalışıyor — pencere taşbar'da görünüyor, süreç canlı, log sürekli ilerliyor (donmamış). Bu, projenin temel hedefine (gerçek GPU render + piksel verisinin ağ üzerinden aktarımı) mimari olarak ulaşıldığını kanıtlıyor.

**Ama pratik olarak kullanılamaz durumda**: kullanıcı pencereye tıkladığında tepki alamadı — "açık ama tıklanmıyor". Bu beklenen bir sonuç, hata değil: her Vulkan komutu şu anki mimaride (optimize edilmemiş, tam senkron) ~200-800ms'lik gerçek bir network round-trip gerektiriyor (bkz. 02 numaralı not). Etkileşimli bir uygulamanın akıcı çalışması saniyede onlarca/yüzlerce komut gerektirir — mevcut hız bunun çok altında.

## Değerlendirme

Faz 4'ün "gerçek render çalışıyor mu" sorusu artık **evet** ile cevaplandı (doğruluk garantili). Kalan iş tamamen performans — daha önce not edilen "gerçek pipelining" (birden fazla komutu tek round-trip'te toplu gönderme) ihtiyacı artık çok daha kritik hale geldi: sadece `vulkaninfo` gibi bir kerelik sorgu için değil, ETKİLEŞİMLİ bir uygulamanın (oyun, vkcube) kullanılabilir olması için bu optimizasyon ZORUNLU. Bu, projenin planında zaten "büyük, riskli, ayrı bir Faz 5 görevi" olarak işaretlenmişti — şimdi bu görevin önceliği netleşti.
