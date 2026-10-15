---
layout: post
title: "Görünmez Dalgaların Geometrisi: Bir Radarın Anatomisi ve FMCW Matematiği"
date: 2026-10-15 18:00:00 +0300
author: "Kerem Uysal"
categories: [Donanım, Radar]
tags: [fmcw, radar, rf, sinyal-isleme, dsp]
lang: tr-TR
ref: fmcw-radar-geometrisi
math: true
description: "Işık hızında hareket eden elektromanyetik dalgaların zamanı frekansa kodlama sanatı: FMCW radar donanım mimarisi, chirp matematiği, 2D-FFT sinyal işleme algoritmaları ve milimetre-dalga PCB tasarım zorlukları."
---

> 🌐 **Dil / Language / Sprache:** **Türkçe** | [English](/posts/geometry-of-invisible-waves-anatomy-of-a-radar-and-fmcw-mathematics-en/) | [Deutsch](/posts/geometrie-unsichtbarer-wellen-anatomie-eines-radars-und-fmcw-mathematik-de/)
{: .prompt-info }

## 1. Giriş: Işık Hızında Zamana Meydan Okumak

Elektromanyetik bir dalgayı uzaya bıraktığınız anda, evrenin hız limiti olan ışık hızıyla ($c \approx 3 \times 10^8\text{ m/s}$) yarışmaya başlarsınız. Bu hız, günlük makroskobik algımız için kavranması güç bir ölçektedir: Işık, bir saniyede Dünya’nın çevresini yaklaşık yedi buçuk kez döner; bir nanosaniyede ($10^{-9}\text{ s}$) ise yalnızca **30 santimetre** yol alır.

Şimdi masanın üzerindeki bir radar sensörüyle sadece $1.5\text{ metre}$ uzağınızdaki bir nesnenin mesafesini ölçmek istediğinizi düşünün. Gönderdiğiniz elektromanyetik dalganın hedefe çarpıp alıcı antene geri dönmesi (round-trip) toplam $3\text{ metrelik}$ bir yoldur ve bu yolculuk yalnızca **10 nanosaniye** sürer. Eğer hedefiniz $15\text{ santimetre}$ yer değiştirirse, ölçmeniz gereken zaman farkı sadece **1 nanosaniyedir**.

Klasik **Darbeli Radar (Pulse Radar / Time-of-Flight)** mimarisiyle bu problemi çözmeye çalıştığınızda üç büyük donanımsal duvarla karşılaşırsınız:

1. **Kör Bölge (Blind Zone) ve Yüksek Tepe Gücü:** Yakın mesafedeki bir hedefi görebilmek için darbe genişliğini ($\tau_p$) pikosaniye veya birkaç nanosaniye mertebesine indirmeniz gerekir. Darbe süresi kısaldıkça, hedeften anlamlı bir enerji dönebilmesi için darbenin anlık tepe gücünü (Peak Power) yüzlerce Watt veya Kilowatt mertebesine çıkarmak zorundasınızdır. Üstelik verici (TX) bu yüksek güçlü darbeyi basarken alıcı (RX) katındaki hassas yükselteçleri (LNA) yakmamak için alıcıyı kapatmanız gerekir; bu da radarın burnunun dibindeki hedeflere karşı kör kalması demektir.
2. **Ultra-Hızlı ADC Zorunluluğu:** Nanosaniye mertebesindeki bir darbenin varış zamanını doğrudan zaman domeninde (Time Domain) sayısallaştırmak için Giga-Samples Per Second (GSPS) hızında çalışan, yüksek güç tüketen ve son derece pahalı ADC (Analog-to-Digital Converter) donanımlarına ihtiyaç duyulur.
3. **Entegrasyon Kısıtları:** Yüksek tepe güçleri, modern CMOS veya SiGe BiCMOS teknolojileriyle üretilen tek çipli (MMIC - Monolithic Microwave Integrated Circuit) otonom araç, İHA ve endüstriyel sensör mimarilerine fiziksel olarak sığmaz.

