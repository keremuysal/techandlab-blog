---
layout: post
title: "Kısıtların Estetiği: 2 Dolarlık Mikrodenetleyicilerde Yapay Zeka (TinyML)"
author: "Kerem Uysal"
date: 2026-10-15 18:00:00 +0300
categories: [Gömülü Sistemler, Yapay Zeka]
tags: [tinyml, arm-cortex, gomulu-sistemler, edge-ai, optimizasyon]
lang: tr-TR
ref: tinyml-kisitlarin-estetigi
math: true
description: "Bulut sunucularının devasa kaynak tüketimine karşı, 64 KB SRAM ve 64 MHz ARM Cortex-M4 mikrodenetleyiciler üzerinde kuantizasyon, TFLite Micro ve CMSIS-NN ile deterministik yapay zeka mimarisi kurmanın mühendislik disiplini."
---

> 🌐 **Dil / Language / Sprache:** **Türkçe** | [English](/posts/aesthetics-of-constraints-tinyml-en/) | [Deutsch](/posts/aesthetik-der-beschraenkungen-tinyml-de/)
{: .prompt-info }

Modern yazılım dünyası, son on yılda tehlikeli bir konfor alanına hapsoldu: **Sınırsız kaynak illüzyonu.**

Bir model beklenen hızda çalışmıyorsa bulut sağlayıcısından daha büyük bir GPU kümesi kiralamak, bellek yetmiyorsa RAM kapasitesini ikiye katlamak, birkaç satırlık soyutlama katmanının (abstraction layer) arkasına saklanıp gigabaytlarca ağırlığı (weights) VRAM’e yüklemek artık olağan kabul ediliyor. Milyarlarca transistörün saniyede trilyonlarca kez anahtarlama yaptığı veri merkezleri bize muazzam bir hesaplama gücü sunuyor; ancak bu bolluk, beraberinde mühendislik disiplinini körelten bir **mimari obezite** getirdi.

Peki ya prizden sınırsız güç çekemediğiniz, fiber optik hatlara bağlı olmadığınız, bir düğme pil ile yıllarca yaşamak zorunda olan gerçek fiziksel dünyada durum nedir? İşte **TinyML (Mikrodenetleyicilerde Yapay Zeka)** tam bu noktada, bulutun kibrine karşı uç noktanın (edge) zarafetini ve mühendislik disiplinini temsil ediyor.

---

## 1. Giriş: Bulutun Şişmanlığına Karşı Uç Noktanın Çevikliği

Bir fabrikadaki yüzlerce endüstriyel motorun titreşimini veya bir hastanın göğsündeki giyilebilir biyomedikal sensörün EKG/PPG sinyallerini saniyede binlerce kez (kHz mertebesinde) örneklediğinizi düşünün. Geleneksel "Bulut Merkezli IoT" yaklaşımı şunu söyler: *"Veriyi sensörden oku, paketle, Wi-Fi veya hücresel ağ üzerinden AWS'ye gönder, çıkarımı (inference) bulutta yap ve sonucu geri yolla."*

Kağıt üzerinde kusursuz görünen bu mimari, fizik yasalarıyla karşılaştığı ilk anda üç büyük duvara çarpar:

1. **Enerji Maliyeti (Bant Genişliği vs. Hesaplama):** Gömülü sistemlerde en pahalı eylem hesaplama yapmak değil, **radyo frekansı (RF) üzerinden veri göndermektir**. Kablosuz bir arayüzle (BLE, Wi-Fi, LoRa veya LTE-M) havaya 1 bitlik ham veri fırlatmanın harcadığı enerji (nanojoule/bit), yerel bir ARM Cortex-M çekirdeğinde binlerce çarpma-biriktirme (*Multiply-Accumulate - MAC*) işlemi yapmanın enerjisinden katbekat fazladır. Ham sensör verisini sürekli buluta pompalamak, pil ömrünü yıllardan saatlere indirir.
2. **Gecikme (Latency) ve Determinizm:** Kritik bir endüstriyel kesici motorda veya otonom bir mikro-sistemde milisaniyeler hayati önem taşır. Verinin ağ yığınından (TCP/IP) geçmesi, baz istasyonuna ulaşması, bulut sunucusunda kuyruğa girmesi ve geri dönmesi **stokastik (olasılıksal)** bir süreçtir. Oysa gömülü kontrol döngüleri **deterministik** (öngörülebilir ve kesin zamanlı) olmak zorundadır.
3. **Bant Genişliği ve Mahremiyet:** Saniyede 10 kHz ile örneklenen 3 eksenli bir ivmeölçer, günün %99.9’unda yalnızca "her şey normal" anlamına gelen gürültü üretir. Anlamsız veriyi taşımak için bant genişliği kiralamak mimari bir israftır. Üstelik biyomedikal veya savunma/endüstriyel verinin cihaz dışına hiç çıkmadan, doğrudan kaynağında işlenip imha edilmesi en üst düzey güvenlik protokolüdür.

> 💡 **Mühendislik İlkesi:** Ham veriyi bilgiye dönüştürmek için binlerce kilometre ötedeki veri merkezine taşımayın; zekayı sensörün tam dibine, silikonun fiziksel dünyayla temas ettiği uç noktaya indirin. Veri yerinde işlenmeli, buluta yalnızca "anomali tespit edildi" bayrağı (1 bayt) gönderilmelidir.
{: .prompt-tip }

---

## 2. Kısıtlar Dünyası: 64 KB RAM, 256 KB Flash ve 64 MHz Cortex-M4

Bulut tabanlı bir Büyük Dil Modeli (LLM) veya derin görüntü işleme ağı onlarca gigabayt bellek tüketirken, 2 dolarlık bir mikrodenetleyicide (örneğin bir STM32F4, nRF52840 veya benzeri bir ARM Cortex-M4F tabanlı MCU) elimizdeki bütün evren şundan ibarettir:

* **İşlemci:** 64 MHz saat hızı (Bulut sunucularından ~50–60 kat daha yavaş tek çekirdek).
* **Flash Bellek (Kalıcı Depolama):** 256 KB (Hem işletim kodu, hem sürücüler, hem de sinir ağının eğitilmiş ağırlıkları buraya sığmak zorunda).
* **SRAM (Çalışma Belleği):** 64 KB (Yığın, global değişkenler, sensör tamponları ve sinir ağının katmanlar arası aktivasyon tensörleri için tek alan).

Bu donanımda bir yapay sinir ağı nasıl nefes alır? Cevap, **Harvard mimarisinin** doğasını ve bellek hiyerarşisini son baytına kadar sömürmekte yatar.

Mikrodenetleyicilerde eğitilmiş modelin ağırlıkları (weights) ve sapma (bias) değerleri değişmez (immutable) olduğu için `const` olarak tanımlanır ve derleyici tarafından doğrudan **Flash bellekteki `.rodata` (Read-Only Data)** bölümüne yerleştirilir. Çalışma zamanında bu ağırlıklar RAM’e kopyalanmaz; işlemci bunları doğrudan Flash üzerinden (XIP - *eXecute-In-Place* / doğrudan adresleme ile) okur. Değerli olan **64 KB SRAM** ise yalnızca o an hesaplanan katmanın giriş ve çıkış aktivasyon matrisleri için dönüşümlü olarak (Ping-Pong tamponlama ile) kullanılır.

### Mikrodenetleyicide TinyML Bellek Haritası (Memory Map)

