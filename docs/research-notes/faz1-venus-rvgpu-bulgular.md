# Faz 1 - Prior Art İnceleme Bulguları (2026-08-27)

## Mesa Venus Protokolü

**Mimari katmanlar:**
- Guest tarafı: Venus Vulkan driver (uygulamaya Vulkan ICD olarak görünür)
- Transport: guest kernelindeki `virtio-gpu` sürücüsü — host ile konuşmak için 5 özel parametre şart: `VIRTGPU_PARAM_3D_FEATURES`, `VIRTGPU_PARAM_CAPSET_QUERY_FIX`, `VIRTGPU_PARAM_RESOURCE_BLOB`, `VIRTGPU_PARAM_HOST_VISIBLE`, `VIRTGPU_PARAM_CONTEXT_INIT`
- Host tarafı: **virglrenderer** — Vulkan 1.1+ ve platform bazlı external memory extension'ları gerektiriyor

**ÖNEMLİ BULGU — komutlar per-draw-call değil, command-buffer seviyesinde toplu gönderiliyor:**
Venus, `vkEndCommandBuffer` çağrılana kadar tüm komutları biriktirip **tek seferde** host'a gönderiyor. Yani round-trip granularitesi zaten "her draw call için ayrı network turu" değil, "her command buffer için bir network turu" seviyesinde — bizim planımızdaki en büyük risk (binlerce draw call = binlerce round trip) Venus tarafından zaten büyük ölçüde çözülmüş durumda. Bu, WAN transport'a geçişi umduğumuzdan daha uygun hale getiriyor.

**Caching:** `vkGetPhysicalDeviceFeatures2()` gibi sorgular, ilk `vkEnumeratePhysicalDevices()`/`vkEnumeratePhysicalDeviceGroups()` çağrısında zaten fetch edilip cache'leniyor — tekrar host'a gitmiyor. Genel prensip: statik/değişmeyen bilgi lokal cache'de tutulup gereksiz round trip'ler eleniyor.

**Eksik/belirsiz kalan (dokümantasyonda yok, kaynak koddan bakılmalı):**
- Fence/semaphore/timeline senkronizasyonunun tam mekaniği açıklanmamış — sadece gelecekte `virgl_renderer_export_fence` desteğinden bahsediliyor. Bu, **Faz 2/3'te asıl derinlemesine incelenmesi gereken kısım** çünkü senkron bekleme noktaları WAN'da en çok acıyan yer.
- Performans/gecikme rakamı hiç paylaşılmamış.

**Risk notu:** Venus, host-visible `VkDeviceMemory` için spec'i ihlal eden implementasyona-özgü varsayımlar yapıyor (`vkMapMemory` davranışı). Transport'u değiştirirken bu varsayımların bozulmaması gerekecek — mimariyi değiştirirken dikkat edilecek nokta olarak not edildi.

## RVGPU (Panasonic Automotive) — DÜZELTME

Plandaki ilk varsayımın aksine, RVGPU bizim istediğimizin **tam tersi** bir mimari: `rvgpu-proxy` (client) GPU'nun olduğu makinede **lokal render** yapıyor, `rvgpu-renderer` (server) ayrı makinede sadece **görüntüyü alıp gösteriyor** (TCP üzerinden, örn. `127.0.0.1:55667`). Yani bu "uzak GPU, lokal render sonucu göster" değil, klasik "lokal render, uzaktan izle" (VNC/streaming'e daha yakın).

**Kullanım değeri:** Mimari şablon olarak bize uymuyor, ama TCP transport implementasyonu (librvgpu) kod seviyesinde network taşıma katmanı yazımı için referans olabilir — o kısmı ayrı değerlendireceğiz, genel mimari model olarak saymıyoruz artık.

## Sonuç / Faz 2'ye Girdi

Venus + virglrenderer, ilk düşündüğümüzden daha uygun bir temel çünkü command-buffer seviyesinde zaten batching yapıyor. Asıl bilinmeyen: **fence/semaphore senkronizasyonunun WAN gecikmesi altında nasıl davranacağı** — bu, dokümantasyonda yok, virglrenderer kaynak kodundan (`src/venus/` dizini) doğrudan okunması gerekiyor. Faz 2'nin (yerel PoC) bir parçası olarak bunu kaynak kod seviyesinde inceleyeceğiz.