İşte modern otonom sistemlerde, otomotiv radarlarında ve hassas endüstriyel algılayıcılarda darbeli radarlar yerine **FMCW (Frequency-Modulated Continuous Wave — Frekans Modülasyonlu Sürekli Dalga)** radarların tercih edilmesinin temel nedeni budur.

> 💡 **FMCW Paradigması:** FMCW radar, ışık hızındaki bir dalganın uçuş süresini kronometreyle yakalamaya çalışmaz; bunun yerine **zamanı frekansa kodlar**. Gönderilen sinyalin frekansını zamanla doğrusal olarak değiştirerek, ölçülmesi imkânsıza yakın nanosaniyelik zaman gecikmelerini ($\tau$), mikser çıkışında kolayca ölçülebilen kilohertz ($\text{kHz}$) veya birkaç megahertz ($\text{MHz}$) mertebesindeki **vuru frekanslarına (Beat Frequency - $f_b$)** dönüştürür.
{: .prompt-tip }

---

## 2. Bir FMCW Radarın Donanım Anatomisi

Bir FMCW radarın RF ön ucu (RF Front-End) ve temel bant (Baseband) sinyal zinciri, mikrodalga mühendisliği ile sayısal sinyal işlemenin (DSP) kusursuz bir senfonisidir. Sistemin blok diyagramını sinyal akışı üzerinden adım adım inceleyelim:

```text
                  +-----------------------+
                  |  Chirp Sentezleyici   |
                  |     (PLL + VCO)       |
                  +-----------+-----------+
                              |
                     [LO Sinyali: f_TX(t)]
                              |
            +-----------------+-----------------+
            |                                   |
            v                                   v
  +-------------------+               +-------------------+
  |  Güç Yükselteci   |               |     RF Mikser     |
  |       (PA)        |               |   (Downconverter) |
  +---------+---------+               +---------^---------+
            |                                   |
       [TX Anteni]                         [RX Sinyali]
            |                                   |
            v                           +-------+-------+
       )))  Dalga  (((                  | Düşük Gürültü |
                                        |  Yükselteci   |
          [ Hedef ]                     |     (LNA)     |
                                        +-------^-------+
                                                |
                                           [RX Anteni]
```

### Donanım Katmanları ve Kritik Fonksiyonları:

1. **Frekans Sentezleyici (Chirp Generator - PLL & VCO):**
   Sistemin kalbidir. Gerilim Kontrollü Osilatör (VCO) ve Kesirli Sayıcı Tabanlı Faz Kilitlemeli Döngü (Fractional-N PLL), zaman içinde frekansı kusursuz bir doğrusallıkla ($R^2 > 0.999$) artan veya azalan sinüs dalgaları üretir. Frekans modülasyonundaki en ufak bir doğrusallık hatası (chirp non-linearity), spektral saçılmaya ve sahte hedeflere (ghost targets) yol açar.
2. **Güç Yükselteci (PA) ve TX Anteni:**
   Sentezlenen milimetre-dalga sinyalini (örneğin $77\text{ GHz}$) genellikle $+10\text{ dBm}$ ila $+15\text{ dBm}$ ($\sim 10-30\text{ mW}$) seviyesine yükseltir ve yönlü bir mikroşerit yama anten (patch antenna) dizisi üzerinden havaya fırlatır. Darbeli radarların kilovatlık güçlerine kıyasla bu güç seviyesi inanılmaz derecede düşüktür.
3. **Düşük Gürültülü Yükselteç (LNA - Low Noise Amplifier):**
   Hedeften yansıyıp geri dönen sinyal, serbest uzay yayılım kaybı ($1/R^4$ radar denklemi) nedeniyle pikowatt veya femtowatt mertebesine kadar sönümlenmiştir. LNA, bu son derece zayıf sinyali kendi gürültü tabanını (Noise Figure - NF) minimumda tutarak yükseltmek zorundadır.
