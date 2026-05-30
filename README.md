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

### **Kullanılan Araç Zinciri ve Analizin Amacı**

Bu analiz aşamasında, `new-firmware.z1` imajının içsel davranış modelini, bellek organizasyonunu ve yazılım mimarisini çözümlemek amacıyla MSP430 araç zincirinde bulunan `msp430-nm` ve `msp430-readelf` araçları kullanılmıştır. `nm `aracı ile firmware içindeki sembol tablosu (fonksiyonlar ve değişkenler) adres sırasına göre çözümlenmiş; `readelf -S` aracı ile de ELF formatının bölüm başlıkları (Section Headers) incelenerek yazılımın bellek uzayındaki stratejik yerleşimi doğrulanmıştır.

### **Fonksiyon isimleri**

Binary imaj içerisindeki çalıştırılabilir fonksiyonlar Flash bellek (`.text` ve `.far.text` segmentleri) üzerinde yer almaktadır. Sistem, işletim sistemini donanım seviyesinde başlatan `platform_init_stage_one` fonksiyonu ile ayağa kalkmakta ve `netstack_init` gibi fonksiyonlarla ağ hiyerarşisini kurmaktadır. `0x313e` adresinde konumlanan ana giriş noktası (`main`), sistemin temel çalışma döngüsünü başlatmaktadır.

### **Global değişkenler**

Sistem genelinden erişilebilen ve durum yönetimi için kritik olan global değişkenler RAM bölgesinde barındırılmaktadır. Analiz sonucunda ağ kimliğini tutan `mac_pan_id` (`0x1148`), düğümün eşsiz adresini belirten `node_id` (`0x14e6`) ve log seviyelerini kontrol eden `curr_log_level_main` (`0x1160`) gibi değişkenler tespit edilmiştir. Bu değişkenlerin **D** (Data) segmentinde yer alması, cihaza enerji verildiğinde bu değerlerin varsayılan başlangıç değerleriyle yüklendiğini göstermektedir.

### **Static değişkenler**

Yalnızca tanımlandıkları dosya (scope) içerisinden erişilebilen statik değişkenler, kapsülleme (encapsulation) ve bellek güvenliği amacıyla kullanılmıştır. Ağ paketlerinin zaman damgasını tutan `last_packet_timestamp` (`0x126c`) ve komşu düğümlerin bellek havuzunu yöneten `neighbor_memb` (`0x1118`) yapıları bu kategoriye girmektedir.

### **ISR (interrupt) fonksiyonları**

Cihazın dış dünyaya ve asenkron donanım olaylarına gerçek zamanlı tepki verebilmesi için Kesme Yöneticileri (ISR) kullanılmıştır. Sembol tablosunda yer alan `port1_isr` (`0x353e`) GPIO kesmelerini, `watchdog_interrupt` (`0x37d8`) sistem kilitlenmelerini denetleyen zamanlayıcıyı, `cc2420_timerb1_interrupt` (`0x35fe`) ise radyo entegresinden gelen sinyalleri işlemektedir. Bu ISR'ler, `0xffc0` adresinden başlayan donanımsal `.vectors` tablosuna kayıtlıdır.

### **Contiki process entry’leri**

İşletim sisteminin olay güdümlü (event-driven) yapısını oluşturan "thread" yapıları `process_thread_` ön ekiyle tespit edilmiştir. `process_thread_tcpip_process, process_thread_sensors_process ve process_thread_hello_world_process` sembolleri; firmware'in sensör okuması, ağ trafiği yönetimi ve ana uygulama görevlerini eşzamanlı (cooperative multitasking) bir yapı içerisinde yürüttüğünü kanıtlamaktadır.

### **Radio driver fonksiyonları**

Cihazın fiziksel haberleşme (PHY/MAC) katmanını yöneten radyo sürücüleri incelendiğinde, cihazın Texas Instruments CC2420 (2.4 GHz 802.15.4) çipini kullandığı saptanmıştır. `cc2420_init, cc2420_transmit` ve iletişim gücünün enerji verimliliği için dinamik ayarlanmasına olanak tanıyan `cc2420_set_txpower` fonksiyonları sürücü katmanının temelini oluşturmaktadır.

### **Timer callback’leri**

Sistemin görev zamanlaması ve asenkron beklemeler için Contiki'nin zamanlayıcı (Timer) kütüphanelerine yoğun biçimde başvurduğu görülmüştür. `etimer_expired, ctimer_set` ve donanım seviyesinde çalışan `rtimer_run_next` fonksiyonları; periyodik sensör okumalarının ve ağ beacon'larının (sinyallerinin) zamanında iletilmesini sağlamaktadır.

### **Networking callback’leri**

Ağ yığını (Network Stack) analiz edildiğinde, cihazın IPv6 ve RPL tabanlı bir yönlendirme şeması koşturduğu netleşmiştir. `uip_icmp6_input` IPv6 ICMP mesajlarını işlerken; rpl_process_dio, `rpl_process_dao` ve `rpl_dag_root_start` fonksiyonları cihazın ağaç topolojisinde (DAG) yönlendirme metrikleri hesapladığını ve bir yönlendirici (router) profiline sahip olduğunu göstermektedir.

### **Sensor handler’ları**

Düğümün çevreyle etkileşimini sağlayan donanım sensörlerine ait okuma rutinleri tespit edilmiştir. `tmp102_read_temp_x100` sembolü ortam sıcaklık sensörünün, `accm_read_axis` sembolü ise ADXL345 dijital ivmeölçerinin entegre edildiğini ve sistem tarafından veri toplamak amacıyla aktif olarak kullanıldığını doğrulamaktadır.

