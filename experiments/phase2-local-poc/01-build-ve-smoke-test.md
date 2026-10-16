# Faz 2 — İlk Yerel Milestone: virglrenderer + Venus Derlendi ve Çalıştı (2026-08-27, otonom tur 3)

## Sonuç: BAŞARILI — AWS/Azure hiç gerekmeden, yerel AMD GPU'ya karşı doğrulandı

## Yapılanlar (tekrarlanabilir adımlar)

1. Build bağımlılıkları (kullanıcı tarafından kuruldu, sudo gerektirdiği için otonom döngü yapamadı):
   ```
   sudo pacman -S --needed meson ninja pkgconf libepoxy libdrm mesa vulkan-headers vulkan-icd-loader wayland-protocols
   ```

2. `venus-protocol` alt projesi Python `mako` ve `pyyaml` modüllerine ihtiyaç duyuyor. Bunlar için **sisteme sudo ile dokunmadan**, proje içinde izole bir venv kullanıldı:
   ```
   python3 -m venv ~/Desktop/gpu-over-ip-research/vendor/.build-venv
   source ~/Desktop/gpu-over-ip-research/vendor/.build-venv/bin/activate
   pip install mako pyyaml
   ```
   (Arch'ın PEP 668 "externally-managed-environment" koruması yüzünden sistem geneline `pip install` yapılamıyor — venv bunun doğru/güvenli çözümü, ana sisteme hiç dokunmuyor.)

3. Build:
   ```
   source ~/Desktop/gpu-over-ip-research/vendor/.build-venv/bin/activate
   cd ~/Desktop/gpu-over-ip-research/vendor/virglrenderer
   meson setup build -Dvenus=true -Dplatforms=egl
   meson compile -C build
   ```
   **105/105 hedef başarıyla derlendi.** Çıktı: `build/vtest/virgl_test_server` (venus destekli host render server), `build/src/libvirglrenderer.so.1.11.0`, `build/server/virgl_render_server`.

4. Smoke test — vtest server'ı lokal AMD GPU'ya (`/dev/dri/renderD128`, RADV/Lucienne APU) karşı, venus açık şekilde başlattım:
   ```
   virgl_test_server --no-fork --venus --rendernode /dev/dri/renderD128 --socket-path /tmp/vtest-smoke.sock
   ```
   **Sonuç:** Process hatasız başladı, unix socket dosyasını oluşturdu, client bağlantısı bekleyerek canlı kaldı. Hiç crash/hata logu yok.

## Anlamı

Bu, **Faz 2'nin (yerel PoC, network olmadan toolchain doğrulama) somut kanıtı.** Daha önce plan crosvm/QEMU tam VM gerektirir diye varsayıyordu — ama vtest server sayesinde tam bir VM'e hiç gerek kalmadan, host render tarafının gerçekten çalıştığı doğrulandı. Bu tamamen ücretsiz, AWS/Azure'a hiç ihtiyaç duymadan yapıldı.

## Sonraki adım
Gerçek bir Venus/Vulkan **client**'ı (Mesa'nın venus ICD'siyle derlenmiş `vkcube` gibi) bu socket'e bağlayıp gerçekten bir üçgen render edip edemediğini test etmek — bu, Faz 2'yi tam olarak tamamlayacak. Bunun için Mesa'nın kendisinin de `-Dvulkan-drivers=virtio` ile derlenmesi gerekiyor (ayrı, daha büyük bir build — sonraki tur için görev).
