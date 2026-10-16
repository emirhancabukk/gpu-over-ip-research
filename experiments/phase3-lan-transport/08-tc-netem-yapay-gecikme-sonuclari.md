# Faz 3 — tc netem Yapay Gecikme Testi: 10/20/40ms Hepsi BAŞARILI (2026-08-28, otonom tur 12 devamı)

> **Sonradan not (2026-10-16):** Bu testler yalnızca gecikme ekledi, bant genişliği
> sınırı yoktu. Gecikmenin etkisi için geçerli bir yordayıcı, ama bant genişliğine bağlı
> sorunları göremez — faz 4'teki 258 MB'lık boşa transfer (`phase4-wan-aws/07-*.md`)
> tam da bu yüzden burada hiç görünmedi. Yeni testlerde `netem ... rate <N>mbit`
> eklenmeli.

## Yöntem: root gerekmeden izole network namespace

`tc qdisc add dev lo ...` normalde `sudo`/root gerektiriyor (denendi, `sudo -n` parola istedi, otonom olarak devam edilemedi). Bunun yerine **ayrıcalıksız (unprivileged) bir user+network namespace** kullanıldı:

```bash
unshare --user --net --map-root-user sleep 3600 &   # izole ns'i canlı tutan "çapa" process
nsenter --target <pid> --net --user --preserve-credentials bash -c "
  ip link set lo up
  tc qdisc add dev lo root netem delay <N>ms
  # ... server ve client'ı da AYNI nsenter içinde çalıştır ...
"
```

Bu, gerçek host'un network yığınına HİÇ dokunmadan (tamamen izole edilmiş kendi `lo`'su olan bir namespace içinde) test yapmayı sağladı — plan'ın "bare host'ta sistem geneline kalıcı etkisi olan şeyler yapılmayacak" kısıtına tam uyumlu, hatta `sudo`'dan bile daha güvenli (gerçek sisteme SIFIR etki, namespace son process çıkınca otomatik temizleniyor).

## Sonuçlar

| Yapay gecikme (tek yön) | Sonuç | Süre (`vulkaninfo --summary` tam çalışma) |
|---|---|---|
| 0ms (Faz 3 temel testi, önceki not) | ✅ BAŞARILI (5/5) | ~birkaç saniye |
| 10ms | ✅ BAŞARILI | 20.75s |
| 20ms | ✅ BAŞARILI | 40.82s |
| 40ms | ✅ BAŞARILI | 81.16s |

**Hepsi `driverName = venus` ile temiz çıktı verdi, hiç çökme/assertion/CS error olmadı.**

## Yorum

1. **Doğruluk (correctness) tarafı sağlam görünüyor.** 0'dan 40ms'ye kadar (Türkiye'den Frankfurt gibi gerçek bir AWS bölgesine gerçekçi bir RTT aralığı) hiçbir bozulma/çökme yaşanmadı. Bu, Faz 3'ün bu tur ana bulgularının (ring senkronizasyonu, reply-pool tazelik hook'u, sync-wait bypass'ı) gerçekten sağlam olduğunu, sadece "en iyi durumda çalışan kırılgan bir demo" olmadığını gösteriyor.

2. **Süre gecikmeyle DOĞRUSAL ölçekleniyor** (10ms→20.75s, 20ms→40.82s, 40ms→81.16s — yaklaşık 2x gecikme = 2x süre). Bu, sistemin sabit sayıda ARDIŞIK round-trip yaptığını gösteriyor (paralel/pipeline edilmiş değil) — beklenen bir sonuç, çünkü bu turun aktif push/pull hook'ları (`vn_ring_submit_internal`, `vn_ring_wait_seqno`) HER senkron Venus çağrısı için ayrı, sıralı network round-trip'leri gerektiriyor, ve arka plan poll thread'i artık ring için hiç çalışmıyor (bkz. 07 numaralı not, Bug #8 düzeltmesi).

3. **Gerçekçi rakam:** 40ms tek-yön gecikme (yaklaşık 80ms RTT, kötü bir WAN senaryosuna denk gelir) için `vulkaninfo`'nun TÜM enumeration'ı ~81 saniye sürdü. Gerçek bir oyun/render iş yükü için bu KABUL EDİLEMEZ derecede yavaş olur — ama bu, planın baştan beri öngördüğü, Faz 5'in konusu olan bir OPTİMİZASYON sorunu, mimari bir bloker değil. Şu anki tasarım "her adımda dur, bir round-trip yap, devam et" şeklinde — gerçek bir üretim tasarımı, komutları PIPELINE etmeli (host'un cevap vermesini beklemeden birden fazla komut arka arkaya gönderip, cevapları toplu işlemek), tam da Faz 1'in erken bulgularının öngördüğü gibi.

## Sonraki Adımlar

- **Faz 3 artık TAMAMEN tamamlandı** (LAN + yapay WAN gecikmesi, hepsi doğrulandı).
- **Faz 4'e geçiş kararı kullanıcıya bırakılmalı**: gerçek AWS GPU testi (g5.xlarge, A10G) için AWS quota'sının onaylanıp onaylanmadığı hâlâ bilinmiyor (önceki turlarda ikinci kez reddedilmişti) — bu, otonom döngünün kontrol edebileceği bir şey değil, kullanıcı tarafında bir AWS konsolu kontrolü gerektiriyor.
- **Faz 5'in erken bir parçası olarak düşünülebilecek** (performans optimizasyonu olmadan önce) bir sonraki teknik adım: pipelining — `vn_ring_wait_seqno`'nun HER iterasyonda tam bir round-trip yapması yerine, birden fazla bekleyen komutu tek bir round-trip'te toplu kontrol etme. Bu, WAN gecikmesinin etkisini büyük ölçüde azaltabilir (N komut için N round-trip yerine ~1 round-trip).
