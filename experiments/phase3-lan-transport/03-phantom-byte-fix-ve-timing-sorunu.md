# Faz 3 — "Phantom Byte" Bug Bulundu ve Düzeltildi, Şimdi Zamanlama Sorunu (2026-08-27, otonom tur 8)

## Bu turun özeti: 4 gerçek bug, 3'ü düzeltildi

Bu oturumda TCP transport denemesinde art arda 4 farklı, gerçek bug bulundu ve çoğu düzeltildi — sırayla:

1. ~~SCM_RIGHTS çökmesi~~ → düzeltildi (önceki tur, yerel anonim mmap + sync thread)
2. ~~TRANSFER_GET2/PUT2'nin data_size'ı okumaması~~ → düzeltildi (önceki tur, eski GET/PUT opcode'larına geçildi)
3. ~~transfer_get/put'un blob kaynaklarda çalışmaması~~ → düzeltildi (bu tur, `vtest_resource_create_blob`'da HOST3D blob'lar için de `res->iov` doldurulup transfer_get/put_internal'da blob'lar için doğrudan memcpy fast-path eklendi)
4. **YENİ bulunan "phantom byte" bug'ı** → düzeltildi

## Bug #4: `vtest_send_fd()` TCP'de sessizce 1 fazla byte gönderiyor

`vtest_send_fd()` (server, `vtest_renderer.c:439`), `sendmsg()` ile 1 baytlık dummy payload + SCM_RIGHTS ancillary data gönderiyor. **AF_INET/TCP soketinde SCM_RIGHTS ancillary data kernel tarafından sessizce yok sayılıyor ama 1 baytlık dummy payload YİNE DE gönderiliyor** (`sendmsg` hata döndürmüyor, `size>=0`). TCP-modundaki client, fd almayı zaten atladığı için (Bug #1'in düzeltmesi) bu 1 fazla byte'ı **hiç okumuyor** — stream'de kalıyor ve bir sonraki komutun header'ının başlangıcını 1 byte kaydırıyor. Bu, her `vtest_send_fd()` çağrısında (blob create, sync create, fence export, vb. — kod tabanında ~10 çağrı yeri var) tekrarlanan, birikimli bir stream bozulması.

**Kanıt zinciri:** İlk blob (ring, boyut 131268) için sorun görünmüyordu (rastlantısal — sonraki komut byte-hizası kazara toleranslıydı), ama İKİNCİ blob (reply shmem, boyut 1048576) oluşturulurken client'ın response-header assertion'ı (`vtest_hdr[VTEST_CMD_LEN]==1`) patlıyordu. Sync thread'i geçici olarak `VTEST_TCP_NO_SYNC_THREAD=1` env değişkeniyle kapatıp aynı hatanın YİNE oluştuğunu görünce (bkz. `vendor/mesa/.../vn_renderer_vtest.c`'deki geçici env-gate), bunun bir **threading race değil, deterministik bir framing bug'ı** olduğu kesinleşti — bu, zamandan tasarruf sağlayan iyi bir teşhis adımıydı.

**Düzeltme:** `vtest_send_fd()`'ye `getsockname()` ile soket ailesini kontrol eden bir kontrol eklendi — `AF_UNIX` değilse (yani TCP ise) fonksiyon hiçbir şey göndermeden `0` (başarı) dönüyor.

## Düzeltmeden sonraki durum: framing bug'ı gitti, şimdi bir ZAMANLAMA sorunu var

Phantom-byte düzeltmesinden sonra:
- Sync thread kapalıyken: `vn_ring_wait_seqno` sonsuza kadar bekleyip `vn_relax` üzerinden abort ediyor (**beklenen/doğru davranış** — sync yoksa ring ilerlemez, bu bizim tasarımımızın öngördüğü bir sonuç).
- Sync thread açıkken (2 blob için 2 ayrı sync thread çalışıyor, `transfer_put`/`get` çiftini 2ms'de bir yapıyorlar): **hâlâ aynı yerde** (`vn_ring_wait_seqno` → `vn_relax` → abort) takılıyor, ama artık framing hatası değil — muhtemelen sync thread'lerin senkronizasyon **hızı** `vn_relax`'ın sabit retry sayısına göre çok yavaş kalıyor (`vn_relax`, iterasyon sayısına göre pes ediyor gibi görünüyor, wall-clock süreye göre değil — TCP round-trip'lerimiz bunu aşıyor).

## Sonraki Tur İçin Somut Görev
1. `vn_common.c`'deki `vn_relax()` fonksiyonunu oku — kaç iterasyonda/ne kadar sürede pes ettiğini netleştir.
2. Sync thread'in 2ms'lik `nanosleep`'ini kaldırıp (veya çok kısaltıp) mümkün olduğunca hızlı poll etmeyi dene — en azından "çalışıyor mu" kanıtlamak için, performans endişesi olmadan (Faz 5'in işi).
3. Alternatif: sadece TEK sync thread'in (ring için olan) polling'ini test için çok agresif (spin-loop, sleep yok) yapıp, reply-shmem thread'i ayrı bırakarak izole test et — hangi ring'in gecikmesi kritik anlaşılır.
4. Eğer agresif polling de yetmezse, `vn_relax`'ın retry sınırını build-time'da artırmayı (deneysel, sadece TCP prototipimiz için) değerlendir.

## Not: env-gate geçici debug kodu
`vendor/mesa/.../vn_renderer_vtest.c`'de `VTEST_TCP_NO_SYNC_THREAD` env-değişkeni geçici bir teşhis aracı olarak eklendi — kalıcı bir özellik değil, ileride temizlenebilir ama şimdilik hızlı izole testler için faydalı, bırakıldı.
