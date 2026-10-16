# Faz 5 — Kök Neden: Arka Plan Thread'i Koşu Başına 258 MB Boşa Transfer Ediyor (2026-09-04)

## Özet

Not 06'nın "yavaşlığın sebebi yerel internet bağlantısının throughput kapasitesi"
sonucu **muhtemelen yanlış**. Ölçüm, client'ın kendi indirme kapasitesini
doldurduğunu gösteriyor: 266 saniyelik bir koşuda reply-pool arka plan thread'i
**258 kez tam 1 MB** çekti (258 MB), aynı koşuda faydalı veri **1.5 KB**'ti.

## Kurulum

Aynı AWS `g5.xlarge` (Frankfurt, NVIDIA A10G, sürücü 580.173.02), yeni bir
public IP ile. Sunucu binary'si 29 Ağustos'tan, yeniden derlenmedi. Sunucudaki üç
kaynak dosya (`vtest_server.c`, `vtest_renderer.c`, `vkr_ring.c`) depodaki
sürüm kontrollü kopyalarla **byte-byte aynı** olduğu doğrulandı — deney
tekrarlanabilir.

Security group'un 33445 kuralı eski ev IP'sine pinliydi; güncel ev IP'sine
(`/32`) güncellendikten sonra bağlantı açıldı.

Koşu öncesi ve sonrası TCP-connect RTT: medyan **89-91 ms**, yayılma **27-30 ms**
(31 Ağustos'ta bölge endpoint'ine 66.1 ms medyan / 3.3 ms yayılma ölçülmüştü).

## Koşu sonucu: çöktü

`vulkaninfo --summary`, `exit=134` (SIGABRT), **266 saniye**, hiç çıktı üretmeden.
282 `sync_now` çağrısı tamamlanmış (251 ring + 31 reply-pool). Geçen turun 353
ring sync'ine ulaşamadı.

Coredump backtrace (`coredumpctl info 53468`):

```
abort
vtest_read                                  <-- burada
vtest_vcmd_transfer_get2
vtest_shmem_sync_ring_cycle
vtest_shmem_sync_now
vn_renderer_shmem_sync_now
vn_ring_submit_command
vn_submit_vkGetPhysicalDeviceImageFormatProperties2
```

Ring dolması assertion'ı **değil**; socket okuması başarısız oldu. Sunucu logunun
son satırı `client: VTEST_CLIENT_ERROR_INPUT_READ` — yani önce sunucunun client'tan
okuması başarısız oldu, sunucu bağlantıyı kapattı, sonra client'ın okuması patladı.

