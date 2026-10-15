---
layout: post
title: "Die Geometrie unsichtbarer Wellen: Anatomie eines Radars und FMCW-Mathematik"
date: 2026-10-15 18:00:00 +0300
author: "Kerem Uysal"
categories: [Hardware, Radar]
tags: [fmcw, radar, hf, signalverarbeitung, dsp]
lang: de-DE
ref: fmcw-radar-geometrisi
math: true
description: "Die Kunst, Zeit bei Lichtgeschwindigkeit in Frequenz zu kodieren: FMCW-Radar-Hardwarearchitektur, Chirp-Mathematik, 2D-FFT-Signalverarbeitung und Herausforderungen im Millimeterwellen-Leiterplattendesign."
---

> 🌐 **Sprache / Language / Dil:** [Türkçe](/posts/gorunmez-dalgalarin-geometrisi-radar-anatomisi-ve-fmcw-matematigi/) | [English](/posts/geometry-of-invisible-waves-anatomy-of-a-radar-and-fmcw-mathematics-en/) | **Deutsch**
{: .prompt-info }

## 1. Einleitung: Wettlauf mit der Lichtgeschwindigkeit

Sobald eine elektromagnetische Welle in den freien Raum abgestrahlt wird, beginnt ein Wettlauf mit der universellen Geschwindigkeitsgrenze: der Lichtgeschwindigkeit ($c \approx 3 \times 10^8\text{ m/s}$). Auf menschlichen Maßstäben ist dieses Tempo kaum fassbar: In einer Sekunde umrundet Licht die Erde siebeneinhalb Mal; in einer Nanosekunde ($10^{-9}\text{ s}$) legt es gerade einmal **30 Zentimeter** zurück.

Stellen Sie sich vor, Sie möchten mit einem Radarsensor auf Ihrem Schreibtisch den Abstand zu einem $1.5\text{ Meter}$ entfernten Objekt messen. Die Gesamtlaufzeit des Signals (Hin- und Rückweg über $3\text{ Meter}$) beträgt lediglich **10 Nanosekunden**. Bewegt sich das Ziel um $15\text{ Zentimeter}$, verschiebt sich die Laufzeit um gerade einmal **1 Nanosekunde**.

Ein klassisches **Pulsradar (Time-of-Flight)** stößt bei dieser Aufgabe an fundamentale Hardwaregrenzen:

1. **Blindzonen und extreme Spitzenleistung:** Um Objekte im Nahbereich zu detektieren, müsste die Pulsdauer ($\tau_p$) auf Pico- oder wenige Nanosekunden komprimiert werden. Damit überhaupt messbare Energie zurückreflektiert wird, müsste die Impulsspitzenleistung auf Hunderte Watt oder Kilowatt steigen. Gleichzeitig muss der rauscharme Vorverstärker (LNA) im Empfänger während des Sendepulses abgeschaltet werden, um Zerstörung zu verhindern – wodurch das Radar im unmittelbaren Nahbereich blind bleibt.
2. **Extreme ADC-Abtastraten:** Das direkte Digitalisieren von Nanosekundenpulsen im Zeitbereich erfordert Wandler mit zig Giga-Samples pro Sekunde (GSPS), was enorme Verlustleistungen und extreme Kosten nach sich zieht.
3. **Integrationsgrenzen:** Hohe Impulsleistungen lassen sich nicht in kompakten Silizium-CMOS- oder SiGe-BiCMOS-Monolith-Chips (MMICs) für Automotive- und Robotikanwendungen integrieren.

Aus diesen Gründen setzt die moderne Sensorik autonomer Fahrzeuge und der Industrie auf **FMCW-Radar (Frequency-Modulated Continuous Wave — Frequenzmoduliertes Dauerstrichradar)**.