### **Kullanılan kütüphaneler**

Cihaz belleğinde standart C kütüphanelerinin (örneğin; bellek kopyalama için `memcpy`, dizgi formatlama için `printf/snprintf`) yanı sıra Contiki'ye özgü çekirdek kütüphaneler yer almaktadır. Özellikle RAM bloklarının yönetimi için `memb_init` ve dinamik bağlı listeler için `list_add` kütüphanelerinin yoğun kullanımı, sistemin kısıtlı bellek (RAM) kaynaklarını dinamik bir şekilde yönettiğine işaret etmektedir.

### **Kullanılmayan (dead) fonksiyonlar**

Geliştirme araç zincirinde, kod bloğunda bulunmasına rağmen programın hiçbir yerinde çağırılmayan (dead code) fonksiyonların analizi için ELF Section başlıkları (`readelf -S`) ve Sembol Tablosu incelenmiştir. Çıktıda `.text.fonksiyon_adi` şeklinde izole edilmiş fonksiyon blokları yerine, boyutları oldukça büyük tekil `.text` (0x976e) ve `.far.text` (0x4a78) blokları gözlemlenmiştir. Bununla birlikte, dosyada `.debug_info`, `.debug_line` gibi hata ayıklama (debug) sembollerinin tam boyutuyla tutulduğu (toplamda ~0x4000 byte civarı) görülmektedir. ELF dosyasının yapısı değerlendirildiğinde, bağlayıcı (Linker) aşamasında kullanılmayan kodların sistemden tamamen atılması işleminin (Dead Code Elimination / `--gc-sections`) bu binary dosyası oluşturulurken izole bir iz bırakmadığı anlaşılmıştır. Dolayısıyla, son ELF dosyasında "atılmış" fonksiyonların listesi, derleyici tarafından sessizce temizlendiği için doğrudan görünmemektedir.


### **Function Address Mapping (Fonksiyon Adres Haritalaması)**

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

```
furkann@DESKTOP-CPNNO3A:~/contiki-ng/examples/rpl-udp$ msp430-strings new-firmware.z1
...
ADXL345 sensor
Accelerometer process
CC2420 driver
...
Starting Contiki-NG-release/v4.8-625-g8518cbaff-dirty
- 802.15.4 PANID: 0x%04x
- 802.15.4 Default channel: %u
Node ID: %u
Tentative link-local IPv6 address:
...
created a new RPL DAG
initialized DAG with instance ID %u, DAG ID
...
GCC: (GNU) 4.7.2 20120920 (mspgcc dev 20120911)
/home/user/tmp/gcc-4.7.2-msp430/msp430/mcpu-430x/mmpy-16/msr20/mc20/libgcc
...
```

### **Kullanılan Araç Zinciri ve Analizin Amacı**
Bu bölümde, derlenmiş firmware imajı (`new-firmware.z1`) içerisinde insan tarafından okunabilir (ASCII) metinleri, log formatlarını ve gömülü yapılandırma değerlerini ortaya çıkarmak amacıyla `msp430-strings` aracı kullanılmıştır. Bu araç, binary dosya içindeki ardışık yazdırılabilir karakterleri tarayarak yazılımın çalışma zamanında (runtime) konsola basacağı debug mesajlarını, ağ protokollerinin bıraktığı izleri ve donanım kimliklerini açığa çıkarmaktadır.

### **Debug Mesajları ve `printf` Logları**
Sistem içerisindeki metinler incelendiğinde, cihazın çalışma durumunu bildiren çok sayıda `INFO` (Bilgi) ve `WARN` (Uyarı) seviyesinde log tespit edilmiştir.

  * **Başlangıç İzleri:** Sistemin ayağa kalktığını gösteren `"Starting Contiki-NG-release..."` ve `"Hello, EK-D103"` gibi boot (başlatma) mesajları bulunmaktadır.

  * **Durum ve Hata Bildirimleri:** Radyo ve kuyruk yönetimine ait `"scheduling transmission in %u ticks", "packet sent to , seqno %u", "Neighbor queue full" ve "failed to create packet"` gibi metinler, sistemin MAC ve kuyruk (queuebuf) katmanında detaylı hata ayıklama (debug) verisi ürettiğini kanıtlamaktadır.

### **Ağ Parametreleri ve Protokol İzleri (IPv6, RPL, 6LoWPAN)**
Firmware'in ağ yetenekleri, kodun içine gömülmüş uyarı ve durum mesajları sayesinde kesin olarak doğrulanmıştır:

- **Fiziksel / MAC Katmanı:** `"- 802.15.4 PANID: 0x%04x"` ve `"- 802.15.4 Default channel: %u"` ifadeleri, IEEE 802.15.4 kablosuz standardının kullanıldığını göstermektedir.
- **RPL (Yönlendirme):** Cihazın RPL protokolündeki davranışlarını anlatan `"created a new RPL DAG", "initiating global repair", "DAG expired, poison and leave" ve "received a %s-DIO from"` gibi stringler yoğun bir şekilde yer almaktadır. Bu durum cihazın ağaç topolojisinde aktif bir yönlendirme aktörü (router) olduğunu doğrular.
- **IPv6 ve 6LoWPAN:** `"Tentative link-local IPv6 address:", "Sending ICMPv6 ERROR message" ve "reassembly: failed to store new fragment session"` mesajları, cihazın IPv6 yığınına sahip olduğunu ve büyük paketleri 6LoWPAN adaptasyon katmanında parçalayıp (fragmentation) birleştirdiğini kanıtlamaktadır.

