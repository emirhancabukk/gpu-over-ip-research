# Faz 1/2 — virglrenderer Kaynak Kodu İncelemesi (Otonom döngü, tur 1)

## Yapılanlar
- `~/Desktop/gpu-over-ip-research/vendor/virglrenderer/` içine gerçek repo clone edildi (gitlab.freedesktop.org HTTP tarayıcı Anubis anti-bot koruması yüzünden WebFetch ile okunamıyordu, ama `git clone` protokol seviyesinde çalıştı — bunu not düşüyorum, ileride tekrar HTTP üzerinden bir şey çekmemiz gerekirse `git clone`/`curl` deneyelim, browser-fetch değil).
- Lokal makine KVM/sanallaştırma desteğini kontrol ettim: **AMD-V var, `/dev/kvm` mevcut, `kvm_amd` modülü zaten yüklü.** Yani VM tabanlı bir yaklaşım (Faz 2/3 için) bu makinede teknik olarak mümkün. `qemu` henüz kurulu değil.
- `src/venus/vkr_transport.c`, `src/venus/vkr_ring.c`, `vtest/vtest_shm.c`, `vtest/vtest.h` dosyalarını okudum.

## KRİTİK BULGU 1 — Venus'un ring buffer'ı gerçek paylaşımlı bellek üzerine kurulu, network'e taşınamaz (as-is)

`vkr_ring.c` içindeki mekanizma şöyle çalışıyor:
- Guest ve host, **aynı fiziksel bellek bölgesini** (virtio-gpu blob resource, `VIRGL_RESOURCE_FD_SHM` tipinde) paylaşıyor.
- Host tarafında ayrı bir thread (`vkr_ring_thread`) sürekli bu paylaşımlı bellekteki `head`/`tail` pointer'larını okuyup (`atomic_load_explicit`) yeni komut var mı diye **polling** yapıyor (busy-wait → üstel geri çekilmeli sleep, `vkr_ring_relax`).
- Bu, guest'in host'a komut "göndermesi" değil, guest'in belleğe yazması ve host'un aynı belleği izlemesi — **fiziksel olarak aynı makinede olmayı gerektiriyor.**

Sonuç: Venus'un ring transport katmanını doğrudan "bir soket üzerinden taşı" diyerek WAN'a taşıyamayız — bu, planın ilk taslağındaki varsayımdan daha zor bir problem. Bu paylaşımlı bellek/polling modelini, gerçek bir mesaj-geçişi (host'a "yeni komut var" bildirimi + veriyi soket üzerinden gönderme) modeline dönüştürmemiz gerekiyor.

## KRİTİK BULGU 2 — vtest'in kaynak paylaşımı da memfd + fd-passing kullanıyor, bu da local-only

`vtest_shm.c`: `vtest_new_shm()` → `memfd_create()` ile bellek-destekli bir dosya tanımlayıcısı (fd) oluşturuyor. Bu fd'nin client↔server arasında paylaşılması, Unix domain socket üzerinden **SCM_RIGHTS** (fd-passing) ile yapılıyor olmalı (kod bu dosyada değil ama memfd_create + Unix socket kombinasyonu klasik SCM_RIGHTS pattern'i). **Bu da network üzerinden (TCP/QUIC) çalışmaz** — dosya tanımlayıcıları OS sınırları içinde geçerli, bir network soketi üzerinden "fd gönderemezsin", sadece byte gönderebilirsin.

Yani: sadece unix socket'i TCP ile değiştirmek yetmiyor — kaynak veri transferinin (texture/buffer içeriği) fd-passing yerine gerçek byte-stream serialization'a çevrilmesi gerekiyor.

## UMUT VERİCİ BULGU — vtest protokolünde zaten mesaj-tabanlı sync ve transfer komutları var

`vtest.h`'de şu fonksiyonlar tanımlı:
- `vtest_transfer_get` / `vtest_transfer_put` (ve `_get2`/`_put2` varyantları) — bunlar, paylaşımlı bellek yerine **açık veri kopyalama komutları**, isimlerinden anlaşıldığı kadarıyla veriyi protokol mesajı olarak taşıyorlar. Bu, WAN transport'umuz için doğal entegrasyon noktası olabilir — ring/shm yerine bu transfer komutlarını temel alan bir yol.
- `vtest_sync_create/unref/read/write/wait`, `vtest_drm_sync_*` (timeline signal/wait dahil) — senkronizasyon burada da mesaj/komut seviyesinde tanımlı, ham paylaşımlı bellek yerine.

**Sonraki adım önerisi:** `vtest_renderer.c` (81KB, henüz okunmadı) içinde `vtest_transfer_get/put` ve `vtest_sync_*` implementasyonlarını okuyup, bunların gerçekten network-friendly (tam byte-stream, fd-passing yok) olup olmadığını doğrulamak. Eğer öyleyse, Faz 3'ün hedefi netleşiyor: Venus'un SHM ring'ini değil, **vtest'in transfer/sync komut protokolünü** WAN transport'un temeli yapmak.

## Plan Güncellemesi (öneri, henüz plana işlenmedi)
Faz 2/3 stratejisi güncellenebilir: crosvm/QEMU tam VM kurmak yerine, önce **vtest_server** (host) + Venus/Mesa Vulkan client (guest tarafı, VM olmadan da libvenus ICD ile çalışabiliyor) ikilisini önce local unix socket ile ayağa kaldırıp doğrulamak, sonra unix socket yerine TCP + explicit resource serialization ile değiştirmek — bu, tam bir VM'den çok daha az hareketli parça içeriyor. Bu değişikliği plana yazmadan önce vtest_renderer.c'yi okuyup mekanizmanın gerçekten uygun olduğunu teyit edeceğim.

## Sonraki Tur İçin Somut Görev
1. `vtest_renderer.c` içinde transfer_get/put ve sync fonksiyonlarının implementasyonunu oku.
2. Eğer network-friendly ise, plan dosyasındaki Faz 2/3 açıklamasını güncelle (crosvm/QEMU yerine vtest tabanlı yaklaşım).
3. `meson`/`ninja` kurup (sıradan geliştirme paketi, ana sisteme kalıcı riskli etkisi yok) virglrenderer'ı `-Dvenus=true` ile derlemeyi dene — lokal AMD APU'nun Vulkan/RADV sürücüsü zaten var, host tarafını yerelde (AWS'siz, ücretsiz) test edebiliriz.