4. **RF Mikser (Downconversion):**
   FMCW mimarisinin matematiksel sihrinin gerçekleştiği noktadır. Vericiye gönderilen anlık sinyal (Local Oscillator - LO) ile LNA'dan gelen gecikmiş hedef sinyali (RX) analog olarak birbiriyle çarpılır. İki sinüsün çarpımı trigonometrik olarak toplam ($f_{TX} + f_{RX} \approx 154\text{ GHz}$) ve fark ($f_{TX} - f_{RX} \sim \text{kHz}-\text{MHz}$) bileşenlerini üretir.
5. **Temel Bant Filtreleme ve ADC (Analog-to-Digital Converter):**
   Toplam frekansı yok etmek için bir Alçak Geçiren Filtre (LPF) kullanılır. Ayrıca doğrudan TX anteninden RX antenine sızan yüksek güçlü kaçak sinyalleri (TX-to-RX Leakage) ve radarın hemen önündeki zemin yansımalarını bastırmak için ayarlanabilir bir Yüksek Geçiren Filtre (HPF) devrededir. Temizlenen vuru sinyali ($f_b$), yalnızca **birkaç megasample/saniye (MSPS)** hızında çalışan standart bir ADC ile sayısallaştırılır.

---

## 3. Chirp Matematiği ve Mesafe (Range) Kestirimi

Bir FMCW radarın gönderdiği temel sinyal darbesine **Chirp (Cıvıltı)** adı verilir. Doğrusal Frekans Modülasyonlu (LFM) bir chirp sinyalinin anlık frekansı zamanla doğrusal olarak değişir:

$$f_{TX}(t) = f_c + S \cdot t \quad (0 \le t \le T_c)$$

Burada:
* $f_c$: Başlangıç taşıyıcı frekansı (örneğin $77\text{ GHz}$),
* $T_c$: Chirp süresi (örneğin $40\text{ \mu s}$),
* $B$: Chirp bant genişliği (örneğin $4\text{ GHz}$ — $77\text{ GHz}$ ile $81\text{ GHz}$ arası),
* $S$: Chirp eğimi / frekans rampası ($S = \frac{B}{T_c} = \frac{4\text{ GHz}}{40\text{ \mu s}} = 100\text{ MHz/\mu s} = 10^{14}\text{ Hz/s}$).

```text
Frekans
  ^
  |        TX Sinyali (Gönderilen)
f_c + B +      /             /
        |     /             /
        |    /             /    RX Sinyali (Gecikmeli Gelen)
        |   /  |<- tau ->| /
        |  /   |         |/
        | /    |         /
    f_c +/-----+--------/------------> Zaman
        |<-- T_c ------>|
```

Radardan $R$ kadar uzaklıktaki duran bir hedefe giden ve yansıyan dalganın gidiş-dönüş uçuş süresi ($\tau$):

$$\tau = \frac{2R}{c}$$

Hedef sinyali alıcıya ulaştığında, verici osilatörü çoktan frekansını artırmış durumdadır. Mikser girişinde karşılaştırılan iki sinyal arasındaki anlık frekans farkına **Vuru Frekansı (Beat Frequency - $f_b$)** denir:

$$f_b = S \cdot \tau = S \cdot \frac{2R}{c} = \frac{2 B R}{c T_c}$$

Bu denklem, mesafenin frekansa nasıl dönüştüğünün temel kanıtıdır. Mesafeyi doğrudan vuru frekansını ölçerek hesaplayabiliriz:

$$R = \frac{c T_c}{2 B} f_b = \frac{c}{2 S} f_b$$

### Somut Sayısal Örnek:
* $B = 4\text{ GHz}$, $T_c = 40\text{ \mu s}$, $S = 100\text{ MHz/\mu s}$.
* Hedef $R = 15\text{ metre}$ uzakta olsun.
* Uçuş süresi: $\tau = \frac{2 \times 15}{3 \times 10^8} = 100\text{ nanosaniye}$.
* Vuru frekansı: $f_b = S \cdot \tau = 10^{14}\text{ Hz/s} \times 100 \times 10^{-9}\text{ s} = 10\text{ MHz}$.

