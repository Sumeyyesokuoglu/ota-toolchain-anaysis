# BİL 304 İşletim Sistemleri Dönem Projesi - Görev 2 (Araştırma ve Analiz Raporu)

Bu depo, Over-The-Air (OTA) Firmware Güncelleme projesinin 2. Görevi olan **"ELF ve Toolchain Analizi"** raporunu içermektedir. Proje kapsamında `new-firmware.z1` imaj dosyası, `msp430` araç zinciri kullanılarak detaylıca analiz edilmiştir.

---

## 1. Binary Kimlik Analizi

<img width="1150" height="488" alt="image" src="https://github.com/user-attachments/assets/97f7b2f4-5a33-468a-9059-d29e7e11adf4" />


Yukarıdaki görselde `new-firmware.z1` bellenim (firmware) dosyası üzerinde `msp430-readelf -h` komutu çalıştırılarak dosyanın temel kimlik ve mimari bilgileri elde edilmiştir. Çıktıdaki parametrelerin analizi şu şekildedir:

*   **Hedef Platform Analizi:** Dosyanın uzantısının `.z1` olması, bu firmware imajının Zolertia Z1 sensör düğümleri (motes) için üretildiğini doğrulamaktadır.
*   **MSP430 Mimari Tipi:** Çıktıdaki `Machine: Texas Instruments msp430 microcontroller` bilgisi, kodun 16-bit RISC tabanlı, ultra düşük güç tüketimli MSP430 çekirdeği için derlendiğini ispatlamaktadır.
*   **ELF Format Bilgisi:** Dosyanın yapısı `Class: ELF32` olarak görünmektedir. Bu, dosyanın rastgele 0 ve 1'lerden oluşan ham bir binary (raw binary) olmadığını, içinde işletim sisteminin okuyabileceği başlıklar ve haritalar bulunduran yapısal bir formatta (Executable and Linkable Format) olduğunu gösterir.
*   **Endianness Bilgisi:** `Data: 2's complement, little endian` ifadesi, verilerin hafızada "Little Endian" formatında tutulduğunu gösterir. Yani çok baytlı verilerin en anlamsız baytı (LSB), hafızadaki en düşük adrese yerleştirilecektir.
*   **Entry Point Adresi:** `0x3100`. Bu çok kritik bir adrestir. Sensöre enerji verildiğinde veya resetlendiğinde, cihazın çalışmaya (kod okumaya) başlayacağı ilk fiziksel hafıza adresidir.
*   **ABI (Application Binary Interface) Bilgisi:** `OS/ABI: Standalone App` ifadesi, bu firmware'in çalışmak için altında devasa bir işletim sistemine (Windows/Linux gibi) ihtiyaç duymadığını, cihaz donanımı üzerinde doğrudan ve bağımsız olarak koşan gömülü bir yazılım olduğunu belirtir.
*   **Debug Sembolleri Analizi:** `Number of section headers: 21` ifadesi, dosyada çok sayıda bölüm (section) olduğunu gösterir. Bu kadar fazla bölüm olması, derleme sırasında dosyanın içinden hata ayıklama (debug) sembollerinin silinmediğini (not stripped) kanıtlar.

---
