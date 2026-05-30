# MSP430 `.z1` / `.sky` / `ARM M4F(CC1352R)` / `cooja-native` Platformları için Üretilmiş Firmware’ler Üzerinde Yapılabilecek Analiz Türleri Kontrol Listesi

---
##### (* ARM Mimarisinde derlenmiş firmware analizi yapmak isteyen gruplar MSP430 Toolchain yanında ARM-Toolchain araçlarını da indirip, kullanmalıdırlar.)

``` bash
  $ wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-rm/9-2020q2/gcc-arm-none-eabi-9-2020-q2-update-x86_64-linux.tar.bz2
  $ tar -xjf gcc-arm-none-eabi-9-2020-q2-update-x86_64-linux.tar.bz2
```
---
##### ** Analiz etmeniz için farklı platformlarda oluşturulmuş örnek firmware arşivi bil.omu drive linki için [tıklayınız](https://drive.google.com/file/d/1oLrZWPmDyuznWe5qS7zOsfSyyyPcQbBG/view?usp=sharing) .


---

# 1. Binary Kimlik Analizi

* Hedef platform analizi (`.z1` / `.sky` / `ARM M4F(CC1352R)` / `cooja-native`)
* MSP430 mimari tipi
* ELF format bilgisi
* Endianness nedir ve Endianness bilgisi
* Entry point adresi
* ABI nedir ve ABI bilgisi
* Compiler izi
* Toolchain versiyonu
* Optimization level tahmini
* Debug symbol var/yok analizi

Araçlar:

* `msp430-readelf`
* `msp430-objdump`
* `msp430-strings`
* `Ve üstteki araçların ARM versiyonları...`
---

# 2. Bellek Kullanım Analizi

* Flash, RAM, Stack, Heap anlamları
* Flash kullanım miktarı
* RAM kullanım miktarı
* `.text` boyutu
* `.data` boyutu
* `.bss` boyutu
* Stack kullanım tahmini
* Heap var/yok analizi
* Section dağılımı
* Memory map analizi
* Büyük veri yapılarının tespiti

Araçlar:

* `msp430-size`
* `msp430-readelf`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---

# 3. Symbol / Function Analizi

```
furkann@DESKTOP-CPNNO3A:~/contiki-ng/examples/rpl-udp$ msp430-nm -n new-firmware.z1
         U gpio_hal_arch_init
         U spi_arch_has_lock
00000000 A __IE1
0000001a A __P3DIR
...
00001100 D __data_start
0000110c D cc2420_process
00001148 d mac_pan_id
...
00001250 B __bss_start
000014e6 B node_id
...
00003100 A __stack
0000313e T main
0000353e T port1_isr
00004034 t process_thread_cc2420_process
00004436 T cc2420_init
000079a0 T rpl_process_dio
...
0000ffc0 T __vectors_start
00010000 T _vectors_end
```

* ### **Kullanılan Araç Zinciri ve Analizin Amacı**

Bu analiz aşamasında, `new-firmware.z1` imajının içsel davranış modelini, bellek organizasyonunu ve yazılım mimarisini çözümlemek amacıyla MSP430 araç zincirinde bulunan `msp430-nm` ve `msp430-readelf` araçları kullanılmıştır. `nm `aracı ile firmware içindeki sembol tablosu (fonksiyonlar ve değişkenler) adres sırasına göre çözümlenmiş; `readelf -S` aracı ile de ELF formatının bölüm başlıkları (Section Headers) incelenerek yazılımın bellek uzayındaki stratejik yerleşimi doğrulanmıştır.

* ### **Fonksiyon isimleri**

Binary imaj içerisindeki çalıştırılabilir fonksiyonlar Flash bellek (`.text` ve `.far.text` segmentleri) üzerinde yer almaktadır. Sistem, işletim sistemini donanım seviyesinde başlatan `platform_init_stage_one` fonksiyonu ile ayağa kalkmakta ve `netstack_init` gibi fonksiyonlarla ağ hiyerarşisini kurmaktadır. `0x313e` adresinde konumlanan ana giriş noktası (`main`), sistemin temel çalışma döngüsünü başlatmaktadır.

