# BİL 304 İşletim Sistemleri Dönem Projesi - Görev 2 (Araştırma ve Analiz Raporu)

Bu depo, Over-The-Air (OTA) Firmware Güncelleme projesinin 2. Görevi olan **"ELF ve Toolchain Analizi"** raporunu içermektedir. Proje kapsamında `new-firmware.z1` imaj dosyası, `msp430` araç zinciri (toolchain) kullanılarak analiz edilmiştir.

---

## 1. Binary Kimlik Analizi (ELF Sınıfı, Mimarisi ve Giriş Adresi)

Firmware dosyasının temel özelliklerini öğrenmek için `msp430-readelf -h` komutu kullanılmıştır:

<img width="1122" height="489" alt="image" src="https://github.com/user-attachments/assets/0b99b83d-54cf-4d2e-ba89-7980f202e209" />

**Analiz ve Yorumlar:**
* **ELF Sınıfı (Class):** Çıktıdaki `ELF32` ifadesi, bu dosyanın 32-bit nesne yapısında (Executable and Linkable Format) olduğunu gösterir. Yani bu dosya ham (raw) bir binary değil, işletim sistemi veya loader tarafından okunup anlamlandırılabilecek başlık ve hafıza haritası bilgilerine sahip yapısal bir dosyadır.
* **Mimari (Machine):** `Texas Instruments msp430 microcontroller` ifadesi, bu firmware'in TI Z1 Mote veya Sky Mote gibi düşük güç tüketimli 16-bit RISC işlemciler için özel olarak derlendiğini ispatlamaktadır.
* **Giriş Adresi (Entry Point):** `0x3100`. Cihaza enerji verildiğinde veya resetlendiğinde, cihazın çalışmaya (kodları okumaya) başlayacağı ilk fiziksel ROM hafıza adresidir.

---

## 2. Bellek Mimarisi ve Temel Bölümler (Sections) Analizi

Dosyanın içindeki verilerin hafızanın nerelerine (RAM/ROM) dağılacağını görmek için `msp430-readelf -S` komutu kullanılmıştır. Aşağıda temel bölümler (sections) izah edilmiştir:

* **`.text` Bölümü:** C dilinde yazdığımız fonksiyonların ve mantıksal kodların makine diline (0 ve 1'lere) çevrilmiş ve çalıştırılabilir halidir. Cihazın **Flash (ROM)** belleğine yazılır ve kalıcıdır.
* **`.rodata` (Read-Only Data):** Kodumuzdaki sadece okunabilir sabit değerlerin (örneğin `const` diziler, stringler, `firmware_data.h` içindeki devasa hex dizileri) tutulduğu bölümdür. Bu da **Flash (ROM)** belleğe yazılır.
* **`.data` Bölümü:** İlk değeri baştan atanmış (örneğin `int sayi = 5;`) global ve statik değişkenlerin tutulduğu yerdir. Cihaz kapalıyken ROM'da saklanır, ancak cihaz açıldığında işlemlerin hızlı yapılabilmesi için **RAM'e kopyalanır**.
* **`.bss` Bölümü:** Başlangıç değeri atanmamış veya 0 olarak bırakılmış global değişkenlerdir. `.bss` dosyada (diskte) yer kaplamaz, sadece cihaz açıldığında bu değişkenler için **RAM'de boş yer ayrılır (rezerve edilir).**
* **`.vectors` Bölümü:** Cihazda oluşacak donanım kesmelerinin (Interrupts - örneğin paket geldi, butona basıldı) cihazı hangi adrese yönlendireceğini tutan vektör tablosudur.

---

## 3. Kod ve Veri Boyutları (Memory Footprint)

Firmware'in Z1 cihazının hafıza sınırlarına (92 KB ROM, 8 KB RAM) uygun olup olmadığını denetlemek için `msp430-size` aracı kullanılmıştır:

```bash
$ msp430-size new-firmware.z1
   text    data     bss     dec     hex filename
  71715     336    5706   77757   12fbd new-firmware.z1
```

**Analiz ve Yorumlar:**
* **Flash (ROM) Tüketimi:** `.text` (71715 bayt) ve `.data` (336 bayt) bölümlerinin toplamı, cihazın ROM belleğine yazılacak net boyutu verir. Bu imaj ROM'da yaklaşık **70.3 KB** yer kaplamaktadır. Z1 Mote'un 92 KB'lık sınırı düşünüldüğünde firmware ROM'a sığmaktadır.
* **RAM Tüketimi:** Cihaz çalıştığında RAM üzerinde kilitlenecek alan `.data` (336 bayt) ve `.bss` (5706 bayt) toplamıdır. Bu da yaklaşık **5.9 KB** statik RAM kullanımı demektir. Z1 cihazının 8 KB RAM'i olduğu düşünülürse, geriye kalan yaklaşık 2 KB'lık alan çalışma anındaki fonksiyon yığınları (Stack) için kullanılacaktır. Ağ tamponlarının (packet buffer) büyüklüğü nedeniyle `.bss` boyutu oldukça yüksektir.

---

## 4. Sembol Tablosu Analizi

Sembol tablosu, firmware içerisindeki fonksiyon isimlerinin ve değişkenlerin RAM/ROM üzerindeki adres haritasıdır. Analiz için `msp430-nm` aracı kullanılmıştır:

```bash
$ msp430-nm -n new-firmware.z1 | head -n 10
         U button_hal_button_count
         U gpio_hal_arch_init
         U spi_arch_has_lock
00000000 A __IE1
00000000 T __far_bss_end
...
```

**Analiz ve Yorumlar:**
Çıktıdaki semboller, kodun işletim sistemi (Contiki-NG) donanım katmanıyla (Hardware Abstraction Layer - HAL) nasıl bağlandığını gösterir. Örneğin `gpio_hal_arch_init` sembolü, cihazın pin giriş/çıkış ayarlarının yapıldığı donanımsal başlatma fonksiyonudur.

---

## Sonuç: Bu Dosya Neden "Ham Binary" Değil de "ELF"?

Eğer bu dosya sadece "0 ve 1"lerden oluşan ham bir binary (örneğin `.bin`) dosyası olsaydı, içerisinde sadece `71715` baytlık saf makine kodu bulunurdu. 

Ancak bir dosyanın **ELF (Executable and Linkable Format)** olması, kodun fiziksel cihazda nereye yerleşeceğini (örneğin `0x3100` Entry Point adresi), değişkenlerin isimlerini, boyutlarını ve hangi işletim sistemi mimarisinde koşacağını belirten bir "harita (metadata)" barındırması anlamına gelir. İşletim sistemi veya bootloader, bu ELF başlıklarını okuyarak .text bölümünü ROM'a, .bss bölümünü ise RAM'e hatasız bir şekilde haritalayabilir. OTA işlemlerinde dosya doğrulama ve güvenli yükleme için ELF formatının sağladığı bu hiyerarşi kritik öneme sahiptir.
