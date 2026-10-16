# Faz 1 — vtest_renderer.c: transfer_get/put ve sync mekanizması (Otonom döngü, tur 2)

## Bulgu — vtest'in transfer/sync komutları GERÇEKTEN network-friendly, memfd/fd-passing kullanmıyor

`vtest/vtest_renderer.c` içinde `vtest_transfer_get_internal`/`vtest_transfer_put_internal` (satır ~1568-1671) ve `vtest_sync_wait` (satır ~2152) okundu:

- **`vtest_transfer_put`**: veriyi `ctx->input->read(ctx->input, data_iov.iov_base, ...)` ile okuyor — yani soket/fd üzerinden **düz byte akışı**. Sonra `virgl_renderer_transfer_write_iov()` ile GPU kaynağına yazıyor.
- **`vtest_transfer_get`**: GPU kaynağından `virgl_renderer_transfer_read_iov()` ile okuyup `vtest_block_write(ctx->out_fd, ...)` ile **düz byte akışı** olarak soket üzerinden geri gönderiyor.
- **`vtest_sync_wait`**: sync ID'leri, flag'ler, timeout değerleri hep `ctx->input->read()` ile **mesaj/komut verisi** olarak okunuyor — paylaşımlı bellek yok, fd-passing yok.

**Sonuç:** Önceki turda bulduğumuz "vtest'in kaynak oluşturma tarafı (`vtest_new_shm`) memfd+fd-passing kullanıyor, bu network'e taşınmaz" endişesi hâlâ geçerli AMA bu sadece **büyük/kalıcı kaynaklar** (texture, buffer) için. **transfer_get/put ve sync_wait komutlarının kendisi zaten tam bir byte-stream protokolü** — unix socket yerine TCP/QUIC soketine bağlamak, bu komutlar için ek bir çeviri katmanı gerektirmez, sadece alttaki soket implementasyonunu değiştirmek yeterli olur.

**Pratik çıkarım:** WAN prototipi için iki kaynak yönetim stratejisi mümkün:
1. Basit ama daha yavaş: tüm kaynak veri transferini `transfer_get/put` komutlarıyla yap (memfd/shared-memory hiç kullanma) — bu yavaş olabilir (her erişimde explicit kopyalama) ama **hemen TCP üzerinden çalışır**, ilk prototip için ideal.
2. Daha hızlı ama karmaşık: lokal makinede memfd/shared-memory hızlı yolunu koru, sadece host gerçekten uzaktaysa (network transport tespit edilirse) transfer_get/put yoluna düş (hybrid).

Faz 3 prototipi için **strateji 1** ile başlamak mantıklı — basit, çalışır, sonra optimize edilir.

## Bu Turda Karşılaşılan Engel: yerel build denemesi sudo gerektiriyor

`meson`/`ninja` ve build bağımlılıklarını (`libepoxy`, `libdrm`, `mesa`, `vulkan-headers` vb.) kurmaya çalıştım (`sudo pacman -S`), ama bu ortamda **interaktif olmayan (otonom) bir sudo şifre istemi başarısız oldu** ("a password is required", terminal yok). Bu, ana sisteme paket kurmanın kendisiyle ilgili bir güvenlik/mimari kısıtı değil (plan bunu "sıradan geliştirme paketi" olarak zaten izin veriyordu) — sadece otonom döngünün şifre girme yeteneği yok.

**Kullanıcıdan gereken:** Bu adımı ilerletmek için kullanıcının kendisinin şu komutu bir seferliğine çalıştırması gerekiyor (ya da pacman için passwordless sudo tanımlarsa döngü kendi kurabilir, ama bunu önermiyorum — güvenlik tercihi kullanıcıya ait):

```
sudo pacman -S --needed meson ninja pkgconf libepoxy libdrm mesa vulkan-headers vulkan-icd-loader wayland-protocols
```

Bu çalıştırılana kadar döngü, build denemesi dışındaki kaynak-kod-okuma temelli araştırmaya devam edecek.
