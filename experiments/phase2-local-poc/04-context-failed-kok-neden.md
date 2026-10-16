# Faz 2 — CONTEXT_FAILED Kök Nedeni Bulundu (2026-08-27, otonom tur 6)

## Önceki turun hipotezi YANLIŞTI, ama debug ile hızlıca düzeltildi

Önceki tur "renderer name string'i partial-read ediliyor olabilir" diye tahmin etmişti. `vtest_create_context()`'e geçici debug satırı eklenip test edildiğinde: `expected length=11 read_ret=11` — **tam eşleşiyor, bu fonksiyon başarıyla dönüyor.** Hipotez elendi.

## Gerçek kök neden: `proxy_context_create()` NULL dönüyor → ENOMEM

`vtest_lazy_init_context()`'e ikinci bir debug satırı eklenip tekrar test edildi:
```
[DEBUG] vtest_lazy_init_context: capset_id=4 ret=12 errno=... (gürültü, dikkate alınmadı)
```
`capset_id=4` = `VIRTGPU_DRM_CAPSET_VENUS`. `ret=12` = **ENOMEM** (Linux'ta errno 12). `src/virglrenderer.c:246-268` (`virgl_renderer_context_create_with_flags`) içine bakıldığında:

```c
case VIRTGPU_DRM_CAPSET_VENUS:
   if (!state.proxy_initialized)
      return EINVAL;
   ctx = proxy_context_create(ctx_id, ctx_flags, nlen, name);
   break;
...
if (!ctx)
   return ENOMEM;   // <-- buraya düşüyoruz
```

Yani **`proxy_context_create()` NULL dönüyor.** Meson build çıktısında gördüğümüz `render server mode: process` / `render server worker: process` ayarlarına bakılırsa, virglrenderer'ın venus desteği güvenlik amaçlı olarak gerçek Vulkan işini **ayrı bir alt process'e (render server)** yaptırıyor — "proxy" ismi de bunu doğruluyor. `proxy_context_create()`'in NULL dönmesi muhtemelen bu **alt process'in başlatılamadığını/fork+exec'in başarısız olduğunu** gösteriyor (sandboxlama, izin, ya da bu ortamdaki bir kısıtlama yüzünden olabilir).

## Yapılan geçici değişiklikler (proje-içi, sisteme etkisi yok)
`vendor/virglrenderer/vtest/vtest_renderer.c` içine iki geçici `fprintf(stderr, ...)` debug satırı eklendi (git diff ile görülebilir, upstream'e gönderilecek bir şey değil, sadece bizim debug amaçlı). İstenirse `git checkout` ile geri alınabilir, ama sonraki tur muhtemelen buna ihtiyaç duyacak, şimdilik bırakıldı.

## Sonraki Tur İçin Somut Görev
`proxy_context_create()` fonksiyonunu bul (muhtemelen `src/proxy/proxy_context.c` veya benzeri, meson çıktısında `proxy_proxy_context.c.o` gördük) ve NULL dönme koşullarını oku — muhtemelen bir `fork()`/`exec()` veya `minijail` çağrısının başarısız olduğu bir noktada. Eğer render server subprocess spawn'ı bu ortamda (örn. bir sandbox/container kısıtlaması yüzünden) çalışmıyorsa, `--render-server-worker` build seçeneğini `process` yerine `thread` yaparak yeniden derlemeyi dene (meson_options.txt'de bu seçenek zaten görülmüştü) — bu, alt process yerine aynı process içinde thread kullanır, subprocess spawn kısıtlamasını atlayabilir.