### **İşletim Sistemi (Process) ve Sensör İsimleri**
Bir önceki sembol analizinde adresleri tespit edilen Contiki-NG işlemlerinin (processes) ve donanım bileşenlerinin, bellek içerisine gömülmüş string karşılıkları başarıyla çıkarılmıştır:
* **İşletim Sistemi Görevleri:** `"Hello world process", "TCP/IP process", "Ctimer process", "Event timer" ve "Accelerometer process".`
* **Donanım ve Sürücüler:** `"ADXL345 sensor"` (İvmeölçer), `"TMP102 sensor"` (Sıcaklık), `"Button"` (Fiziksel Tuş) ve haberleşme çipi olan `"CC2420 driver"` isimleri düz metin olarak kodda bulunmaktadır.

### **Hardcoded (Gömülü) Config Değerleri ve Geliştirici İzleri**
Geliştirme ortamına ve derleyiciye (Compiler) ait son derece kritik geliştirici izleri (metadata) saptanmıştır:

* **İşletim Sistemi Versiyonu:** Cihaza yüklenen Contiki-NG versiyonu `"Contiki-NG-release/v4.8-625-g8518cbaff-dirty"` olarak tespit edilmiştir. Buradaki *dirty* bayrağı (flag), derleme (compile) işlemi yapılırken Git deposunda henüz commit edilmemiş (kaydedilmemiş) değişiklikler olduğunu gösteren önemli bir detaydır.
* **Derleyici ve Dizin İzleri:** Kodu derleyen araç zincirinin versiyonu `"GCC: (GNU) 4.7.2 20120920 (mspgcc dev 20120911)"` ve asbmler versiyonu `"GNU AS 2.22"` olarak kodun içine kazınmıştır.
* **Geliştirici Bilgisayarı Yolları:** Derleme işleminin yapıldığı bilgisayarın mutlak dosya yolları (Absolute Paths) ELF formatında kalmıştır: `"/home/user/tmp/gcc-4.7.2-msp430/msp430/..."` ve `"/home/user/mspgcc-4.7.2/lib/gcc/msp430/..."`. Bu bilgi, firmware'in bir Linux ortamında ve "user" adlı bir kullanıcı dizininde derlendiğini kesin olarak ispatlamaktadır.

| Kategori | Tespit Edilen Örnek Metin (String) |	Yorum / Teknik Anlamı |
|---|---|---|
| Boot & İşletim Sistemi | `Starting Contiki-NG-release...` |	Sistemin başlangıç mesajı ve kullanılan OS versiyonu. |
| Fiziksel Ağ (MAC) | `- 802.15.4 PANID: 0x%04x` |	Cihazın `IEEE 802.15.4` kablosuz standardında çalıştığı kanıtlanmıştır. |
| Yönlendirme & IPv6 | `Tentative link-local IPv6 address: created a new RPL DAG` |	Cihazın IPv6 kullandığı ve RPL yönlendirme ağacı (DAG) oluşturabildiği görülmüştür. |
| Sensör & Donanım | `ADXL345 sensor CC2420 driver` |	İvmeölçer sensörünün ve radyo çipinin sürücü metinleridir. |
| Geliştirici İzleri | `GCC: (GNU) 4.7.2 /home/user/tmp/gcc-4.7.2...` |	Kodu derleyen GCC versiyonu ve derlemenin yapıldığı bilgisayarın mutlak dosya yoludur. |
---

# 5. Assembly / Instruction Analizi

```
furkann@DESKTOP-CPNNO3A:~/contiki-ng/examples/rpl-udp$ msp430-objdump -d new-firmware.z1 | grep -A 20 "<main>:"
0000313e <main>:
    313e:       b0 13 e4 6a     calla   #27364          ;0x06ae4
    3142:       b0 13 3a 45     calla   #17722          ;0x0453a
    3146:       b0 13 da aa     calla   #43738          ;0x0aada
    314a:       b0 13 7c 6d     calla   #28028          ;0x06d7c
    314e:       0e 43           clr     r14             ;
    3150:       3f 40 3c 11     mov     #4412,  r15     ;#0x113c
    3154:       b0 13 8e 6e     calla   #28302          ;0x06e8e
    3158:       b0 13 34 50     calla   #20532          ;0x05034
    315c:       b1 13 8a 3d     calla   #81290          ;0x13d8a
    3160:       b0 13 2a 51     calla   #20778          ;0x0512a
    3164:       b0 13 f2 bf     calla   #49138          ;0x0bff2
    3168:       b0 13 f6 6a     calla   #27382          ;0x06af6
    316c:       b0 13 dc 6e     calla   #28380          ;0x06edc
    3170:       b0 13 c2 68     calla   #26818          ;0x068c2
    3174:       b0 13 1c 69     calla   #26908          ;0x0691c
    3178:       b2 90 03 00     cmp     #3,     &0x1160 ;
    317c:       60 11
    317e:       0e 38           jl      $+30            ;abs 0x319c
    3180:       30 12 3a ca     push    #-13766 ;#0xca3a
    3184:       30 12 3f ca     push    #-13761 ;#0xca3f
```
```
furkann@DESKTOP-CPNNO3A:~/contiki-ng/examples/rpl-udp$ msp430-objdump -d new-firmware.z1 | grep -A 20 "<port1_isr>:"
0000353e <port1_isr>:
    353e:       3f 14           pushm.a #4,     r15     ;20-bit words
    3540:       f2 b0 40 00     bit.b   #64,    &0x0023 ;#0x0040
    3544:       23 00
    3546:       15 24           jz      $+44            ;abs 0x3572
    3548:       e2 b2 23 00     bit.b   #4,     &0x0023 ;r2 As==10
    354c:       12 20           jnz     $+38            ;abs 0x3572
    354e:       3f 40 54 12     mov     #4692,  r15     ;#0x1254
    3552:       b0 13 fc c6     calla   #50940          ;0x0c6fc
    3556:       0f 93           cmp     #0,     r15     ;r3 As==00
    3558:       32 24           jz      $+102           ;abs 0x35be
    355a:       3d 40 20 00     mov     #32,    r13     ;#0x0020
    355e:       0e 43           clr     r14             ;
    3560:       3f 40 54 12     mov     #4692,  r15     ;#0x1254
    3564:       b0 13 e0 c6     calla   #50912          ;0x0c6e0
    3568:       5f 42 23 00     mov.b   &0x0023,r15     ;0x0023
    356c:       7f f0 bf ff     and.b   #-65,   r15     ;#0xffbf
    3570:       18 3c           jmp     $+50            ;abs 0x35a2
    3572:       5f 42 23 00     mov.b   &0x0023,r15     ;0x0023
    3576:       4f 93           cmp.b   #0,     r15     ;r3 As==00
    3578:       1b 34           jge     $+56            ;abs 0x35b0
```
### Kullanılan Araç Zinciri ve Analizin Amacı 
Bu analizde, derlenmiş makine kodunu insan tarafından okunabilir Assembly diline çevirmek (Disassembly) amacıyla `msp430-objdump -d` aracı kullanılmıştır. Analizin temel amacı; fonksiyon çağrı yapılarını, donanım kesme (ISR) rutinlerinin CPU seviyesindeki davranışlarını, derleyici optimizasyonlarını ve işletim sistemine özgü bellek/zamanlayıcı geçişlerini makine komutları düzeyinde incelemektir.

