# Faz 0 — Bulut Sağlayıcı Önceliği Değişti: Azure (2026-08-27)

## Karar
AWS'de GPU instance quota'sı (`Running On-Demand G and VT instances`) hâlâ onay bekliyor ve bloke ediyordu. Kullanıcının Azure'da $200 ücretsiz kredisi olduğu ortaya çıktı — gerçek para değil, quota beklemeden hemen kullanılabilir. Birincil bulut hedefi **Azure**'a çevrildi. AWS paralel olarak arka planda bekletiliyor (quota onaylanırsa karşılaştırma/doğrulama için kullanılacak, $30 tavanla).

## Instance eşleniği
AWS g5.xlarge (1x NVIDIA A10G, tam GPU) ≈ Azure **NV36ads_A10_v5** (1x NVIDIA A10, tam GPU — Azure bu seride GPU'yu partition'lıyor: NV6ads/NV12ads/NV18ads kısmi GPU dilimleri, NV36ads tam GPU'yu veriyor). Not: NV36ads çok daha fazla vCPU/RAM (36 vCPU) ile geliyor, bizim mimaride (uzak taraf sadece host render process'i çalıştırıyor) bu gereksiz ama ücretsiz kredi olduğu için sorun değil.

## Bölge
Azure'da NVadsA10 v5 serisi Germany West Central, West Europe (Hollanda), Poland Central dahil çok sayıda AB bölgesinde mevcut. **Germany West Central** öneriliyor — AWS Frankfurt ölçümümüzle coğrafi olarak aynı/çok yakın, karşılaştırma yapılabilir olacak.

## Maliyet tavanı
Otonom döngü, Azure tarafında toplam **$150**'ye kadar kullanıcıdan onay almadan harcayabilir (kullanıcı onayladı, ücretsiz kredi olduğu için AWS'deki $30 tavanından daha rahat). Instance her deney sonunda durdurulacak, boşta bırakılmayacak.

## Sonraki adım
Kullanıcı Azure Portal'dan VM'i kendisi açacak (AWS'de yaptığımız gibi), bana public IP + SSH erişimi (private key dosya yolu) verecek.