Not: backtrace'teki `vtest_vcmd_transfer_get2` ismi, ölü sokak #15'e (GET2/PUT2
opcode'ları) dönüldüğü anlamına gelmez — bu client tarafı yardımcı fonksiyonun
adı, gönderdiği opcode hâlâ eski 4.

## Ölçüm: trafiğin dağılımı

Sunucu logu (`/home/ubuntu/vtest-20260904-123435.log`) sayıldı:

| Reply-pool `transfer_get` kaynağı | Sayı | Transfer boyutu |
|---|---|---|
| Arka plan thread'i | **258** | `offset=0 size=1048576` (tam 1 MB) |
| Aktif `sync_now` hook'u | 31 | 24-64 byte, artan offset (delta doğru çalışıyor) |

Sunucudaki `fprintf`'in bastığı `size` alanı `data_size`, yani istenen gerçek
transfer boyutu — kaynağın toplam boyutu değil (`vtest_renderer.c:1656`).
Dolayısıyla `1048576` gerçekten aktarılan bayt sayısıdır.

31 sayısı, client'ta `[PROF]` satırlarından sayılan `is_ring=0` sayısıyla birebir
aynı. Yani delta yolunu **yalnızca aktif hook** kullanıyor; arka plan thread'i
`hint_size` vermediği için `vtest_shmem_sync_now`'daki delta dalına
(`hint_size > 0 && hint_size <= mmap_size`) hiç girmiyor ve son `else` dalındaki
tam-havuz GET'ine düşüyor.

**258 MB / 266 s ≈ 7.8 Mbps sürekli, tamamen boşa giden indirme.**

## Neden bu, not 06'nın teşhisini geçersiz kılıyor

Not 06 şu üç gözlemden "sorun ne kodda ne host'ta, kalan tek açıklama yerel
bağlantının throughput kapasitesi" sonucuna varmıştı: RTT sabit ~72-75 ms, host
tamamen boşta, kod bilinen-iyi haline döndürülmüş.

Üç gözlem de doğru ama sonuç eksik. Sürekli 7.8 Mbps'lik faydasız indirme ev
bağlantısını doyurduğunda ortaya çıkan tablo tam olarak bu:

- **Küçük TCP-connect probe'ları etkilenmez** — minik paketler kuyruğu geçer, o
  yüzden RTT ölçümü sabit görünür.
- **Gerçek veri transferleri çöker** — ring sync'leri, koşunun başında ölçülen
  174-201 ms'den (2 RTT, tasarımın teorik tabanı) çökme anında **755-1051 ms**'ye
  çıkmış durumda.
- **Host boşta görünür** — GPU gerçekten çalışmıyor, çünkü darboğaz ağ.

Yani "kod değil" çıkarımı, aranan kodun ölçülen kod olmamasından kaynaklandı:
suçlu, `sync_now`'un profillenen aktif yolu değil, hiç profillenmeyen arka plan
thread'i.

Not 01'in "örneklemin %15-20'si 1-7 saniye, rastgele dağılmış, GPU %0" bulgusu da
aynı mekanizmanın belirtisi: 100 ms'de bir başlayan 1 MB'lık indirme, o aralığa
denk gelen ring sync'ini kuyrukta bekletiyor.

Optimizasyon #4'ün (thread periyodu 2 ms → 100 ms, "~2.8x hızlanma") neden işe
yaradığı da böylece açıklanıyor: kök nedeni düzeltmedi, tam-havuz çekmelerini 50
kat seyrekleştirdi. 2 ms periyotta talep saniyede ~500 MB olur ki bu imkânsız —
ilk ölçümlerdeki 780-900 ms'lik reply-pool sync'lerinin sebebi de bu.

## Bu zaten biliniyordu, ölçülmemişti

`vn_renderer_vtest.c:729` civarındaki yorum durumu doğru tarif ediyor: *"reply-pool
dalı her çalıştığında KÖR, tam 1 MB GET yapıyor — ring'in aksine delta takibi yok
... gerçek WAN üzerinde büyük, sürekli, çoğu boşa giden bir trafik akışı."*

Eksik olan, bunun **ölçülmüş büyüklüğü** ve **not 06'daki yavaşlığın sebebi
olduğunun** görülmesiydi. Yorum bunu bir verimsizlik olarak kaydetmiş, tıkanmanın
kök nedeni olarak değil.

## Düzeltme yönü ve neden "sil gitsin" olmadığı

Aynı yorum bir ölü sokak da kaydediyor: bu dalı **tamamen kaldırma denemesi geri
alınmış** — 10 dakikada yalnızca 8 senkron çağrı, ~485 ms ortalama (100 ms'lik
sürümde ~63 ms'ti), RTT o sırada yeniden ölçülüp değişmediği doğrulanmışken.
Kök neden izole edilmemiş. Yani aktif hook tek başına yeterli bir ikame değil,
sebebi hâlâ bilinmiyor.

Bu yüzden düzeltme, thread'i kaldırmak değil **çektiği miktarı sınırlamak**
olmalı: aynı 100 ms periyodunda kalıp, `offset=0 size=1048576` yerine
`shmem->last_synced_size`'dan başlayan sınırlı bir pencere (örn. 4-8 KB) çekmek.
Bir reply küçüktür; bu dalın işi zaten yalnızca aktif hook'un kaçırdığı nadir
durumu yakalamak. Beklenen etki: 100 ms'de 1 MB yerine 100 ms'de ~4 KB, yani
~250 kat azalma, güvenlik ağı korunarak.

Kaldırma denemesini bozan mekanizma hâlâ bilinmediği için, periyodu ve dalın
varlığını değiştirmemek bilinçli bir tercih — tek değişken transfer boyutu olsun.

## Yanlışlama testi

Bu hipotez ucuz biçimde çürütülebilir: yukarıdaki sınırlı-pencere değişikliğinden
sonra aynı koşuyu tekrarlamak. Hipotez doğruysa sunucu logunda `size=1048576`
satırları kaybolur, ring sync süreleri koşu boyunca 174-201 ms bandında kalır
(sonuna doğru 750 ms+'ye tırmanmaz) ve koşu çökmeden tamamlanır. Hipotez yanlışsa
yavaşlama transfer boyutundan bağımsız olarak devam eder.

## Ayrıca bulunan: `vtest_read` hata yolunda dayanıksız

```c
const ssize_t ret = read(vtest->sock_fd, ptr, size);
if (unlikely(ret < 0)) { vn_log(...); abort(); }
ptr += ret; size -= ret;
} while (size);
```

İki sorun:

1. `ret < 0` → doğrudan `abort()`. Geçici bir TCP hatası tüm koşuyu öldürüyor;
   yeniden deneme veya temiz hata döndürme yok. WAN üzerinde çalışan bir sistem
   için bu, uzun koşuların yapısal olarak kırılgan olması demek.
2. `ret == 0` (karşı taraf bağlantıyı düzgün kapattı) hata sayılmıyor: `size`
   hiç azalmadığı için döngü sonsuza gidiyor — EOF durumunda meşgul bekleme.

Bu, kök nedenden bağımsız ayrı bir dayanıklılık işi.

## Durum (güncellendi 2026-10-16)

- Sınırlı-pencere düzeltmesi 4 Eylül'de, bu not yazıldıktan sonra uygulandı ve
  temiz derlendi; 16 Ekim'de `patches/mesa/0002-*.patch` olarak commit'lendi.
  **Doğrulama koşusu henüz yapılmadı** — yukarıdaki yanlışlama testi hâlâ açık.
- Ders: security group kuralı ev IP'sine pinliyse, IP değiştiğinde bağlantı
  sessizce kırılır — `connection refused` değil `timeout` görülür.

## Sonradan not (2026-10-16): netem bu sorunu neden göremedi

Faz 3'teki netem testleri (`phase3-lan-transport/08-*.md`) yalnızca gecikme
ekledi (`netem delay <N>ms`), bant genişliği sınırı yoktu. Sınır olmayınca 1 MB'lık
bir çekme birkaç round trip'te biter; 8 Mbps'lik bir ev bağlantısında ise ~1 saniye
sürer ve bu süre boyunca tek socket'i (ve `sock_mutex`'i) tutar. 258 MB sorununun
yerel testlerde hiç görünmemesinin sebebi bu: netem gecikmeyi doğru modelliyor, bant
genişliğini hiç modellemiyor.

İyi tarafı: yanlışlama testi AWS gerektirmiyor. netem'e bir bant genişliği sınırı
eklemek (örneğin `netem delay 35ms rate 10mbit`) ev bağlantısını yerelde taklit
eder; düzeltmeli ve düzeltmesiz client aynı koşulda karşılaştırılabilir.
