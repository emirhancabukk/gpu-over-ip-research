# Faz 5 — Reply-Pool Arka Plan Thread'i Düzeltmesi + vkcube'un Kaldığı Yer (2026-08-29)

## Bulunan ikinci darboğaz

vkcube testinde host log'unda (`VIRGL_LOG_LEVEL=debug`) aynı satırın yüzlerce kez tekrarlandığı görüldü:
```
[DEBUG] server transfer_get on reply-pool res: handle=2 offset=0 size=1048576 ...
```

Kök neden: reply-pool'un arka plan senkronizasyon thread'i (`vtest_shmem_sync_thread_func`, `vn_renderer_vtest.c`), Faz 3'ten beri **her 2ms'de bir, tam 1MB'lık kör bir GET** yapıyordu — aktif `sync_now` hook'unun (Faz 4'te eklenen `hint_size`/delta optimizasyonuna sahip) yanında, bu arka plan thread hiç optimize edilmemişti. Gerçek internet üzerinden her 2ms'de 1MB çekmeye çalışmak, sürekli, büyük ölçüde gereksiz bir trafik + mutex çekişmesi yaratıyordu (ring'in kendi arka plan thread'i için zaten 5ms'ye çıkarılmıştı, ama reply-pool dalı unutulmuştu).

## Düzeltme

Reply-pool arka plan thread'inin periyodu 2ms'den **100ms**'ye çıkarıldı (`vn_renderer_vtest.c`, `vtest_shmem_sync_thread_func`). Aktif `sync_now` hook'u zaten asıl işi (delta-pull) yapıyor; bu arka plan thread'in sadece nadir "unutulmuş" senaryolar için düşük frekanslı bir güvenlik ağı olması yeterli.

## Sonuç

vkcube testinde: düzeltme öncesi 90 saniyede 8 senkron çağrı tamamlanıyordu (~11.25s/çağrı), düzeltme sonrası **60 saniyede 15 senkron çağrı** (~4s/çağrı) — **~2.8x hızlanma**.

## vkcube'un şu anki durumu

Süreç ~150 saniye sonra 16. senkron çağrıda takılı (15'te sabit kaldı), ring senkronizasyonları ~950ms-1.2s civarında devam ediyor ama tamamlanamıyor — `vn_relax`'ın exponential backoff'una girmiş durumda. **Crash yok**, sadece çok yavaş. Bu, muhtemelen device-memory/image (`vtest_bo_create_from_device_memory`, `vtest_vcmd_resource_create_blob`) oluşturma zincirinin bir parçası — bu, Venus ring'inin komut-akışından **tamamen ayrı bir RPC mekanizması** kullanıyor (doğrudan `vtest_write`+`vtest_read`, senkron), bu yüzden bu oturumdaki pipelining düzeltmesinden (ring'e özel) hiç yararlanamıyor.

Pencere henüz hiç açılmadı — swapchain oluşturma zincirinin tamamlanmasına daha çok var.

## Bu turun özeti

Bu oturumda, `vulkaninfo`'dan `vkcube` denemesine geçtikten sonra bulunan/düzeltilen sorunlar sırasıyla:
1. Device-memory transport eksikliği (crash → çalışır hale getirildi, 03 numaralı not).
2. Ring submit-time'da koşulsuz push (pipelining engeli → kaldırıldı, %46 hızlanma, 04 numaralı not).
3. Reply-pool arka plan thread'inin aşırı agresif (2ms, tam 1MB) senkronizasyonu (→ 100ms'ye düşürüldü, ~2.8x hızlanma, bu not).

Üçü de gerçek, ölçülebilir kazanımlar. Kalan açık soru: device-memory/blob-create RPC zincirinin kendisinin neden bu kadar yavaş olduğu — bu, ayrı bir teşhis turu gerektiriyor (muhtemelen benzer bir "arka plan thread agresifliği" ya da "gereksiz büyük transfer" deseni, ama henüz doğrulanmadı).
