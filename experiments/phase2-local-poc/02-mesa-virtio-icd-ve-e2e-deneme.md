# Faz 2 — Mesa virtio Vulkan ICD Derlendi, Uçtan Uca Deneme (2026-08-27, otonom tur 4)

## Yapılanlar

1. Mesa kaynak kodu clone edildi (`vendor/mesa`, shallow clone, 546MB).
2. Meson ile sadece virtio Vulkan sürücüsü derlendi (OpenGL/EGL/Gallium kapalı, build süresini kısaltmak için):
   ```
   source vendor/.build-venv/bin/activate
   cd vendor/mesa
   meson setup build -Dvulkan-drivers=virtio -Dplatforms=x11,wayland -Dgallium-drivers= -Dglx=disabled
   meson compile -C build
   ```
   Eksik Python modülü (`packaging`) venv'e eklendi (`pip install packaging`, sudo gerekmedi).
   **273/273 hedef başarıyla derlendi.** Çıktı: `build/src/virtio/vulkan/libvulkan_virtio.so` (client-side Venus/vtest Vulkan ICD).

3. **Önemli bulgu:** Kaynak kodda `vn_renderer_vtest.c` diye ayrı bir dosya var — Mesa'nın venus sürücüsü, gerçek virtio-gpu kernel aygıtı yanında **vtest unix socket'ine de client olarak bağlanabiliyor**. Bunu tetiklemek için:
   - `VN_DEBUG=vtest` (vtest fallback'ini zorla)
   - `VTEST_SOCKET_NAME=/path/to.sock` (hangi socket'e bağlanacağı)
   - `VK_ICD_FILENAMES=.../virtio_devenv_icd.x86_64.json` (kurulum yapılmadan, build dizinindeki .so'yu gösteren "devenv" ICD dosyası — meson bunu otomatik üretiyor, `meson install` gerekmiyor)

## Uçtan uca deneme sonucu: KISMİ BAŞARI

- `virgl_test_server --venus` (host, önceki turda derlenmişti) lokal AMD GPU'ya karşı başlatıldı.
- `vkcube` ile denendi → `VK_KHR_surface` bulunamadı hatası (WSI/pencere sorunu, ayrı konu).
- `vulkaninfo --summary` ile (pencere gerektirmeyen, sadece instance/device enumeration) denendi:
  - İlk denemede ICD yolu yanlıştı (`/usr/local/lib/...` — kurulum yapılmadığı için dosya yoktu). **Çözüm:** meson'un otomatik ürettiği `virtio_devenv_icd.x86_64.json` kullanıldı, bu build dizinindeki gerçek .so yolunu gösteriyor.
  - Doğru ICD ile tekrar denendi: **client, sunucuya gerçekten bağlandı** (önceki "ICD not found" hatası kayboldu) ama sunucu tarafında **`VTEST_CLIENT_ERROR_CONTEXT_FAILED`** hatası alındı, `vulkaninfo` core dump etti.

## Anlamı

Bu, önemli bir eşik: **artık path/config sorunu değil, gerçek Venus protokol seviyesinde bir hata var.** ICD yüklendi, socket'e bağlandı, ama host tarafında Vulkan context oluşturma başarısız oluyor. Olası nedenler (doğrulanmadı, sonraki tur için hipotez):
1. AMD/RADV üzerinde Venus host desteği, Intel(ANV)/NVIDIA kadar olgun olmayabilir — virglrenderer/Venus üst akış geliştirmesi büyük ölçüde Intel donanımıyla test ediliyor.
2. `virgl_test_server` çağrısına eksik bir flag/context tipi belirtilmemiş olabilir.
3. Protokol versiyon uyuşmazlığı (Mesa'nın çektiği venus-protocol sürümü ile virglrenderer'ın çektiği farklı olabilir — ikisi ayrı ayrı clone edildi, aynı commit olmayabilir).

## Sonraki Tur İçin Somut Görev
1. `virgl_test_server`'ı `-v` (verbose) gibi bir bayrakla veya `VN_DEBUG=vtest,init,result` gibi daha ayrıntılı debug env değişkenleriyle çalıştırıp context hatasının tam sebebini bulmaya çalış.
2. Mesa ve virglrenderer'ın çektiği `venus-protocol` alt proje commit'lerinin aynı olup olmadığını kontrol et (`subprojects/venus-protocol-1.1` her iki repoda da var, versiyon farkı context negotiation'ı bozabilir).
3. AMD/RADV yerine, eğer sistemde başka bir GPU/sürücü yolu varsa (yok, tek GPU AMD) — bu yüzden bu debug AWS/gerçek NVIDIA GPU'da tekrarlanmalı, RADV'a özgü bir sorun olup olmadığını ayırt etmek için.