* ### **Global değişkenler**

Sistem genelinden erişilebilen ve durum yönetimi için kritik olan global değişkenler RAM bölgesinde barındırılmaktadır. Analiz sonucunda ağ kimliğini tutan `mac_pan_id` (`0x1148`), düğümün eşsiz adresini belirten `node_id` (`0x14e6`) ve log seviyelerini kontrol eden `curr_log_level_main` (`0x1160`) gibi değişkenler tespit edilmiştir. Bu değişkenlerin **D** (Data) segmentinde yer alması, cihaza enerji verildiğinde bu değerlerin varsayılan başlangıç değerleriyle yüklendiğini göstermektedir.

* ### **Static değişkenler**

Yalnızca tanımlandıkları dosya (scope) içerisinden erişilebilen statik değişkenler, kapsülleme (encapsulation) ve bellek güvenliği amacıyla kullanılmıştır. Ağ paketlerinin zaman damgasını tutan `last_packet_timestamp` (`0x126c`) ve komşu düğümlerin bellek havuzunu yöneten `neighbor_memb` (`0x1118`) yapıları bu kategoriye girmektedir.

* ### **ISR (interrupt) fonksiyonları**

Cihazın dış dünyaya ve asenkron donanım olaylarına gerçek zamanlı tepki verebilmesi için Kesme Yöneticileri (ISR) kullanılmıştır. Sembol tablosunda yer alan `port1_isr` (`0x353e`) GPIO kesmelerini, `watchdog_interrupt` (`0x37d8`) sistem kilitlenmelerini denetleyen zamanlayıcıyı, `cc2420_timerb1_interrupt` (`0x35fe`) ise radyo entegresinden gelen sinyalleri işlemektedir. Bu ISR'ler, `0xffc0` adresinden başlayan donanımsal `.vectors` tablosuna kayıtlıdır.

* ### **Contiki process entry’leri**

İşletim sisteminin olay güdümlü (event-driven) yapısını oluşturan "thread" yapıları `process_thread_` ön ekiyle tespit edilmiştir. `process_thread_tcpip_process, process_thread_sensors_process ve process_thread_hello_world_process` sembolleri; firmware'in sensör okuması, ağ trafiği yönetimi ve ana uygulama görevlerini eşzamanlı (cooperative multitasking) bir yapı içerisinde yürüttüğünü kanıtlamaktadır.

* ### **Radio driver fonksiyonları**

Cihazın fiziksel haberleşme (PHY/MAC) katmanını yöneten radyo sürücüleri incelendiğinde, cihazın Texas Instruments CC2420 (2.4 GHz 802.15.4) çipini kullandığı saptanmıştır. `cc2420_init, cc2420_transmit` ve iletişim gücünün enerji verimliliği için dinamik ayarlanmasına olanak tanıyan `cc2420_set_txpower` fonksiyonları sürücü katmanının temelini oluşturmaktadır.

* ### **Timer callback’leri**

Sistemin görev zamanlaması ve asenkron beklemeler için Contiki'nin zamanlayıcı (Timer) kütüphanelerine yoğun biçimde başvurduğu görülmüştür. `etimer_expired, ctimer_set` ve donanım seviyesinde çalışan `rtimer_run_next` fonksiyonları; periyodik sensör okumalarının ve ağ beacon'larının (sinyallerinin) zamanında iletilmesini sağlamaktadır.

* ### **Networking callback’leri**

Ağ yığını (Network Stack) analiz edildiğinde, cihazın IPv6 ve RPL tabanlı bir yönlendirme şeması koşturduğu netleşmiştir. `uip_icmp6_input` IPv6 ICMP mesajlarını işlerken; rpl_process_dio, `rpl_process_dao` ve `rpl_dag_root_start` fonksiyonları cihazın ağaç topolojisinde (DAG) yönlendirme metrikleri hesapladığını ve bir yönlendirici (router) profiline sahip olduğunu göstermektedir.