$100\text{ nanosaniyelik}$ bir zaman aralığını zaman domeninde doğrudan ölçmek için devasa bir osiloskop gerekirdi; oysa $10\text{ MHz}$'lik bir sinüs dalgasını $25\text{ MSPS}$'lik sıradan bir ADC ve mikroişlemci rahatlıkla sayısallaştırabilir!

### Mesafe Çözünürlüğü (Range Resolution):
İki farklı nesnenin radar tarafından tek bir leke olarak değil, iki ayrı hedef olarak ayırt edilebilmesi için frekans farklarının en az bir spektral çözünürlük ($\Delta f = 1/T_c$) kadar ayrışması gerekir:

$$f_{b2} - f_{b1} \ge \frac{1}{T_c} \implies \frac{2 B \Delta R}{c T_c} \ge \frac{1}{T_c}$$

Süreler sadeleştiğinde geriye radar fiziğinin en zarif formülü kalır:

$$\Delta R = \frac{c}{2B}$$

> ⚠️ **Mühendislik Çıkarımı:** Mesafe çözünürlüğü chirp süresine veya taşıyıcı frekansına bağlı **değildir**; yalnızca ve yalnızca **Bant Genişliğine ($B$)** bağlıdır.
> * $24\text{ GHz}$ ISM bandında kullanılabilen $200\text{ MHz}$ bant genişliği ile: $\Delta R = \frac{3 \times 10^8}{2 \times 200 \times 10^6} = 75\text{ cm}$.
> * $77-81\text{ GHz}$ bandındaki $4\text{ GHz}$ bant genişliği ile: $\Delta R = \frac{3 \times 10^8}{2 \times 4 \times 10^9} = \mathbf{3.75\text{ cm}}$!
> Otomotiv radar endüstrisinin 24 GHz'i terk edip 77 GHz bandına göç etmesinin birinci temel nedeni budur.
{: .prompt-warning }

---

## 4. Faz Bilgisi, Doppler Kayması ve Hız Kestirimi

Bir hedef hareket ediyorsa, yansıyan sinyalin frekansı **Doppler Etkisi** ile kayar:

$$f_d = \frac{2v}{\lambda} = \frac{2v f_c}{c}$$

$77\text{ GHz}$ bandında ($\lambda \approx 3.9\text{ mm}$), $100\text{ km/h}$ ($27.8\text{ m/s}$) hızla giden bir aracın oluşturduğu Doppler frekansı:

$$f_d = \frac{2 \times 27.8}{0.0039} \approx 14.25\text{ kHz}$$

$14.25\text{ kHz}$'lik bir Doppler kayması, $10\text{ MHz}$ mertebesindeki vuru frekansının ($f_b$) yanında ihmal edilebilecek kadar küçük bir kaymadır ve tek bir chirp içinde doğrudan frekans analiziyle ayırt edilemez. 

Peki modern FMCW radarlar hızı milimetrik hassasiyetle nasıl ölçer? Cevap, vuru sinyalinin **Fazında ($\phi$)** gizlidir.

### Fazın Hassasiyeti:
Mikserden çıkan vuru sinyalinin matematiksel ifadesi tam olarak şöyledir:

$$s_{IF}(t) = A \cos\left( 2\pi f_b t + \phi_0 \right)$$

Buradaki faz terimi ($\phi_0$), dalganın hedefe ulaşıp döndüğü mesafeyle doğrudan ilişkilidir:

$$\phi_0 = 2\pi f_c \tau = 2\pi f_c \left(\frac{2R}{c}\right) = \frac{4\pi R}{\lambda}$$

Taşıyıcı dalga boyumuz $\lambda = 3.9\text{ mm}$ olduğuna göre, hedef masadan yalnızca **$1\text{ mm}$** uzaklaşsa bile faz değeri:

$$\Delta \phi = \frac{4\pi \times 1\text{ mm}}{3.9\text{ mm}} \approx 3.22\text{ radyan} \approx 185^\circ$$

