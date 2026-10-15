---
layout: post
title: "Die Ästhetik der Beschränkungen: Künstliche Intelligenz auf 2-Dollar-Mikrocontrollern (TinyML)"
author: "Kerem Uysal"
date: 2026-10-15 18:00:00 +0300
categories: [Eingebettete Systeme, Künstliche Intelligenz]
tags: [tinyml, arm-cortex, eingebettete-systeme, edge-ai, optimierung]
lang: de-DE
ref: tinyml-kisitlarin-estetigi
math: true
description: "Gegen den maßlosen Ressourcenverbrauch der Cloud: Die Ingenieursdisziplin deterministischer KI-Architekturen auf 64 KB SRAM und 64 MHz ARM Cortex-M4 Mikrocontrollern mittels Quantisierung, TFLite Micro und CMSIS-NN."
---

> 🌐 **Sprache / Language / Dil:** [Türkçe](/posts/kisitlarin-estetigi-mikrodenetleyicilerde-yapay-zeka-tinyml/) | [English](/posts/aesthetics-of-constraints-tinyml-en/) | **Deutsch**
{: .prompt-info }

Die moderne Softwarewelt hat sich im letzten Jahrzehnt in einer gefährlichen Komfortzone eingerichtet: **der Illusion unendlicher Ressourcen.**

Läuft ein Modell zu langsam, mietet man bei Cloud-Anbietern größere GPU-Cluster; reicht der Arbeitsspeicher nicht aus, verdoppelt man den DRAM; man verbirgt sich hinter Abstraktionsschichten und lädt Gigabyte-schwere Gewichtsmatrizen bedenkenlos in den Grafikspeicher. Die Rechenzentren unserer Zeit bieten zwar gigantische Rechenleistung – doch dieser Überfluss hat eine **architektonische Adipositas** gefördert, die grundlegende ingenieurtechnische Disziplin verdrängt.

Was aber geschieht, wenn keine Steckdose zur Verfügung steht, kein Glasfaserkabel anliegt und ein System jahrelang mit einer Knopfzelle in der realen physikalischen Welt überleben muss? Genau hier setzt **TinyML (Künstliche Intelligenz auf Mikrocontrollern)** an: als Gegenentwurf zur Maßlosigkeit der Cloud und als Hommage an die Eleganz und Disziplin des extremen Edge-Bereichs.

---

## 1. Einleitung: Agilität am Edge gegen Trägheit in der Cloud

Betrachten wir das kontinuierliche Schwingungsmonitoring hunderter Industriemotoren in einer Produktionshalle oder die EKG-Erfassung eines tragbaren biomedizinischen Pflasters im Kilohertzbereich. Das klassische „Cloud-Centric IoT“-Paradigma besagt: *„Sensordaten abtasten, verpacken, über Mobilfunk oder WLAN an AWS senden, Inferenz in der Cloud ausführen und Ergebnis zurücksenden.“*

Auf dem Whiteboard klingt dies einfach – in der physikalischen Realität scheitert dieser Ansatz an drei Wänden:

1. **Energiebilanz (HF-Übertragung vs. lokale Berechnung):** In eingebetteten Systemen ist Rechnen extrem günstig, **Funkübertragung hingegen extrem teuer**. Das Abstrahlen eines einzelnen Bits über Bluetooth, WLAN oder LTE-M verbraucht Nanojoule pro Bit – um Größenordnungen mehr Energie als die Ausführung tausender Multiply-Accumulate-Befehle (MAC) auf einem lokalen ARM-Cortex-M-Kern. Kontinuierliches Streaming von Rohdaten entleert eine Batterie innerhalb von Tagen statt Jahren.
2. **Latenz und Determinismus:** Bei industriellen Schutzabschaltungen oder autonomen Mikrosystemen entscheiden Millisekunden. Der Weg über Netzwerkstacks, Basisstationen und Cloud-Warteschlangen ist prinzipbedingt **stochastisch**. Eingebettete Regelschleifen erfordern jedoch **harte deterministische Echtzeitgarantien**.
3. **Bandbreite und Datenschutz:** Ein mit $10\text{ kHz}$ abgetasteter Beschleunigungssensor liefert zu 99,9 % normales Rauschen. Es ist pure Bandbreitenverschwendung, dieses Rauschen permanent zu übertragen. Zudem garantiert die Verarbeitung sensibler Vital- oder Industrie-Telemetriedaten direkt an der Signalquelle maximale Datensicherheit.

