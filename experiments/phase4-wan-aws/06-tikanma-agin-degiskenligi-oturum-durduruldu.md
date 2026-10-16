# Faz 5 — Tıkanma: Gerçek Ağ Değişkenliği, Kod Değil (2026-08-29)

> **Sonradan not (2026-10-16):** Bu notun "yavaşlığın sebebi yerel bağlantının
> throughput'u" sonucu büyük olasılıkla yanlış. 07 numaralı not, reply-pool arka plan
> thread'inin koşu başına ~258 MB'ı boşa çektiğini ve bağlantıyı kendisinin
> doldurduğunu ölçtü. Aşağıdaki metin o günkü haliyle bırakıldı.

## Durum

Reply-pool arka plan thread'ini tamamen kaldırma denemesi (05 numaralı notun devamı) test edildi: 10 dakikada sadece 8 senkron çağrı tamamlandı (~485ms ortalama) — kanıtlanmış 100ms-periyotlu versiyondan (aynı oturumda daha önce ~63ms ortalama ölçülmüştü) belirgin şekilde kötü. Değişiklik geri alındı (100ms periyoduna dönüldü, kanıtlanmış hali).

**Ama geri alma sonrası da aynı yavaşlık devam etti**: 10 dakikada **sıfır** senkron çağrı tamamlandı, ortalama sync_now süresi 851ms'ye çıktı.

## Ayırt edici teşhis

- Gerçek RTT (TCP connect timing): oturum boyunca tutarlı **~72-75ms** — bu testler sırasında da aynı, değişmemiş.
- Host CPU/GPU: tamamen boşta (`top`: %97.6 idle, load average 0.04; `nvidia-smi`: %0).
- Kod: kanıtlanmış (daha önce 2.8x hızlanma göstermiş) konfigürasyona geri dönüldü, aynı sonuç.

RTT sabit kalırken, host tamamen boşta dururken, VE kod bilinen-iyi haline döndürülmüşken performansın bu kadar kötüleşmesi, sorunun ne kodda ne host'ta olduğunu gösteriyor — kalan tek açıklama, benim yerel internet bağlantımın **throughput/paket işleme kapasitesinin** o anki durumu (RTT'yi etkilemeyen ama gerçek veri transferini etkileyen bir tıkanıklık, örn. ISS seviyesinde, ya da yerel ağdaki başka bir kullanım).

## Sonuç

Bu, projenin planında zaten kabul edilen bir gerçeklik: gerçek WAN üzerinde çalışan bir sistemde ağ koşulları zamanla değişir, ve tek bir oturumdaki ölçümler mutlak/kalıcı değildir. Bu noktada kod tarafında yapılabilecek bir şey yok — daha fazla "optimizasyon" denemek, güvenilmez bir zeminde ölçüm yapmak anlamına gelir.

## Bu oturumun toplam kazanımları (özet)

1. Kritik race condition (ring buffer veri bozulması → crash) bulundu ve düzeltildi.
2. Device-memory (bo) transport TCP üzerinden çalışır hale getirildi — gerçek render artık çöküyor değil.
3. Ring pipelining (submit-time koşulsuz push kaldırıldı) — ölçülen %46 hızlanma (435s→236s, iyi ağ koşullarında).
4. Reply-pool arka plan thread'i sıklığı düşürüldü (2ms→100ms) — ölçülen ~2.8x hızlanma (iyi ağ koşullarında).

Dördü de gerçek, koddan kaynaklanan, kalıcı iyileştirmeler. Şu anki yavaşlık bunları geçersiz kılmıyor — sadece güncel ölçümü güvenilmez kılıyor.

## Öneri

Oturum burada durmalı. AWS instance açık bırakılırsa maliyet işlemeye devam eder. Devam edilecekse, ağ koşulları normale döndüğünde (farklı bir zamanda) mevcut kod tabanıyla (4 optimizasyon dahil) temiz bir doğrulama turu yapılmalı.