> 💡 **Das FMCW-Paradigma:** FMCW-Radar misst die Signallaufzeit nicht mit einer Nanosekunden-Stoppuhr, sondern **kodiert Zeit in Frequenz**. Durch kontinuierliche, lineare Frequenzmodulation des Sendesignals werden unmessbar kurze Nanosekunden-Laufzeiten ($\tau$) am Ausgang eines Mischers in leicht digitalisierbare **Schwebungsfrequenzen (*Beat Frequency* - $f_b$)** im Kilohertz- oder niedrigen Megahertz-Bereich transformiert.
{: .prompt-tip }

---

## 2. Hardware-Anatomie eines FMCW-Radars

Das HF-Frontend und die Basisband-Signalkette eines FMCW-Radars bilden eine harmonische Symbiose aus Mikrowellentechnik und digitaler Signalverarbeitung (DSP):

```text
                  +-----------------------+
                  |    Chirp-Synthesizer  |
                  |       (PLL + VCO)     |
                  +-----------+-----------+
                              |
                     [LO-Signal: f_TX(t)]
                              |
            +-----------------+-----------------+
            |                                   |
            v                                   v
  +-------------------+               +-------------------+
  | Leistungsverst.   |               |     HF-Mischer    |
  |       (PA)        |               |   (Downconverter) |
  +---------+---------+               +---------^---------+
            |                                   |
       [TX-Antenne]                        [RX-Signal]
            |                                   |
            v                           +-------+-------+
       )))  Welle  (((                  |   Rauscharmer |
                                        |   Verstärker  |
          [ Ziel ]                      |     (LNA)     |
                                        +-------^-------+
                                                |
                                           [RX-Antenne]
```

### Die Baugruppen im Überblick:

1. **Frequenzsynthesizer (PLL & VCO):**
   Das Herz des Radars. Ein spannungsgesteuerter Oszillator (VCO) mit Fractional-N-Phasenregelschleife (PLL) erzeugt Frequenzrampen mit hoher Linearität ($R^2 > 0.999$). Nichtlinearitäten im Sweep führen unweigerlich zu spektraler Aufweitung und Geisterzielen.
2. **Leistungsverstärker (PA) & Sendeantenne (TX):**
   Verstärkt das Millimeterwellensignal (z. B. $77\text{ GHz}$) auf moderate Pegel ($+10\text{ dBm}$ bis $+15\text{ dBm}$, ca. $10-30\text{ mW}$) und strahlt es über Microstrip-Patch-Arrays ab.
3. **Rauscharmer Vorverstärker (LNA - Low Noise Amplifier):**
   Das reflektierte Echo ist durch die $1/R^4$-Pfaddämpfung der Radargleichung extrem abgeschwächt (Pikowattbereich). Der LNA verstärkt dieses Signal bei minimaler Rauschzahl (*Noise Figure*).
4. **HF-Mischer (Downconversion):**
   Hier vollzieht sich die Frequenztransformation. Das aktuelle Sendesignal (LO) wird mit dem zeitverzögerten Echosignal (RX) multipliziert. Durch trigonometrische Multiplikation entstehen Summen- ($f_{TX} + f_{RX} \approx 154\text{ GHz}$) und Differenzfrequenzen ($f_{TX} - f_{RX} \sim \text{kHz}-\text{MHz}$).
5. **Basisbandfilterung und ADC:**
   Ein Tiefpassfilter (LPF) eliminiert die $154\text{ GHz}$-Summenkomponente. Ein Hochpassfilter (HPF) unterdrückt direktes Antennenübersprechen (TX-to-RX Leakage) und Nahbereichsclutter. Das bereinigte Schwebungssignal ($f_b$) wird von einem Standard-ADC mit wenigen **Mega-Samples pro Sekunde (MSPS)** digitalisiert.

---

## 3. Chirp-Mathematik und Entfernungsmessung (Range Estimation)

Das grundlegende Sendesignal eines FMCW-Radars ist ein linear frequenzmodulierter Wellenzug, bezeichnet als **Chirp**:

$$f_{TX}(t) = f_c + S \cdot t \quad (0 \le t \le T_c)$$

