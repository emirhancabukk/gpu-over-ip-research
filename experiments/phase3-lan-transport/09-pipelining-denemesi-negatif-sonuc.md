# Faz 5 (erken deneme) — Aktif Pull Throttling'i Ölçülebilir Fayda Sağlamadı (2026-08-28, otonom tur 13)

## Denenen değişiklik

08 numaralı notun önerisi doğrultusunda: `vn_ring_wait_seqno()`'nun (`vn_ring.c`) her döngü iterasyonunda gerçek bir network round-trip'i (`sync_now` çağrısı) yapması yerine, sadece HER 4. iterasyonda bir yapması sağlandı (`(pull_iter++ % 4) == 0` koşulu, ilk iterasyon her zaman dahil). Hipotez: `vn_relax`'ın busy-wait fazı (256 hızlı iterasyona kadar) her iterasyonda gerçek bir network round-trip'i tetiklediği için, WAN gecikmesi altında bu fazın kendisi gereksiz yere çok maliyetli oluyor olabilir.

## Sonuç: ölçülebilir fayda YOK

40ms yapay gecikmeyle (aynı `unshare --user --net` + `tc netem` yöntemi, 08 numaralı nota bakınız) tekrar test edildi:

| | Süre |
|---|---|
| Düzeltme ÖNCESİ (08 numaralı not) | 81.16s |
| Düzeltme SONRASI (bu tur, 3 farklı çalıştırma) | 80.95s, ve iki doğrulama çalıştırması aynı aralıkta |

**Pratikte hiç fark yok.** Doğruluk korundu (3/3 başarılı, `driverName = venus`), ama hız kazancı yok.

## Neden işe yaramadı (analiz)

Bir çalıştırmanın log'unda `"client push"` satırları sayıldığında **289** ayrı push işlemi bulundu (`grep -c "client push" ...`). Her biri kendi başına gerçek bir network round-trip'i. Bu, hipotezimin YANLIŞ olduğunu gösteriyor: darboğaz, `vn_relax`'ın busy-wait fazında GEREKSİZ YERE tekrar tekrar deneme yapması değil -- neredeyse her `wait_seqno` çağrısı zaten İLK denemede (iter=0, throttle'dan bağımsız olarak her zaman gerçek bir pull yapılan durum) başarıyla sonuçlanıyor. Yani throttle'ın "kazandırmayı" hedeflediği fazla iterasyonlar zaten pratikte hiç gerçekleşmiyormuş.

**Gerçek maliyet kalemi:** her bir Venus komutunun kendisi zaten minimum ~2-3 zorunlu round-trip gerektiriyor (aktif push = 1, aktif pull'un ilk/tek gerekli denemesi = 1, varsa reply-pool senkronizasyonu = 1) ve **289 gibi yüksek sayıda ayrı komut** oluyor (`vulkaninfo --summary`'nin tam bir Vulkan enstans+cihaz sorgulama dizisi için). Toplam süre ≈ (komut başına zorunlu round-trip sayısı) × (komut sayısı) × (RTT gecikmesi) -- bu, TASARIM GEREĞİ sıralı/senkron bir mimaride kaçınılmaz bir maliyet, "daha az deneme yap" ile düzeltilebilecek bir israf değil.

## Doğru çözüm: gerçek pipelining (çok daha büyük bir iş)

Gerçek bir hız kazancı için, MÜMKÜNSE, birden fazla Venus komutunun host'un HER BİRİNİN cevabını ayrı ayrı beklemeden art arda ring'e yazılıp TEK bir round-trip'te toplu gönderilmesi gerekir (host bunları sırayla işler, sonuçları biriktirir, client sadece EN SON komutun tamamlanmasını bir kez bekler). Bu, `vn_ring_submit_command`/`vn_call_*` katmanının çağrı düzenini (şu an her senkron çağrı kendi push+wait+reply-pull üçlüsünü tek başına yapıyor) yeniden tasarlamayı gerektiren, bu tur içinde güvenle yapılamayacak kadar büyük bir mühendislik değişikliği. Bu, ayrı, daha büyük bir Faz 5 görevi olarak bırakılmalı, spesifik "throttle" gibi küçük ayarlarla çözülemez.

## Değerlendirme ve öneri

Faz 3'ün asıl hedefi (doğruluk kanıtı) zaten sağlam şekilde tamamlandı (07 ve 08 numaralı notlar). Performans optimizasyonu (gerçek pipelining) hem büyük bir mühendislik yatırımı hem de muhtemelen gerçek AWS WAN rakamları eldeyken (Faz 4) daha bilinçli önceliklendirilebilir bir konu -- 10-40ms yapay gecikmeyle "çalışıyor ama yavaş" zaten kanıtlandı, gerçek rakamlar (Türkiye→Frankfurt gerçek RTT) değişmeden büyük bir yeniden-tasarım yatırımı yapmak riskli olabilir.

**Öneri:** Otonom döngü, AWS quota durumu netleşene kadar (kullanıcı tarafında bir konsol kontrolü gerektiriyor, otonom olarak kontrol edilemedi -- `aws` CLI bu makinede kurulu değil) bu noktada duraklayabilir/kullanıcıya durumu net şekilde özetleyebilir. Devam edilecekse, bir sonraki mantıklı adım ya (a) gerçek pipelining tasarımına başlamak (büyük, çok turlu bir iş) ya da (b) kullanıcının AWS quota durumunu bizzat kontrol edip bildirmesini beklemek olabilir.