### Instruction Sequence ve Function Call Graph Analizi
Sistemin ana döngüsü olan `main` fonksiyonunun başlangıcı incelendiğinde, programın doğrusal bir akış (sequence) yerine yoğun bir şekilde alt rutinlere (subroutine) dallandığı görülmüştür.
```
0000313e <main>:
    313e:       b0 13 e4 6a     calla   #27364          ;0x06ae4
    3142:       b0 13 3a 45     calla   #17722          ;0x0453a
    3146:       b0 13 da aa     calla   #43738          ;0x0aada
    ...
```
Yukarıdaki kesitte art arda gelen `calla` (Call Absolute) komutları, Contiki işletim sisteminin başlatma (boot) hiyerarşisini göstermektedir. İşlemci sırasıyla `0x06ae4` (`platform_init_stage_one`), `0x0453a` (`clock_init`) ve `0x0aada` (`rtimer_init`) gibi alt rutinlere zıplayıp geri dönmektedir. Bu durum, modüler bir **Call Graph** (Çağrı Grafiği) yapısının kanıtıdır.

### Register Kullanımı ve Function Prologue
Fonksiyonların başlangıcında, o anki donanım durumunu korumak amacıyla CPU yazmaçlarının (Register) Yığın (Stack) bölgesine yedeklenmesi (Prologue) işlemi gözlemlenmiştir.
```
    3180:       30 12 3a ca     push    #-13766 ;#0xca3a
    3184:       30 12 3f ca     push    #-13761 ;#0xca3f
```
`main` fonksiyonu içinde yer alan `push` komutları, fonksiyonun işleyişi sırasında üzerine yazılacak olan Register değerlerini veya yerel değişkenleri Stack uzayına atarak korumaya almaktadır.

###  ISR Akışı ve Context Switching (Bağlam Değişimi)
Donanımdan gelen bir kesmenin (Örneğin GPIO tetiklemesi) standart bir C fonksiyonundan ne kadar farklı çalıştığı `port1_isr` fonksiyonu üzerinden analiz edilmiştir.
```
0000353e <port1_isr>:
    353e:       3f 14           pushm.a #4,     r15     ;20-bit words
    3540:       f2 b0 40 00     bit.b   #64,    &0x0023 ;#0x0040
    ...
```
Normal fonksiyonlar tekil `push` komutlarıyla başlarken, ISR'nin `pushm.a #4, r15` (Push Multiple) komutuyla başlaması son derece kritiktir. Kesme geldiğinde CPU o an yaptığı işi aniden bırakmak zorundadır; bu nedenle 

tüm kritik yazmaçlar 20-bit boyutunda topluca yığına kopyalanarak "Bağlam Değişimi" (Context Switching) güvenli bir şekilde gerçekleştirilmiş olur.

### Donanım Register Etkileşimi (Hardware Access)
ISR bloğundaki `bit.b #64, &0x0023` komutu, doğrudan bir donanım adresine müdahaleyi gösterir. Bellek analizinde `0x0023` adresinin `__P1IFG` (Port 1 Interrupt Flag Register) olduğu bilinmektedir. Kesme yöneticisi, hangi pinden kesme geldiğini anlamak için bu donanım register'ındaki 6. biti (Değeri 64) test etmektedir.