Hierbei gilt:
* $f_c$: Träger-Startfrequenz (z. B. $77\text{ GHz}$),
* $T_c$: Chirp-Dauer (z. B. $40\text{ \mu s}$),
* $B$: Sweep-Bandbreite (z. B. $4\text{ GHz}$ von $77\text{ GHz}$ bis $81\text{ GHz}$),
* $S$: Frequenzsteigung / Rampenrate ($S = \frac{B}{T_c} = \frac{4\text{ GHz}}{40\text{ \mu s}} = 100\text{ MHz/\mu s} = 10^{14}\text{ Hz/s}$).

```text
Frequenz
  ^
  |        TX-Signal (Gesendet)
f_c + B +      /             /
        |     /             /
        |    /             /    RX-Signal (Verzögertes Echo)
        |   /  |<- tau ->| /
        |  /   |         |/
        | /    |         /
    f_c +/-----+--------/------------> Zeit
        |<-- T_c ------>|
```

Für ein ruhendes Ziel im Abstand $R$ beträgt die Signallaufzeit ($\tau$):

$$\tau = \frac{2R}{c}$$

Bis das Echo den Empfänger erreicht, hat der Sender seine Frequenz bereits erhöht. Die Differenz am Mischerausgang ist die **Schwebungsfrequenz (*Beat Frequency* - $f_b$)**:

$$f_b = S \cdot \tau = S \cdot \frac{2R}{c} = \frac{2 B R}{c T_c}$$

Daraus ergibt sich die direkte Proportionalität zwischen Distanz und Frequenz:

$$R = \frac{c T_c}{2 B} f_b = \frac{c}{2 S} f_b$$

### Rechenbeispiel:
* $B = 4\text{ GHz}$, $T_c = 40\text{ \mu s}$, $S = 100\text{ MHz/\mu s}$.
* Zielabstand $R = 15\text{ Meter}$.
* Laufzeit: $\tau = \frac{2 \times 15}{3 \times 10^8} = 100\text{ Nanosekunden}$.
* Schwebungsfrequenz: $f_b = 10^{14}\text{ Hz/s} \times 100 \times 10^{-9}\text{ s} = 10\text{ MHz}$.

Anstelle einer extrem schwierigen $100\text{-Nanosekunden}$-Direktzeitmessung tastet ein gewöhnlicher $25\text{-MSPS}$-ADC mühelos ein stabiles $10\text{-MHz}$-Sinussignal ab.

### Entfernungsauflösung ($\Delta R$):
Zwei benachbarte Objekte sind trennbar, wenn sich ihre Schwebungsfrequenzen um mindestens eine FFT-Frequenzlinie unterscheiden ($\Delta f = 1/T_c$):

$$f_{b2} - f_{b1} \ge \frac{1}{T_c} \implies \frac{2 B \Delta R}{c T_c} \ge \frac{1}{T_c} \implies \Delta R = \frac{c}{2B}$$

> ⚠️ **Ingenieurerkenntnis:** Die Entfernungsauflösung hängt nicht von der Chirpdauer oder Trägerfrequenz ab, sondern ausschließlich von der **HF-Bandbreite ($B$)**.
> * $24\text{ GHz}$-Band ($200\text{ MHz}$ Bandbreite): $\Delta R = \frac{3 \times 10^8}{2 \times 200 \times 10^6} = 75\text{ cm}$.
> * $77-81\text{ GHz}$-Band ($4\text{ GHz}$ Bandbreite): $\Delta R = \frac{3 \times 10^8}{2 \times 4 \times 10^9} = \mathbf{3.75\text{ cm}}$.
> Dies begründet den Technologiewechsel der Automobilindustrie von 24 GHz zu 77 GHz.
{: .prompt-warning }

---

## 4. Phasenanalyse, Doppler-Verschiebung und Geschwindigkeitsmessung

Bewegt sich ein Ziel mit der Radialgeschwindigkeit $v$, verschiebt sich die Frequenz über den **Doppler-Effekt**:

$$f_d = \frac{2v}{\lambda} = \frac{2v f_c}{c}$$

Bei $77\text{ GHz}$ ($\lambda \approx 3.9\text{ mm}$) erzeugt ein Fahrzeug bei $100\text{ km/h}$ ($27.8\text{ m/s}$) eine Doppler-Verschiebung von:

$$f_d \approx 14.25\text{ kHz}$$

$14.25\text{ kHz}$ sind gegenüber der $10\text{ MHz}$-Schwebungsfrequenz vernachlässigbar klein und lassen sich innerhalb eines einzelnen Chirps nicht zuverlässig auflösen.

Die Lösung liegt in der **Phase ($\phi$)** des Schwebungssignals:

$$s_{IF}(t) = A \cos\left( 2\pi f_b t + \phi_0 \right)$$

Die Phase $\phi_0$ spiegelt die exakte Wellenlaufzeit wider:

$$\phi_0 = 2\pi f_c \tau = \frac{4\pi R}{\lambda}$$

Da $\lambda \approx 3.9\text{ mm}$, führt bereits eine Zielbewegung von nur **$1\text{ mm}$** zu einer Phasendrehung von:

$$\Delta \phi = \frac{4\pi \times 1\text{ mm}}{3.9\text{ mm}} \approx 3.22\text{ Radiant} \approx 185^\circ$$

### Chirp-Frame-Verarbeitung
Um die Geschwindigkeit zu messen, sendet das Radar einen Burst aus $N$ aufeinanderfolgenden Chirps im Abstand $T_c$:

Zwischen zwei Chirps bewegt sich das Ziel um $\Delta R = v \cdot T_c$. Dies erzeugt eine Phasenverschiebung zwischen den Range-Peaks aufeinanderfolgender Chirps:

$$\Delta \phi = \frac{4\pi v T_c}{\lambda} \implies v = \frac{\lambda \cdot \Delta \phi}{4\pi T_c}$$

Die maximale eindeutig messbare Geschwindigkeit (**Maximum Unambiguous Velocity**) beträgt:

$$v_{max} = \frac{\lambda}{4 T_c}$$

Die Geschwindigkeitsauflösung über einen Frame aus $N$ Chirps lautet:

$$\Delta v = \frac{\lambda}{2 N T_c}$$

---

## 5. Zweidimensionale FFT (2D-FFT): Die Range-Doppler-Heatmap

Die abgetasteten ADC-Werte werden im Speicher als zweidimensionale Matrix (Radar Data Cube) organisiert:

### 1. Schritt: Fast-Time FFT (Range FFT)
Auf jede Spalte ($M$ Abtastwerte eines Chirps) wird eine 1D-FFT angewendet. Die Spitzenwerte liefern den **Entfernungsbereich (*Range Bin*)**.

### 2. Schritt: Slow-Time FFT (Doppler FFT)
Über alle $N$ Chirps hinweg wird entlang der Zeilen für jeden detektierten Entfernungsbereich eine zweite 1D-FFT gerechnet. Dies wandelt die Phasendrehgeschwindigkeit in die **Radialgeschwindigkeit (*Doppler Bin*)** um.

Das Resultat ist die **Range-Doppler-Heatmap**:

```text
Doppler (Geschwindigkeit) [m/s]
     ^
 +30 |             . (Annäherndes Fahrzeug)
     |
   0 |-------------------+------- (Statischer Clutter / Leitplanke)
     |                   . (Sich entfernender Fußgänger)
 -30 |
     +-----------------------------------> Entfernung (Range) [m]
     0        20        40        60
```

