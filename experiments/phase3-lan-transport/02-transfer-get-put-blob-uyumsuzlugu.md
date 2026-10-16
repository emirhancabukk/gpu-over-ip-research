# Faz 3 — transfer_get/put Blob Kaynaklarını Desteklemiyor (2026-08-27, otonom tur 7 devamı)

## Bu turda yapılan düzeltmeler (sırayla)

1. **SCM_RIGHTS çökmesi düzeltildi**: `vtest_vcmd_resource_create_blob`, TCP modunda (`vtest->is_tcp`) artık `vtest_receive_fd()` çağırmıyor, `res_fd=-1` dönüyor. `vtest_shmem_create`, `res_fd<0` ise gerçek fd yerine **yerel anonim mmap** (`MAP_ANONYMOUS`) kullanıp, arka planda bir **senkronizasyon thread'i** başlatıyor (`vtest_shmem_sync_thread_func`) — bu thread periyodik olarak (2ms) local belleği host'a `transfer_put` ile gönderip `transfer_get` ile geri çekiyor.

2. **İlk bug**: `VCMD_TRANSFER_GET2`/`PUT2` (opcode 13/14) kullandım — bunlar sunucuda **data_size alanını hiç okumuyor** (`vtest_transfer_decode_args2`'de `DATA_SIZE` field'ı parse edilmiyor), her zaman `data_size=0` ile çağrılıyor, yani **0 byte transfer ediyor.** Client'ım `size` byte yazıp/okumayı beklerken sunucu 0 byte gönderip/okuyunca stream bozuluyordu (sonraki komutun header'ı yanlış yerden okunmaya başlıyordu — `Assertion vtest_hdr[VTEST_CMD_LEN]==1 failed`, `VTEST_CLIENT_ERROR_INPUT_READ`).

3. **Düzeltme**: Eski `VCMD_TRANSFER_GET`/`PUT` (opcode 4/5, 11-dword header, `DATA_SIZE` alanı gerçekten okunuyor — `vtest_transfer_decode_args`'de `*data_size = thdr_buf[VCMD_TRANSFER_DATA_SIZE]`) komutlarına geçtim. Stream bozulması düzeldi, artık **temiz bir protokol-seviyesi hata** alıyoruz.

## Yeni (ve muhtemelen temel) engel: transfer_get/put, blob tipi kaynaklarda çalışmıyor

```
vtest_transfer_put_internal called virgl_renderer_transfer_write_iov which failed (-1)
client: VTEST_CLIENT_ERROR_COMMAND_DISPATCH
```

`virgl_renderer_transfer_write_iov()` (`src/virglrenderer.c:316`) kaynağı okundu: çağrıyı context'in `transfer_3d()` callback'ine yönlendiriyor (`ctx->transfer_3d(ctx, res, &transfer_info, VIRGL_TRANSFER_TO_HOST)`). Bizim context'imiz **Venus proxy context** (capset_id=4) — bu proxy'nin `transfer_3d` implementasyonu **blob tipi kaynaklar için başarısız oluyor.**

**Mantıklı açıklama:** Blob kaynaklar (host-visible `VkDeviceMemory` / ring buffer belleği), tasarım gereği **doğrudan mmap ile erişilecek** şekilde düşünülmüş — "transfer" (box-copy) API'si, klasik virgl 2D/3D texture kaynakları (level/format/stride'lı) için var. Blob'un byte'larına box-copy API'siyle erişmenin bir yolu **muhtemelen hiç yok** — bu, vtest protokolünün üzerine kurulduğu tasarımın doğal bir sonucu, bizim bir şeyi yanlış kullanmamızdan değil.

## Sonuç / Genel Değerlendirme

Faz 1'de umutla not ettiğimiz "transfer_get/put zaten byte-stream, WAN'a uygun" bulgusu **kısmen yanlış çıktı**: transfer_get/put mekanik olarak byte-stream taşıyor (doğru), ama **sadece klasik texture kaynaklar için**, blob kaynaklar (Venus'un ihtiyaç duyduğu tam olarak bu) için **çalışmıyor**. Bu, mevcut vtest protokolünde **hazır bir çözüm olmadığı** anlamına geliyor — WAN üzerinden blob senkronizasyonu için ya:

1. **vtest protokolüne yeni bir komut eklemek** gerekiyor — sunucu tarafında blob'un backing belleğine (muhtemelen `res->iov` veya proxy context'in kendi bellek alanı) doğrudan memcpy yapan, box/format kavramı olmayan basit bir "blob_get/blob_put" komutu. Bu, virglrenderer'ın sunucu tarafına (vtest_renderer.c + vtest_protocol.h + protokolü işleyen proxy context kodu) gerçek bir değişiklik yazmak demek — küçük ama gerçek bir protokol uzantısı.
2. Ya da: blob'un proxy context içinde (render server worker process/thread'inde) nasıl saklandığını bulup, oradan host-side'da doğrudan bellek erişimi sağlayan farklı bir API/fonksiyon var mı araştırmak (örn. `vkr_renderer.c`/`proxy_context.c` içinde blob'a "map" eden bir fonksiyon olabilir, onu yeni bir vtest komutuyla dışa açmak).

## Sonraki Tur İçin Somut Görev
1. `vendor/virglrenderer/src/proxy/` (ya da meson çıktısında gördüğümüz `proxy_proxy_*.c` dosyaları) içinde blob resource'ların host tarafında nasıl temsil edildiğini bul — muhtemelen bir `void *` pointer + size olarak saklanıyor (çünkü mmap edilebilir olması gerekiyordu).
2. O pointer'a erişimi sağlayan (ya da eklenebilecek) bir fonksiyon var mı bak.
3. Bulursan, `vtest_protocol.h`'a yeni bir basit komut (örn. `VCMD_BLOB_READ`/`VCMD_BLOB_WRITE`, sadece res_id + offset + size + data) ekleyip hem `vtest_server.c`/`vtest_renderer.c`'ye (host) hem `vn_renderer_vtest.c`'ye (client) minimal bir implementasyon yaz. Bu, gerçek bir protokol uzantısı yazmak — büyük bir adım ama net bir hedef.