###  Branch (Dallanma), Karar Yapıları ve Optimizasyon Analizi
Kod içerisindeki `if/else` karar yapılarının derleyici tarafından nasıl Assembly döngülerine çevrildiği ve optimize edildiği tespit edilmiştir.
```
    3178:       b2 90 03 00     cmp     #3,     &0x1160 ;
    317c:       60 11
    317e:       0e 38           jl      $+30            ;abs 0x319c
```
Burada `cmp` (Compare) komutu, RAM'deki `0x1160` adresinde bulunan `curr_log_level_main` değişkeninin değerini `3` sabiti ile karşılaştırmaktadır. Ardından gelen `jl` (Jump if Less) komutu, eğer log seviyesi 3'ten küçükse aradaki `push` komutlarını atlayarak doğrudan `0x319c` adresine dallanmaktadır. Bu tür mantıksal sıçramalar, gereksiz kod icrasını önleyen tipik derleyici optimizasyonlarıdır.

### Loop (Döngü) Davranışları
Benzer bir şartlı dallanma (Branching) durumu ISR içerisinde de görülmektedir.
```
    3556:       0f 93           cmp     #0,     r15     ;r3 As==00
    3558:       32 24           jz      $+102           ;abs 0x35be
```
Burada `r15` register'ındaki değer sıfır ise `jz` (Jump if Zero) komutu tetiklenmekte ve CPU kod bloğunda 102 byte ileriye (`0x35be` adresine) atlayarak bir karar döngüsünü (Loop) tamamlamaktadır. Bu yapı, C dilindeki `while()` veya `if()` bloklarının makine koduna dönüşmüş (expanded) halidir.

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

Bu bölümde, `new-firmware.z1` imajının ağ üzerindeki rolü, paket yönlendirme stratejileri ve MAC/Fiziksel katman etkileşimleri detaylı olarak incelenmiştir. Analizlerde `msp430-nm` ve `msp430-strings` araçlarından elde edilen veriler referans alınmıştır.

### Unicast kullanım tespiti
Ağdaki iki düğüm (node) arasında noktadan noktaya doğrudan iletişimi sağlayan Unicast haberleşme yapısı, firmware içerisinde aktif olarak kullanılmaktadır. `msp430-nm` çıktılarında tespit edilen `uip_ds6_route_lookup` ve `uip_ds6_route_nexthop` sembolleri, gelen paketlerin hedef IP adresine göre yönlendirme tablosunda eşleştirildiğini göstermektedir. Ayrıca `msp430-strings` çıktısındaki `"unicast DIS, reply to sender"` hata ayıklama mesajı, sistemin RPL protokolü kapsamında hedefi belli olan (unicast) mesajlar üretebildiğini kesin olarak ispatlamaktadır.

### Broadcast kullanım tespiti
Yönlendirme ağacı oluşturulurken (DAG) veya komşu keşif (Neighbor Discovery) aşamalarında ağdaki tüm düğümlere aynı anda ulaşmak için Broadcast mekanizması kullanılmaktadır. Sembol tablosunda açıkça görülen `frame802154_is_broadcast_addr` ve `packetbuf_holds_broadcast` fonksiyonları, MAC katmanında paketin hedef adresinin `0xFFFF` (`802.15.4` broadcast adresi) olup olmadığını denetler. Cihaz ağa ilk katıldığında yönlendirici bulmak amacıyla bu mekanizmaya sıklıkla başvurur.

### Multicast tespiti
IPv6 ağlarının bel kemiği olan Multicast (çoklu yayın) mekanizması, özellikle RPL protokolündeki ağ topolojisi mesajları için bu firmware'de aktif kılınmıştır. Sembol listesindeki `rpl_multicast_addr` değişkeni ve `msp430-strings` analizinde karşımıza çıkan `"invalid DAG MC"` ve `"sending a %s-DIO with rank %u to"` logları, cihazın spesifik bir gruba (Örneğin tüm RPL router'larına) çoklu yayın yapabildiğini göstermektedir.

### IPv6 stack kullanımı
Firmware, kısıtlı IoT cihazları için optimize edilmiş `Contiki uIP` (Micro IP) ve `uIPv6` yığınını barındırmaktadır. `uip_process`, `tcpip_process`, `uip_icmp6_input` ve `uip_ds6_init` sembolleri bu durumun temel kanıtlarıdır. Cihaz, IPv6 adreslemesini kullanırken, `802.15.4` radyo paketlerinin dar boyutuna (127 byte MTU) sığabilmek için `sicslowpan_init` sembolü ile tanımlanan **6LoWPAN** sıkıştırma (Header Compression) mimarisini koşturmaktadır. `msp430-strings` loglarındaki `"Tentative link-local IPv6 address:"` ibaresi cihazın kendi kendine IPv6 adresi atayabildiğini kanıtlar.

### RPL routing analizi
RPL (Routing Protocol for Low-Power and Lossy Networks), bu cihazın ana yönlendirme stratejisidir. `rpl_dag_root_start`, `rpl_process_dio` (DODAG Information Object) ve `rpl_process_dao` (Destination Advertisement Object) sembolleri, cihazın yönlendirme ağacında (DAG - Directed Acyclic Graph) bir rol üstlendiğini gösterir. Cihaz, çevresindeki düğümlerden DIO mesajları alarak köke (root) olan uzaklığını (Rank) hesaplamakta ve DAO mesajları göndererek kendi varlığını kök düğüme bildirmektedir. Loglardaki `"created a new RPL DAG"` ve `"significant rank update %u->%u"` mesajları RPL mekanizmasının tamamen aktif çalıştığını doğrular.

### TSCH scheduler çağrıları
Yapılan detaylı sembol tablosu (`msp430-nm`) analizinde, **TSCH** (Time Slotted Channel Hopping) planlayıcısına ait herhangi bir fonksiyon veya sembol (örneğin `tsch_process` veya `tsch_init`) tespit edilememiştir. Bu durum cihazın zaman senkronizasyonlu ve frekans atlamalı bir MAC katmanı kullanmadığını, paket gönderimlerinin slotlara (zaman dilimlerine) bölünmediğini açıkça göstermektedir.

