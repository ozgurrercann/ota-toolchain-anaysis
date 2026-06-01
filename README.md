# 📑 OTA Toolchain Analysis (Araştırma İş Parçacığı Raporu)

## 1. Introduction
Bu analiz, Nesnelerin İnterneti (IoT) ve kablosuz algılayıcı ağlarda (WSN) kullanılan havadan bellenim güncelleme (OTA) mimarilerinin araç zincirlerini (toolchain), derleme adımlarını ve güvenlik doğrulama süreçlerini incelemek amacıyla hazırlanmıştır.

## 2. Compiler and Linker Analysis
* **Kullanılan Derleyici (Compiler):** Contiki-NG altyapısında, Texas Instruments CC1352R gibi ARM Cortex-M4F mimarisine sahip donanımlar için derleme aşamasında `arm-none-eabi-gcc` araç zinciri (cross-compiler) kullanılır. Native emülasyon ve yerel platform testlerinde ise doğrudan ana işletim sisteminin yerel `gcc` derleyicisinden yararlanılır.
* **Derleme Süreci (Compilation):** Geliştirilen kaynak kodlar (`test-arq.c`, `udp-client.c` vb.), derleyici vasıtasıyla mimariye özgü makine diline çevrilir ve nesne dosyaları (`.o` / `.obj`) üretilir.
* **Bağlayıcı (Linker) Görevi:** `Linker`, derleme sonucu oluşan bağımsız nesne dosyalarını, projenin bağımlı olduğu sistem kütüphanelerini (`os/sys/log.c`, `lib/crc16.c` vb.) ve donanımın bellek sınırlarını belirten `linker script` dosyasını bir araya getirir. Bellek yönetimindeki `.text` (kod), `.rodata` (sabitler), `.data` (başlatılmış değişkenler) ve `.bss` (başlatılmamış değişkenler) kesitlerinin RAM ve Flash üzerindeki fiziksel adres yerleşimlerini hatasız bir şekilde organize ederek nihai çalıştırılabilir dosyayı (`.native` veya `.bin/.hex`) üretir.

## 3. Firmware Image Formats
OTA güncelleme süreçlerinde, hedef sensör düğümüne iletilecek bellenim imajları işlevlerine göre farklı formatlarda yapılandırılabilir:
* **`.bin` (Raw Binary):** Sadece hedef işlemci tarafından doğrudan yürütülecek saf makine kodlarını (opcode) içerir. Herhangi bir adresleme veya başlık bilgisi barındırmadığı için dosya boyutu oldukça küçüktür ve havadan transfer (OTA) için en ideal formattır.
* **`.hex` (Intel Hex):** Saf makine kodlarının yanı sıra, bu kodların hafızada tam olarak hangi fiziksel adres alanlarına yazılması gerektiğini belirten ASCII formatında satır etiketleri ve adres bilgileri barındırır.
* **`.elf` (Executable and Linkable Format):** İçerisinde sembolik tablolar, hata ayıklama (debugging) verileri ve sistem mimarisine ait detaylı meta veriler taşır. Dosya boyutu çok büyük olduğundan doğrudan kablosuz düğümlere yüklenmez; ancak simülasyon ortamlarında ve geliştirme aşamasındaki analizlerde kritik rol oynar.

## 4. Verification and Security (Hash Algoritmaları)
Kablosuz sensör ağlarında havadan taşınan bellenim paketlerinin elektromanyetik parazitler veya hat gürültüleri nedeniyle bozulma riski (Veri Bütünlüğü - Integrity) oldukça yüksektir. Bozuk bir kodun cihaza yazılması sistemin tamamen kilitlenmesine (bricking) yol açar.
* **Kullanılan Yöntem ve Seçim Nedeni:** Bu projede veri bütünlüğünü garanti altına almak amacıyla, Contiki-NG kütüphanesinde gömülü ve optimize olarak yer alan **CRC16 (Cyclic Redundancy Check)** algoritması tercih edilmiştir. Kaynak kısıtlı IoT cihazlarında SHA-256 gibi kriptografik hash algoritmaları yüksek işlemci döngüsü ve RAM tükettiği için, hata tespiti odağında CRC16 matematiksel olarak çok daha hafif (lightweight) ve verimlidir.
* **Çalışma Mantığı:** Dağıtımı yapan UDP Sunucusu, göndereceği her 64 Byte'lık payload bloğu için polinom tabanlı bir CRC16 hata kontrol kodu hesaplar ve paketin sonuna ekler. İstemci (Client) düğüm paketi aldığında, gelen veri üzerinde aynı CRC16 algoritmasını yeniden koşturur. Eğer istemcinin hesapladığı değer ile paketten çıkan değer birebir eşleşiyorsa bütünlük doğrulanır (`CRC16 Check: SUCCESS`) ve paket onaylanarak (ACK) Coffee CFS ile hafızaya yazılır. Eşleşme başarısız ise paket sessizce düşürülerek ARQ mekanizması üzerinden yeniden iletimi (Retransmission) beklenir.

## 5. Comparison Table
Farklı bellenim güncelleme yaklaşımlarının teknik parametrelere göre karşılaştırma tablosu aşağıdadır:

| Parametre | Full Image Update (Tam İmaj Güncelleme) | Differential Update (Delta / Fark Güncelleme) |
| :--- | :--- | :--- |
| **Bant Genişliği Tüketimi** | Yüksek (Tüm firmware dosyası baştan sona havadan taşınır) | Çok Düşük (Sadece eski kod ile yeni kod arasındaki fark/yama iletilir) |
| **İşlemci / RAM Yükü** | Düşük (Gelen paket blokları doğrudan kalıcı hafızaya/Flash ardışık işlenir) | Yüksek (Eski imajla gelen fark paketini birleştirmek için cihazda ağır bir algoritma döner) |
| **Güvenilirlik Seviyesi** | Çok Yüksek (Sıfırdan, temiz ve bağımsız bir kurulum gerçekleştirilir) | Orta (Eski bellenim sektörlerinde bir bozulma varsa delta yaması sistemi çökertebilir) |
| **Enerji Tüketimi** | Telsiz donanımı (RF Core) uzun süre açık kaldığından pil tüketimi yüksektir | Paket boyutu küçük ve telsiz süresi kısa olduğundan pil dostudur |
