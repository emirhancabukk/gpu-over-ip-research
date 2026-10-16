# Faz 3 — TCP Transport İlk Deneme: Faz 1 Tahmini Doğrulandı (2026-08-27, otonom tur 7)

## Yapılan değişiklik

Hem client (`vendor/mesa/src/virtio/vulkan/vn_renderer_vtest.c`) hem host (`vendor/virglrenderer/vtest/vtest_server.c`) tarafına, mevcut unix-socket davranışını bozmadan, **"tcp:" önekiyle TCP soketi de destekleyen** geriye-uyumlu bir değişiklik eklendi:

- Client: `VTEST_SOCKET_NAME=tcp:<host>:<port>` → `vtest_connect_socket_tcp()` (AF_INET connect, TCP_NODELAY açık)
- Server: `--socket-path tcp:<port>` → `vtest_server_open_socket_tcp()` (AF_INET bind INADDR_ANY, listen)

İkisi de derlendi, localhost üzerinde (`tcp:127.0.0.1:15150` ↔ `--socket-path tcp:15150`) test edildi.

## Sonuç: KISMİ BAŞARI + Faz 1'in tahmini birebir doğrulandı

- Vulkan **context oluşturma TCP üzerinden de başarılı** (`vtest_lazy_init_context: ret=0` — Faz 2'deki aynı başarı, transport farketmiyor).
- Ama client çöktü (SIGABRT). `gdb -batch -ex run -ex bt` ile tam stack trace alındı:

```
#3  vtest_receive_fd (...) at vn_renderer_vtest.c:197   <-- abort() burada
#4  vtest_vcmd_resource_create_blob (...) at vn_renderer_vtest.c:404
#5  vtest_shmem_create (...) at vn_renderer_vtest.c:838
#7  vn_renderer_shmem_create (...)
#8  vn_ring_create (...) at vn_ring.c:303
#9  vn_instance_init_ring (...)
#10 vn_CreateInstance (...)
```

`vtest_receive_fd()` kaynak kodu (satır 172-201): `recvmsg()` ile **`SCM_RIGHTS`** ancillary data okuyor — bu, klasik Unix-domain-socket **file descriptor passing** mekanizması. `cmsg` geçerli/SCM_RIGHTS değilse `abort()` çağırıyor (satır 196-198), tam da bizim gördüğümüz çökme.

## Bu, Faz 1'de yazdığımız tahminin tam doğrulanması

`docs/research-notes/faz1-kaynak-kod-bulgular.md`'da şunu yazmıştık: *"vtest'in kaynak oluşturma tarafı (vtest_new_shm) memfd+fd-passing kullanıyor, bu network'e taşınmaz."* Şimdi gerçek bir çalıştırma ve tam stack trace ile **kanıtlanmış oldu**: `vn_ring_create` (Venus'un guest↔host komut halkasını kurmak için ilk yaptığı şey) hemen `vn_renderer_shmem_create` çağırıyor, o da `vtest_shmem_create` → `vtest_vcmd_resource_create_blob` → sunucudan bir shared-memory fd'sini **SCM_RIGHTS ile almaya çalışıyor**. TCP soketinde SCM_RIGHTS ancillary data taşınamıyor (bu bir OS/kernel kısıtı, Unix-domain socket'e özgü bir özellik), sunucu muhtemelen sessizce gönderemiyor veya hiç göndermiyor, client geçersiz/boş cmsg alıp abort ediyor.

## Anlamı: Faz 3'ün gerçek zorluğu netleşti

Basit "soket tipini değiştir" yaklaşımı **yetmiyor** çünkü Venus'un komut halkası (ring) kurulumunun kendisi bile shmem+fd-passing'e bağımlı — bu sadece büyük texture/buffer'lar için değil, protokolün en temelinde (`vn_instance_init_ring`, yani instance oluşturmanın hemen başında) kullanılıyor. Yani WAN transport'u çalıştırmak için ya:

1. **`vtest_shmem_create`'i TCP modunda farklı bir mekanizmaya yönlendirmek gerekiyor** — fd-passing yerine, sunucudan `malloc` edilmiş düz bellek kullanıp, ring buffer içeriğini periyodik olarak explicit `transfer_get/put` komutlarıyla senkronize eden bir "TCP shmem emülasyonu" yazmak (Faz 1'de bulduğumuz transfer_get/put'un byte-stream olduğu bilgisi burada işe yarayacak). Bu, ring'in artık "gerçek paylaşımlı bellek" değil, "periyodik senkronize edilen bir yerel kopya" olması anlamına gelir — performans etkisi olur ama WAN'da çalışır.
2. Ya da: TCP bağlantısı üzerinde ek bir yan-kanal (side-channel) kurup (örn. ilk bağlantıda bir Unix socket pair oluşturup sadece fd-passing gereken anlarda onu kullanmak) — ama bu, host ve client aynı makinede olmayı gerektirdiği için WAN senaryosunda anlamsız.

**Sonuç:** Yol 1 (shmem emülasyonu / ring'i transfer_get/put üzerinden senkronize etme) tek gerçekçi seçenek. Bu, başta düşündüğümüzden daha büyük bir mühendislik işi — Venus'un ring mekanizmasını (guest tarafında `vn_ring_create`/`vn_renderer_shmem_create`, host tarafında `vtest_shmem_create`'in karşılığı) TCP-uyumlu bir shmem emülasyon katmanıyla değiştirmek gerekiyor.

## Sonraki Tur İçin Somut Görev
1. `vtest_vcmd_resource_create_blob`'un TAM kodunu oku (`vn_renderer_vtest.c` satır ~380-410), sunucu tarafındaki karşılığını (`vtest_renderer.c`'de resource_create_blob handler) da oku — fd-passing'in tam olarak nerede/nasıl yapıldığını, blob resource'un ne için kullanıldığını (sadece ring için mi, yoksa texture/buffer'lar için de mi) netleştir.
2. Basit bir "TCP shmem emülasyonu" tasarımı taslağı çıkar: sunucu tarafında blob'u gerçek bellekte tut, `transfer_get/put` (ya da yeni bir özel komut) ile client'ın periyodik senkronize etmesini sağla. Bunun ring buffer semantiğini (head/tail pointer polling) nasıl etkileyeceğini düşün — bu, Faz 1'de bulduğumuz "ring gerçek paylaşımlı bellek + polling üzerine kurulu" bulgusuyla doğrudan bağlantılı, ikisi aynı köke iniyor.