* ### **Sensor handler’ları**

Düğümün çevreyle etkileşimini sağlayan donanım sensörlerine ait okuma rutinleri tespit edilmiştir. `tmp102_read_temp_x100` sembolü ortam sıcaklık sensörünün, `accm_read_axis` sembolü ise ADXL345 dijital ivmeölçerinin entegre edildiğini ve sistem tarafından veri toplamak amacıyla aktif olarak kullanıldığını doğrulamaktadır.

* ### **Kullanılan kütüphaneler**

Cihaz belleğinde standart C kütüphanelerinin (örneğin; bellek kopyalama için `memcpy`, dizgi formatlama için `printf/snprintf`) yanı sıra Contiki'ye özgü çekirdek kütüphaneler yer almaktadır. Özellikle RAM bloklarının yönetimi için `memb_init` ve dinamik bağlı listeler için `list_add` kütüphanelerinin yoğun kullanımı, sistemin kısıtlı bellek (RAM) kaynaklarını dinamik bir şekilde yönettiğine işaret etmektedir.

* ### **Kullanılmayan (dead) fonksiyonlar**

Geliştirme araç zincirinde, kod bloğunda bulunmasına rağmen programın hiçbir yerinde çağırılmayan (dead code) fonksiyonların analizi için ELF Section başlıkları (`readelf -S`) ve Sembol Tablosu incelenmiştir. Çıktıda `.text.fonksiyon_adi` şeklinde izole edilmiş fonksiyon blokları yerine, boyutları oldukça büyük tekil `.text` (0x976e) ve `.far.text` (0x4a78) blokları gözlemlenmiştir. Bununla birlikte, dosyada `.debug_info`, `.debug_line` gibi hata ayıklama (debug) sembollerinin tam boyutuyla tutulduğu (toplamda ~0x4000 byte civarı) görülmektedir. ELF dosyasının yapısı değerlendirildiğinde, bağlayıcı (Linker) aşamasında kullanılmayan kodların sistemden tamamen atılması işleminin (Dead Code Elimination / `--gc-sections`) bu binary dosyası oluşturulurken izole bir iz bırakmadığı anlaşılmıştır. Dolayısıyla, son ELF dosyasında "atılmış" fonksiyonların listesi, derleyici tarafından sessizce temizlendiği için doğrudan görünmemektedir.


* ### **Function Address Mapping (Fonksiyon Adres Haritalaması)**

`readelf -S` aracı ile elde edilen ELF Section başlıkları ve `nm` çıktıları birleştirilerek, yazılımın hedef cihazdaki nihai bellek haritası (Memory Map) aşağıda özetlenmiştir. Bu adres haritalaması, cihazın çalışma zamanında (runtime) vereceği olası "Crash (Çökme)" loglarındaki hata adreslerini çözümlerken referans alınacak ana yapı niteliğindedir.


| Bellek Bölgesi | Başlangıç Adresi | Bitiş Adresi | İçerik Tipi | Örnek Semboller |
|---|---|---|---|---|
| Donanım Register | `0x0000` | `0x10FF` | Mutlak Adresler | `__P1IN, __ADC12CTL0` |
| RAM (Data & BSS) | `0x1100` | `0x2898` | Değişkenler | `mac_pan_id, node_id` |
| Flash (Text/Kod) | `0x3100` | `0x14A78` | Çalıştırılabilir Kod | `main, cc2420_init` |
| Kesme Vektörleri | `0xFFC0` | `0x10000` | ISR (Interrupts) | `port1_isr, watchdog_interrupt` |


(Not: Temel çalıştırılabilir kodlar `.text` bölümünde `0x3100` adresinden başlarken, genişletilmiş kod blokları `.far.text` bölümünde `0x10000` adresinde saklanmaktadır. Salt okunur sabitler ise `0xc870` adresine yerleştirilmiştir.)


---

# 4. String ve Metadata Analizi