```text
+-------------------------------------------------------------------+
|                   FLASH BELLEK (256 KB - Kalıcı)                  |
+-------------------------------------------------------------------+
| 0x0800_0000 | Kesme Vektör Tablosu (ISR Vector Table)      [ ~1 KB] |
|-------------|-----------------------------------------------------|
| .text       | Uygulama Kodu + HAL + TFLite Micro Çekirdeği [~65 KB] |
|             | + CMSIS-NN Optimize Operatörler (SIMD/DSP)          |
|-------------|-----------------------------------------------------|
| .rodata     | Kuantize Model Ağırlıkları (INT8 FlatBuffer) [~90 KB] |
|             | (SRAM'e kopyalanmaz, doğrudan Flash'tan okunur)     |
+-------------------------------------------------------------------+

+-------------------------------------------------------------------+
|                    SRAM BELLEK (64 KB - Uçucu)                    |
+-------------------------------------------------------------------+
| 0x2000_0000 | .data & .bss (Global/Statik Değişkenler)     [ ~6 KB] |
|-------------|-----------------------------------------------------|
|             | DMA Sensör Çift Tamponu (Ping-Pong Buffer)   [ ~4 KB] |
|-------------|-----------------------------------------------------|
| STATİK      | +-------------------------------------------------+ |
| TENSOR      | | Katman Aktivasyonları (Tampon A <-> Tampon B)   | |
| ARENA       | | Çıkarım (Inference) Geçici Çizik Alanı (Scratch)| |
| [ 40 KB ]   | | TFLite Micro Çalışma Zamanı Graf Metaverisi     | |
|             | +-------------------------------------------------+ |
|-------------|-----------------------------------------------------|
| HEAP        | [ 0 KB - YASAKLANDI! Dinamik tahsis (malloc) yok ]  |
|-------------|-----------------------------------------------------|
| 0x2001_0000 | STACK (Yığın - Aşağı doğru büyür)            [~14 KB] |
+-------------------------------------------------------------------+
```

Yukarıdaki şemada görüldüğü üzere, $N$ katmanlı bir sinir ağında tüm katmanların çıktılarını aynı anda bellekte tutmayız. $L_1$ katmanının çıktısı `Tampon A`'dan okunup $L_2$ katmanı tarafından işlenirken sonuç `Tampon B`'ye yazılır; bir sonraki katmanda ise `Tampon A`'nın üzerine yazılarak bellek yeniden kullanılır.

---

## 3. Model Küçültme Sanatı: Quantization (Kuantizasyon) ve Pruning (Budama)

Masaüstü GPU'larda sinir ağları genellikle **32-bit Kayan Noktalı Sayılar (FP32 - IEEE 754)** ile eğitilir. Her bir ağırlık bellekte 4 bayt yer kaplar. 250.000 parametreli mütevazı bir model bile FP32 formatında tam **1 MB** yer tutar ki bu, 256 KB Flash belleğimizin 4 katıdır!

Modeli mikrodenetleyiciye sığdırmanın iki temel matematiksel silahı vardır: **Kuantizasyon (Quantization)** ve **Ağırlık Budama (Pruning)**.

### 3.1. FP32’den INT8’e İndirme Matematiği (Affine Quantization)

Bir sinir ağındaki ağırlıklar ve aktivasyonlar genellikle dar bir istatistiksel dağılım içinde kümelenir. $[-3.25, +3.18]$ aralığındaki sürekli (continuous) gerçek sayıları temsil etmek için $2^{32}$ farklı duruma ihtiyacımız yoktur. Bu aralığı $[-128, +127]$ arasındaki **8-bitlik işaretli tam sayılara (INT8)** doğrusal (afin) bir dönüşümle eşleyebiliriz:

$$q = \text{clip}\left( \text{round}\left( \frac{r}{S} \right) + Z, \, -128, \, 127 \right)$$

Burada:
* $r$: Gerçek (FP32) değer,
* $q$: Kuantize edilmiş 8-bit tam sayı (INT8) karşılığı,
* $S$ (**Scale**): Ölçekleme çarpanı (FP32 aralığının INT8 aralığına oranı: $S = \frac{r_{\max} - r_{\min}}{255}$),
* $Z$ (**Zero-Point**): Gerçek $0.0$ değerinin tam sayı uzayında tam olarak hangi değere karşılık geldiğini belirten sıfır noktası kaydırmasıdır (sıfır dolgulu — *zero-padding* — evrişim işlemlerinde kritik öneme sahiptir).

Geri dönüşüm (dequantization) denklemi ise şöyledir:

$$r \approx S \cdot (q - Z)$$