Yani hedefteki mikroskobik bir mesafe değişimi, vuru sinyalinin başlangıç fazında devasa bir açısal dönüş yaratır!

### Ardışık Chirp (Chirp Frame) Yöntemi:
Hızı ölçmek için radar, aralarında $T_c$ kadar süre bulunan $N$ adet ardışık chirp gönderir (bir chirp karesi / burst).

```text
Chirp 1         Chirp 2         Chirp 3                 Chirp N
  /               /               /                       /
 /               /               /                       /
/               /               /                       /
|<-   T_c   ->| |<-   T_c   ->| |<-   T_c   ->| ...
```

İki ardışık chirp arasında hedef $v$ hızıyla $\Delta R = v \cdot T_c$ kadar hareket eder. Bu küçük hareket, ardışık chirplerin FFT tepe noktalarındaki vuru sinyali arasında şu faz farkını ($\Delta \phi$) doğurur:

$$\Delta \phi = \frac{4\pi \Delta R}{\lambda} = \frac{4\pi v T_c}{\lambda}$$

Bu faz farkını ölçtüğümüz anda hedefin bağıl hızını doğrudan türetiriz:

$$v = \frac{\lambda \cdot \Delta \phi}{4\pi T_c}$$

### Hız Belirsizliği ve Çözünürlüğü:
Faz açısının belirsizliğe düşmeden ($-\pi < \Delta \phi < +\pi$) ölçülebilmesi için ölçülebilecek maksimum net hız (**Maximum Unambiguous Velocity**):

$$v_{max} = \frac{\lambda}{4 T_c}$$

$N$ adet chirp içeren bir çerçevenin hız çözünürlüğü ($\Delta v$) ise toplam çerçeve süresine ($N \cdot T_c$) bağlıdır:

$$\Delta v = \frac{\lambda}{2 N T_c}$$

---

## 5. İki Boyutlu Fourier Dönüşümü (2D-FFT): Range-Doppler Haritası

Bir FMCW radarın işlemci çekirdeğinde (DSP, FPGA veya donanım hızlandırıcılı MCU) ham ADC verisi iki boyutlu bir tensör (Radar Data Cube) olarak yapılandırılır:

```text
               Yavaş Zaman (Slow Time) — Chirp İndeksi (0, 1, ..., N-1)
              +--------------------------------------------------------+
              | Chirp 0 | Chirp 1 | Chirp 2 | ... | Chirp N-1          |
     +--------+---------+---------+---------+-----+--------------------+
Hızlı| Örnek 0|         |         |         |     |                    |
Zaman| Örnek 1|         |         |         |     |                    |
     | Örnek 2|         |         |         |     |                    |
(Fast| ...    |         |         |         |     |                    |
Time)| Örnek M|         |         |         |     |                    |
     +--------+---------+---------+---------+-----+--------------------+
```

Sinyal işleme zinciri iki adımdan oluşur:

### 1. Adım: Hızlı Zaman FFT (Range FFT)
Matrisin her bir sütununa (tek bir chirpten okunan $M$ adet ADC örneğine) Pencereleme (Hanning/Blackman) ve 1D-FFT uygulanır. 
* Çıktı: Zaman domeni sinyali frekans domenine taşınır.
* Her bir tepe noktası, hedefin radara olan **Mesafesini (Range Bin)** verir.

### 2. Adım: Yavaş Zaman FFT (Doppler FFT)
Range FFT sonucunda belirli bir mesafe kutusunda bir hedef yakalanmışsa, o satır boyunca (tüm $N$ chirp boyunca) faz değişimini izlemek için satır yönünde ikinci bir 1D-FFT uygulanır.
* Çıktı: Fazın dönüş hızı frekansa dönüştürülür.
* Elde edilen spektral tepe, hedefin **Radyal Hızını (Doppler Bin)** verir.

