# Faz 0 - Network Ölçüm Notları (2026-08-27)

## Yöntem
ICMP (ping/traceroute) bu ortamda engelli/çalışmadı (`ping` 5/5 paket kaybı, `traceroute` kurulu değil).
Onun yerine `curl` ile TCP connect + TLS handshake süresi ölçüldü (AWS EC2 API endpoint'lerine HTTPS istekleri).
`time_connect` kabaca 1 RTT'ye (SYN → SYN/ACK) denk gelir.

## Sonuçlar (AWS EU bölgeleri, TCP connect time)

| Bölge | Endpoint | TCP connect | TLS tamam | Toplam |
|---|---|---|---|---|
| eu-central-1 (Frankfurt) | ec2.eu-central-1.amazonaws.com | 64ms | 238ms | 427ms |
| eu-west-1 (İrlanda) | ec2.eu-west-1.amazonaws.com | 127ms | 317ms | 513ms |
| eu-west-2 (Londra) | ec2.eu-west-2.amazonaws.com | 151ms | 324ms | 511ms |
| eu-south-1 (Milano) | ec2.eu-south-1.amazonaws.com | 128ms | 304ms | 491ms |

**En düşük gecikme: Frankfurt (eu-central-1), ~64ms TCP connect.**

## Önemli Uyarı / Doğrulanması Gereken Nokta

Bu ölçüm Claude Code'un Bash aracı üzerinden yapıldı — bu, gerçekte kullanıcının oyun/masaüstü makinesinin gerçek ISP yolundan mı geçiyor yoksa farklı bir network path'i mi kullanıyor **kesin değil**. 64ms rakamı, iyi bir fiber peering için beklenenden (İstanbul-Frankfurt genelde 25-35ms bandında olur) yüksek — bu ya (a) gerçek ISP routing kalitesi, (b) ilk bağlantı (TCP+TLS) overhead'i saf RTT'den fazlasını yansıtıyor, ya da (c) bu araç farklı bir ağ yolundan ölçüyor olabilir.

**Yapılması gereken:** Aynı ölçümü kullanıcının gerçek oyun bilgisayarından, ideal olarak ham ICMP ping ile (`ping -c 20 <frankfurt-ip>`) veya `mtr` ile tekrarlamak, ve mümkünse gerçek bir AWS g5 instance'ı ayağa kaldırıp ona doğrudan ping atmak (EC2 API endpoint'i değil, instance'ın kendi IP'si — API endpoint routing'i instance routing'inden farklı olabilir).

## Sonraki Adım
AWS hesabı + CLI kurulumu yok (`aws` komutu bulunamadı). Gerçek bir GPU instance açıp ona ping atmadan önce:
1. AWS hesabı hazır mı, kredi kartı/bütçe onayı var mı netleştir (g5.xlarge ~$1/saat, kapatılmazsa saatlik ücretlendirilir).
2. AWS CLI kur + credentials yapılandır.
3. Frankfurt'ta bir g5.xlarge aç, açık kaldığı süre boyunca hem ICMP hem TCP ölçümü yap, sonra hemen kapat.