İki matrisin çarpımında (örneğin $Y = W \cdot X + B$), işlemci artık yavaş ve enerji oburu kayan nokta (FPU) komutlarını kullanmaz. Tüm matris çarpımı tamamen **8-bit tam sayılarla** yapılır, çarpım sonuçları taşmayı (overflow) önlemek için **32-bit tam sayı biriktiricide (INT32 accumulator)** toplanır ve katman sonunda sabit noktalı (fixed-point) bir ölçek çarpanı ve bit kaydırma (`>>`) işlemiyle tekrar INT8'e indirgenir.

> 🚀 **Kuantizasyonun Mühendislik Kazancı:** FP32'den INT8'e geçiş, modelin Flash ve RAM ayak izini **tam 4 kat (%75 oranında)** küçültür. Üstelik *Quantization-Aware Training (QAT - Kuantizasyon Farkındalıklı Eğitim)* uygulandığında, model eğitim sırasında yuvarlama gürültüsüne adapte olduğu için doğruluk (accuracy) kaybı genellikle **%1'in altında** kalır.
{: .prompt-tip }

### 3.2. Ağırlık Budama (Pruning): Beyindeki Sinaptik Temizlik

Eğitilmiş bir sinir ağındaki bağlantıların önemli bir kısmı sıfıra çok yakın ağırlıklara sahiptir ve nihai karara anlamlı bir katkı sunmaz. **Budama (Pruning)**, belirli bir eşik değerinin ($\epsilon$) altındaki ağırlıkların sıfırlanması veya tümüyle ağdan çıkarılması işlemidir.

Ancak gömülü sistemlerde kritik bir ayrım vardır:
* **Yapılandırılmamış Budama (Unstructured Pruning):** Matris içindeki rastgele tekil ağırlıkları sıfırlar. Sıkıştırıldığında (Gzip/LZ4) dosya boyutunu küçültür; fakat çalışma zamanında seyrek matris (sparse matrix) indekslerini çözmek işlemciye ek dallanma (`if/else`) ve bellek erişim yükü bindirir.
* **Yapılandırılmış Budama (Structured Pruning):** Tekil ağırlıkları değil, katkısı düşük olan **tüm bir evrişim filtresini (channel/filter)** veya nöronu bütünsel olarak siler. Böylece matris boyutları doğrudan küçülür ve mikrodenetleyicinin yapacağı MAC işlem sayısı doğrusal olarak azalır.

### Sayısal Temsil ve Donanım Maliyeti Karşılaştırması

| Metrik / Özellik | FP32 (32-bit Float) | FP16 (16-bit Half) | INT8 (8-bit Tam Sayı) |
| :--- | :--- | :--- | :--- |
| **Parametre Başına Bellek** | 4 Bayt (32 bit) | 2 Bayt (16 bit) | **1 Bayt (8 bit)** |
| **Bellek Tasarrufu** | Referans (1x) | %50 (2x küçülme) | **%75 (4x küçülme)** |
| **Aritmetik Birim** | FPU (Kayan Nokta Birimi) | FPU / Vektör | **ALU + DSP SIMD (Tam Sayı)** |
| **Saat Darbesi / MAC (Cortex-M4)** | ~1 - 3 Döngü | Donanım bağımlı | **0.5 Döngü (SIMD ile 2 MAC/döngü)** |
| **Tipik Doğruluk Kaybı (QAT ile)** | %0 (Referans) | <%0.1 | **%0.2 – %1.0** |
| **Enerji Tüketimi (Bağıl)** | Yüksek (1.0x) | Orta (~0.45x) | **Çok Düşük (~0.20x)** |

---

## 4. Yazılım Yığını: TFLite Micro ve ARM CMSIS-NN Optimizasyonları

Bir modeli Python ortamında küçültmek işin yalnızca yarısıdır. Onu işletim sistemi bile olmayan (*bare-metal*) veya FreeRTOS/Zephyr üzerinde koşan bir mikrodenetleyicide çalıştırmak için standart kütüphanelerden çok farklı bir yazılım yığınına ihtiyaç duyarız: **TensorFlow Lite for Microcontrollers (TFLite Micro)** ve **ARM CMSIS-NN**.

