---
layout: post
title: "AlphaFold ve Protein Katlanması: Biyolojinin 'Transistör' Anı mı?"
date: 2026-10-15 17:50:00 +0300
author: "Kerem Uysal"
categories: [Biyomühendislik, Yapay Zeka]
tags: [alphafold, protein-folding, levinthal-paradox, derin-ogrenme, biyomühendislik]
lang: tr-TR
ref: alphafold-transistor-ani
math: true
description: "Levinthal Paradoksu'ndan Evoformer ve Difüzyon mimarilerine, pLDDT/PAE metriklerinden de novo enzim tasarımına: AlphaFold'un biyomühendislikte başlattığı yapısal devrimin teknik analizi."
---

> 🌐 **Dil / Language / Sprache:** **Türkçe** | [English](/posts/alphafold-and-protein-folding-biologys-transistor-moment-en/) | [Deutsch](/posts/alphafold-und-proteinfaltung-der-transistor-moment-der-biologie-de/)
{: .prompt-info }

Selamlar, Tech&Lab'deki ilk teknik incelemelerimizden biri olan [**"Transistör Savaşları"**](/posts/dijital-evrenin-temel-tasi/) yazımda, modern dijital uygarlığın insan saç telinden on binlerce kat küçük silikon anahtarlar üzerine nasıl inşa edildiğini incelemiştik. Örneğin bir Apple M4 işlemcisinin içinde yer alan 28 milyar adet transistör, saniyede trilyonlarca kez açılıp kapanarak dijital evrenin mantık kapılarını çalıştırır.

Peki ya karbon tabanlı biyolojik evrenin "transistörleri" nelerdir?

Canlılık dediğimiz o muazzam biyokimyasal fabrikayı ayakta tutan görünmez motorlar **proteinlerdir**. Ancak bir biyomühendis için protein, ders kitaplarındaki iki boyutlu harf dizilerinden (`M-A-L-W-M-R...`) ibaret değildir; uzayda atomik hassasiyetle katlanan, konformasyon değiştiren, elektron ve proton transfer eden üç boyutlu birer **nano-makinedir**. Yarım yüzyıl boyunca biyolojinin en büyük darboğazı, tek boyutlu genetik koddan bu üç boyutlu makine mimarisine nasıl geçildiğini hesaplayamamaktı.

Bugün, yarı iletken endüstrisinin 2 boyutlu düzlemsel yapıdan (Planar FET) 3 boyutlu FinFET ve GAAFET mimarilerine geçerek fiziksel sınırları aşmasına benzer bir kırılmayı, yapısal biyolojide **AlphaFold** ile yaşıyoruz.

---

## 1. Giriş: Şeklin Fonksiyonu Belirlediği Evren

Moleküler biyolojinin santral dogması bize bilginin $\text{DNA} \rightarrow \text{RNA} \rightarrow \text{Protein}$ yönünde aktığını söyler. Ribozomdan çıkan yeni sentezlenmiş bir polipeptit zinciri, 20 standart amino asidin peptit bağlarıyla uç uca eklendiği **1 boyutlu (1D)** doğrusal bir dizidir (primer yapı). Ancak bu zincir, hücre içi sulu ortamda (sitozol) kendi haline bırakıldığı an, saniyenin kesirleri içinde bükülür, alfa-sarmallar ($\alpha$-helices) ve beta-tabakalar ($\beta$-sheets) oluşturur ve nihayetinde **3 boyutlu (3D)** özgün bir topolojiye (tersiyer ve kuaterner yapı) kavuşur.

Biyomühendislikte temel aksiyom şudur: **Dizilim yapıyı, yapı ise fonksiyonu belirler.**