### CFAR: Ziele aus dem Rauschen filtern
Thermische Fluktuationen und Bodenechos erzeugen Scheinsignale. Radarsysteme nutzen **CFAR-Algorithmen (Constant False Alarm Rate)** mit dynamischen Schwellenwerten:
* **CA-CFAR (Cell Averaging):** Ermittelt den Rauschpegel benachbarter Referenzzellen um die Testzelle herum, isoliert durch Schutzzellen (*Guard Cells*).
* **OS-CFAR (Ordered Statistic):** Sortiert Referenzzellen nach Amplitude, um zu verhindern, dass benachbarte echte Ziele den Schwellenwert verfälschen.

---

## 6. Millimeterwellen-HF-Design (77 GHz) und Leiterplatten-Herausforderungen

Bei $77\text{ GHz}$ beträgt die Wellenlänge lediglich **$3.9\text{ Millimeter}$**. Auf der Leiterplatte wirkt jede Lötstelle und jede Leiterbahnkante als parasitärer Resonator.

### 1. Warum FR-4 ungeeignet ist
* **Verlustfaktor ($\tan\delta$):** FR-4 besitzt einen hohen Verlustfaktor ($\approx 0.02$). Bei $77\text{ GHz}$ wird das Signal nach wenigen Zentimetern als dielektrische Wärme dissipiert.
* **Inhomogenität von $\varepsilon_r$:** Glasfasergewebe und Epoxidharz erzeugen lokale Schwankungen der Dielektrizitätskonstante, was zu Phasenfehlern führt.

77-GHz-Radare erfordern PTFE-Keramik-Substrate wie **Rogers RO3003, RO4835** oder **Isola Astra MT77** ($\tan\delta \approx 0.0013$) in **Hybrid-Stack-up**-Konfigurationen (Rogers als HF-Außenlage, FR-4 für Innenlagen).

### 2. Skin-Effekt und Oberflächenrauhigkeit
Bei $77\text{ GHz}$ sinkt die Skintiefe ($\delta$) in Kupfer auf:

$$\delta = \sqrt{\frac{\rho}{\pi f \mu}} \approx 0.23\text{ \mu m} \quad (230\text{ nm})$$

Da der Strom nur noch in einer Schicht von einem Viertel Mikrometer fließt, erhöht mikroskopische Kupferrauhigkeit den HF-Widerstand drastisch. Es kommen elektropoliertes VLP- oder HVLP-Kupfer zum Einsatz.

### 3. Einfallswinkel (Angle of Arrival - AoA) und MIMO
Durch den Einsatz mehrerer RX-Antennen mit Abstand $d = \lambda/2$ entsteht eine räumliche Phasenverschiebung:

$$\Delta \phi_{spatial} = \pi \sin\theta \implies \theta = \arcsin\left( \frac{\Delta \phi_{spatial}}{\pi} \right)$$

Mittels **MIMO (Multiple-Input Multiple-Output)** erzeugen $N_{TX}$ Sende- und $N_{RX}$ Empfangsantennen ein **virtuelles Antennenarray** von $N_{TX} \times N_{RX}$ Elementen, was hohe Winkeltrennung ohne mechanische Drehmechanik ermöglicht.

---

## 7. Fazit: Wellen, die zu Geometrie werden

Die FMCW-Radararchitektur gehört zu den elegantesten Ingenieursleistungen der Hochfrequenztechnik.

Statt Lichtpulse mit extremen Taktfrequenzen zu stoppen, modulieren wir Frequenzen kontinuierlich. Über analoge Abwärtsmischung transformieren wir Gigahertz-Signale in akustische Frequenzen, trennen Entfernung und Geschwindigkeit über 2D-FFTs und erfassen die Umwelt mit Submillimeter-Präzision.

Wenn Starkregen, dichter Nebel oder Finsternis Kameras blenden und LiDAR-Strahlen streuen, navigieren autonome Systeme sicher weiter – dank der unsichtbaren Geometrie elektromagnetischer FMCW-Wellen.

---

**Autor:** Kerem Uysal — *Gründer, Tech&Lab*  
**Kontakt:** [info@keremuysal.com](mailto:info@keremuysal.com) | [info@techandlab.com](mailto:info@techandlab.com)