### 4.1. Neden Dinamik Bellek (`malloc`) Kesinlikle Yasaktır?

Masaüstü yazılımlarda veya üst seviye dillerde çalışma zamanında `malloc()` veya `new` ile bellekten yer istemek sıradan bir reflekstir. Ancak 64 KB RAM’e sahip, 7/24 kesintisiz yıllarca çalışması gereken endüstriyel veya medikal bir gömülü sistemde dinamik bellek tahsisi (**Heap kullanımı**) mühendislik intiharıdır. Neden?

1. **Bellek Parçalanması (Heap Fragmentation):** Farklı boyutlardaki tensörler sürekli tahsis edilip serbest bırakıldıkça (`malloc`/`free`), RAM üzerinde İsviçre peyniri gibi küçük boşluklar oluşur. Toplamda 15 KB boş yeriniz olsa bile, ardışık (contiguous) 8 KB blok bulunamadığı anda sistem `NULL` döner ve çöker.
2. **Deterministik Olmayan Çalışma Süresi:** `malloc` fonksiyonunun uygun bir bellek bloğu bulmak için serbest liste (free list) üzerinde ne kadar süre arama yapacağı önceden bilinemez. Gerçek zamanlı (Real-Time) bir sistemde zamanlama garantisini bozar.
3. **Stack-Heap Çarpışması:** Dinamik bellek yukarı doğru büyürken çağrı yığını (Stack) aşağı doğru büyür. Sessizce birbirlerinin üzerine yazdıkları an, tespit edilmesi en zor donanım hataları (*HardFault*) meydana gelir.

> ⚠️ **Altın Kural (MISRA-C & Güvenlik Kritik Sistemler):** Gömülü yapay zekada tüm bellek yerleşimi **derleme zamanında (compile-time)** veya sistemin ilk açılış anında (**statik olarak**) kesinleşmelidir. Çalışma zamanında (runtime) tek bir bayt bile dinamik olarak tahsis edilmez.
{: .prompt-warning }

Helmuth von Moltke'nin askeri stratejide vurguladığı gibi; kaynakların ilk yığınağı (*Aufmarsch*) tamamen deterministik olmak zorundadır, çünkü başlangıçtaki yerleşimde yapılacak bir hata operasyon boyunca bir daha telafi edilemez. Gömülü yazılımda bu deterministik hazırlığın karşılığı **Static Tensor Arena** yapısıdır:

```cpp
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "model_data.h" // Flash (.rodata) içindeki FlatBuffer dizisi

// Dinamik bellek (malloc) YOK! 
// Derleme zamanında BSS/Data bölgesinde ayrılmış, 16-bayt hizalı statik havuz:
constexpr int kTensorArenaSize = 40 * 1024; // 40 KB
alignas(16) static uint8_t tensor_arena[kTensorArenaSize];

void tinyml_init() {
    // 1. Modeli doğrudan Flash bellekten (Zero-Copy) haritalandır
    const tflite::Model* model = tflite::GetModel(g_anomaly_model_data);

    // 2. Sadece modelin kullandığı operatörleri bağla (Flash tasarrufu)
    static tflite::MicroMutableOpResolver<4> resolver;
    resolver.AddFullyConnected();
    resolver.AddConv2D();
    resolver.AddRelu();
    resolver.AddQuantize();

    // 3. Statik Tensor Arena üzerinde yorumlayıcıyı (Interpreter) kur
    static tflite::MicroInterpreter static_interpreter(
        model, resolver, tensor_arena, kTensorArenaSize);
    
    // Tensör adreslerini arena içinde deterministik olarak yerleştir
    TfLiteStatus allocate_status = static_interpreter.AllocateTensors();
    if (allocate_status != kTfLiteOk) {
        // Arena boyutu yetersizse daha ilk açılışta (boot) yakalanır!
        Error_Handler();
    }
}
```