### MAC layer interaction
Fiziksel katman ile ağ katmanı arasındaki köprüyü kuran MAC etkileşimleri, `802.15.4` standartlarına uygun olarak tasarlanmıştır. `framer_802154_create` ve `framer_802154_parse` sembolleri, IP paketlerine `IEEE 802.15.4` çerçeve başlıklarının (Frame Header) eklendiği ve çözüldüğü noktalardır. Ayrıca mac_sequence_init fonksiyonu ile MAC seviyesinde paketlere sıra numarası (Sequence Number) verilerek tekrar eden (duplicate) paketlerin MAC katmanında hızla reddedilmesi sağlanmıştır.

### Packet buffer kullanımı
Contiki-NG'nin RAM kısıtları nedeniyle, sistemde paketleri tutmak için devasa kuyruklar yerine ortak bir hafıza bloğu olan packetbuf mimarisi kullanılmıştır. Sembol tablosundaki packetbuf_copyfrom, packetbuf_hdralloc, packetbuf_set_attr ve packetbuf_dataptr gibi fonksiyonlar, radyo çipinden alınan veya uygulamadan radyoya gönderilecek olan verilerin tek bir statik tampon (buffer) üzerinde işlendiğini, başlık (header) ekleme/çıkarma işlemlerinin bu tampon üzerinde yapıldığını ispatlamaktadır.

### Neighbor table erişimi
IPv6 ve RPL protokollerinin sağlıklı çalışabilmesi için cihazın sinyal menzilindeki diğer düğümleri tanıması gerekir. `uip_ds6_nbr_add`, `uip_ds6_neighbors_init` ve `nbr_table_register` sembolleri komşu tablosu yönetimini ifade eder. Cihazın kısıtlı belleği nedeniyle komşu sayısının bir sınırı vardır; bu durum `msp430-strings` çıktısındaki `"neighbor table full, dropping DIO"` (komşu tablosu dolu, DIO paketi atılıyor) hata logu ile desteklenmektedir.

### Radio transmission akışı
Bir paketin cihaz içindeki yazılım katmanlarından donanıma aktarılma süreci hiyerarşik bir çağrı silsilesi izler. Analizlere göre yukarıdan aşağıya akış şu şekildedir:
* Yüksek katman `tcpip_ipv6_output` fonksiyonunu çağırır.
* Paket MAC katmanına inerek `csma_output_packet` fonksiyonuna iletilir.
* Donanım seviyesinde radyo sürücüsüne ulaşılarak `cc2420_transmit` ve `cc2420_send` fonksiyonları tetiklenir ve paket anten üzerinden fiziksel olarak ortama yayılır.

### Retransmission logic
Kablosuz iletişimdeki kayıpları tolere edebilmek için yeniden iletim algoritmaları koda gömülmüştür. Ağ katmanı düzeyinde RPL protokolünün `schedule_dao_retransmission` sembolü kullanılarak hedefine ulaşamayan yönlendirme metriklerinin tekrar gönderilmesi planlanmaktadır. MAC düzeyinde ise CSMA sürücüsü ortamın dolu olması durumunda üstel geri çekilme (exponential backoff) algoritması uygulayarak paketi daha sonra göndermeyi dener.

### ACK mekanizmaları
Paketlerin hedefine başarıyla ulaşıp ulaşmadığını denetleyen onay mekanizması hem donanımsal hem de yazılımsal olarak konfigüre edilmiştir. Radyo çipi (`CC2420`) sürücüsündeki `set_auto_ack` sembolü, `IEEE 802.15.4` donanımsal ACK (Hardware Acknowledgement) özelliğinin aktifleştirildiğini gösterir. Böylece, hedef cihaz paketi aldığında işlemciyi yormadan doğrudan radyo çipi üzerinden milisaniyeler içinde onay (ACK) sinyali dönebilmektedir.

### CSMA/TSCH farkları
TSCH tespit edilememesi üzerine kod mimarisi incelendiğinde, firmware'in MAC katmanı olarak CSMA (Carrier-Sense Multiple Access) kullandığı kesinleşmiştir. Sembol tablosundaki `csma_driver`, `csma_output_packet` ve `csma_security_create_frame` değişkenleri bunun doğrudan kanıtıdır.

**Fark Analizi:** TSCH, cihazların yüksek doğruluklu zamanlayıcılarla anlaşıp belirli milisaniyelik dilimlerde uyanarak haberleşmesini sağlarken (deterministik); bu cihazda bulunan CSMA yaklaşımı, radyo kanalının anlık olarak dinlenmesi (Carrier Sense) ve ortam boşsa paketin asenkron olarak iletilmesi prensibine (fırsatçı yaklaşım) dayanır.

### Contiki network API kullanımı
Sistemin uygulama katmanı ile ağ yığını arasındaki iletişim, Contiki'nin olay güdümlü (Event-driven) ağ API'leri üzerinden sağlanmaktadır. `tcpip_event` global değişkeni ve `tcpip_input` sembolü bunun merkezindedir. Sistemdeki ana süreçler (örneğin `hello_world_process`), yeni bir ağ paketi geldiğinde asenkron olarak tetiklenen `tcpip_event` sinyalini bekler (Process Yield/Wait mantığı) ve ancak paket belleğe düştüğünde ağ API'si üzerinden bu veriyi çekerek işler.

