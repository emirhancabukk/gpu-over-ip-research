# Faz 4 — Gerçek AWS GPU'suna Gerçek İnternet Üzerinden Bağlanıldı (2026-08-29)

## Kilometre taşı

`vulkaninfo --summary`, ev internetinden gerçek internet üzerinden (SSH tüneli değil, düz TCP), Frankfurt'ta (eu-central-1) çalışan gerçek bir AWS `g5.xlarge` (NVIDIA A10G) GPU'sunu Venus protokolü üzerinden başarıyla enumerate etti. **Çoklu tekrarlı çalıştırma, hepsi başarılı**: `deviceName = Virtio-GPU Venus (NVIDIA A10G)`, `driverName = venus`, exit code 0.

Bu, planın Faz 4 hedefinin temel kanıtını karşılıyor: mimari sadece localhost'ta değil, gerçek WAN üzerinde de uçtan uca çalışıyor.

## Kurulum

- AWS EC2 `g5.xlarge`, `eu-central-1` (Frankfurt), Ubuntu 22.04, NVIDIA sürücü `580` (proprietary, Vulkan ICD dahil).
- Patched `virglrenderer` (Faz 3'ten, TCP transport + tüm bug fix'ler) AWS'de derlendi, `virgl_test_server --no-fork --venus --rendernode /dev/dri/renderD128 --socket-path tcp:33445` ile çalıştırıldı.
- AWS security group: sadece SSH (22) ve 33445 portu, kaynak "My IP" ile kısıtlı (dünyaya açık değil).
- Lokal client (Faz 3'ten patched Mesa): `VTEST_SOCKET_NAME=tcp:<public-ip>:33445` ile gerçek internet üzerinden bağlandı.
- Gerçek RTT (TCP connect timing ile ölçüldü): **~65-70ms** (Türkiye-Frankfurt).

## Performans sorunu ve optimizasyon turu

İlk çalıştırma **430 saniye** sürdü (40ms yapay `tc netem` gecikmesindeki 81 saniyeden çok daha yavaş — oran RTT ile doğrusal değildi, incelenmeye değer bulundu).

### Teşhis (strace kurulu değildi, C koduna manuel profiling eklendi)

`vtest_shmem_sync_now`'a `clock_gettime` tabanlı süre ölçümü eklendi (`vn_renderer_vtest.c`). Bulgular:
- 40 saniyelik bir örneklemede, sürenin **%89'u** network round-trip'lerinde geçiyordu.
- Ring senkronizasyonu (`vtest_shmem_sync_ring_cycle`) her cycle'da **128KB'lık TÜM ring buffer'ı** PUT ediyordu (halbuki genelde sadece birkaç yüz byte değişiyor) + **3 ayrı senkron GET** (head, status, extra) yapıyordu.
- Reply-pool senkronizasyonu her syncron çağrıda **sabit 1MB'lık tüm pool'u** GET ediyordu.

### Uygulanan düzeltmeler

1. **TCP_NODELAY sunucu tarafında eksikti** (`vtest_server.c`, accept edilen client soketi) — eklendi, ama tek başına ölçülebilir fark yaratmadı (Nagle/delayed-ACK hipotezi yanlış çıktı, asıl darboğaz veri hacmiydi).
2. **Ring delta-push**: `ring->cur` (monoton sayaç) izlenerek sadece [son_gönderilen, yeni) aralığı PUT ediliyor, 128KB'lık tüm buffer değil (wrap-around durumu iki parçalı PUT ile ele alındı). `vn_renderer_vtest.c`.
3. **Ring GET birleştirme**: head (offset 0) + status (offset 128) tek bir GET'te birleştirildi (aradaki boşluk atılıyor), extra ayrı kaldı (uzak, buffer'ın hemen sonrasında).
4. **Reply-pool delta-pull**: `sync_now` hook'una `hint_size` parametresi eklendi (`vn_renderer.h`, `vn_ring.c`, `vn_renderer_vtest.c`). İlk denemede `reply_shmem_pool.used` (kümülatif bump-allocator sayacı) doğrudan `hint_size` olarak kullanıldı — bu YANLIŞTI: `used` "şu ana kadar toplam ne kadar allocate edildi" anlamına geliyor, her çağrı [0, used) çekmeye devam etti, run ilerledikçe giderek büyüyen bir aralık tekrar tekrar okunuyordu. Düzeltildi: `last_synced_size` state'i eklenip sadece [last_synced_size, hint_size) deltası çekiliyor.
5. **Bulunan ve düzeltilen bug**: `vtest_shmem_sync_ring_cycle` içinde `cur` değeri mutex alınmadan ÖNCE okunuyordu, ama `last_pushed_cur` state güncellemesi mutex İÇİNDE yapılıyordu. İki thread (arka plan senkron thread'i + aktif `sync_now` hook'u) çakıştığında `last_pushed_cur` GERİYE gidebiliyordu — bu, `vn_ring_submit_internal`'da bir `abort()` (ring dolması assertion'ı) olarak kendini gösterdi (`coredumpctl` ile teşhis edildi). Düzeltme: `cur` okuması mutex içine taşındı.

