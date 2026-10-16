# Faz 5 — Gerçek Pipelining: 435s → 236s (%46 Hızlanma) (2026-08-29)

## Fikir

`vn_ring_submit_internal`'da (Faz 3'ten beri), HER submit'ten (sync veya async fark etmeksizin) sonra ring'i host'a **koşulsuz, hemen** push ediyorduk. Bu, async komutların (fire-and-forget, `vn_async_vkFoo`, `vn_ring_wait_seqno`'yu hiç çağırmayan) bile kendi network round-trip'ini ödemesine neden oluyordu.

09/10 numaralı notlarda bu tam olarak tespit edilmiş ama düzeltme denemesi (koşulsuz push'u kaldırmak) o zaman **doğruluğu bozmuştu** — çünkü o an itibariyle ring'in arka plan senkronizasyon thread'i tamamen kaldırılmıştı (Bug #8 düzeltmesi), yani push'suz kalan async veri için HİÇBİR teslimat yolu yoktu.

## Neden şimdi güvenli

Bu oturumda iki şey değişti:
1. **Arka plan thread'i geri geldi** (11 numaralı not) — düşük frekansta (5ms), saf-async komut dizileri için sınırlı gecikmeli (en kötü ~5ms) bir güvenlik ağı.
2. **Race condition düzeltmesi** (02 numaralı not) — `vn_ring_write_buffer` artık bir kilit altında, arka plan thread'in yarım-yazılmış veriyi okuması artık mümkün değil.

Bu iki güvenlik ağı sayesinde, submit-time push'u **tamamen kaldırmak** artık güvenli: veri, ya (a) bir sonraki SENKRON çağrının kendi `vn_ring_wait_seqno` döngüsü (zaten her zaman ilk iterasyonda push+pull cycle'ı tetikliyor) tarafından, ya da (b) saf-async dizilerde arka plan thread'i tarafından, er ya da geç host'a ulaşıyor — ama artık HER TEK komut için değil, BİRİKTİRİLMİŞ halde.

## Değişiklik

`vn_ring.c`, `vn_ring_submit_internal()`: `store_tail`'den sonraki koşulsuz `vn_renderer_shmem_sync_now(...)` çağrısı **tamamen kaldırıldı**. Geri kalan her şey (lock/unlock, store_tail) aynı kaldı.

## Sonuç

Temiz bir AWS testinde (aynı server, aynı gerçek internet bağlantısı):

| | Önceki (koşulsuz push) | Sonraki (pipelining) |
|---|---|---|
| Toplam süre | 435s | **236s** |
| Ring sync çağrı sayısı | 578 | **353** |
| Crash | Yok | Yok |
| Sonuç | `driverName=venus`, NVIDIA A10G | Aynı |

**%46 hızlanma, %39 daha az ring round-trip, doğruluk korundu.** Bu, planın "büyük ve riskli, ayrı bir Faz 5 görevi" olarak işaretlediği pipelining işinin, önceden tahmin edilenden çok daha küçük bir değişiklikle (tek bir fonksiyon çağrısını kaldırmak) mümkün olduğunu gösteriyor — çünkü zaten var olan `wait_seqno` mekanizması, sync çağrılar için doğal bir "batch noktası" görevi görüyordu, sadece bunu KULLANMIYORDUK.

## Sonraki adım

vkcube'da (gerçek render, çok sayıda ardışık async komut içeren bir iş yükü) test edilecek — vulkaninfo'dan çok daha fazla async komut oranına sahip olduğu için, pipelining'in etkisi orada muhtemelen daha da büyük olacak.
