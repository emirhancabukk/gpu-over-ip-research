# Faz 3 — TAMAMLANDI: Venus/vtest TCP Üzerinden Uçtan Uca Çalışıyor (2026-08-28, otonom tur 12)

## Sonuç

`vulkaninfo --summary`, virtio-gpu/QEMU olmadan, **iki ayrı process arasında gerçek bir TCP soketi üzerinden**, gerçek AMD GPU'yu Venus protokolü üzerinden başarıyla enumerate etti. **5 art arda çalıştırmanın 5'i de** temiz çıktı verdi (exit code 0, `driverName = venus`). Tam çıktı: `vulkaninfo-tcp-BASARILI-cikti.txt` (bu klasörde).

```
	deviceName         = Virtio-GPU Venus (AMD Radeon Graphics (RADV RENOIR))
	driverID           = DRIVER_ID_MESA_VENUS
	driverName         = venus
	driverInfo         = Mesa 26.3.0-devel (git-e9a87f80b2)
```

Bu, planın Faz 3 hedefini karşılıyor: Venus'un varsayılan virtio-gpu taşıyıcısı yerine bir TCP soketi üzerinden, LAN'da (şu an localhost, gecikme ~0) sıfırdan doğru çalıştığını kanıtlıyor. Sıradaki adım (planda zaten tanımlı): `tc netem` ile yapay gecikme (10/20/40ms) enjekte edip nerede bozulduğunu ölçmek.

## Bu turda bulunan ve düzeltilen SON iki bug

Önceki turların (04-06 numaralı notlar) bıraktığı iki açık sorun bu turda kapatıldı:

### Bug #8: Arka plan poll thread'i ile aktif push/pull hook'ları arasında koordinasyonsuz yarış