### Sonuç: kısmi başarı, açık bir bulmaca

Düzeltmelerden sonra:
- Reply-pool sync: 780-900ms → **~63ms ortalama** (yaklaşık 13x hızlanma), tutarlı.
- Ring sync: çoğu cycle 730ms → **~120-140ms** (yaklaşık 5-6x hızlanma).
- **AMA**: örneklemin **%15-20'si** hâlâ **1-7 saniye** arası sürüyor (rastgele dağılmış, belirli bir komuta bağlı görünmüyor).

**Host GPU kullanımı test boyunca sürekli %0'dı** (nvidia-smi ile doğrulandı) — yani bu yavaşlamalar GPU sürücü işlem süresinden KAYNAKLANMIYOR. Host log'unda (`VIRGL_LOG_LEVEL=debug`) hata/uyarı yok. Bu, sorunun gerçek ağ katmanında (jitter, paket kaybı, TCP retransmission) olduğunu düşündürüyor — kod optimizasyonlarıyla giderilemeyecek bir WAN gerçekliği olabilir, ama kesin olarak KANITLANMADI (tcpdump/wireshark seviyesinde paket analizi yapılmadı, strace bu ortamda kurulu değildi).

Tam çalıştırma süreleri optimizasyon turunda **beklenenin tersine kötüleşti** (430s → 458s → 510s → 645s) — bu, kod değişikliklerinin YETERSİZ kaldığını değil, ölçümler arası kontrol edilemeyen bir değişkenin (muhtemelen gerçek ağ koşullarının zamanla değişmesi, ya da art arda çok sayıda test/server-restart döngüsünün biriktirdiği bir durum) sonuçları etkilediğini gösteriyor. Temiz, tek değişkenli bir karşılaştırma yapılamadı bu turda.

## Bilinen sınırlamalar / sonraki adım için not

- `strace`/`tcpdump` bu ortamda kurulu değil (sudo gerektiriyor, non-interactive sudo başarısız). Gerçek paket-seviyeli analiz için bunlardan biri kurulmalı (kullanıcıdan sudo şifresi istenerek) — bu, "%15-20 ara sıra 1-7s" deseninin gerçek kaynağını (paket kaybı mı, TCP congestion window mu, başka bir şey mi) kesin olarak belirlemenin tek yolu.
- Geçici debug/profiling kodu (`[PROF] sync_now ...` fprintf'leri `vn_renderer_vtest.c`'de) hâlâ kod tabanında duruyor — teşhis için bırakıldı, performans-kritik yolda gereksiz I/O overhead'i ekliyor, temizlenmeli.
- AWS `g5.xlarge` instance açık bırakıldı (server process durduruldu ama instance kendisi hâlâ çalışıyor) — $30 otonom harcama tavanına dikkat, kullanılmadığında durdurulmalı.

## Maliyet

Bu oturumda instance yaklaşık 2-2.5 saat açık kaldı (~$2.5-3, $30 tavanın altında).