Dikkat ederseniz `MicroMutableOpResolver<4>` kullanarak kütüphanedeki yüzlerce operatörü değil, yalnızca modelimizin ihtiyaç duyduğu 4 operatörü ikili dosyaya (binary) dahil ediyoruz. Bu sayede `linker`, kullanılmayan tüm operatör kodlarını (dead-code stripping) Flash bellekten atıyor.

### 4.2. ARM CMSIS-NN: Silikonun Damarlarındaki SIMD Gücü

TFLite Micro referans çekirdekleri taşınabilir C++ ile yazılmıştır, ancak ARM Cortex-M4/M7/M33/M55 işlemcilerde donanımın gerçek gücünü ortaya çıkaran katman **CMSIS-NN** kütüphanesidir.

Cortex-M4 işlemcilerde bulunan DSP uzantıları, **SIMD (Single Instruction, Multiple Data - Tek Komut, Çoklu Veri)** yeteneği sunar. 32-bitlik standart bir işlemci yazmacına (register) dört adet 8-bitlik (`INT8`) ağırlık veya aktivasyon paketlenebilir.

Standart bir C döngüsünde her çarpma-toplama ayrı birer komutken, CMSIS-NN; verileri 16-bit'e genişletip **`__SMLAD` (Signed Multiply Accumulate Dual)** komutunu kullanarak **tek bir saat darbesinde (single clock cycle) iki adet 16-bitlik çarpma ve bir adet 32-bitlik biriktirme** işlemini paralel olarak gerçekleştirir. Bu donanım seviyesi optimizasyon, saf C koduna kıyasla evrişim (Convolution) ve tam bağlantılı (Fully Connected) katmanlarda **4x ile 5x arasında doğrudan hızlanma** ve aynı oranda enerji tasarrufu sağlar.

---

## 5. Gerçek Dünya Uygulaması: Kestirimci Bakım ve Biyomedikal Anomali Tespiti

Kısıtlı donanımda yapay zeka kurmanın en büyük sırrı, **her şeyi sinir ağına yıkmamaktır.** İyi bir gömülü sistem mühendisi, klasik **Dijital Sinyal İşleme (DSP)** ile **TinyML**'i hibrit bir boru hattında (pipeline) birleştirir.

İster endüstriyel bir asenkron motorda rulman aşınmasını yakalayan bir **kestirimci bakım (predictive maintenance)** düğümü, isterse giyilebilir bir sensörde **EKG aritmi / biyomedikal anomali** takip sistemi tasarlayın; uçtan uca deterministik mimari şu 4 aşamayla kurulur:

```text
[MEMS Sensör / AFE] --(SPI/I2C + DMA)--> [Ping-Pong RAM Tamponu]
                                                  |
                                       (CMSIS-DSP: Pencereleme + FFT)
                                                  v
[Anomali Bayrağı / Kesme] <--(Eşik: ||x - x̂|| > τ)-- [TFLite Micro: INT8 Autoencoder]
```

### Adım 1: Sıfır İşlemci Yüküyle Veri Toplama (DMA + Ping-Pong Buffer)
Üç eksenli bir MEMS ivmeölçerden (veya biyopotansiyel AFE çipinden) saniyede $1.6\text{ kHz}$ hızında veri okurken işlemciyi `while` döngüsünde bekletmek (polling) kabul edilemez. **DMA (Direct Memory Access)** birimi yapılandırılarak sensörden gelen baytlar işlemci çekirdeği **Derin Uyku (Sleep Mode)** halindeyken doğrudan SRAM’deki `Buffer_A` dizisine yazılır. `Buffer_A` dolduğu anda DMA donanımı otomatik olarak `Buffer_B`’ye geçer ve işlemciyi bir kesme (interrupt) ile uyandırır.