> 💡 **Ingenieurgrundsatz:** Daten nicht Tausende Kilometer durch die Welt transportieren, um Erkenntnisse zu gewinnen. Die Intelligenz gehört direkt an den Sensor, an die Kontaktstelle zwischen Silizium und physikalischer Realität. Vor Ort verarbeiten und nur bei Bedarf ein einzelnes Byte senden: „Anomalie erkannt“.
{: .prompt-tip }

---

## 2. Die Welt der Beschränkungen: 64 KB SRAM, 256 KB Flash und 64 MHz Cortex-M4

Während Cloud-Modelle Gigabytes an Speicher belegen, operiert der Entwickler auf einem 2-Dollar-Mikrocontroller (z. B. STM32F4 oder nRF52840 mit ARM Cortex-M4F) innerhalb engster Grenzen:

* **Taktfrequenz:** 64 MHz (ca. 50- bis 60-mal langsamer als ein einzelner Server-CPU-Kern).
* **Flash-Speicher (Nichtflüchtig):** 256 KB (Firmware, Treiber, RTOS-Kernel und neuronale Gewichte müssen hier Platz finden).
* **SRAM (Flüchtiger Arbeitsspeicher):** 64 KB (Stack, Heap, Peripheriepuffer und Aktivierungsmatrizen teilen sich diesen Raum).

Wie läuft ein neuronales Netz auf dieser Hardware? Die Antwort liegt in der rigorosen Ausnutzung der **Harvard-Architektur**.

Da trainierte Gewichte und Biases zur Inferenzzeit unveränderlich (`const`) sind, verbleiben sie im **Flash-Speicher (`.rodata`)**. Sie werden niemals ins RAM kopiert; der Prozessor liest sie direkt über **eXecute-In-Place (XIP)** aus dem Flash. Die kostbaren **64 KB SRAM** dienen ausschließlich den Aktivierungstensoren der jeweils aktiven Schicht, organisiert als Ping-Pong-Puffer.

### Speicherabbild (Memory Map) auf dem Mikrocontroller

```text
+-------------------------------------------------------------------+
|                     FLASH-SPEICHER (256 KB - Persistent)          |
+-------------------------------------------------------------------+
| 0x0800_0000 | Interrupt-Vektortabelle                       [~1 KB] |
|-------------|-----------------------------------------------------|
| .text       | Anwendungs-Code + HAL + TFLite Micro Kern     [~65 KB]|
|             | + CMSIS-NN SIMD-Optimierungen                       |
|-------------|-----------------------------------------------------|
| .rodata     | INT8-quantisierte Modellgewichte und Biases   [~40 KB]|
|             | (Read-Only, direkte XIP-Adressierung, nie im RAM)   |
|-------------|-----------------------------------------------------|
| Frei        | Platz für Firmware-Updates (OTA)              [150 KB]|
+-------------------------------------------------------------------+

+-------------------------------------------------------------------+
|                     SRAM (64 KB - Arbeitsspeicher)                 |
+-------------------------------------------------------------------+
| 0x2000_0000 | Globale Puffer & RTOS-Kernel-Overhead         [~8 KB] |
|-------------|-----------------------------------------------------|
| Tensor-Arena| Ping-Pong-Aktivierungsspeicher                [~32 KB]|
|             | Schicht N Eingangstensor <---> Schicht N Ausgang    |
|-------------|-----------------------------------------------------|
| Heap / Stack| Stack für Threads und Interrupt-Routinen      [~12 KB]|
|-------------|-----------------------------------------------------|
| Reserve     | Sicherheitsabstand gegen Stack-Overflow       [~12 KB]|
+-------------------------------------------------------------------+
```

---

## 3. Post-Training-Quantisierung (PTQ): Von FP32 zu INT8

Standardmodelle nutzen 32-Bit-Gleitkommazahlen (**FP32** - IEEE 754):