Bu iki dönüşümün matris çarpımı bize radar mühendisliğinin en ikonik çıktısını sunar: **Range-Doppler Haritası (Range-Doppler Heatmap)**.

```text
Doppler (Hız) [m/s]
     ^
 +30 |             . (Yaklaşan Araç)
     |
   0 |-------------------+------- (Duran Bariyerler / Clutter)
     |                   . (Uzaklaşan Yaya)
 -30 |
     +-----------------------------------> Mesafe (Range) [m]
     0        20        40        60
```

### CFAR Algoritması: Gürültü İçinden Hedef Avlamak
Range-Doppler matrisindeki her parlak pik gerçek bir hedef midir? Hayır. Zemin yansımaları, yağmur damlaları ve termal gürültü sürekli sahte hedefler üretir. 

Sabit bir eşik değeri (threshold) kullanmak gürültülü ortamlarda sistemi kör eder. Bu yüzden **CFAR (Constant False Alarm Rate — Sabit Yanlış Alarm Oranı)** algoritmaları kullanılır:
* **CA-CFAR (Cell Averaging):** Test edilen hücrenin (Cell Under Test - CUT) etrafındaki referans hücrelerin ortalama gücü hesaplanır, araya koruma hücreleri (Guard Cells) konularak hedef enerjisinin gürültü hesabına sızması engellenir. Dinamik bir eşik hesaplanır: $T = \alpha \cdot P_{noise}$.
* **OS-CFAR (Ordered Statistic):** Çoklu hedeflerin yan yana olduğu karmaşık senaryolarda ortalama almak yerine referans hücreler sıralanır ($k$. en büyük değer seçilir); böylece komşu bir hedefin diğerini maskelemesi önlenir.

---

## 6. Milimetre-Dalga (77 GHz) RF ve Donanım Tasarım Zorlukları

Kağıt üzerinde veya MATLAB simülasyonunda kusursuz çalışan bu matematik, devre kartı (PCB) üzerine döküldüğünde elektromanyetik fiziğin acımasız gerçekleriyle karşılaşır. $77\text{ GHz}$'de dalga boyu yalnızca **$3.9\text{ milimetredir}$**. Bu ölçekte PCB üzerindeki her lehim damlası, her köşe ve her bakır pürüzlülüğü birer rezonatör veya anten gibi davranmaya başlar.

### 1. FR-4 Neden Kullanılamaz?
Standart elektronik kartlarda kullanılan ucuz **FR-4** laminatı, $77\text{ GHz}$ için tam bir felakettir:
* **Kayıp Tanjantı ($\tan\delta$):** FR-4'ün kayıp faktörü $\approx 0.02$ civarındadır. $77\text{ GHz}$'de sinyal birkaç santimetre ilerleyemeden dielektrik malzemenin içinde ısıya dönüşerek tamamen yok olur.
* **Dielektrik Sabiti Kararsızlığı ($\varepsilon_r$):** FR-4 cam elyaf ve epoksi örgüsünden oluşur. Liflerin yoğun olduğu yer ile reçinenin olduğu yer arasında dielektrik sabiti homojen değildir. Bu durum, mikrodalga hatlarında faz kaymalarına ve empedans uyumsuzluklarına yol açar.

Bu nedenle mmWave radarlarda **Rogers RO3003, RO4835** veya **Isola Astra MT77** gibi seramik dolgulu PTFE özel laminatlar ($\tan\delta \approx 0.0013$) kullanılır. Maliyeti düşürmek için genellikle üst katmanı Rogers, alt besleme ve dijital katmanları standart FR-4 olan **Hibrit Çok Katmanlı (Hybrid Stack-up)** PCB mimarileri tercih edilir.

### 2. Deri Etkisi (Skin Effect) ve Bakır Yüzey Pürüzlülüğü
Frekans arttıkça elektronlar iletkenin merkezinden kaçarak dış yüzeye doğru yığılır (**Skin Effect**). $77\text{ GHz}$ frekansta bakır içindeki deri derinliği ($\delta$):

