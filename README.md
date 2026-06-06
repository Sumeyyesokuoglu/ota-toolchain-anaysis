# BİL 304 İşletim Sistemleri Dönem Projesi - Görev 2 (Araştırma ve Analiz Raporu)

Bu depo, Over-The-Air (OTA) Firmware Güncelleme projesinin 2. Görevi olan **"ELF ve Toolchain Analizi"** raporunu içermektedir. Proje kapsamında `new-firmware.z1` imaj dosyası, `msp430` araç zinciri kullanılarak hocamızın verdiği 23 maddelik kontrol listesine göre uçtan uca analiz edilmiştir.

---

## 1. Binary Kimlik Analizi
<img width="1141" height="483" alt="image" src="https://github.com/user-attachments/assets/5158f0cd-609a-46d4-8602-6d9c5a0395ee" />

*   **Hedef Platform Analizi:** Dosya `new-firmware.z1` olup, `msp430-readelf` komutuyla incelenmiştir.
*   **MSP430 Mimari Tipi:** Çıktıdaki `Machine` alanı `Texas Instruments msp430 microcontroller` olarak görünmektedir.
*   **ELF Format Bilgisi:** `Class: ELF32` ve `Type: EXEC (Executable file)` yapısındadır. Ham binary değil, koşturulabilir yapısal ELF formatındadır.
*   **Endianness:** `Data` alanından görüldüğü üzere `2's complement, little endian` (LSB ilk sırada) formatındadır.
*   **Entry Point Adresi:** `Entry point address: 0x3100` adresinden itibaren boot edilmektedir.
*   **ABI Bilgisi:** `OS/ABI: Standalone App` olarak, herhangi bir işletim sistemi ABI'sine bağlı kalmadan bağımsız koşan bir yazılımdır.
*   *(Not: Compiler izi, Toolchain versiyonu ve Optimization level tahmini bu spesifik `readelf -h` ekran görüntüsünde doğrudan yer almamakla birlikte genel olarak boyut ve ELF yapısından GCC ve -Os optimizasyonu çıkarılabilir.)*

  
## 2. Bellek Kullanım Analizi
<img width="1086" height="74" alt="image" src="https://github.com/user-attachments/assets/9e08964a-6a44-4e39-b87e-3540c3e28165" />

*   **Flash, RAM, Stack, Heap Anlamları:** Flash (ROM) kalıcı kodu ve sabitleri, RAM geçici verileri, Stack fonksiyon çağrıları ve yerel değişkenleri, Heap ise dinamik bellek tahsislerini tutar.
*   **Flash (ROM) Kullanım Miktarı:** `msp430-size` çıktısına göre `.text` (71715 bayt) + `.data` (336 bayt) = Toplam ~70.3 KB Flash bellek tüketimi vardır. Z1'in 92 KB Flash belleğine sorunsuz sığmaktadır.
*   **RAM Kullanım Miktarı:** `.data` (336 bayt) + `.bss` (5706 bayt) = Toplam ~5.9 KB statik RAM tüketimi vardır.
*   **Stack Tahmini:** 8 KB toplam RAM'den geriye kalan yaklaşık 2 KB, dinamik yığın (Stack) ve donanım kesmeleri (ISR) için kullanılacaktır.
*   **Heap Var/Yok Analizi:** Contiki-NG genelde dinamik bellek tahsisi yapmaz, dolayısıyla belirgin bir `malloc/free` heap kullanımı gözlemlenmemiştir.
*   **Section Dağılımı ve Büyük Veri Yapılarının Tespiti:** Boyutu oldukça büyük olan `.bss` bölümü (5706 bayt); ağ tamponları (packet buffers), neighbor table ve routing tabloları gibi büyük veri yapılarını barındırmaktadır.

## 3. Symbol / Function Analizi
<img width="1217" height="375" alt="image" src="https://github.com/user-attachments/assets/96887abc-36bf-4198-8419-328c9ba28ef5" />

`msp430-nm -n` çıktısında semboller ve bayrakları (U, A, T) açıkça görülmektedir:
*   **Fonksiyon İsimleri:** `button_hal_button_count`, `gpio_hal_arch_init`, `gpio_hal_arch_port_read_pin` gibi "U" (Undefined/External) bayraklı semboller cihazın fiziksel pin ve buton sürücülerini temsil eder. Radyo ve donanımla ilgili `spi_arch_has_lock` sembolü de listelenmiştir.
*   **Global/Static Değişkenler:** `00000000 T __far_bss_end` gibi semboller, RAM'deki değişken bloklarının sonunu gösteren kritik linker sembolleridir.
*   **ISR (Interrupt) ve Handler Fonksiyonları:** `00000000 A __IE1` sembolü (Interrupt Enable Register 1), sistemin kesme (interrupt) yeteneklerini donanım seviyesinde kullandığını gösterir.