| Analiz Kriteri| Tespit Edilen Sembol / Log (Kanıt) | Sonuç ve Kullanım Amacı |
|---|---|---|
| Yönlendirme (Routing) | rpl_process_dio, rpl_process_dao | Ağaç (DAG) yapısında RPL protokolü aktif olarak koşturulmaktadır. |
| IP ve Adaptasyon | tcpip_process, sicslowpan_init | IPv6 adreslemesi ve 802.15.4 için 6LoWPAN başlık sıkıştırması kullanılmaktadır. |
| Yayın Tipleri | uip_ds6_route_lookup, "unicast DIS | "Hem doğrudan (Unicast) hem de çoklu/genel (Broadcast/Multicast) yayın aktiftir. |
| MAC ve Çarpışma | csma_driver, csma_output_packet| TSCH yerine, fırsatçı kanal dinleme mantığına dayanan CSMA MAC protokolü aktiftir. |
| Bellek Yönetimi | packetbuf_copyfrom, packetbuf_hdralloc | Paketler için dinamik bellek yerine statik packetbuf mimarisi kullanılmaktadır. |


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

Bu bölümde, firmware imajının çekirdeğini oluşturan Contiki-NG işletim sisteminin olay güdümlü (event-driven) yapısı, görev yönetimi ve ağ yığını etkileşimleri detaylı olarak analiz edilmiştir.

### PROCESS_THREAD recovery

İşletim sisteminde koşan bağımsız görevlerin (thread) tespiti (recovery) işlemi, `msp430-nm` sembol tablosu üzerinden başarıyla gerçekleştirilmiştir. Contiki mimarisinde her süreç, bellekte özel bir isim uzayı (namespace) ile tutulur. Analizde `process_thread_hello_world_process`, `process_thread_tcpip_process`, `process_thread_sensors_proces`s ve `process_thread_ctimer_process` sembolleri açıkça izole edilmiştir. Bu durum, cihazın tek bir monolitik döngü (`while(1)`) yerine, ağ katmanını, sensör okumalarını ve kullanıcı uygulamasını birbirinden bağımsız iş parçacıkları halinde yürüttüğünü kanıtlamaktadır.

(Not: Protothread makrolarının genişletilmiş (expanded) `switch-case` formlarını doğrudan gözlemleyebilmek için derleme öncesinde kaynak kodlar (.c) üzerinde `msp430-cpp` (C Preprocessor) aracının koşturulması gerekmektedir. Ancak analiz derlenmiş `.z1 `ELF imajı üzerinden yürütüldüğü için bu genişleme süreci disassembly ve sembol adresleri üzerinden reverse-engineering yöntemleriyle doğrulanmıştır.)

### Protothread expansion

Contiki-NG, her bir görev için ayrı bir yığın (stack) belleği ayırmak yerine RAM tasarrufu sağlamak için "Protothread" mimarisini kullanır. Tespit edilen görevler geleneksel işletim sistemlerindeki (örneğin Linux) thread'ler gibi kendi izole RAM alanlarına sahip değildir. Bunun yerine hepsi aynı yığını (stack) paylaşır. Derleme aşamasında (expansion), C dilinde yazılan protothread'ler, her görevin nerede kaldığını hafızada tutan `pt->lc` (Local Continuation) isimli 2 baytlık bir durum değişkenine (state variable) indirgenir. Böylece her thread için yüzlerce bayt RAM harcamak yerine, thread başına sadece birkaç bayt ile görev yönetimi (multitasking) sağlanır.

### Event-driven scheduler analizi

Sistemin görev zamanlayıcısı (Scheduler), geleneksel "Round-Robin" (sürekli sırayla çalıştırma) yöntemi yerine olay güdümlü (Event-Driven) çalışmaktadır. `msp430-nm` tablosunda tespit edilen `process_post`, `process_run` ve `process_poll` fonksiyonları zamanlayıcının kalbini oluşturur. Bir donanım kesmesi (ISR) oluştuğunda veya ağdan bir paket geldiğinde, çekirdeğe `process_post` ile bir olay (event) fırlatılır. Çekirdek, `process_run` döngüsü içerisinde olay kuyruğunu (event queue) tarar ve sadece kendisine olay atanmış (tetiklenmiş) süreçleri uyandırır. Olay beklemeyen süreçler CPU döngüsü tüketmez.


### etimer/ctimer usage

Contiki-NG'nin görevleri periyodik olarak uyandırmak için kullandığı iki farklı zamanlayıcı mimarisi kod içinde aktif olarak bulunmuştur:
* **etimer (Event Timer):** `etimer_set` ve `etimer_expired` sembolleriyle görülür. Belirlenen süre dolduğunda, işletim sistemi zamanlayıcısı üzerinden o etimer'ı kuran sürece bir "Zamanlayıcı Olayı" (Timer Event) göndererek süreci uyandırır.
* **ctimer (Callback Timer):** `ctimer_set` sembolü ile görülür. Herhangi bir sürece bağlı kalmadan, süre dolduğunda doğrudan bellekteki bir fonksiyonu (callback pointer) çağırır. `process_thread_ctimer_process` sembolü, sistemin arka planda ctimer'ları yönetmek için özel bir süreç ayırdığını kanıtlar.

### PROCESS_BEGIN/END expansion

Contiki-NG'de süreçlerin gövdesini oluşturan `PROCESS_BEGIN()` ve `PROCESS_END()` makroları, C Preprocessor (Ön İşlemci) tarafından arka planda büyük bir durum makinesine (State Machine) çevrilir.
* `PROCESS_BEGIN()` makrosu aslında görünmez bir `switch (pt->lc) { case 0:` satırını başlatır.
* `PROCESS_END()` makrosu ise bu switch-case yapısını kapatır, `pt->lc = 0;` atamasını yapar ve süreci sonlandıran `return PT_ENDED;` komutunu işletir. Assembly düzeyinde `process_thread...` fonksiyonlarının incelenmesi, bu switch-case yapısına ait dallanma (jump/branch) komutlarını açıkça doğrulamaktadır.