* Debug mesajları
* printf logları
* IPv6 adresleri
* MAC adresleri
* Network node ID’leri
* Sensor isimleri
* Process isimleri
* Routing protokol isimleri
* TSCH/6LoWPAN/RPL stringleri
* Hidden diagnostic message’lar
* Hardcoded config değerleri
* Developer notları

Araçlar:

* `msp430-strings`
* `Ve üstteki aracın ARM versiyonu...`

---

# 5. Assembly / Instruction Analizi

* Instruction sequence analizi
* Function prologue/epilogue
* Register kullanımı
* Stack frame yapısı
* ISR akışı
* Loop yapıları
* Branch analizi
* Jump table analizi
* Function call graph
* Inline function tespiti
* Compiler optimization davranışı
* Delay loop analizi
* Busy-wait yapıları
* Context switching
* Protothread expansion
* Scheduler davranışı

Araçlar:

* `msp430-objdump`
* `msp430-as`
* `Ve üstteki araçların ARM versiyonları...`

---

# 6. Source-Level Mapping Analizi

(Debug build varsa)

* Address → source line eşleme
* Function → source file eşleme
* ISR → source mapping
* Crash address çözümleme
* Optimization sonrası source mapping
* Inline edilmiş kodların tespiti

Araçlar:

* `msp430-addr2line`
* `msp430-objdump -S`
* `Ve üstteki araçların ARM versiyonları...`

---

# 7. ELF Yapısı Analizi

* ELF header
* Section header
* Program header
* Symbol table
* Relocation entries
* Debug sections
* DWARF info
* Linker-generated metadata
* Startup section
* Vector table
* Initialization routines

Araçlar:

* `msp430-readelf`
* `msp430-elfedit`
* `Ve üstteki araçların ARM versiyonları...`

---

# 8. Interrupt ve Donanım Analizi

* Interrupt vector table
* GPIO access pattern
* Timer interrupt kullanımı
* UART ISR
* Radio interrupt handler
* ADC access
* Sensor polling
* Low-power mode geçişleri
* Clock configuration
* MSP430 register erişimleri

Araçlar:

* `msp430-objdump`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

---

# 9. Networking Analizi

* Unicast kullanım tespiti
* Broadcast kullanım tespiti
* Multicast tespiti
* IPv6 stack kullanımı
* RPL routing analizi
* TSCH scheduler çağrıları
* MAC layer interaction
* Packet buffer kullanımı
* Neighbor table erişimi
* Radio transmission akışı
* Retransmission logic
* ACK mekanizmaları
* CSMA/TSCH farkları
* Contiki network API kullanımı

Araçlar:

* `msp430-nm`
* `msp430-objdump`
* `msp430-strings`
* `Ve üstteki araçların ARM versiyonları...`

---

# 10. Wireless / TSCH Analizi

* TSCH slot operation
* Channel hopping logic
* ASN handling
* Radio timing loops
* Synchronization routines
* Schedule management
* Packet timing
* MAC timing critical path
* Drift compensation
* Low-power radio behavior

Araçlar:

* `msp430-objdump`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---

# 11. Sensor ve Peripheral Analizi

* Button handler
* LED driver
* UART usage
* SPI access
* I2C access
* ADC routines
* Sensor polling interval
* Interrupt-driven sensor logic
* GPIO toggle behavior
* Peripheral initialization sequence

Araçlar:

* `msp430-objdump`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---

# 12. Algoritma Koşma / DSP / Matematiksel Analiz

* Floating-point kullanımı
* Fixed-point kullanımı
* Trigonometric computation
* Multiply/divide routines
* Software floating-point emulation
* DSP benzeri loop’lar
* Matrix operation izleri
* Signal processing pattern’leri
* Computational hotspot’lar
* Numerical optimization

Araçlar:

* `msp430-objdump`
* `msp430-gprof`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---

# 13. Güç ve Performans Analizi

* Low-power mode geçişleri
* CPU-intensive function’lar
* Busy-wait detection
* Sleep/wakeup flow
* Timer usage intensity
* Radio duty cycle tahmini
* ISR yoğunluğu
* Function execution cost
* Flash/RAM efficiency
* Energy-heavy computation bölgeleri