### Adım 2: Fiziksel Alan Bilgisini Modele Enjekte Etmek (CMSIS-DSP)
Ham zaman serisi verisini (örneğin 1024 örnek) doğrudan sinir ağına vermek, ağın frekans bileşenlerini öğrenmek için çok daha fazla katmana ihtiyaç duymasına yol açar. Bunun yerine, `CMSIS-DSP` kütüphanesinin donanım hızlandırmalı **RFFT (Real Fast Fourier Transform)** fonksiyonuyla sinyali milisaniyeler içinde frekans uzayına geçirir, **Hanning pencerelemesi** uygular ve enerji spektrumunu 32 veya 64 frekans kutusuna (bin) sıkıştırırız. Böylece sinir ağının giriş boyutu 1024'ten 32'ye düşer!

### Adım 3: INT8 Kuantize Otokodlayıcı (Autoencoder) ile Anomali Yakalama
Gerçek dünyada bir motorun veya kalbin nasıl bozulacağına dair elimizde binlerce "arızalı" veri seti bulunmaz; ancak sistemin "sağlıklı" çalışma rejimi bolca mevcuttur. Bu nedenle $32 \rightarrow 16 \rightarrow 8 \rightarrow 16 \rightarrow 32$ mimarisine sahip darboğazlı bir **Otokodlayıcı (Autoencoder)** yalnızca sağlıklı frekans spektrumlarını yeniden üretmek (reconstruct) üzere eğitilir ve INT8 olarak kuantize edilir (~4 KB Flash boyutu).

Çalışma zamanında giriş vektörü $x$ ile modelin yeniden ürettiği $\hat{x}$ arasındaki Ortalama Mutlak Hata ($L_1$ Rekonstrüksiyon Hatası) tam sayı uzayında hesaplanır:

$$E = \frac{1}{N} \sum_{i=1}^{N} |x_i - \hat{x}_i|$$

* Motor sağlıklı çalışırken model spektrumu tanıdığı için $E < \tau$ (hata düşüktür).
* Rulmanda mikroskobik bir çatlak başladığında veya biyomedikal sinyalde anormal bir ritim belirdiğinde, frekans harmonikleri değişir. Model bu yeni dağılımı sıkıştırıp geri açamaz ve $E > \tau$ (anomali eşiği aşılır).

### Adım 4: Enerji Bütçesi ve Sonuç
64 MHz'lik bir Cortex-M4 üzerinde 1024 noktalı FFT ve ardından çalışan INT8 Autoencoder çıkarımı toplamda **yalnızca ~2.4 milisaniye** sürer. İşlemci zamanının %97’sinde uyku modunda ($\mu\text{A}$ seviyesinde) kalır, sadece anomali eşiği ($\tau$) aşıldığında radyo modülünü uyandırıp merkeze birkaç baytlık teşhis paketini iletir.

---

## 6. Sonuç: Gerçek Mühendislik Kısıtlar Altında Parıldar

Bugün yapay zeka dünyasında popüler olan akım, daha fazla veri merkezi kurmak, nükleer santrallerle beslenen gigawatt'lık GPU çiftliklerinde trilyon parametreli modeller koşturmaktır. Elbette bu devasa modellerin yeri ayrıdır; ancak sınırsız bellek ve sınırsız enerjiyle çalışan bir sistemde verimsizliği gizlemek çok kolaydır.

64 KB SRAM içinde ise saklanacak hiçbir yer yoktur.

Orada her bir baytın, her bir bellek hizalamasının (`alignas(16)`), her bir saat darbesinin ve her bir mikroamperin hesabını vermek zorundasınızdır. İnanılmaz derecede karmaşık bir matematiksel modeli alıp; onu doğruluktan ödün vermeden 4 kat sıkıştırmak, dinamik bellek kullanmadan statik bir arenaya kilitlemek ve 2 dolarlık bir silikon parçası üzerinde milisaniyeler içinde deterministik olarak koşturmak... İşte **kısıtların estetiği** tam olarak budur.

Unutmayalım ki; sınırsız kaynaklarla herkes bir şeyler inşa edebilir, ancak **gerçek mühendislik sert fiziksel kısıtlar altında parıldar.**

---

**Yazar:** Kerem Uysal — *Kurucu, Tech&Lab*  
**İletişim:** [info@keremuysal.com](mailto:info@keremuysal.com) | [info@techandlab.com](mailto:info@techandlab.com)