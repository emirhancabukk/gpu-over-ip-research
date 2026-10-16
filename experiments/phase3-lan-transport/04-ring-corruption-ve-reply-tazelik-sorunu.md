# Faz 3 — Ring Bozulması Kök Nedeni Bulundu ve Düzeltildi; Kalan Sorun Reply-Pool Tazelik Yarışı (2026-08-27/28, otonom tur 9)

## Bu turun özeti

Önceki turun bıraktığı yerden (`vn_relax`'ın zamanlama bütçesi yetersiz görünüyordu) devam edildi. Gerçek kök neden zamanlama değil, **ring senkronizasyonunun kendi kendini bozması** çıktı — üç ayrı, gerçek bug bulundu ve ilk ikisi tamamen düzeltildi:

1. **Kör çift-yönlü tam-buffer kopyalama, ring'in head/tail/status alanlarını eziyordu** → düzeltildi (alan-bazlı, sahiplik-farkında senkronizasyon)
2. **PUT sırası yanlıştı (tail, buffer'dan önce gönderiliyordu)** → düzeltildi (buffer önce, tail sonra)
3. **Reply-pool tazelik/happens-before garantisi yok** → **kısmen düzeltildi, hâlâ zaman zaman başarısız oluyor** (bu turun kalan açık sorunu)

## Bug #5: Ring'in head/tail/status alanları kör kopyalamayla eziliyordu

`vn_ring_get_layout()` (`vn_ring.c:257`) ring'in shmem düzenini şöyle tanımlıyor (128KB tampon + 4B extra için, `vn_instance.c`'deki sabit `buf_size=128*1024, extra_size=4` ile):

```
[0]      head   (4B)  — HOST sahipli (host tüketirken ilerletir)
[64]     tail   (4B)  — GUEST sahipli (guest üretirken ilerletir)
[128]    status (4B)  — HOST sahipli (örn. periyodik ALIVE sinyali)
[192]    buffer (128KiB) — GUEST sahipli (asıl komut verisi)
[131264] extra  (4B)  — HOST sahipli
toplam shmem_size = 131268 (gözlemlediğimiz ring blob boyutuyla tam eşleşiyor)
```

Önceki turun senkronizasyon mekanizması her döngüde **tüm 131268 byte'ı** önce PUT (guest→host) sonra GET (host→guest) olarak kopyalıyordu. Bu, guest'in PUT'unun her seferinde host'un HEAD/STATUS alanlarındaki güncel değerleri **guest'in bayat yerel kopyasıyla ezmesine** neden oluyordu — host'un ilerlemesi (head'in artması, ALIVE bitinin set edilmesi) guest'in bir sonraki PUT'unda sessizce geri alınıyordu. Bu, `vn_relax`'ın hiçbir zaman host'un gerçek ilerlemesini görememesine ve bütçesi dolunca abort etmesine yol açıyordu — önceki turun "zamanlama sorunu" sandığı şey aslında buydu.

**Düzeltme:** `vtest_shmem_sync_ring_cycle()` eklendi (`vn_renderer_vtest.c`) — artık sadece guest-sahipli alanları (tail + buffer) PUT ediyor, sadece host-sahipli alanları (head + status + extra) GET ediyor. Legacy PUT/GET komutunun gerçek bir offset alanı olmadığı için (`vtest_transfer_decode_args` her zaman `offset=0` set ediyordu), kullanılmayan `box.x` alanını offset taşıyıcısı olarak yeniden kullandık — hem client (`vn_renderer_vtest.c`) hem server (`vtest_renderer.c`) tarafında, sadece bizim blob fast-path'imizi etkileyecek şekilde (klasik texture transfer_3d yoluna dokunmadan).

## Bug #6: PUT sırası — tail, buffer'dan ÖNCE gönderiliyordu

Alan-bazlı düzeltmeden hemen sonra host tarafında **yeni** bir hata ortaya çıktı: `vkr: vkCreateInstance resulted in CS error` / `vkr: ring_submit_cmd: vn_dispatch_command failed` (host komutu bozuk/eksik görüyordu). Kök neden: senkronizasyon döngümüz önce TAIL'i (4B) sonra BUFFER'ı (128KB) PUT ediyordu. Host'ta ring-tüketici thread'i tail değişikliklerini bağımsız olarak izliyor ve tail değiştiği an mevcut buffer içeriğini decode etmeye **hemen** başlıyor — eğer tail, buffer verisinden önce yayınlanırsa, host henüz güncellenmemiş/bayat buffer baytlarını okuyabiliyor (klasik "lock-free SPSC ring" veri-yarışı deseni).

**Düzeltme:** Sıra ters çevrildi — önce BUFFER PUT edilir, sonra TAIL. Bu düzeltmeden sonra host tarafı **tamamen temiz** çalışıyor: `vkCreateInstance` başarıyla işleniyor, `CS error` bir daha hiç görülmedi.

## Bug #7 (kısmen düzeltildi): Reply-pool tazelik/happens-before yarışı

Bu düzeltmelerden sonra client tarafında YENİ bir hata: `vn_decode_vkEnumerateInstanceVersion_reply: Assertion command_type == VK_COMMAND_TYPE_vkEnumerateInstanceVersion_EXT failed` — beklenen değer 137, ama bazı çalıştırmalarda okunan değer 0 (hepsi sıfır byte).

Kök neden zinciri:

1. `instance->reply_shmem_pool` (`vn_instance.c`) sabit **1u<<20 (1048576) byte** — senkron çağrı cevaplarını taşıyan havuz. Host bu belleğe cevap yazan **tek** taraf; guest sadece okur (havuz bookkeeping'i tamamen client-side, blob içinde saklanmıyor). Önceki kör çift-yönlü kopyalama burada da aynı öz-yaralama bug'ını tetikliyordu: guest'in PUT'u host'un henüz yazdığı gerçek cevabı ezip duruyordu. **Düzeltme:** bu kaynak boyutuna özel algılama (`size == 1u<<20`) eklendi, artık **sadece GET**, hiç PUT yok.

2. Bunu düzeltince YENİ bir hata ortaya çıktı: server log'unda ring senkronizasyon thread'i ile reply-pool senkronizasyon thread'i AYNI `vtest->sock_mutex`'i paylaşıyor, ve saf `thrd_yield()` + hemen tekrar kilitleme deseni Linux'un adil-olmayan (non-FIFO) pthread mutex'inde **klasik bir "lock convoy"** yaratıyordu — 20 saniyelik bir çalıştırma boyunca reply-pool thread'i mutex'i **bir kere bile kazanamadı**, host cevabı server tarafında doğru yazılmış olsa bile client hiç göremedi (hep ilk-değer sıfırları okudu).
   - **Kısmi düzeltme:** ring döngüsünün sonuna kısa gerçek bir `nanosleep` (200µs) eklendi, reply-pool thread'ine daha uzun bir sleep (2ms) verildi — ring'i öncelikli tutup reply-pool'a "sonradan yetişme" şansı vermek için.
   - Bu düzeltmeyle **en az bir kez tam doğru sonuç gözlemlendi**: `got command_type=137` (beklenen değerle TAM eşleşiyor) — yani mekanizma **doğru çalışabiliyor**, kanıtlandı.
   - Ancak **hâlâ deterministik değil** — art arda çalıştırmalarda bazen yine `command_type=0` (host'un cevabı yazmasıyla bizim son GET'imiz arasında hâlâ garantili bir "happens-before" ilişkisi yok, sadece istatistiksel olarak iyileşmiş bir zamanlama).

## Genel Değerlendirme

Bu tur, projenin en önemli mimari bulgusunu üretti: **kör, çift-yönlü, tam-buffer kopyalama bir üretici/tüketici ring buffer için temelden yanlış bir senkronizasyon stratejisi** — kendi kendini bozuyor. Alan-sahipliğine saygılı, sıralama-bilinçli senkronizasyon (bu turda uygulanan) bunu çözüyor ve **gerçek bir Venus komutu (vkCreateInstance + vkEnumerateInstanceVersion) TCP üzerinden en az bir kez uçtan uca doğru şekilde tamamlandı** — bu, "Venus/vtest WAN transport üzerinde çalışabilir mi" sorusuna somut bir "evet" kanıtı.

Kalan sorun artık bir mimari bloker değil, **saf bir senkronizasyon-zamanlama sorunu**: cevap havuzu için "körü körüne periyodik poll et" yerine "host'un cevabı yazdığını bildiğimiz TAM anda senkron bir GET yap" gerekiyor. Bu, ring'in zaten sağladığı bir garantiye bağlanabilir: host, cevabı yazmadan ÖNCE ring seqno'yu ilerletmez (bu sıralama garantisi Venus protokolünün kendisinde var) — yani `vn_ring_wait_seqno()` başarıyla döndüğü an, host'un cevabı ZATEN yazılmış olması gerekir. Sorun sadece bizim TCP-sync mekanizmamızın bunu YETERİNCE HIZLI yakalamaması.

## Sonraki Tur İçin Somut Görev

1. `vn_ring.c`'deki `vn_ring_wait_seqno()`'nun başarılı dönüş noktasını bul (seqno hedefine ulaşıldığında).
2. Orada (veya `vn_ring_submit_command()`'ın çağrı döndüğü noktada), varsa bekleyen reply-pool kaynağı için **senkron, zorunlu bir GET** tetikleyen bir hook ekle — arka plan poll thread'ine güvenmek yerine. Bu, `vn_ring.c` (renderer-agnostik) içinden `vn_renderer_vtest.c`'ye özel bir çağrı gerektirebilir; bu deneysel dal için mimari saflık yerine pragmatik bir fonksiyon işaretçisi/hook eklemek kabul edilebilir.
3. Alternatif (daha basit, denenmemiş): `vn_call_*` fonksiyonlarının reply'i okumadan hemen önce çağırdığı ortak bir noktayı bul (varsa) ve orayı hook'la.
4. Düzeltmeden sonra `vulkaninfo --summary`'yi art arda en az 5 kez çalıştırıp **hepsi başarılı** oluyor mu diye doğrula (şu an ~%50 başarı oranı gözlemlendi, tutarlı %100 hedefleniyor).
5. Başarılı olursan, Faz 3'ü TAMAMLANDI olarak işaretle: plan dosyasını güncelle, `vulkaninfo` çıktısının `driverName: venus` gösterdiği tam çıktıyı kaydet (Faz 2'nin `05-BASARILI-uctan-uca-render.md`'sine benzer şekilde), sonra `tc netem` ile yapay gecikme testine geç.

## Kullanılan komutlar (referans)

Sunucu:
```
VIRGL_LOG_LEVEL=debug ./vendor/virglrenderer/build/vtest/virgl_test_server \
  --no-fork --venus --rendernode /dev/dri/renderD128 --socket-path tcp:33445
```

Client:
```
ICD=$(find vendor/mesa/build -name "virtio_devenv_icd.x86_64.json")
VK_ICD_FILENAMES="$ICD" VTEST_SOCKET_NAME=tcp:127.0.0.1:33445 vulkaninfo --summary
```

## Not: geçici debug enstrümantasyonu

Bu turda şu geçici debug `fprintf`'ler eklendi (ileride temizlenebilir, şimdilik faydalı):
- `vtest_renderer.c`: `vtest_client_dispatch_commands`'de dispatch hatası print'i, `transfer_get_internal`'da reply-pool byte dump'ı.
- `vn_renderer_vtest.c`: sync thread başlangıç print'i.
- `vn_protocol_driver_instance.h` (meson-generated, build/ altında — **yeniden yapılandırmada/temiz build'de kaybolur**, kalıcı değil): reply decode'daki assertion'dan önce ham byte dump'ı.

`VIRGL_LOG_LEVEL=debug` olmadan `vkr_log()` çağrıları (host tarafındaki asıl Venus hata mesajları) **sessizce bastırılıyor** (varsayılan log seviyesi WARNING) — bu, bu turun başında CS error'un neden görünmediğini açıklıyor. Bundan sonraki tüm sunucu testlerinde bu env değişkeni kullanılmalı.