Araçlar:

* `msp430-gprof`
* `msp430-objdump`
* `msp430-size`
* `Ve üstteki araçların ARM versiyonları...`

---

# 14. Coverage ve Profiling Analizi

* Function call frequency
* Execution hotspot
* Unused branch’ler
* Rarely executed path’ler
* Test coverage
* Critical execution path
* Runtime bottleneck’ler

Araçlar:

* `msp430-gcov`
* `msp430-gprof`
* `Ve üstteki araçların ARM versiyonları...`

---

# 15. Reverse Engineering Analizi

* Firmware behavior recovery
* Unknown firmware classification
* Feature inference
* Protocol inference
* ISR purpose discovery
* Hardware interaction recovery
* State machine extraction
* Scheduler reconstruction
* Event-flow reconstruction
* Network role inference

Araçlar:

* `msp430-objdump`
* `msp430-nm`
* `msp430-readelf`
* `msp430-strings`
* `Ve üstteki araçların ARM versiyonları...`

---

# 16. Compiler ve Optimization Analizi

* `-O0/-O2/-Os` farkları
* Inlining behavior
* Dead code elimination
* Constant folding
* Loop optimization
* Register allocation
* Tail-call optimization
* Branch optimization
* Macro expansion
* Preprocessor etkileri

Araçlar:

* `msp430-gcc`
* `msp430-cpp`
* `msp430-objdump`
* `Ve üstteki araçların ARM versiyonları...`

---

# 17. Linker ve Build Sistemi Analizi

* Section placement
* Link order
* Static library linkage
* Startup code
* Linker script behavior
* Vector placement
* Symbol resolution
* Relocation behavior

Araçlar:

* `msp430-ld`
* `msp430-ar`
* `msp430-ranlib`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

---

# 18. Binary Transformation Analizi

* ELF → HEX conversion
* ELF → binary conversion
* Section extraction
* Symbol stripping
* Debug removal
* Firmware minimization
* Binary patch preparation

Araçlar:

* `msp430-objcopy`
* `msp430-strip`
* `Ve üstteki araçların ARM versiyonları...`

---

# 19. Library ve Archive Analizi

* Static library içeriği
* Object file extraction
* Archive symbol table
* Linked module analizi

Araçlar:

* `msp430-ar`
* `msp430-gcc-ar`
* `msp430-ranlib`
* `Ve üstteki araçların ARM versiyonları...`

---

# 20. Contiki-NG Özel Analizler

* PROCESS_THREAD recovery
* Protothread expansion
* Event-driven scheduler analizi
* etimer/ctimer usage
* PROCESS_BEGIN/END expansion
* PROCESS_YIELD flow
* NETSTACK interaction
* Packetbuf lifecycle
* uIP callback chain
* Rime stack usage

Araçlar:

* `msp430-cpp`
* `msp430-objdump`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---

# 21. Güvenlik ve Robustness Analizi

* Hardcoded credential arama
* Debug backdoor izleri
* Buffer handling
* Unsafe memory access
* Stack-heavy routines
* Potential overflow bölgeleri
* Assert/debug remnants
* Information leakage string’leri

Araçlar:

* `msp430-strings`
* `msp430-objdump`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

---

# 22. Karşılaştırmalı Firmware Analizi

İki firmware arasında:

* Code size farkı
* RAM farkı
* Function count farkı
* ISR yoğunluğu
* Networking complexity
* Radio stack farkı
* Symbol farkı
* Optimization farkı
* Assembly complexity farkı



---

# 23. Eğitimsel Reverse Engineering Görevleri

* Bir firmware’in ne yaptığını bulma
* hangi protokolü kullandığını çıkarma
* button/LED mapping bulma
* ISR’leri tanıma
* network role çıkarımı
* Kullandığı algoritmik blok tespiti
* energy-heavy bölgeleri bulma
* stripped firmware çözümleme


---