### PROCESS_YIELD flow

Bir süreç `PROCESS_YIELD()` makrosunu çağırdığında sistemin nasıl tepki verdiği işletim sistemi teorisi açısından kritiktir. Bu makro çağrıldığında:
* Derleyici o satırın numarasını (veya etiketini) `pt->lc` değişkenine kaydeder.
* `return PT_YIELDED;` komutuyla süreç o noktada CPU'yu gönüllü olarak terk eder (Yielding).
* CPU, işletim sistemi zamanlayıcısına geri döner ve sıradaki diğer olayları işler.
* Bu sürece yeniden bir olay geldiğinde, `PROCESS_BEGIN` altındaki switch bloğu `pt->lc` değerini okur ve fonksiyonu tam olarak `YIELD` edildiği satırdan yeniden başlatır. Bu asenkron yapı, sistemin meşgul bekleme (busy-waiting) yapmasını engeller.

### NETSTACK interaction

İşletim sistemi süreçleri ile donanımsal ağ katmanı arasındaki iletişim `NETSTACK` makroları üzerinden izole edilmiştir. Sembol tablosundaki `netstack_init` ve `tcpip_event` bulguları, ağın süreçlerle nasıl konuştuğunu gösterir. Radyo donanımına bir paket düştüğünde donanım kesmesi (ISR) tetiklenir, bu kesme `tcpip_process'e` bir sinyal (poll) yollar. Ağ yığını paketi işledikten sonra, ilgili kullanıcı uygulamalarına (Örneğin `hello_world_process`) global bir uyarı olan `tcpip_event` sinyalini yollayarak uygulamanın paketi okumasını sağlar.

### Packetbuf lifecycle

RAM kapasitesi kısıtlı olan bu mimaride, her gelen/giden paket için işletim sisteminden dinamik RAM (`malloc/free`) talep edilmez. Bunun yerine "Sıfır Kopya" (Zero-copy) mimarisine yaklaşan statik `packetbuf` yapısı kullanılır. Sembollerde tespit edilen yaşam döngüsü şu şekildedir:
* **`packetbuf_clear`:** Önceki paketten kalan veriler tampondan silinir.
* **`packetbuf_copyfrom`:** Yeni gönderilecek veri (payload) bu ortak RAM bloğuna kopyalanır.
* **`packetbuf_hdralloc`:** Paket MAC ve IP katmanlarından aşağı indikçe, verinin önüne yeni protokol başlıkları için yer açılır. Paket iletildikten sonra aynı tampon bir sonraki işlem için serbest bırakılır.

### uIP callback chain

İşletim sistemindeki ağ durum güncellemeleri, klasik if-else kontrolleri yerine tetikleyici (callback) zincirleriyle yönetilmektedir. Sembol tablosundaki `uip_ds6_link_callback` ve `netstack_process_ip_callbac`k fonksiyonları bu zincirin temel yapıtaşlarıdır. RPL yönlendirme protokolü komşu cihazlardan yeni bir yol (route) bulduğunda veya mevcut bir bağlantı koptuğunda, işletim sistemi bu **"callback"** fonksiyonlarını otomatik olarak çağırarak (chaining) üst katmandaki uygulamaların ağ değişikliklerinden anında haberdar olmasını sağlar.

### Rime stack usage

Yapılan kapsamlı String ve Sembol analizlerinde, Contiki'nin eski ve hafif haberleşme yığını olan **"Rime Stack"** (örneğin `rime_init`, `mesh_open`, `unicast_send`) mekanizmalarına ait hiçbir kanıt bulunamamıştır. Analizler sonucunda, cihazdaki Rime mimarisinin derleyici seviyesinde tamamen devre dışı bırakıldığı, iletişimin bütünüyle modern ve standartlara uygun olan uIP (IPv6) ve 6LoWPAN ağ yığını üzerinden yürütüldüğü kesin olarak raporlanmıştır.

İşletim Sistemi Konsepti | Tespit Edilen Sembol / Kanıt | Teknik İşlev ve Mimari Karşılığı |
|---|---|---|
|Protothread (Görevler) | process_thread_hello_world... process_thread_tcpip_process | Süreçlerin izole RAM yığınları yerine, bellek tasarrufu için pt->lc (switch-case) mantığıyla ortak yığında yürütülmesi. | 
| Olay Güdümlü Zamanlayıcı | process_post, process_run process_poll | Döngüsel (Round-robin) çalıştırma yerine, süreçlerin sadece donanım/ağ olayı (event) geldiğinde uyandırılması.Zaman Yönetimi (Timers)etimer_set, ctimer_setGörevlerin, sistemi kilitlemeden (busy-wait olmadan) olay (etimer) veya geri çağrı (ctimer) ile zamanlanması. | 
| Ağ Bellek Yönetimi | packetbuf_copyfrom packetbuf_hdralloc | Ağ paketleri için dinamik bellek (malloc/free) yerine, "sıfır kopya" mantığına dayanan statik tampon (packetbuf) kullanımı. |
| Donanım-Ağ Etkileşimi | tcpip_event, netstack_init | Radyo çipinden gelen kesmelerin işletim sistemine asenkron bir sinyal (tcpip_event) olarak aktarılması süreci. |


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