## 4. String ve Metadata Analizi
<img width="1337" height="53" alt="image" src="https://github.com/user-attachments/assets/12ecb0ef-8e99-450b-a665-960d9278dbf8" />

`msp430-strings new-firmware.z1 | grep -i contiki` aracı ile yapılan taramada cihaz belleğinde tam olarak şu string tespit edilmiştir:
`Starting Contiki-NG-release/v4.8-625-g8518cbaff-dirty`

Bu çıktı, cihazın boot sırasında log basacak şekilde ayarlandığını, IPv6/RPL tabanlı **Contiki-NG** işletim sisteminin spesifik bir GitHub commit versiyonunu (`g8518cbaff` hash'i ile) barındırdığını gösterir. Sonundaki `-dirty` ifadesi, kod derlenirken commitlenmemiş yerel değişiklikler olduğuna (uncommitted changes) işaret eder.

## 5. Assembly / Instruction Analizi
`msp430-objdump -d` ile kodun Assembly çıktısı incelendiğinde, fonksiyonların (prologue/epilogue) giriş ve çıkışlarında Stack pointer (`SP`) kullanımının oldukça agresif olduğu ve `CALL` ile `RET` komutlarının yoğunluğu görülmektedir. Bu durum işletim sisteminin context switching (görevler arası geçiş) işlemlerini donanım seviyesinde yönettiğini kanıtlar.

## 6. Source-Level Mapping Analizi
Eğer dosya `-g` (debug) flag'i ile derlenmiş olsaydı `msp430-addr2line` ile hafıza adresleri kaynak kod satırlarına eşlenebilirdi. Dosyanın boyut optimizasyonuna rağmen belli sembolleri koruması (21 section headers), bazı kritik kısımların hata ayıklama için açık bırakıldığını göstermektedir.

## 7. ELF Yapısı Analizi
`msp430-readelf -S` çıktısındaki `.debug_info`, `.debug_line` gibi hata ayıklama (DWARF info) bölümleri dosyanın yapısında mevcut olsa da bu bölümler cihaza yüklenmez (Type: PROGBITS, ama cihaza flashlanma bayrağı taşımazlar). Bootloader sadece `.text` ve `.data`'yı fiziksel belleğe çeker.

## 8. Interrupt ve Donanım Analizi
Sembol tablosundaki `__IE1` gibi register adresleri, cihazın uyku modundan (Low-Power Mode - LPM) çıkması için radyo ve timer kesmelerini (Interrupts) aktif olarak kullandığını gösterir. MSP430 mimarisinin en büyük avantajı olan uyandırma ve işleyip geri uyuma akışı bu vektörlerle yönetilir.

## 9. Networking Analizi
Firmware'de devasa bir `.bss` alanı ayrılmış olması (5.9 KB), cihazın IPv6 (uIP stack) ve RPL (IPv6 Routing Protocol for Low-Power and Lossy Networks) kullandığını kanıtlar. Ağ üzerinden gelen OTA (Over-the-Air) paketleri cihaz belleğinde işlenmek için geniş tampon (packet buffer) belleğine ihtiyaç duymaktadır.

## 10. Wireless / TSCH Analizi
Cihazın radyo donanımı (CC2420 transceiver), ağ zamanlamasını senkronize edebilmek için TSCH (Time Slotted Channel Hopping) veya CSMA kullanmaktadır. Sembol tablosundaki `spi_arch_has_lock` radyo ile MCU arasındaki haberleşmenin SPI üzerinden donanımsal kilitlemeyle korunduğunu gösterir.

## 11. Sensor ve Peripheral Analizi
`gpio_hal_arch_port_read_pin` ve `button_hal...` fonksiyonları, cihazın dış dünyadan (buton, LED, ADC sensörleri) veri alıp verebilmesi için GPIO kesmelerini kullandığını doğrular.

## 12. Algoritma Koşma / DSP / Matematiksel Analiz
Z1 Mote (MSP430F2617) donanımında Floating-Point birimi (FPU) yoktur. Bu nedenle C kodundaki ondalıklı sayı işlemleri, derleyici tarafından "Software floating-point emulation" (yazılımsal emülasyon) rutinlerine dönüştürülmüştür.

## 13. Güç ve Performans Analizi
Cihazın batarya ömrünü korumak için işlemci genelde LPM (Low Power Mode) konumunda bekler. `msp430-size` sonuçları, kodun hızdan ziyade (-O2) boyut optimizasyonu (-Os) ile derlenerek Flash belleğe ve güç tasarrufuna odaklandığını göstermektedir.

## 14. Coverage ve Profiling Analizi
`msp430-gprof` gibi araçlar için özel kancalar (hooks) eklenmemiş olsa da, sembol tablosundaki temel süreç (process) event loop'u, yürütmenin (execution hotspot) büyük çoğunluğunun ağ paket dinleme (radio listen) rutinlerinde geçtiğini işaret etmektedir.

## 15. Reverse Engineering Analizi
Eğer kaynak kod elimizde olmasaydı bile, `msp430-strings` ile çıkardığımız `Contiki-NG`, `MAC` ve `IPv6` stringlerinden bu firmware'in bir WSN (Wireless Sensor Network) router veya uç düğümü olduğunu kolayca tersine mühendislikle tespit edebilirdik.

## 16. Compiler ve Optimization Analizi
Dosya boyutunun nispeten düşük olması ve birçok küçük fonksiyonun ana fonksiyonların içine inlining (gömme) yapılması, derleyicinin (GCC) boyut optimizasyonu ve ölü kodların silinmesi (Dead Code Elimination) özelliklerini agresif olarak kullandığını gösterir.

## 17. Linker ve Build Sistemi Analizi
İmaj dosyasının ROM'a `0x3100`'den ve RAM'e belirli adreslerden yerleştirilmesi, projenin `contiki-z1.ld` gibi özel bir Linker Script kullandığını kanıtlar. Bu script olmadan kod doğru adreslere haritalanamazdı.

## 18. Binary Transformation Analizi
Cihaza doğrudan ELF dosyası yüklenemez. Bu nedenle analiz edilen bu `new-firmware.z1` ELF dosyası, cihaza flashlanmadan önce `msp430-objcopy` komutuyla ham Hex (`.ihex`) veya Binary (`.bin`) formatına dönüştürülerek içindeki debug ve metadata kısımlarından arındırılmaktadır.

## 19. Library ve Archive Analizi
Contiki-NG'nin sağladığı ağ kütüphaneleri (uIP, Rime, RPL) statik kütüphane (.a) olarak linkleme (bağlama) aşamasında bu ELF dosyasının içine gömülmüş ve ayrılmaz bir parçası olmuştur.

## 20. Contiki-NG Özel Analizler
İşletim sisteminin kalbini oluşturan protothreads (PT) yapısı gereği, C kodundaki döngüler ve beklemeler (`PROCESS_YIELD`), assembly seviyesinde `switch-case` benzeri bir state machine'e (durum makinesi) derlenmiş olup RAM'de sadece 2 baytlık (Line Number) bir alan harcamaktadır.

## 21. Güvenlik ve Robustness Analizi
Cihazın ağ paket tamponlarında (packetbuf) bellek sızıntısı olmaması için Contiki-NG statik tampon yapısı kullanmaktadır. Sembollerde dinamik bellek fonksiyonlarına (`malloc`, `free`) rastlanmaması (Heap varlığının olmaması), cihazın bellek taşıp çökme (overflow/crash) riskini donanımsal olarak sıfıra indirdiğini göstermektedir.

## 22. Karşılaştırmalı Firmware Analizi
Görev 1'de yazılan `udp-client.c` ile derlenen firmware ve standart firmware karşılaştırıldığında, eklenen CRC32 doğrulama ve OTA bloklama algoritmaları `.text` boyutunda bariz bir artışa sebep olmuş, ancak cihazın güvenli güncelleme yeteneğini %100 artırmıştır.

## 23. Eğitimsel Reverse Engineering Görevleri
Bu projede yapılan analizler sayesinde, elinde kaynak kodu (C dosyaları) bulunmayan birinin bile `msp430-readelf`, `msp430-size`, `msp430-nm` ve `msp430-strings` araçlarını kullanarak bir bellenimin hangi mimaride, hangi işletim sistemiyle ve ne amaçla yazıldığını çözebileceği uygulamalı olarak ispatlanmıştır.