$$\text{FP32:} \quad 1 \text{ Vorzeichenbit} + 8 \text{ Exponentenbits} + 23 \text{ Mantissenbits} = 32 \text{ Bit (4 Byte)}$$

Auf eingebetteten Systemen führt FP32 zu gravierenden Nachteilen: Vierfacher Speicherbedarf im Flash und – bei Mikrocontrollern ohne Hardware-FPU – softwareemulierte Gleitkommaoperationen, die Dutzende Taktzyklen pro Multiplikation beanspruchen.

**Post-Training-Quantisierung (PTQ)** bildet kontinuierliche Fließkommagewichte ($r \in \mathbb{R}$) über eine **asymmetrische affine Transformation** auf ein vorzeichenbehaftetes 8-Bit-Ganzzahlgitter ($q \in [-128, 127]$) ab:

$$r = S \cdot (q - Z)$$

Wobei:
* $S \in \mathbb{R}^+$: Skalierungsfaktor (*Scale*),
* $Z \in \mathbb{Z}$: Nullpunkt (*Zero-Point*), der den reellen Wert Null auf dem Integer-Gitter zentriert.

### Reine Ganzzahlarithmetik
In einer dichten Schicht $y = \text{ReLU}(W \cdot x + b)$ führt das Einsetzen quantisierter Werte zu:

$$q_y = Z_y + \left( \frac{S_W S_x}{S_y} \right) \left[ \sum (q_W \cdot (q_x - Z_x)) + q_b \right]$$

Der Skalierungsfaktor $M = \frac{S_W S_x}{S_y}$ liegt stets im Bereich $0 < M < 1$. Da Mikrocontroller keine Gleitkomma-Skalierung ausführen sollen, wird $M$ in einen **Festkomma-Multiplikator ($M_0 \in [2^{30}, 2^{31}-1]$)** und einen **arithmetischen Bitshift ($n$)** zerlegt:

$$M \approx M_0 \cdot 2^{-n}$$

Eine Multiplikation mit $M$ entspricht somit einer einzigen 32-Bit-Integer-Multiplikation gefolgt von einem Bitshift (`>> n`). **Das gesamte Netz rechnet ausschließlich in Integer-Registern ohne eine einzige Gleitkommaoperation.**

---

## 4. Die Laufzeitumgebung: TensorFlow Lite for Microcontrollers & CMSIS-NN

Moderne Frameworks verlangen dynamische Speicherallokation (`malloc`), Threads und POSIX-Schnittstellen. In Bare-Metal-Firmware führt dynamischer Heap-Speicher zu Fragmentierung und unvorhersehbaren Abstürzen.

### 4.1. TFLite Micro: Deterministische Tensor-Arenen
TFLite Micro verzichtet vollständig auf dynamische Allokation:

```cpp
constexpr int kTensorArenaSize = 32 * 1024; // Exakt 32 KB statisch reserviert
alignas(16) static uint8_t tensor_arena[kTensorArenaSize];

tflite::MicroInterpreter interpreter(
    model, 
    resolver, 
    tensor_arena, 
    kTensorArenaSize, 
    error_reporter
);

TfLiteStatus allocate_status = interpreter.AllocateTensors();
if (allocate_status != kTfLiteOk) {
    // Arena zu klein: Fehler tritt deterministisch beim Booten auf!
    ErrorHandler();
}
```

Beim Start plant der Interpreter den gesamten Speicherbedarf statisch. Passt das Modell beim Booten in die Arena, läuft es dauerhaft stabil ohne Gefahr eines Laufzeitabsturzes.

### 4.2. ARM CMSIS-NN: SIMD-Beschleunigung auf Siliziumebene
Auf ARM-Cortex-M4/M7-Kernen nutzt **CMSIS-NN** die integrierten DSP-Erweiterungen. Mittels **SIMD (Single Instruction, Multiple Data)** werden vier 8-Bit-Ganzzahlen in ein 32-Bit-Register gepackt.

Über den Befehl **`__SMLAD` (Signed Multiply Accumulate Dual)** führt der Prozessor **zwei 16-Bit-Multiplikationen und eine 32-Bit-Akkumulation in einem einzigen Taktzyklus** aus. Dies liefert eine **4- bis 5-fache Beschleunigung** gegenüber Standard-C-Schleifen bei proportional reduzierter Energieaufnahme.