$$\delta = \sqrt{\frac{\rho}{\pi f \mu}} \approx 0.23\text{ mikrometre } (230\text{ nm})$$

İletkenin akım taşıyan derinliği bir mikrometrenin çeyreğine indiğinde, bakır folyonun mikroskobik yüzey pürüzlülüğü ($R_q$) akımın yolunu uzatır ve RF direncini dramatik biçimde artırır. Bu frekansta standart bakır folyolar yerine elektrolitik olarak parlatılmış ultra-düşük profilli bakırlar (VLP / HVLP) kullanılmak zorundadır.

### 3. TX-RX İzolasyonu ve Sızıntı (Leakage)
Verici anten sürekli olarak $+12\text{ dBm}$ güç basarken, hemen birkaç milimetre yanındaki alıcı anten $-100\text{ dBm}$ gücündeki zayıf yankıyı dinlemeye çalışır. PCB yüzeyinden sızan yüzey dalgaları (Surface Waves), LNA girişini doyuma (saturation) sokabilir. Bu sızıntıyı kesmek için anten dizilerinin arasına **SIW (Substrate Integrated Waveguide)** çitleri ve toprak körükleri (Via Fences) dikilir.

### 4. Açı Kestirimi (Angle of Arrival - AoA) ve MIMO Mimarisi
Radar sadece mesafeyi ve hızı değil, hedefin yataydaki açısını ($\theta$) da bilmek zorundadır. Tek bir alıcı anten açıyı göremez. Aralarında $d = \lambda/2$ mesafe bulunan iki RX anteni kullanıldığında, dalganın antenlere varış yolu arasındaki geometrik fark bir faz farkı ($\Delta \phi_{spatial}$) doğurur:

$$\Delta \phi_{spatial} = \frac{2\pi d \sin\theta}{\lambda} = \frac{2\pi (\lambda/2) \sin\theta}{\lambda} = \pi \sin\theta$$

$$\theta = \arcsin\left( \frac{\Delta \phi_{spatial}}{\pi} \right)$$

Modern radarlar, fiziksel olarak yüzlerce anten döşemek yerine **MIMO (Multiple-Input Multiple-Output)** teknolojisini kullanır. $N_{TX}$ adet verici ve $N_{RX}$ adet alıcı anten zaman, frekans veya kod bölmeli olarak modüle edildiğinde, sinyal işlemci katmanında $N_{TX} \times N_{RX}$ boyutunda **Sanal bir Anten Dizisi (Virtual Antenna Array)** oluşturulur. Örneğin 3 TX ve 4 RX antenine sahip kompakt bir çip, sanki kartın üzerinde 12 kanallı devasa bir faz dizili radar varmış gibi yüksek açısal çözünürlük sunar.

---

## 7. Özet: Geometriye Dönüşen Dalgalar

FMCW radar mimarisi, insan zekasının fiziksel sınırlara karşı geliştirdiği en zarif çözümlerden biridir. 

Işık hızındaki dalgaların nanosaniyelik uçuş sürelerini pahalı donanımlarla kovalamak yerine, frekansı doğrusal olarak rampalayarak zamanı frekansa kodlarız. Mikserin analog trigonometrik çarpımıyla bu farkı duyulabilir vuru frekanslarına indirger, iki boyutlu Fourier dönüşümüyle mesafeyi ve hızı birbirinden ayırır, milimetre-dalga PCB sanatıyla da bu dalgaları silikonun içine hapsederiz.

Karanlıkta, yoğun siste, şiddetli fırtınada kameraların kör olduğu ve LiDAR sensörlerinin saçıldığı anlarda bir otonom aracın veya İHA'nın dünyayı kusursuz bir netlikle görmeye devam etmesinin arkasındaki sır; işte bu görünmez dalgaların kusursuz geometrisidir.

---

**Yazar:** Kerem Uysal — *Kurucu, Tech&Lab*  
**İletişim:** [info@keremuysal.com](mailto:info@keremuysal.com) | [info@techandlab.com](mailto:info@techandlab.com)