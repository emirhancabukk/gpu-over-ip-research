# Faz 2 — VTEST_CLIENT_ERROR_CONTEXT_FAILED Debug (2026-08-27, otonom tur 5)

## Elenen hipotezler

1. **Versiyon uyuşmazlığı değil.** virglrenderer ve Mesa, `venus-protocol` alt projesinin **aynı commit'ini** (`e94b12f301b9eb27ebead757128a18420b4f7994`) çekiyor. Bu ihtimal elendi.

2. **AMD/RADV desteksizliği değil (muhtemelen).** Web araştırması: virglrenderer resmen RADV 21.1+ ile Venus'u destekliyor (bizim RADV çok daha güncel, Mesa 26.x). "Deneysel" olarak işaretlenmiş olsa da desteklenmiyor değil. (Not: aramada "1.10'dan beri AMDGPU native context desteği var" gibi bir ifade çıktı ama bizim virglrenderer sürümü `1.3.0` — bu rakamlar tutarsız görünüyor, muhtemelen arama özeti güvenilir değil, dikkate almadım.)

3. **Handshake/protokol-version desync değil.** Sunucu kodu (`vtest_client_dispatch_commands`), ilk komut `VCMD_CREATE_RENDERER` değilse `VTEST_CLIENT_ERROR_CONTEXT_MISSING` döndürüyor — biz **`VTEST_CLIENT_ERROR_CONTEXT_FAILED`** aldık, yani sunucu ilk komutu doğru tanıdı (CREATE_RENDERER olarak), bu da `VN_DEBUG=vtest`'in tetiklediği ping/protocol-version negotiation'ın byte hizasını bozmadığını gösteriyor.

## Kesinleşen hata konumu

Hata, `vtest/vtest_renderer.c:744` içindeki `vtest_create_context()` fonksiyonunda oluşuyor — bu fonksiyon sadece 2 yerde -1 dönebiliyor:
- `length > 1024*1024` (isim çok uzun) — imkansız, isim kısacık bir string.
- `ctx->input->read(ctx->input, vtestname, length)` çağrısının döndürdüğü byte sayısı `length`'e eşit değilse.

Yani sunucu, istemcinin gönderdiği "renderer name" string'ini **tam olarak beklenen uzunlukta okuyamıyor.** Client tarafı (`vn_renderer_vtest.c:170-181`, `vtest_vcmd_create_renderer`) header'ı ve ismi **iki ayrı `write()` çağrısıyla** gönderiyor. Sunucunun okuma fonksiyonunun bunu doğru şekilde birleştirip birleştirmediği (yani soket üzerinde partial read'leri düzgün loop'layıp loop'lamadığı) doğrulanmadı.

## Sonraki Tur İçin Somut Görev
Runtime izleme gerekiyor (statik kod okuması yetersiz kaldı):
1. `strace -f -e trace=read,write -s 200` ile hem client hem server process'ini izleyip, CREATE_RENDERER komutu sırasında gerçekte kaç byte gönderilip kaç byte okunduğunu karşılaştır.
2. Alternatif: `vtest_create_context()`'e geçici bir `fprintf(stderr, ...)` debug satırı ekleyip (`ret` değerini ve `length`'i yazdır) yeniden derle, tam olarak read'in kaç döndürdüğünü gör.
3. Eğer gerçekten partial-read sorunu ise, bu virglrenderer'ın vtest okuma fonksiyonunda (muhtemelen `vtest_block_read`, henüz okunmadı) bir loop eksikliği/edge-case olabilir — bu durumda üst akış (upstream) bug tracker'ında bilinen bir issue olup olmadığına bakılmalı.