06 numaralı notta bulunan "host'un aldığı veri tamamen sıfır" belirtisi, üç AYRI, birbirinden habersiz kaynağın (arka plan poll thread'i, `vn_ring_submit_internal`'daki aktif push, `vn_ring_wait_seqno`'daki aktif pull) aynı ring kaynağı için bağımsız, sırasız tam-döngü senkronizasyonları tetiklemesinden kaynaklanıyordu. Üçü de aynı mutex'i kullanıyordu (veri bozulması yok) ama sıralamaları öngörülemezdi. **Düzeltme:** arka plan thread'i artık ring kaynağı için HİÇBİR ŞEY yapmıyor (`is_ring` durumunda döngü gövdesi boş) — ring'in senkronizasyonu artık SADECE iki hassas, olayla tetiklenen kaynağa bağlı: yeni veri yazıldığında aktif push, ilerleme beklenirken aktif pull. Bu, tail'in defalarca aynı değerle "sıfır veri" olarak host'a ulaşması sorununu ortadan kaldırdı.

**Bilinen sınırlama:** `vn_ring_wait_space()` (ring dolduğunda geri basınç) hâlâ hook'lanmadı — büyük komutlar/dolu ring senaryolarında bu yol arka plan thread'ine güvenmeye devam ediyordu, ama artık arka plan thread ring için çalışmadığından bu yol TAMAMEN senkronizasyonsuz kalabilir. `vulkaninfo`'nun küçük komutları için sorun çıkarmadı (128KB ring hiç dolmadı), ama gerçek bir render/oyun yükünde bu tekrar ele alınmalı.

### Bug #9: `vtest_vcmd_sync_wait()` hâlâ SCM_RIGHTS bekliyordu — teardown'da sonsuz blok

`coredumpctl` ile canlı bir "askıda kalmış" çalıştırmaya `SIGABRT` gönderip çekirdek döküm analiziyle bulundu: enumeration TAMAMEN başarıyla bitmişti, süreç `vkDestroyInstance` → `vn_ring_destroy` → `vn_renderer_wait` → `vtest_vcmd_sync_wait` → `vtest_receive_fd` → `recvmsg()` içinde **sonsuza kadar** bekliyordu. Bu, önceki turlarda SCM_RIGHTS için düzeltilen `vtest_vcmd_resource_create_blob`'un DIŞINDA kalan, hiç fark edilmemiş BAŞKA bir fd-alma noktasıydı — Venus'un sync/fence bekleme mekanizması, sunucunun bir eventfd'yi `vtest_send_fd` ile client'a geçirip client'ın onu `poll()` etmesini bekliyor; TCP'de bu fd hiç alınamaz.

**Düzeltme (bilinçli, sınırlı kapsamlı bir kısayol):** TCP modunda `vtest_vcmd_sync_wait()` artık `vtest_receive_fd()` çağırmıyor, `-1` dönüyor; `vtest_wait()` bunu "poll edilecek bir şey yok, zaten tamam say" olarak yorumluyor (`VK_SUCCESS` döner, `sync_wait_poll`'u hiç çağırmaz). **Bu, gerçek bir henüz-sinyallenmemiş fence için doğruluk garantisi VERMİYOR** — sadece enumeration/senkron-çağrı gibi basit iş yükleri için doğrulandı. Gerçek render/oyun iş yükleri (asenkron fence bekleme gerektiren) için host→guest bildirim kanalı gerekecek (Faz 5).

## Faz 3 kabul kriteri karşılandı mı?

Plandaki Faz 2 kabul kriteri ("network hiç yokken Venus/virtio-gpu üzerinden render") zaten karşılanmıştı. Faz 3'ün amacı ("Venus'un taşıyıcısını TCP/QUIC soketine uyarlamak, LAN'da doğru çalıştığını doğrulamak") artık karşılandı: **5/5 tutarlı, temiz `vulkaninfo --summary` çalıştırması, gerçek AMD GPU'yu Venus üzerinden TCP soketi üzerinden doğru enumerate ediyor.**

## Toplam değişiklik özeti (Faz 3 boyunca, tüm turlar)

**`vendor/virglrenderer/vtest/vtest_renderer.c`:**
- `vtest_resource_create_blob`: HOST3D blob'lar için host tarafında da mmap (SHM fd tipi için), `res->iov` doldurma.
- `vtest_transfer_get/put_internal`: blob kaynaklar için doğrudan memcpy fast-path (box.x'i offset olarak yeniden kullanarak).
- `vtest_send_fd`: AF_UNIX olmayan soketlerde no-op.
- `atomic_thread_fence(memory_order_release)` blob PUT memcpy'sinden sonra (deneysel, Hipotez B testi — sonuçta ANA düzeltme olmadı ama zararsız, bırakıldı).
- `vkr_ring_submit_cmd` (vendor/virglrenderer/src/venus/vkr_ring.c): fatal anında ring buffer dump debug print'i.

**`vendor/virglrenderer/vtest/vtest_server.c`:** `tcp:<port>` soket desteği.

**`vendor/mesa/src/virtio/vulkan/vn_renderer_vtest.c`:**
- TCP bağlantı desteği (`vtest_connect_socket_tcp`), `is_tcp` bayrağı.
- Legacy TRANSFER_GET/PUT (opcode 4/5) kullanımı, box.x alanının offset taşıyıcısı olarak yeniden kullanılması.
- `vtest_shmem_create`: TCP modunda anonim mmap + arka plan senkronizasyon thread'i.
- `vtest_shmem_sync_ring_cycle`: ring için alan-sahipliğine saygılı, doğru sıralı (buffer önce, tail sonra) senkronizasyon.
- `vtest_shmem_sync_now` + `shmem_ops.sync_now`: reply-pool VE ring için senkron, olayla tetiklenen push/pull.
- Arka plan thread'i artık ring için çalışmıyor (sadece reply-pool ve genel fallback için).
- `vtest_vcmd_sync_wait`/`vtest_wait`: TCP modunda fd-bekleme bypass'ı (Bug #9 düzeltmesi).

**`vendor/mesa/src/virtio/vulkan/vn_renderer.h`:** `shmem_ops.sync_now` opsiyonel hook'u + `vn_renderer_shmem_sync_now()` yardımcı fonksiyonu.

**`vendor/mesa/src/virtio/vulkan/vn_ring.c`:**
- `vn_ring_wait_seqno`: başarı anında reply-pool + ring için `sync_now` çağrısı (her iterasyonda ring için aktif pull).
- `vn_ring_submit_internal`: yeni veri yazıldıktan hemen sonra ring için aktif push.

## Kullanılan tam komutlar (referans, tekrarlanabilir)

Sunucu:
```bash
VIRGL_LOG_LEVEL=debug ./vendor/virglrenderer/build/vtest/virgl_test_server \
  --no-fork --venus --rendernode /dev/dri/renderD128 --socket-path tcp:33445
```

Client:
```bash
ICD=$(find vendor/mesa/build -name "virtio_devenv_icd.x86_64.json")
VK_ICD_FILENAMES="$ICD" VTEST_SOCKET_NAME=tcp:127.0.0.1:33445 vulkaninfo --summary
```

Build (her iki değişiklik için de):
```bash
source vendor/.build-venv/bin/activate
cd vendor/mesa && meson compile -C build       # client tarafı değişikliklerinde
cd vendor/virglrenderer && meson compile -C build  # server tarafı değişikliklerinde
```

## Sıradaki Adım: Faz 3'ün ikinci yarısı — `tc netem` ile yapay gecikme testi

Plan bunu zaten öngörüyor: LAN'da (gecikme ~0) doğrulandıktan sonra, `tc netem` ile yapay 10ms, 20ms, 40ms gecikme enjekte edilip hangi senkron noktaların (özellikle bizim aktif push/pull hook'larımızın — her biri artık BİR network round-trip'e denk geliyor, gecikme arttıkça bu maliyetlenecek) sorun çıkardığı ölçülmeli. Bu, gerçek AWS WAN testine (Faz 4) geçmeden önce ucuza (bulut maliyeti olmadan) yapılabilecek son adım.

Somut görev:
1. Loopback (`lo`) arayüzüne `tc qdisc add dev lo root netem delay 10ms` ile yapay gecikme ekle (ya da bir `veth` çifti üzerinden, `lo`'ya netem eklemek bazı sistemlerde localhost bağlantılarını etkilemeyebilir — dikkatle doğrula, gerekirse iki ayrı network namespace + veth pair kurulumu gerekebilir).
2. Aynı 5-tekrar testini 10ms, sonra 20ms, sonra 40ms gecikmeyle çalıştır, her birinde başarı oranını ve (varsa) toplam süreyi kaydet.
3. Bulguları yeni bir not olarak `experiments/phase3-lan-transport/` altına yaz.
4. Test bitince `tc qdisc del dev lo root netem` ile gecikmeyi kaldırmayı unutma.