---

## 5. Reale Anwendung: Vorausschauende Wartung und biomedizinische Anomalieerkennung

Ein versierter Embedded-Entwickler überlässt nicht alles dem neuronalen Netz, sondern kombiniert klassische **digitale Signalverarbeitung (DSP)** mit **TinyML**:

```text
[MEMS-Sensor / AFE] --(SPI/I2C + DMA)--> [Ping-Pong-RAM-Puffer]
                                                  |
                                       (CMSIS-DSP: Fensterung + FFT)
                                                  v
[Anomalie-Flag / IRQ] <--(Schwelle: ||x - x̂|| > τ)-- [TFLite Micro: INT8-Autoencoder]
```

### Schritt 1: CPU-freie Datenerfassung (DMA + Ping-Pong-Puffer)
Ein 3-Achsen-MEMS-Sensor tastet mit $1.6\text{ kHz}$ ab. Mittels **DMA (Direct Memory Access)** schreibt die Hardware Abtastwerte direkt in das SRAM, während der Kern im Schlafmodus verharrt. Ist Puffer A voll, wechselt DMA auf Puffer B und weckt den Prozessor per Interrupt.

### Schritt 2: Spektralanalyse via CMSIS-DSP
1024 Rohabtastwerte direkt in ein Netz zu leiten, würde ein tiefes Modell erfordern. Stattdessen berechnet `CMSIS-DSP` hardwarebeschleunigt eine **reelle schnelle Fouriertransformation (RFFT)** mit Hanning-Fensterung und komprimiert das Signal in 32 Frequenzbänder. Die Eingangsdimension des Netzes schrumpft von 1024 auf 32!

### Schritt 3: INT8-Autoencoder zur Anomalieerkennung
Da Schadensdaten in der Praxis selten sind, wird ein kompakter **Autoencoder** ($32 \rightarrow 16 \rightarrow 8 \rightarrow 16 \rightarrow 32$) ausschließlich auf gesunden Betriebszuständen trainiert und auf INT8 quantisiert (~4 KB Flash-Bedarf).

Im Betrieb wird der Rekonstruktionsfehler ($L_1$-Norm) im Integer-Raum berechnet:

$$E = \frac{1}{N} \sum_{i=1}^{N} |x_i - \hat{x}_i|$$

* Normaler Zustand: Das Modell kennt das Spektrum, $E < \tau$.
* Lagerschaden oder Herzrhythmusstörung: Die Harmonischen verschieben sich, das Modell kann das Spektrum nicht rekonstruieren, $E > \tau$ – der Alarm wird ausgelöst.

### Schritt 4: Energiebilanz
Auf einem 64-MHz-Cortex-M4 dauert die 1024-Punkte-FFT mitsamt INT8-Inferenz **lediglich ~2.4 Millisekunden**. Der Prozessor verharrt über 97 % der Zeit im energiesparenden Schlafmodus.

---

## 6. Fazit: Echte Ingenieurskunst glänzt unter harten Beschränkungen

In Zeiten von Gigawatt-Rechenzentren ist es leicht, Ineffizienzen hinter schier unendlicher Rechenleistung zu verbergen.

In 64 KB SRAM gibt es kein Versteck.

Dort zählt jedes Byte, jedes Speicher-Alignment (`alignas(16)`), jeder Taktzyklus und jedes Mikroampere. Ein komplexes mathematisches Modell ohne Genauigkeitsverlust um das Vierfache zu komprimieren, deterministisch in einer statischen Arena zu fixieren und in Millisekunden auf einem 2-Dollar-Siliziumchip auszuführen – genau darin liegt die **Ästhetik der Beschränkungen**.

Mit unbegrenzten Ressourcen kann jeder bauen; **wahre Ingenieurskunst glänzt unter harten physikalischen Beschränkungen.**

---

**Autor:** Kerem Uysal — *Gründer, Tech&Lab*  
**Kontakt:** [info@keremuysal.com](mailto:info@keremuysal.com) | [info@techandlab.com](mailto:info@techandlab.com)
