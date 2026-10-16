# Faz 5 (erken deneme #2) — "Sadece idle'da push" Denemesi Doğruluğu Bozdu, Geri Alındı (2026-08-28, otonom tur 14)

## Hipotez ve deneme

09 numaralı nottan devam: gerçek darboğazın, `vulkaninfo`'nun ürettiği ~289 komuttan her birinin kendi zorunlu round-trip'i olduğu belirlenmişti. Bu turda önce doğrulandı: Venus protokolünün üretilen kodunda gerçekten `vn_async_vkFoo` (cevap beklemeyen, "fire-and-forget") ve `vn_call_vkFoo`/`vn_submit_vkFoo` (cevap bekleyen) ayrımı VAR (örn. `vn_async_vkCreateInstance`, `vn_async_vkEnumerateInstanceVersion` — `vn_protocol_driver_instance.h`). Async çağrılar `reply_size=0` ile submit ediliyor, bu da `vn_ring_submit_command()`'ın `vn_ring_wait_seqno()`'yu HİÇ çağırmadığı anlamına geliyor.

**Sorun tespiti:** bizim `vn_ring_submit_internal()`'a eklediğimiz aktif push hook'u (Faz 3, tur 11) HER submission'da (sync veya async fark etmeksizin) koşulsuz çalışıyor — yani async çağrılar bile gereksiz yere senkron bir round-trip ödüyor.

**Denenen düzeltme:** push'u koşullu yap — sadece host'un EN SON gözlemlendiğinde idle (`VK_RING_STATUS_IDLE_BIT_MESA`) olduğu durumda zorla push et; host meşgulse arka plan thread'inin veya sonraki bir çağrının onu zaten teslim edeceğini varsay.

## Sonuç: BOZULDU, geri alındı

Test (0ms gecikme, sadece 3 çalıştırma yeterliydi göstermeye): **3/3 çalıştırma timeout'a takıldı** (30s içinde hiç bitmedi). Log incelemesi net bir kısır döngü gösterdi: `tail=228`'de sıkışıp kaldı, client aynı tail değerini defalarca "push" olarak logluyordu ama server tarafında bu veri HİÇ görünmüyordu (host'un aldığı son reply verisi hep eski/aynı kalıyordu).

**Kök neden:** async bir çağrı (a) push'u atlıyor (host idle değildi varsayımıyla) VE (b) `vn_ring_wait_seqno`'yu hiç çağırmıyor (async olduğu için) → bu komutun verisi YEREL bellekte yazılı kalıyor ama HİÇBİR mekanizma onu host'a iletmiyor. Normalde arka plan poll thread'i bunu er ya da geç yakalardı, AMA 07 numaralı nottaki Bug #8 düzeltmesinde arka plan thread'i ring için TAMAMEN devre dışı bırakılmıştı (sadece aktif push/pull hook'larına güvenilecek şekilde). Bu iki değişiklik (Bug #8'in "arka plan thread ring'e dokunmasın" kararı + bu turun "idle değilse push etme" denemesi) BİRLİKTE bir teslimat garantisi boşluğu yaratıyor: hiçbir yol veriyi host'a göndermeyi garanti etmiyor.

**Düzeltme:** push tekrar koşulsuz hale getirildi (`vn_ring.c`, `vn_ring_submit_internal`). 3/3 doğrulama çalıştırması tekrar temiz (`driverName = venus`).

## Çıkarım

Ring'in senkronizasyon mimarisi şu an **tamamen** aktif push/pull hook'larına dayanıyor (arka plan thread'i devre dışı) — bu, HER submission'ın kendi teslimatından sorumlu olması gerektiği, "birisi zaten görür" varsayımının GÜVENLİ OLMADIĞI anlamına geliyor. Gerçek bir optimizasyon için iki seçenekten biri gerekiyor:

1. **Arka plan thread'ini ring için YENİDEN etkinleştirmek** (bir fallback/güvenlik ağı olarak, ama bu sefer 09 numaralı nottaki Bug #8'in nedenini tekrar tetiklemeyecek şekilde — örn. çok DAHA UZUN bir periyotla, örn. 5-10ms, sadece "biri unutursa" diye bir yedek, aktif hook'larla yarışmayacak kadar seyrek).
2. **Doğru pipelining:** async çağrıları GERÇEKTEN biriktirip (push etmeden) bir sonraki SYNC çağrı geldiğinde (ki o zaten bekleyecek, push zorunlu) hepsini TEK round-trip'te göndermek. Bu, "hiç push etme" değil, "erteleyip TOPLU push et" demek — 1 numaralı seçenekten daha karmaşık ama daha büyük kazanç potansiyeli var (N async çağrı = hâlâ sadece son sync çağrının 1 push'u).

Bu turun kapsamı dışında (zaman/karmaşıklık nedeniyle) — sonraki tur için somut, net bir başlangıç noktası bu.

## Sonraki Tur İçin Somut Görev

1. **Seçenek 1'i dene (düşük risk, hızlı test):** arka plan thread'ini ring için 5ms gibi uzun bir periyotla yeniden etkinleştir (SADECE fallback, aktif hook'lar hâlâ birincil mekanizma). Doğruluğu ve tc netem 40ms süresini ölç.
2. Zaman kalırsa Seçenek 2'yi (gerçek erteleme + toplu push) düşünmeye başla — ama bu, `vn_ring_submit_internal`'ın "async mı sync mı" bilgisine ihtiyaç duyması nedeniyle üst katmanlardan (`vn_ring_submit_command`) bir parametre geçirmeyi gerektirir, daha invaziv bir değişiklik.