1960'larda Christian Anfinsen'in ribonükleaz A üzerinde yaptığı klasik denatürasyon/renatürasyon deneyleri (**Anfinsen'in Termodinamik Hipotezi**), bir proteinin üç boyutlu doğal konformasyonunun (*native state*), fizyolojik koşullar altında doğrudan amino asit dizisinde kodlandığını ve sistemin Gibbs serbest enerjisinin ($\Delta G$) global minimumuna karşılık geldiğini kanıtladı:

$$\Delta G = \Delta H - T\Delta S < 0$$

Bir enzimin substratını tanıyan katalitik cebi (aktif bölgesi), bir antikorun antijene kenetlenen CDR halkaları veya hücre zarındaki bir iyon kanalının seçici filtresi; tamamı Ångström ($1\text{ \AA} = 10^{-10}\text{ m}$) ölçeğindeki uzaysal koordinatların eseridir.

### Yanlış Katlanmanın Biyolojik Maliyeti
Kusursuz çalışan bir mikroişlemcide tek bir kapı oksidinin delinmesi ve kaçak akım oluşturması devreyi nasıl kilitlerse, proteinlerin yanlış katlanması (*protein misfolding*) da hücresel homeostazı öyle çökertir. 

Normal şartlarda proteinin iç çekirdeğine gömülmesi gereken hidrofobik (su sevmez) amino asit yan zincirleri, yanlış katlanma sonucu sitozole açıldığında termodinamik olarak kararsız bir durum oluşur. Bu açık hidrofobik yüzeyler birbirine yapışarak çözünmeyen, proteazlara dirençli **çapraz-beta ($\beta$-sheet) amiloid fibrilleri** oluşturur:

* **Alzheimer Hastalığı:** Amiloid-$\beta$ ($\text{A}\beta_{42}$) peptitlerinin hücre dışı plaklar ve hiperfosforile Tau proteinlerinin nörofibriler yumaklar halinde çökmesi.
* **Prion Hastalıkları (Creutzfeldt-Jakob, Deli Dana):** Genetik dizisi tamamen aynı olan normal hücresel prion proteininin ($\text{PrP}^\text{C}$, ağırlıklı olarak $\alpha$-sarmal yapıda), enfeksiyöz ve yanlış katlanmış izoforma ($\text{PrP}^\text{Sc}$, ağırlıklı olarak $\beta$-tabaka yapıda) dönüşmesi. Daha da çarpıcı olan, $\text{PrP}^\text{Sc}$'nin otokatalitik bir şablon gibi davranarak sağlıklı $\text{PrP}^\text{C}$ moleküllerini de kendi bozuk geometrisine zorlamasıdır.

---

## 2. Levinthal Paradoksu: $10^{300}$ Olasılık İçinde Doğru Adresi Bulmak

Madem 1D amino asit dizisi 3D yapıyı deterministik olarak kodluyor, neden kuantum mekaniği ve moleküler dinamik denklemlerini yazıp her proteinin şeklini kaba kuvvetle (*brute-force*) hesaplayamadık?

Bu sorunun cevabını 1969 yılında moleküler biyolog **Cyrus Levinthal** verdi.

Bir polipeptit ana zincirinde (*backbone*) peptit bağı ($\text{C}-\text{N}$, $\omega$ açısı) kısmi çift bağ karakteri nedeniyle düzlemsel ve rijittir ($\omega \approx 180^\circ$ veya $0^\circ$). Ancak her bir amino asit kalıntısının (rezidü) alfa karbonu ($\text{C}_\alpha$) etrafında dönebilen iki temel burulma (dihedral) açısı vardır:
1. $\phi$ (Phi): $\text{N}-\text{C}_\alpha$ bağı etrafındaki dönüş açısı.
2. $\psi$ (Psi): $\text{C}_\alpha-\text{C}$ bağı etrafındaki dönüş açısı.

Buna ek olarak yan zincirlerin ($\chi_1, \chi_2, \dots$) rotamerik serbestlik dereceleri mevcuttur. Ortalama büyüklükte bir proteinin $N = 100$ ila $300$ amino asitten oluştuğunu düşünelim:

* Çok iyimser ve ayrık (*discrete*) bir yaklaşımla, sadece $100$ amino asitlik küçük bir zincirde her bir bağın uzayda yalnızca $3$ kararlı konformasyon alabildiğini varsaysak bile, olası ana zincir durum sayısı $3^{198} \approx 10^{94}$ olur.
* Gerçekçi açısal çözünürlükler, yan zincir rotamerleri ve standart bir protein boyutu hesaba katıldığında, olası konformasyon uzayı astronomik bir sayıya, yaklaşık **$10^{300}$** olası duruma ulaşır (karşılaştırmak gerekirse, gözlemlenebilir evrendeki toplam atom sayısı $\approx 10^{80}$ civarındadır).

Şimdi bir düşünce deneyi yapalım: Bu proteinin doğru katlanmayı (*native state*) bulmak için tüm konformasyonları atomik titreşim frekansı hızında, yani pikosaniye/femtosaniye ($\tau \approx 10^{-13}\text{ s}$, saniyede $10^{13}$ deneme) mertebesinde rastgele taradığını varsayalım. Sadece $10^{100}$ durumu taramak için bile gereken süre:

$$t_{\text{arama}} = \frac{10^{100}\text{ konformasyon}}{10^{13}\text{ konformasyon/s}} = 10^{87}\text{ saniye} \approx 3.17 \times 10^{79}\text{ yıl}$$

Oysa evrenin yaşı yalnızca $\approx 13.8 \times 10^9\text{ yıl}$ ($\approx 4.35 \times 10^{17}\text{ saniye}$) kadardır.

İşte **Levinthal Paradoksu** budur: Rastgele (stokastik) bir aramayla evrenin yaşından katbekat uzun sürecek olan bu optimizasyon problemini, doğadaki ribozomdan çıkan bir protein **mikrosaniyeler ($\mu\text{s}$) ile milisaniyeler ($\text{ms}$)** içinde nasıl çözer?

### Doğanın Çözümü: Katlanma Hunisi (Folding Funnel)
Doğa, tüm durum uzayını rastgele taramaz. Biyofizikçiler bu süreci çok boyutlu bir **"Katlanma Hunisi" (*Folding Funnel*)** enerji peyzajı ile açıklar. Huninin geniş ağzı yüksek entropili ($\Delta S$ yüksek) ve yüksek serbest enerjili açılmış (*unfolded*) durumları temsil ederken; huninin dar tabanı tekil, düşük enerjili doğal konformasyonu temsil eder.

Polipeptit zinciri sentezlenir sentezlenmez yerel etkileşimler (hidrofobik çöküş, yerel hidrojen bağları ve elektrostatik tuz köprüleri) zinciri yerel enerji gradyanları boyunca aşağı doğru çeker. Tıpkı [Mareşal von Moltke'nin strateji teorisinde](/posts/moltke-strateji-uzerine-1871-tr/) belirttiği gibi; başlangıçtaki fizikokimyasal kurulum (*Aufmarsch*) deterministiktir, ancak katlanma operasyonu başladığında süreç yerel koşullara anlık uyum sağlayan stokastik bir "uyarlamalar sistemi" (*System der Aushülfen*) olarak huninin dibine doğru akar.

---

## 3. Geleneksel Yöntemlerin Sınırları: Islak Laboratuvarın Darboğazı

Algoritmik devrimden önce bir proteinin 3B atomik koordinatlarını (`PDB` formatında $x, y, z$ konumlarını) öğrenmek istiyorsanız, ıslak laboratuvarın (*wet-lab*) zahmetli, pahalı ve çoğu zaman kaprisli deneysel yöntemlerine mecburdunuz:

### 1. X-Işını Kristalografisi (X-Ray Crystallography)
Yapısal biyolojinin tarihsel altın standardıdır ancak mühendislik açısından devasa bir darboğaz barındırır: **Protein kristalizasyonu**.
* Hedef proteini rekombinant olarak (örneğin *E. coli* veya memeli hücrelerinde) yüksek saflıkta üretmeniz, afinite ve boyut dışlama kromatografisi (SEC) ile saflaştırmanız ve ardından milyonlarca protein molekülünü kusursuz bir 3B periyodik kristal örgü (*crystal lattice*) içine girmeye ikna etmeniz gerekir.
* Buhar difüzyonu (*hanging-drop / sitting-drop*) plakalarında yüzlerce farklı pH, tuz ve polietilen glikol (PEG) koşulu taranır. Ancak birçok protein —özellikle ilaç hedeflerinin yaklaşık %50'sini oluşturan hidrofobik **membran proteinleri (GPCR'ler, iyon kanalları)** ve esnek bölgelere sahip proteinler— doğaları gereği kristalleşmeyi reddeder.
* Kristal elde edilse bile senkrotron (synchrotron) X-ışını kırınımında dedektörler yalnızca kırınan dalgaların genliklerini ($I_{hkl} \propto \vert{}F_{hkl}\vert{}^2$) ölçebilir; elektron yoğunluk haritasını $\rho(x,y,z)$ ters Fourier dönüşümüyle hesaplamak için gereken faz açısı ($\alpha_{hkl}$) kaybolur (**Faz Problemi / Phase Problem**):

$$\rho(x,y,z) = \frac{1}{V} \sum_{h}\sum_{k}\sum_{l} \vert{}F_{hkl}\vert{} \cdot e^{i\alpha_{hkl}} \cdot e^{-2\pi i (hx + ky + lz)}$$

### 2. Kriyo-Elektron Mikroskobisi (Cryo-EM)
Kristalizasyon zorunluluğunu ortadan kaldırarak protein çözeltisini sıvı etan içinde hızla dondurma (**vitrifikasyon**) ve yüz binlerce 2B elektron mikroskobu projeksiyonundan 3B rekonstrüksiyon yapma prensibine dayanır. Büyük makromoleküler komplekslerde devrim yaratmış olsa da; milyonlarca dolarlık mikroskop altyapısı (Titan Krios vb.), devasa veri işleme kümeleri gerektirir ve düşük moleküler ağırlıklı küçük proteinlerde ($< 50\text{ kDa}$) düşük sinyal-gürültü oranı (SNR) nedeniyle fiziksel sınırlara çarpar.

### 3. NMR Spektroskopisi
Proteinleri çözelti içinde, fizyolojik dinamikleriyle inceleyebilir; ancak izotop etiketleme ($^{15}\text{N}$, $^{13}\text{C}$) maliyeti ve büyük moleküllerde spektral çakışma/yavaş moleküler dönme (*tumbling*) nedeniyle genellikle $30-40\text{ kDa}$ altındaki küçük proteinlerle sınırlıdır.

**Sonuç:** Tek bir proteinin yapısını deneysel olarak çözmek, ortalama bir doktora öğrencisinin **3 ila 5 yılına** ve laboratuvarın **yüz binlerce dolarlık bütçesine** mal oluyordu. Yeni nesil dizileme (NGS) teknolojileriyle yüz milyonlarca gen dizisi veritabanlarına akarken, Protein Data Bank (PDB) 50 yılda yalnızca ~200.000 deneysel yapı biriktirebilmişti. Dizi ile yapı arasındaki uçurum her geçen gün büyüyordu.

---

## 4. AlphaFold Mimarisi: Bir Biyomühendis Gözüyle Kaputun Altı

2020 yılındaki **CASP14** (*Critical Assessment of protein Structure Prediction*) yarışmasında DeepMind tarafından geliştirilen **AlphaFold 2**, deneysel yapılarla karşılaştırıldığında medyan $0.96\text{ \AA}$ ana zincir hatasına ($\text{RMSD}_{95}$) ulaşarak 50 yıllık problemi fiilen çözdü. Ardından 2024'te gelen **AlphaFold 3**, bu mimariyi tüm biyomoleküler evrene genişletti.

Popüler bilim basını bunu "yapay zeka mucizesi" olarak başlıklandırsa da, kaputun altında son derece zarif bir **geometrik derin öğrenme (geometric deep learning)** ve **evrimsel bilgi kuramı** mimarisi yatar.

```text
[1D Amino Asit Dizisi]
         │
         ├──► [Genetik Veritabanı Arama] ──► MSA Temsili (N_seq × N_res × c_m) ┐
         └──► [Eşlenik İlişkiler & Şablon] ──► Pair Temsili (N_res × N_res × c_z) ┼──► [EVOFORMER (48 Blok)]
                                                                               │     (Üçgen Dikkat & Güncelleme)
                                                                               ▼
                                                                    [STRUCTURE MODULE (8 Blok)]
                                                                   (SE(3) Invariant Point Attention)
                                                                               │
                                                                               ▼
                                                                  [3D Atomik Koordinatlar + pLDDT/PAE]
```

### AlphaFold 2: Evoformer, MSA ve Pair Representation

AlphaFold 2'nin temel başarısı, fizik tabanlı kuvvet alanlarını (*force fields*) simüle etmek yerine, doğanın milyarlarca yıllık evrimsel kayıtlarını geometrik kısıtlara dönüştürmesidir:

#### 1. Multiple Sequence Alignment (MSA - Çoklu Dizi Hizalaması) ve Birlikte Evrim (*Co-evolution*)
Hedef proteinin dizisi, farklı organizmalardaki homolog (akraba) protein dizileriyle hizalanarak $N_{\text{seq}} \times N_{\text{res}}$ boyutlu bir MSA matrisi oluşturulur. Buradaki biyomühendislik sezgisini şöyle açıklayalım:
Dizi üzerinde birbirinden çok uzakta olan $i$. ve $j$. amino asitler (örneğin 15. ve 180. rezidüler), 3B uzayda katlandıklarında birbirleriyle fiziksel temas halinde (örneğin bir tuz köprüsü kurmuş) olabilirler. Evrimsel süreçte 15. pozisyondaki negatif yüklü Glutamat ($\text{Glu}^{-}$), pozitif yüklü Lizin ($\text{Lys}^{+}$) amino asidine mutasyona uğrarsa protein kararsızlaşır; proteinin fonksiyonunu koruması için uzayda ona temas eden 180. pozisyondaki amino asidin de tamamlayıcı bir mutasyon geçirmesi gerekir. MSA matrisindeki bu **kovaryans (birlikte değişim)** sinyali, 3B uzaydaki temas haritasının (*contact map*) şifresidir.

#### 2. Pair Representation (Çift Temsili)
$N_{\text{res}} \times N_{\text{res}} \times c_z$ boyutundaki bu tensör, proteinin her $(i, j)$ amino asit çifti arasındaki uzaysal mesafe ve yönelim ilişkilerini tutan bir graf yapısı gibi çalışır.

#### 3. Evoformer Bloğu (48 Katmanlı Omurga)
AlphaFold 2'nin kalbi olan **Evoformer**, MSA temsili ile Pair temsili arasında sürekli bilgi alışverişi yapan çift kuleli bir Transformer mimarisidir. Ancak standart dil modellerindeki (LLM) 1B dikkat mekanizmasından farklı olarak Evoformer, Pair temsili üzerinde **Üçgen Dikkat Mekanizması (*Triangular Self-Attention*)** ve **Üçgen Çarpımsal Güncelleme (*Triangular Multiplicative Update*)** uygular.

Neden üçgen? Çünkü 3 boyutlu Öklid uzayında yer alan herhangi üç amino asit ($i, j, k$) arasındaki uzaklıklar **üçgen eşitsizliğini** sağlamak zorundadır:

$$d_{ij} \le d_{ik} + d_{kj}$$

Evoformer, $i$ ve $j$ arasındaki kenar bilgisini güncellerken tüm üçüncü $k$ düğümlerinden gelen $(i \rightarrow k)$ ve $(k \rightarrow j)$ kenarlarını hesaba katarak, 2B matrisin henüz 3B koordinatlara geçmeden fiziksel olarak tutarlı bir 3B geometriye yakınsamasını zorlar.

#### 4. Structure Module ve Invariant Point Attention (IPA)
Evoformer'dan çıkan rafine edilmiş temsiller **Structure Module**'e girer. Burada her bir amino asidin ana zincir atomları ($\text{N}-\text{C}_\alpha-\text{C}$), 3B uzayda bir öteleme vektörü ve döndürme matrisi ile tanımlanan bir **katı cisim çerçevesi (*rigid-body backbone frame*)** olarak modellenir:

$$T_i = (R_i, \vec{t}_i) \in SE(3), \quad R_i \in SO(3), \; \vec{t}_i \in \mathbb{R}^3$$

Başlangıçta tüm amino asit üçgenleri uzayın merkezinde $(0,0,0)$ noktasında üst üste yığılmış haldedir (*"black hole initialization"*). **Invariant Point Attention (IPA)** mekanizması, genel döndürme ve ötelemelere karşı değişmez (*SE(3)-equivariant*) bir dikkat hesabı yürüterek bu üçgenleri 8 adımda gerçek 3B uzaydaki yerlerine taşır, yan zincir $\chi_1, \chi_2, \chi_3, \chi_4$ burulma açılarını tahmin eder ve **FAPE (*Frame Aligned Point Error*)** kayıp fonksiyonu ile kiraliteyi (L-amino asit stereokimyasını) koruyarak yapıyı kilitler.

---

### AlphaFold 3: Difüzyon Tabanlı Yapıya Evrim

AlphaFold 2 yalnızca polipeptit zincirlerini tahmin etmek için tasarlanmıştı. Ancak hücre içinde proteinler yalnız çalışmaz; DNA ve RNA sarmallarına bağlanır, küçük molekül ilaçlarla (ligandlar) etkileşir, hem grubu veya $\text{Zn}^{2+}, \text{Mg}^{2+}$ gibi metal iyonlarını ko-faktör olarak kullanır ve glikozilasyon/fosforilasyon gibi post-translasyonel modifikasyonlara (PTM) uğrar.

AlphaFold 2'nin "amino asit üçgen çerçevesi ($SE(3)$ frame + torsion angles)" parametrizasyonu, standart 20 amino asit dışındaki keyfi kimyasal yapıların (örneğin aromatik halkalara sahip sentetik bir ilaç molekülünün) modellenmesine izin vermiyordu. **AlphaFold 3** bu sınırı yıkmak için mimaride iki radikal ameliyat gerçekleştirdi:

1. **Evoformer'dan Pairformer'a Geçiş:** MSA işleme bloğu 48 katmandan yalnızca 4 hafif katmana indirildi. Tüm hesaplama gücü, doğrudan tekil ve çift temsilleri (*single & pair representations*) üzerinde çalışan 48 blokluk **Pairformer** mimarisine aktarıldı.
2. **Structure Module'den Üretken Difüzyon Ağına (Diffusion Module) Geçiş:** Burulma açıları ve katı cisim üçgenleri tamamen terk edildi! Bunun yerine, görüntü üretim modellerinden (Stable Diffusion vb.) aşina olduğumuz **Denoising Diffusion Probabilistic Models (DDPM)** mimarisi doğrudan **3B atomik koordinatlar ($\vec{r} \in \mathbb{R}^{N_{\text{atom}} \times 3}$)** üzerine kuruldu.

$$\vec{x}_t = \sqrt{\alpha_t}\vec{x}_0 + \sqrt{1 - \alpha_t}\vec{\epsilon}, \quad \vec{\epsilon} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$$

AlphaFold 3'ün Difüzyon Modülü, uzaydaki rastgele bir Gauss atom bulutundan ($\vec{x}_T$) başlar ve iteratif gürültü giderme (*denoising*) adımlarıyla; büyük gürültü ölçeklerinde kompleksin global kuaterner topolojisini, küçük gürültü ölçeklerinde ise kovalent bağ uzunluklarını, stereokimyayı ve ligand-protein bağlanma cebi geometrisini doğrudan tüm atomlar (*all-atom*) düzeyinde inşa eder.

| Özellik / Kriter | Geleneksel Deneysel (X-Ray / Cryo-EM) | AlphaFold 2 (2020) | AlphaFold 3 (2024+) |
| :--- | :--- | :--- | :--- |
| **Temel Girdi** | Saflaştırılmış fiziksel protein örneği | 1D Amino Asit Dizisi (FASTA) | Protein, DNA, RNA, Ligand (SMILES/CCD), İyonlar |
| **Merkezi Omurga** | Kırınım Faz Çözümü / 3D Elektron Rekonstrüksiyonu | **Evoformer** (48 Blok, MSA + Pair) | **Pairformer** (48 Blok) + Hafif MSA (4 Blok) |
| **3B Yapı Üretim Mekanizması** | Elektron yoğunluk haritasına atomik model oturtma | **Structure Module** ($SE(3)$ IPA + Burulma Açıları) | **Diffusion Module** (Doğrudan Tüm-Atom 3B Koordinat Difüzyonu) |
| **Moleküler Kapsam** | Kristalleşebilen / görüntülenebilen tüm yapılar | Yalnızca standart polipeptit zincirleri (Monomer/Multimer) | Tüm biyomoleküler kompleksler + Kovalent PTM'ler |
| **Süre ve Maliyet** | Aylar / Yıllar — $10^4 - 10^6$ USD | Dakikalar — GPU hesaplama maliyeti | Dakikalar — GPU hesaplama maliyeti |

---

## 5. pLDDT ve PAE Metrikleri: Bir Biyomühendis AlphaFold Çıktısını Nasıl Okur?

Bir biyomühendisi sıradan bir kullanıcıdan ayıran en önemli nokta, ekranda dönen renkli 3B protein modeline körü körüne inanmamasıdır. Yapısal biyolojide model doğrulama yapmadan moleküler kenetlenme (*molecular docking*) veya mutasyon tasarımı yapmak, temel mukavemet hesabı yapmadan köprü inşa etmeye benzer.

AlphaFold, ürettiği her yapı ile birlikte kendi tahminine ne kadar güvendiğini iki kritik olasılıksal metrikle raporlar: **pLDDT** ve **PAE**.

### 1. pLDDT (*predicted Local Distance Difference Test*)
Her bir amino asit kalıntısı için $[0, 100]$ aralığında verilen **yerel (lokal) güven skorudur**. Modelin, bir rezidünün $\text{C}_\alpha$ atomu ile $15\text{ \AA}$ yarıçapı içindeki komşu atomlar arasındaki mesafeleri deneysel gerçeğe ne kadar yakın tahmin ettiğini (lDDT-$\text{C}_\alpha$) ölçer:

* **$\text{pLDDT} > 90$ (Koyu Mavi):** Çok yüksek doğruluk. Sadece ana zincir (*backbone*) değil, yan zincir (*side-chain* $\chi$) rotamer yönelimleri de atomik hassasiyettedir. Enzim aktif bölgesi analizi ve ilaç kenetlenme (*docking*) çalışmaları için doğrudan kullanılabilir.
* **$70 < \text{pLDDT} \le 90$ (Açık Mavi):** Güvenilir ana zincir tahmini. Proteinin genel katlanması (*fold*) doğrudur ancak yan zincir açılarında yerel esneklikler olabilir.
* **$50 < \text{pLDDT} \le 70$ (Sarı):** Düşük güven; ikincil yapı elemanları doğru olabilir ancak ilmek (*loop*) bölgeleri belirsizdir.
* **$\text{pLDDT} \le 50$ (Turuncu / Kırmızı):** Çok düşük güven.

> **Kritik Biyomühendislik Notu — İntrinsik Düzensiz Bölgeler (IDRs):**
> Bir AlphaFold modelinde $\text{pLDDT} < 50$ olan ve proteinin etrafında "spagetti şeritleri" gibi uzanan turuncu bölgeler gördüğünüzde, bu genellikle algoritmanın başarısızlığı **değildir**. Ökaryotik proteomun yaklaşık %30'u, fizyolojik koşullarda tek bir sabit 3B yapıya katlanmayan **İntrinsik Olarak Düzensiz Bölgeler (*Intrinsically Disordered Regions - IDRs*)** içerir. Bu bölgeler yüksek konformasyonel entropiye sahiptir ve ancak spesifik bir bağlanma partneriyle (örneğin bir kinaz, nükleik asit veya hedef reseptör) karşılaştığında katlanır (*coupled folding and binding*) ya da hücre içi sıvı-sıvı faz ayrışmasında (*LLPS*) görev alır.

### 2. PAE (*Predicted Aligned Error*)
pLDDT yerel doğruluğu ölçerken, **PAE ($N_{\text{res}} \times N_{\text{res}}$ boyutlu asimetrik ısı haritası)** proteinin farklı bölgelerinin birbirine göre **global uzaysal konumlanma doğruluğunu** Ångström ($\text{\AA}$) cinsinden ölçer.

Matristeki $(i, j)$ pikseli şu soruya yanıt verir: *"Eğer tahmin edilen yapıyı gerçek deneysel yapı üzerine $i$. amino asidin koordinatlarını referans alarak çakıştırırsam (hizalarsam), $j$. amino asidin konumu gerçek konumundan kaç Ångström sapar?"*

* **Köşegen Üzerindeki Koyu Yeşil Bloklar ($0-5\text{ \AA}$ hata):** Kendi içinde rijit ve kompakt bir şekilde katlanmış bağımsız **protein domainlerini (alanlarını)** gösterir.
* **Köşegen Dışı (*Off-Diagonal*) Bölgeler:**
  * Eğer iki farklı domain arasındaki kesişim bölgesi **koyu yeşil** ise (düşük PAE), bu iki domain birbirine fiziksel olarak kenetlenmiştir ve aralarındaki uzaysal oryantasyon kesindir.
  * Eğer iki domainin kendi iç blokları koyu yeşil (yüksek pLDDT), ancak kesişim bölgeleri **açık yeşil/beyaz** ise ($> 25\text{ \AA}$ PAE); her iki domain de kendi içinde doğru katlanmıştır fakat aralarında esnek bir menteşe/bağlayıcı (*flexible linker*) bulunduğu için uzayda birbirlerine göre sabit bir duruşları yoktur.
* **Protein-Protein Etkileşimleri (Multimer / Kompleks Analizi):** İki farklı proteinin gerçekten fiziksel bir kompleks oluşturup oluşturmadığını anlamak için zincirler arası (*inter-chain*) PAE bloklarına ve **ipTM (*interface predicted Template Modeling*)** skoruna ($\text{ipTM} > 0.75$) bakılır.

---

## 6. Biyolojinin "Transistör" Anı: Dökümhanelerden Biyodökümhanelere

İlk blog yazım olan [**"Transistör Savaşları"**](/posts/dijital-evrenin-temel-tasi/)nda yarı iletken endüstrisinin modern dünyayı nasıl dönüştürdüğünü üç temel sütun üzerinden incelemiştik. Şimdi aynı mühendislik merceğini AlphaFold sonrası biyomühendisliğe tuttuğumuzda, tarihin birebir aynı mimari örüntüyle tekerrür ettiğini görüyoruz:

### 1. Boyutsal Sıçrama: Planar FET $\rightarrow$ GAAFET ve 1D Dizi $\rightarrow$ 3D/4D Kompleks
Elektronikte sızıntı akımlarını önlemek ve atomik sınırda elektrostatik kontrolü sağlamak için transistörler 2 boyutlu düzlemsel yapıdan (**Planar FET**), kapının kanalı 3 taraftan sardığı **FinFET**'e ve nihayetinde 4 taraftan sardığı **GAAFET (Nanosheet)** mimarisine evrildi. 
Biyoloji de tam olarak aynı boyutsal sıçramayı yaşadı: On yıllarca süren 1 boyutlu dizi okuma (Genomik) çağından, AlphaFold 2 ile 3 boyutlu yapısal proteomik çağına, AlphaFold 3 ve difüzyon modelleriyle ise ligand, nükleik asit ve iyon etkileşimlerini kapsayan çok boyutlu atomik tasarım çağına geçtik.

### 2. Biyolojik Moore Yasası ve Ekonomik Motor
1965'te Gordon Moore'un ortaya koyduğu **Moore Yasası**, yalnızca transistör yoğunluğunun her iki yılda bir ikiye katlanmasını değil, **transistör başına düşen maliyetin radikal biçimde azalmasını** sağlayarak teknolojinin laboratuvarlardan çıkıp cebimize girmesini sağlayan ekonomik motoru ateşlemişti.
AlphaFold, yapısal biyolojide benzer bir maliyet çöküşü yarattı:
* **AlphaFold Öncesi:** 50 yılda milyarlarca dolarlık küresel Ar-Ge bütçesiyle çözülen ~200.000 deneysel protein yapısı.
* **AlphaFold Sonrası:** EMBL-EBI ve DeepMind işbirliğiyle kurulan *AlphaFold Protein Structure Database (AFDB)* üzerinden bilim dünyasına ücretsiz açılan **214 milyondan fazla** protein yapısı — yani bilinen neredeyse tüm kataloglanmış organizmaların yapısal proteomu. Bir proteinin 3B yapısına ulaşmanın marjinal maliyeti yüz binlerce dolardan birkaç dakikalık GPU süresine indi.

### 3. "Fabless" Biyomühendislik ve "Pure-Play Biofoundry" Modeli
1987'de Morris Chang'in **TSMC**'yi kurarak icat ettiği **"Pure-play foundry" (Saf Dökümhane)** modeli, Apple, Nvidia, AMD ve Qualcomm gibi **"Fabless" (fabrikasız)** şirketlerin milyarlarca dolarlık yarı iletken fabrikası kurma maliyetinden kurtulup yalnızca üstün çip mimarileri tasarlamaya odaklanmasını sağlamıştı.

AlphaFold ve kardeş üretken modeller (**RFdiffusion**, **ProteinMPNN**, **ESM3**) bugün biyoteknolojide aynı "Fabless" devrimi başlatıyor:
* **Tersine Katlanma (*Inverse Folding*) ve *De Novo* Tasarım:** Artık sadece doğada var olan proteinlerin nasıl katlandığını tahmin etmiyoruz ($1\text{D Dizi} \rightarrow 3\text{D Yapı}$). Süreci tersine çeviriyoruz: Doğada daha önce hiç evrimleşmemiş bir işlevi yerine getirecek (örneğin PET plastiklerini parçalayacak hiper-termofilik bir sentetik enzim veya sadece kanserli hücre reseptörüne kenetlenecek *de novo* bir mini-protein) **3B atomik geometriyi bilgisayarda çiziyoruz**, difüzyon modelleriyle iskeletini (*backbone*) üretiyor ve tersine katlanma ağlarıyla bu 3B şekle katlanacak 1D amino asit dizisini hesaplıyoruz ($3\text{D Hedef Fonksiyon} \rightarrow 1\text{D Sentetik Dizi}$).
* **Tasarımcı vs. Üretici Ayrımı:** Bir biyomühendislik girişimi artık binlerce varyantı ıslak laboratuvarda körlemesine taramak zorunda değil. Bilgisayar başında (*in silico*) milyonlarca aday diziyi AlphaFold/PAE/ipTM filtrelerinden geçirip en yüksek olasılıklı 10 adayı seçerek, gen sentezini ve yüksek verimli ekspresyon testlerini robotik **Biyodökümhanelere (*Biofoundries*)** sipariş edebiliyor.

---

## 7. Özet: Okuma Çağından Yazma Çağına

1947'de Bell Laboratuvarları'nda ilk nokta temaslı transistör çalıştırıldığında, kimse bunun birkaç on yıl içinde cebimizdeki milyarlarca transistörlü yapay zeka işlemcilerine dönüşeceğini tam olarak kestirememişti. Bugün AlphaFold ile biyomühendislikte tam olarak o "1947 anında" duruyoruz.

Arthur C. Clarke'ın üçüncü yasasında söylediği gibi, *"Yeterince gelişmiş bir teknoloji, büyüden ayırt edilemez"*. Ancak bir mühendisin görevi büyüye kapılmak değil, o mekanizmanın matematiğini, sınırlarını ve hata paylarını anlamaktır. AlphaFold ıslak laboratuvarı veya deneysel yapısal biyolojiyi bitirmedi; aksine kristalografları ve biyomühendisleri "yapı çözme işçiliğinden" kurtarıp "fonksiyon ve mekanizma mühendisliğine" terfi ettirdi. Dinamik konformasyonel geçişler, allosterik regülasyon, hücresel ortamın kalabalık etkisi (*macromolecular crowding*) ve ilaç kinetiği hâlâ deneysel doğrulamaya ve biyomühendislik sezgisine muhtaç.

Unutmayalım ki; **algoritmalar sadece olasılık uzayında hesap yapar, o hesapları yaşamı iyileştirecek gerçek moleküllere dönüştüren ise insan zihni ve mühendislik iradesidir**. Biyolojiyi sadece "okuduğumuz" çağ kapandı; artık atomik ölçekte "yazdığımız" ve "tasarladığımız" yeni bir mühendislik çağı başlıyor.

---

**Yazar:** Kerem Uysal — *Kurucu, Tech&Lab*  
**İletişim:** [info@keremuysal.com](mailto:info@keremuysal.com) | [info@techandlab.com](mailto:info@techandlab.com)