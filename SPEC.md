# ToneToSheet – Spezifikation

> Version 0.2 · Status: Entwurf · Sprache der App: Deutsch (Englisch vorbereitet)
>
> Jede Anforderung hat eine ID (`BEREICH-NNN`) und, wo möglich, einen konkreten **Test**.
> Tests werden **vor** dem Code geschrieben (TDD). Eine Anforderung gilt als erfüllt, wenn alle zugehörigen Tests grün sind.

---

## 0. Projektüberblick

### 0.1 Ziel
ToneToSheet macht aus **gespielter Musik** (Mikrofon oder MIDI) und aus **gescannten Notenblättern** bearbeitbare digitale Noten, die man **transponieren**, **als Einzelstimmen ausgeben**, **abspielen** und **exportieren** kann.

### 0.2 Zielgruppen & Kern-Anwendungsfälle
- **Brass-Quintett-Leiter**
  - Alte Notenblätter scannen → bearbeitbare Partitur.
  - Stück in andere Tonart setzen, Einzelstimmen für Trompete in B, Horn in F, Posaune, Tuba automatisch korrekt transponiert ausgeben.
  - Tonumfang-Warnungen, wenn eine Stimme nach dem Transponieren unspielbar wird.
- **Klavierstudentin**
  - Mit Metronom einspielen (Mikrofon oder Digitalpiano per MIDI) → Klaviersystem (2 Notenzeilen).
  - Transponieren, Üben mit Loop und reduziertem Tempo.
- **CCM-Band (E-Gitarre/Gitarre)**
  - Leadsheets/Chord Charts mit Akkordsymbolen, Nashville Number System, ChordPro, Tabs.
  - Songstruktur (Verse/Chorus/Bridge), Kapodaster-Rechner, Kompositionshilfen.

### 0.3 Plattformen
- **Phase 1:** Android, Windows.
- **Später:** macOS, iOS.
- Offline-first. Netzwerk ist keine Voraussetzung für irgendeine Kernfunktion.

### 0.4 Nicht-Ziele (vorerst)
- Keine Cloud-Synchronisation, keine Benutzerkonten.
- Kein Echtzeit-Mehrbenutzer-Editing.
- Keine Erkennung **handgeschriebener** Noten (nur gedruckte).
- Kein vollwertiger Ersatz für Dorico/Sibelius (kein professioneller Stichsatz-Feinschliff).
- Keine Polyphonie-Erkennung für andere Instrumente als Klavier.

---

## 1. Architektur & Tech-Stack

- **ARC-001** App-Framework: **Flutter** (eine Codebasis für Android, Windows, macOS, iOS).
- **ARC-002** Musik-Kern als **reines Dart-Package** `tonetosheet_core` ohne Flutter- oder Plattform-Abhängigkeiten.
  - *Test:* `dart test` im Core-Package läuft ohne Flutter-SDK-Import; ein CI-Check schlägt fehl, wenn `package:flutter` im Core importiert wird.
- **ARC-003** Modulgrenzen (je eigenes Package/Verzeichnis, nur über Interfaces verbunden):
  - `core` (Datenmodell, Musiktheorie, Transposition, Notensatz-Logik)
  - `io` (MusicXML, MIDI, ChordPro, ABC, Projektformat)
  - `render` (Anbindung Notensatz-Engine)
  - `audio` (Metronom, Aufnahme, MIDI-Eingang, Playback)
  - `transcribe` (Pitch-Detection, Polyphonie-Modell, Drums, Quantisierung)
  - `omr` (Bildvorverarbeitung, Symbolerkennung)
  - `app` (UI)
  - *Test:* Ein Abhängigkeits-Check (z. B. `dependency_validator` oder eigenes Skript) verbietet Importe von `core` → `app`/`audio`/`omr`.
- **ARC-004** Notensatz-Rendering über **Verovio** (C++, LGPL; MusicXML/MEI → SVG), angebunden per FFI (Desktop/Mobile) oder WebView-Fallback. Kein eigener Stichsatz-Algorithmus.
  - *Test:* Eine minimale MusicXML-Datei (1 Takt, 4 Viertel C4) wird zu einem SVG gerendert, das genau 4 Notenkopf-Elemente enthält.
- **ARC-005** ML-Inferenz über **ONNX Runtime** oder **TensorFlow Lite** mit austauschbarem Modell-Interface (`TranscriptionModel`, `OmrModel`).
  - *Test:* Ein Fake-Modell, das feste Ergebnisse liefert, kann ohne Codeänderung anstelle des echten Modells injiziert werden.
- **ARC-006** Audio-I/O nativ (Android: Oboe/AAudio, Windows: WASAPI, Apple: AVAudioEngine) hinter einem gemeinsamen Interface `AudioEngine`.
- **ARC-007** Alle Rechenzeit-intensiven Aufgaben (Transkription, OMR, Rendering großer Partituren) laufen in Isolates/Hintergrund-Threads; die UI friert nie > 100 ms ein.
- **ARC-008** Lizenzen aller Abhängigkeiten werden in `THIRD_PARTY_LICENSES.md` dokumentiert; CI prüft, dass keine nicht gelistete Lizenz hinzukommt.

---

## 2. Datenmodell & Musiktheorie (Kern)

### 2.1 Tonhöhe (Pitch)
- **MOD-001** Eine Tonhöhe besteht aus Stammton (`C D E F G A B`), Alteration (−2 … +2) und Oktave (wissenschaftliche Notation, C4 = eingestrichenes c).
  - *Test:* `Pitch('C', 0, 4).midi == 60`; `Pitch('A', 0, 4).midi == 69`.
- **MOD-002** Umrechnung MIDI ↔ Pitch; bei MIDI → Pitch wird die Schreibweise abhängig von der aktuellen Tonart gewählt.
  - *Test:* MIDI 70 in F-Dur → `Bb4`; in E-Dur → `A#4`.
- **MOD-003** Enharmonische Gleichheit vs. Identität.
  - *Test:* `Pitch('F#',4).isEnharmonic(Pitch('Gb',4)) == true`, aber `==` liefert `false`.
- **MOD-004** Doppelkreuz/Doppel-b werden unterstützt.
  - *Test:* `Pitch('F', +2, 4).midi == 67` (Fisis = G).
- **MOD-005** Grenzfälle über Oktavgrenzen.
  - *Test:* `Cb4.midi == 59`, `B#3.midi == 60`.
- **MOD-006** Frequenz ↔ Pitch mit einstellbarem Kammerton (Standard A4 = 440 Hz, Bereich 415–466 Hz).
  - *Test:* A4 → 440.00 Hz; C4 → 261.63 Hz (±0.01); bei A4 = 442 Hz → A4 = 442.00 Hz.
- **MOD-007** Cent-Abweichung zu nächstem Halbton.
  - *Test:* 445 Hz bei A4 = 440 → `A4`, +19.6 Cent (±0.1).

### 2.2 Intervalle
- **MOD-010** Intervalle mit Größe (Prime … Undezime usw.) und Qualität (rein, groß, klein, übermäßig, vermindert).
  - *Test:* Intervall C4→E4 = große Terz; C4→Eb4 = kleine Terz; C4→F#4 = übermäßige Quarte; C4→Gb4 = verminderte Quinte.
- **MOD-011** Intervall-Halbtöne.
  - *Test:* übermäßige Quarte = 6, verminderte Quinte = 6, große Sexte = 9.
- **MOD-012** Intervall auf Pitch anwenden (auf- und abwärts), mit korrekter Schreibweise.
  - *Test:* E4 + kleine Terz = G4; F#4 + große Terz = A#4 (nicht Bb4); Bb3 − große Sekunde = Ab3.
- **MOD-013** Komplementärintervall / Umkehrung.
  - *Test:* Umkehrung große Terz = kleine Sexte.

### 2.3 Dauer & Rhythmus
- **MOD-020** Dauern werden als **exakte Brüche** gespeichert (kein Float), Ganze = 1.
  - *Test:* Viertel = 1/4; punktierte Viertel = 3/8; doppelt punktierte Halbe = 7/8.
- **MOD-021** Notenwerte von Ganze bis 64tel, plus Brevis.
- **MOD-022** Triolen und allgemeine N-tolen (z. B. 5:4, 3:2, 6:4).
  - *Test:* Achteltriole hat Dauer 1/12; drei Achteltriolen = 1/4.
- **MOD-023** Haltebögen (Tie) als Eigenschaft von Note zu Note; Dauer gebundener Noten summiert sich für Playback.
  - *Test:* Halbe C4 gebunden an Viertel C4 → Playback-Dauer 3/4.
- **MOD-024** Unterscheidung Haltebogen (gleiche Tonhöhe) vs. Legatobogen (Slur, beliebige Noten).
  - *Test:* Haltebogen zwischen C4 und D4 wird als Validierungsfehler gemeldet; Slur ist erlaubt.
- **MOD-025** Vorschlagsnoten (Acciaccatura/Appoggiatura) haben Dauer 0 im Taktmaß.
  - *Test:* Takt mit 4 Vierteln + 1 Vorschlag ist gültig voll.

### 2.4 Note, Pause, Akkord, Stimme
- **MOD-030** Note: Pitch, Dauer, Tie-Start/-Ende, Stimme, Lautstärke (Velocity 1–127), Artikulationen, Notenkopf-Typ.
- **MOD-031** Pause: Dauer, Stimme, optional „Ganztaktpause".
  - *Test:* Ganztaktpause in 3/4 hat Dauer 3/4, wird aber als Ganze-Pausensymbol gekennzeichnet.
- **MOD-032** Akkord = mehrere Noten mit gleichem Startzeitpunkt und gleicher Dauer in einer Stimme.
  - *Test:* Akkord C4-E4-G4 hat genau 3 Pitches, sortiert aufsteigend.
- **MOD-033** Bis zu 4 Stimmen pro Notenzeile.
  - *Test:* Ein Takt mit Stimme 1 (Halbe) und Stimme 2 (2 Viertel) gilt in 2/4 als vollständig.
- **MOD-034** Artikulationen: Staccato, Staccatissimo, Tenuto, Akzent, Marcato, Fermate, Dämpfer-Hinweise (con sord./senza sord.), Triller.
- **MOD-035** Dynamik: ppp–fff, sfz, fp, Crescendo/Decrescendo-Gabeln mit Start/Ende.
  - *Test:* Dynamik `mf` → Default-Velocity 80 (Mapping-Tabelle ist testbar).

### 2.5 Takt, Taktart, Tonart, Schlüssel
- **MOD-040** Taktarten: einfache (2/4, 3/4, 4/4), zusammengesetzte (6/8, 9/8, 12/8), unregelmäßige (5/4, 7/8, 5/8), alla breve (2/2), C-Symbol.
  - *Test:* 6/8 hat 2 Hauptschläge à punktierte Viertel; 4/4 hat 4 Schläge à Viertel.
- **MOD-041** Unregelmäßige Taktarten mit Gruppierung.
  - *Test:* 7/8 mit Gruppierung 2+2+3 liefert Schlagbeginne bei 0, 2/8, 4/8.
- **MOD-042** Takt-Validierung: Summe der Dauern je Stimme = Taktlänge.
  - *Test:* 4/4-Takt mit 3 Vierteln → `MeasureStatus.underfull`; mit 5 Vierteln → `overfull`.
- **MOD-043** Auftakt (Pickup): erster Takt darf kürzer sein und ist als Auftakt markiert.
  - *Test:* Auftakt mit einer Viertel in 4/4 ist gültig; Taktnummerierung beginnt mit 0 bzw. 1 laut Einstellung.
- **MOD-044** Tonarten: alle 15 Dur- und 15 Moll-Tonarten (bis 7 Vorzeichen), dazu Modi (Dorisch, Mixolydisch usw.).
  - *Test:* 2 Kreuze → D-Dur / h-Moll; 3 B → Es-Dur / c-Moll; D-Dorisch → 0 Vorzeichen.
- **MOD-045** Tonleitern aus Tonart ableitbar.
  - *Test:* G-Dur → G A B C D E F#; a-Moll harmonisch → A B C D E F G#.
- **MOD-046** Schlüssel: Violin, Bass, Alt, Tenor, Violin-8va-bassa (Gitarre), Schlagzeugschlüssel.
  - *Test:* Unterste Linie im Violinschlüssel = E4; im Bassschlüssel = G2; im Tenorschlüssel = D3; im Gitarrenschlüssel klingt E4-Position als E3.
- **MOD-047** Tonart-, Takt-, Schlüssel- und Tempowechsel mitten im Stück.
  - *Test:* Tonartwechsel ab Takt 9 → Note in Takt 10 wird mit neuer Tonart interpretiert.

### 2.6 Struktur & Partitur
- **MOD-050** Hierarchie: `Score → Part (Instrument) → Staff → Measure → Voice → Event`.
- **MOD-051** Klavier = 1 Part mit 2 Staffs (Akkolade/Grand Staff).
  - *Test:* Default-Klavierpart hat Staff 1 Violin-, Staff 2 Bassschlüssel.
- **MOD-052** Wiederholungen: Wiederholungszeichen, Volta-Klammern (1./2. Ende), D.C., D.S., Coda, Fine, Segno.
  - *Test:* Struktur `A ||: B :|| C` mit Volta → Playback-Reihenfolge der Takte wird korrekt „ausgerollt" (Unit-Test auf Taktindex-Liste).
- **MOD-053** Probenziffern/-buchstaben, Abschnittsnamen (Verse, Chorus …), Tempoangaben (BPM + Text wie „Andante").
- **MOD-054** Liedtext (Lyrics) pro Note, mehrere Strophen, Silbentrennung und Melismen.
  - *Test:* Silbe „Hal-" mit Trennstrich + „le-" + „lu-" + „ja" ergibt das Wort „Halleluja".
- **MOD-055** Metadaten: Titel, Untertitel, Komponist, Arrangeur, Texter, Copyright, Tonart des Originals.
- **MOD-056** Immutable Datenstrukturen + Command-Pattern für alle Änderungen (Grundlage für Undo/Redo).
  - *Test:* 100 Änderungen + 100× Undo → Score ist identisch (deep equal) mit dem Ausgangszustand.

---

## 3. Notensatz-Regeln (korrekte Darstellung)

Diese Logik liegt **im Core** (nicht in der Render-Engine), damit sie testbar ist. Verovio bekommt bereits korrekt aufbereitete Daten.

### 3.1 Vorzeichen
- **ENG-001** Ein Versetzungszeichen gilt bis zum Ende des Taktes für dieselbe Tonhöhe in derselben Oktave.
  - *Test:* In C-Dur: F#4, dann F4 im selben Takt → zweite Note bekommt ein Auflösungszeichen (♮).
- **ENG-002** Im nächsten Takt wird ohne Zeichen wieder die Tonart angenommen; optional Warnungsvorzeichen (Klammer).
  - *Test:* Takt 1: F#4, Takt 2: F4 → Takt 2 bekommt kein Pflicht-♮, aber (bei aktivierter Option) ein Höflichkeits-♮.
- **ENG-003** Haltebogen über den Taktstrich: Fortsetzung bekommt kein erneutes Vorzeichen.
  - *Test:* F#4 gebunden über Taktstrich → zweite Note ohne ♯-Symbol.
- **ENG-004** Andere Oktave: Vorzeichen gilt nicht automatisch (konservative Regel: Zeichen wird wiederholt).
  - *Test:* F#4, dann F#5 im selben Takt → F#5 bekommt ♯.

### 3.2 Notenhälse & Balken
- **ENG-010** Halsrichtung bei einer Stimme: Note unterhalb der Mittellinie → Hals nach oben, auf/über der Mittellinie → nach unten.
  - *Test:* Violinschlüssel: G4 → oben; B4 → unten; D5 → unten.
- **ENG-011** Bei zwei Stimmen: Stimme 1 Hals oben, Stimme 2 Hals unten, unabhängig von der Tonhöhe.
- **ENG-012** Akkord-Halsrichtung nach der am weitesten von der Mittellinie entfernten Note.
- **ENG-013** Balkengruppierung nach Taktart:
  - 4/4: Achtel in Vierergruppen pro halbem Takt (1–2, 3–4), nie über die Taktmitte.
  - 3/4: Achtel pro Viertel oder ganzer Takt gebalkt (Einstellung).
  - 6/8, 9/8, 12/8: Dreiergruppen.
  - 7/8 (2+2+3): Gruppen 2, 2, 3.
  - *Test:* 8 Achtel in 4/4 → 2 Balkengruppen à 4; 6 Achtel in 6/8 → 2 Gruppen à 3; 7 Achtel in 7/8 (2+2+3) → Gruppen [2, 2, 3].
- **ENG-014** Pausen unterbrechen Balken (Default) bzw. werden auf Wunsch überbalkt.

### 3.3 Rhythmische Darstellung
- **ENG-020** Noten, die über einen Taktstrich hinausgehen, werden in übergebundene Noten geteilt.
  - *Test:* Halbe auf Zählzeit 4 in 4/4 → Viertel (Takt n) + Viertel (Takt n+1) mit Haltebogen.
- **ENG-021** Noten werden an Schlaggrenzen sinnvoll geteilt, damit der Schlag sichtbar bleibt (in zusammengesetzten Taktarten an der punktierten Viertel).
  - *Test:* In 6/8 wird eine Viertel, die auf der 3. Achtel beginnt, als Achtel + Achtel mit Haltebogen dargestellt.
- **ENG-022** Pausen werden konsolidiert und nach Schlägen aufgeteilt (keine „Viertelpause über die Taktmitte" in 4/4).
  - *Test:* Pausen-Lücke Zählzeit 2–3 in 4/4 → zwei Viertelpausen (nicht eine Halbe).
- **ENG-023** Synkopen: Viertel auf „2+" in 4/4 bleibt Viertel (übliche Lesbarkeit), solange sie nicht die Taktmitte überquert; konfigurierbar.
- **ENG-024** Punktierungen werden bevorzugt, wenn sie einen Schlag nicht verdecken.
  - *Test:* Dauer 3/8 ab Zählzeit 1 in 4/4 → punktierte Viertel.

### 3.4 Tonumfang & Lesbarkeit
- **ENG-030** Automatische Schlüsselwahl für eine einzelne Stimme anhand des Tonumfangs.
  - *Test:* Melodie zwischen C2 und C3 → Bassschlüssel; zwischen C4 und C6 → Violinschlüssel.
- **ENG-031** Klavier: Aufteilung auf rechte/linke Hand über Splitpunkt (Default C4) oder Heuristik (Hand-Spannweite ≤ Dezime).
  - *Test:* Splitpunkt C4: Akkord C3-E3-G3 + Melodie E5 → Akkord im Bass-System, E5 im Violin-System.
- **ENG-032** Ab mehr als 4 Hilfslinien wird optional ein Oktavierungszeichen (8va/8vb) vorgeschlagen.
  - *Test:* Folge von Noten über C7 → Vorschlag 8va-Klammer über genau diese Noten.
- **ENG-033** Mehrtaktpausen in Einzelstimmen werden zusammengefasst.
  - *Test:* 12 leere Takte in der Hornstimme → 1 Mehrtaktpause „12".

### 3.5 Layout (Daten für Renderer)
- **ENG-040** Seitenformat (A4/Letter, Hoch/Quer), Ränder, Systemabstand, Notengröße, Takte pro Zeile (auto/fest).
- **ENG-041** Taktnummern an jedem Systemanfang (Default) oder jedem Takt.
- **ENG-042** Kopfzeile mit Titel/Komponist, Stimmname auf jeder Seite, Seitenzahlen.
- **ENG-043** Responsives Layout auf Bildschirm (fließend) vs. Seitenansicht (druckgetreu) umschaltbar.

---

## 4. Transposition

### 4.1 Grundfunktionen
- **TRN-001** Chromatisch um ±N Halbtöne (−24 … +24).
  - *Test:* C4 + 2 = D4; F4 − 1 = E4; B4 + 1 = C5 (Oktavübertrag).
- **TRN-002** Diatonisch/intervallisch (z. B. „große Sekunde aufwärts") mit korrekter Schreibweise.
  - *Test:* F#4 + große Sekunde = G#4 (nicht Ab4).
- **TRN-003** In Ziel-Tonart: Nutzer gibt nur die Tonart an, die App berechnet das Intervall.
  - *Test:* G-Dur → F-Dur: alle F# werden E, Tonartvorzeichnung 1♭.
- **TRN-004** Richtungswahl: „nächstgelegen" (Default), „aufwärts", „abwärts".
  - *Test:* C-Dur → B-Dur nächstgelegen = 1 Halbton abwärts; „aufwärts" = 11 Halbtöne aufwärts.
- **TRN-005** Dur bleibt Dur, Moll bleibt Moll; Modus bleibt erhalten.
  - *Test:* a-Moll um +2 → h-Moll (2♯), nicht D-Dur.
- **TRN-006** Akzidentien außerhalb der Tonart werden relativ korrekt mitgenommen.
  - *Test:* In C-Dur steht F#4 (Zwischendominante); C-Dur → D-Dur → F#4 wird G#4.
- **TRN-007** Enharmonik-Strategie: Zieltonart mit ≤ 6 Vorzeichen bevorzugen; Fis-Dur/Ges-Dur per Einstellung.
  - *Test:* H-Dur + 1 Halbton → C-Dur (nicht His-Dur); Des-Dur + 2 → Es-Dur.
- **TRN-008** Doppelvorzeichen vermeiden, wenn eine einfache enharmonische Schreibweise innerhalb der Tonart passt; sonst korrekt setzen.
  - *Test:* E-Dur-Stück mit B#4 (übermäßige Quinte im Akkord) → in Fis-Dur als C##5 korrekt, in Ges-Dur als C5.
- **TRN-009** Transposition eines **Bereichs** (Takte x–y, ausgewählte Stimmen) ohne Rest des Stücks.
  - *Test:* Nur Takt 5–8 transponiert; Takt 4 und 9 unverändert; Tonartwechsel-Markierungen automatisch an Takt 5 und 9 eingefügt.
- **TRN-010** Oktavtransposition (±12) ohne Tonartänderung.
- **TRN-011** Transposition ist umkehrbar.
  - *Test:* Score + 5 Halbtöne, dann − 5 Halbtöne → identisch zum Original (inkl. Schreibweise).
- **TRN-012** Akkordsymbole, Nashville-Zahlen (bleiben gleich!), Tabs und Tonart-Metadaten werden mittransponiert.
  - *Test:* Akkord „D/F#" in G-Dur → nach A-Dur „E/G#"; NNS-Anzeige bleibt „5/7".

### 4.2 Transponierende Instrumente (Brass-Fokus)
Interne Speicherung immer **klingend (Concert Pitch)**. Auszüge werden beim Anzeigen/Export umgerechnet.

- **TRN-020** Instrumentenprofile mit: Name, Stimmung, Transpositionsintervall, Schlüssel, Tonumfang (absolut / komfortabel), MIDI-Programm.
- **TRN-021** Mindestumfang der Profile:

  | Instrument | Notiert gegenüber klingend | Schlüssel |
  |---|---|---|
  | Trompete in C | gleich | Violin |
  | Trompete/Flügelhorn in B | große Sekunde höher | Violin |
  | Trompete in Es | kleine Terz tiefer | Violin |
  | Trompete in D | große Sekunde tiefer | Violin |
  | Piccolotrompete in A | große Sexte tiefer | Violin |
  | Horn in F | reine Quinte höher | Violin (tiefe Lagen Bass) |
  | Horn in Es | große Sexte höher | Violin |
  | Posaune / Bassposaune | gleich | Bass (Tenor optional) |
  | Posaune in B (Brass-Band-Notation) | große None höher | Violin |
  | Euphonium/Bariton (Bass) | gleich | Bass |
  | Euphonium in B (Violinschlüssel) | große None höher | Violin |
  | Tuba in C / F | gleich | Bass |
  | Tuba in Es (Violinschlüssel) | Oktave + große Sexte höher | Violin |
  | Tuba in B (Violinschlüssel) | 2 Oktaven + große Sekunde höher | Violin |
  | Gitarre | Oktave höher | Violin-8 |
  | E-Bass | Oktave höher | Bass-8 |
  | Klavier | gleich | Violin + Bass |

  - *Test:* klingend C4 → Trompete in B notiert D4; Horn in F notiert G4; Trompete in Es notiert A3; Tuba in Es: klingend C2 → notiert A3.
- **TRN-022** Tonart-Vorzeichnung der Stimme wird mitberechnet.
  - *Test:* Partitur in Es-Dur → Trompete in B: F-Dur (1♭); Horn in F: B-Dur (2♭).
- **TRN-023** Option „Horn ohne Tonartvorzeichnung" (traditionelle Notation mit Akzidentien).
- **TRN-024** Umschalter „klingend / notiert" für die ganze Partitur.
  - *Test:* Umschalten ändert die Anzeige, aber nicht das gespeicherte Datenmodell (Hash bleibt gleich).
- **TRN-025** Tonumfang-Prüfung nach jeder Transposition: Noten außerhalb des komfortablen Umfangs gelb, außerhalb des absoluten Umfangs rot markiert, mit Liste der Stellen.
  - *Test:* Trompete in B, notiert D6 → Warnung „über komfortablem Umfang"; notiert F#3 → ok; notiert E3 → Fehler „unter Umfang".
- **TRN-026** Vorschlag „Oktave tiefer/höher" für einzelne Passagen bei Umfangsproblemen.
- **TRN-027** Instrument einer Stimme nachträglich wechseln (z. B. Trompete in C → in B), Noten klingen danach gleich.
  - *Test:* Instrumentwechsel ändert notierte Noten um eine große Sekunde, klingende MIDI-Werte bleiben identisch.
- **TRN-028** Vorlage „Brass-Quintett" (2 Trp in B, Horn in F, Posaune, Tuba) und „Brass-Quintett in C".

### 4.3 Gitarren-Transposition
- **TRN-030** Kapodaster-Rechner: Klingende Tonart + Kapo-Bund → Griff-Tonart und umgekehrt.
  - *Test:* Song in A-Dur, Kapo 2 → Griffe in G-Dur; Song in Eb-Dur, Kapo 1 → Griffe in D-Dur.
- **TRN-031** Vorschlag der „besten" Kapo-Position (bevorzugte offene Griffe: C, G, D, A, E, Am, Em, Dm).
  - *Test:* Song in Bb-Dur → Vorschläge enthalten „Kapo 3, Griffe in G" und „Kapo 1, Griffe in A".
- **TRN-032** Alternative Stimmungen (Drop D, Halbton tiefer, DADGAD) werden bei Tabs berücksichtigt.

---

## 5. Audio: Metronom, Aufnahme, Transkription

### 5.1 Metronom
- **AUD-001** Tempo 30–300 BPM in 1er-Schritten, Feineinstellung ±0.5.
- **AUD-002** Klicks werden **sample-genau** im Audio-Thread erzeugt (nicht per Timer).
  - *Test:* Offline-Render bei 48 kHz, 120 BPM → Klick-Onsets exakt alle 24 000 Samples (±1 Sample); bei 7/8 mit 2+2+3 liegen die Akzente auf Achtel 1, 3, 5.
- **AUD-003** Langzeit-Stabilität.
  - *Test:* 10 Minuten offline gerendert bei 97 BPM → kumulierte Abweichung < 1 ms.
- **AUD-004** Taktart, Betonung der Eins, Akzentmuster (frei editierbar), Unterteilung (Achtel, Triolen, Sechzehntel).
- **AUD-005** Vorzähler (0, 1 oder 2 Takte) vor der Aufnahme.
  - *Test:* Vorzähler 1 Takt in 4/4 bei 60 BPM → Aufnahme-Zeitpunkt 0 liegt 4.0 s nach Start.
- **AUD-006** Tap-Tempo (Mittelwert der letzten 4–8 Taps, Ausreißer verworfen).
  - *Test:* Taps im Abstand 500, 510, 490, 500 ms → 120 BPM (±1).
- **AUD-007** Visuelles Metronom (blinkender Schlag, Taktposition) für Kopfhörer-lose Nutzung.
- **AUD-008** Verschiedene Klänge (Klick, Holzblock, Cowbell) und Lautstärke getrennt für Akzent/Normal.
- **AUD-009** Latenz-Kalibrierung: Gerät spielt Klicks, Mikrofon nimmt auf, gemessene Roundtrip-Latenz wird gespeichert und bei Transkription abgezogen.
  - *Test:* Simulation mit künstlich verzögertem Eingang (37 ms) → Kalibrierung ermittelt 37 ms ± 2 ms.
- **AUD-010** Tempo-Trainer (optional): Tempo steigt alle N Takte um X BPM.

### 5.2 Eingabequellen
- **AUD-020** **MIDI-Eingang** (USB-MIDI, Bluetooth-MIDI) – genaueste Methode für Digitalpianos.
  - *Test:* Eingespielte MIDI-Datei (simulierter Input-Stream) mit C-Dur-Akkord → 3 Noten C4/E4/G4 mit identischem Zeitstempel.
- **AUD-021** Sustain-Pedal (CC64) wird erfasst und optional als Pedalzeichen notiert.
- **AUD-022** Mikrofon-Aufnahme: 44.1/48 kHz, Mono, Pegelanzeige, Übersteuerungswarnung.
  - *Test:* Eingangssignal mit Spitzen bei 0 dBFS → Clipping-Warnung wird ausgelöst.
- **AUD-023** Import vorhandener Audiodateien (WAV, MP3, M4A, FLAC) zur Transkription.
- **AUD-024** Aufnahme wird immer als Audiodatei gespeichert, damit die Transkription mit anderen Einstellungen wiederholt werden kann.

### 5.3 Monophone Erkennung (eine Stimme)
- **AUD-030** Tonhöhenerkennung per YIN/pYIN (klassisch) oder CREPE-artigem Modell (optional).
  - *Test:* Synthetischer Sinus 440 Hz → A4; Sägezahn 261.63 Hz → C4; 466.16 Hz in F-Dur → Bb4.
- **AUD-031** Erkennungsbereich E1 (Bass) bis C8.
  - *Test:* Synthetische Töne über den gesamten Bereich, Fehlerrate < 1 %.
- **AUD-032** Robust gegen Rauschen.
  - *Test:* Sinus + weißes Rauschen bei 20 dB SNR → korrekter Ton in > 95 % der Frames.
- **AUD-033** Onset-/Offset-Erkennung inkl. wiederholter gleicher Töne.
  - *Test:* Zwei aufeinanderfolgende A4 mit 50 ms Pause → 2 Noten, nicht 1.
- **AUD-034** Vibrato und Bendings (Gitarre) werden auf den Mittelton abgebildet.
  - *Test:* A4 mit ±30 Cent Vibrato bei 5 Hz → eine Note A4.
- **AUD-035** Echtzeit-Anzeige der erkannten Note während des Spielens (Live-Vorschau).

### 5.4 Polyphone Erkennung (Klavier)
- **AUD-040** Polyphone Transkription über vortrainiertes Modell (z. B. Basic Pitch oder Onsets-and-Frames-artiges Piano-Modell), **nach** der Aufnahme (nicht in Echtzeit).
- **AUD-041** Ausgabe: Liste von Noten-Events (Onset, Offset, MIDI-Pitch, Velocity).
  - *Test:* Gerenderte Klavieraufnahme (aus bekannter MIDI-Datei per SoundFont) mit C-Dur-Akkord → genau C4, E4, G4 mit Onsets innerhalb von 30 ms.
- **AUD-042** Qualitätsmessung gegen Testkorpus (siehe Kap. 13).
  - *Test:* Note-F1 (Onset-Toleranz 50 ms) ≥ 0.80 auf dem internen Klavier-Testset aus synthetisch gerenderten Stücken.
- **AUD-043** Konfidenz pro Note; Noten unter Schwelle werden im Editor markiert.
- **AUD-044** Fortschrittsanzeige und Abbrechen-Funktion; Verarbeitung von 3 Minuten Audio auf Windows-Mittelklasse-PC < 60 s.

### 5.5 Quantisierung (Events → lesbare Noten)
- **AUD-050** Grid aus Metronom (Tempo + Taktart) ist bekannt; Onsets werden auf das kleinste eingestellte Raster gezogen (Viertel, Achtel, Sechzehntel).
  - *Test:* Onsets bei 120 BPM mit ±20 ms Zufalls-Jitter → alle korrekt auf Achtelraster.
- **AUD-051** Triolen-Erkennung (optional zuschaltbar).
  - *Test:* 3 gleichmäßige Onsets innerhalb einer Viertel → Achteltriole, nicht Achtel + 2 Sechzehntel.
- **AUD-052** Swing-Erkennung: Swing-Achtel werden als gerade Achtel mit „Swing"-Anweisung notiert.
  - *Test:* Achtelpaare im Verhältnis 2:1 → gerade Achtel + Tempotext „Swing".
- **AUD-053** Notenenden werden zur nächsten sinnvollen Dauer gerundet (Legato-Lücken < 1 Sechzehntel werden geschlossen).
  - *Test:* Viertel gespielt mit 420 ms bei 120 BPM (statt 500 ms) → Viertel, nicht Achtel + Pause.
- **AUD-054** Mindest-Notenwert einstellbar; kürzere Noten werden verworfen oder angehängt.
- **AUD-055** Toleranz gegenüber Tempo-Schwankungen: Beat-Tracking passt das Raster an, wenn der Spieler driftet.
  - *Test:* Aufnahme beschleunigt von 100 auf 104 BPM über 16 Takte → Taktgrenzen weiterhin korrekt.
- **AUD-056** Anschließend Anwendung der Notensatz-Regeln (Kap. 3): Taktaufteilung, Haltebögen, Balken.
- **AUD-057** Tonart-Erkennung aus den Tönen (Krumhansl-Schmuckler), überschreibbar.
  - *Test:* Tonfolge G-Dur-Tonleiter + Kadenz → G-Dur vorgeschlagen.
- **AUD-058** Freier Modus ohne Metronom: Tempo und Taktart werden geschätzt, Nutzer kann korrigieren.

### 5.6 Schlagzeug-Erkennung
- **AUD-060** Onset-Erkennung (Spectral Flux) auf Drum-Aufnahmen.
  - *Test:* Zwei tieffrequente Impulse im Abstand 1 s bei 60 BPM → 2 Onsets bei 0 s und 1 s (±10 ms).
- **AUD-061** Klassifikation mindestens Kick, Snare, Hi-Hat (geschlossen/offen); später Toms, Crash, Ride.
  - *Test:* Synthetisches Testset aus Drum-Samples → Klassifikationsgenauigkeit ≥ 85 % für Kick/Snare/Hi-Hat.
- **AUD-062** Notation im Schlagzeugschlüssel nach üblichem Mapping (Kick unterster Zwischenraum, Snare 3. Zwischenraum, Hi-Hat mit x-Notenkopf über dem System).
  - *Test:* Kick auf Schlag 1 und 3, Snare auf 2 und 4, Hi-Hat Achtel → korrekter Standard-Rockgroove mit 2 Stimmen (Hände oben, Füße unten).
- **AUD-063** MIDI-Drum-Mapping (General MIDI Kanal 10) für Import/Export und E-Drums per MIDI.
  - *Test:* GM-Note 36 → Kick, 38 → Snare, 42 → Hi-Hat geschlossen.

### 5.7 Zusatz-Werkzeuge
- **AUD-070** Stimmgerät (chromatisch, Kammerton einstellbar, Anzeige in Cent).
  - *Test:* 329.63 Hz → E4, 0 Cent ± 1.
- **AUD-071** Gehörbildung/Übung optional später (Intervalle erkennen).

---

## 6. Optical Music Recognition (Notenblatt-Scan)

### 6.1 Import
- **OMR-001** Quellen: Kamera, Bildergalerie (JPG, PNG, HEIC), PDF (mehrseitig), Scanner-Dateien (TIFF).
  - *Test:* 3-seitige PDF → 3 Seitenobjekte in richtiger Reihenfolge.
- **OMR-002** Mehrere Fotos zu einem Stück zusammenfügen, Reihenfolge per Drag & Drop.
- **OMR-003** Kamera-Assistent: Rahmen-Erkennung live, Hinweis bei Unschärfe/zu dunkel.
  - *Test:* Bild mit Laplace-Varianz unter Schwellwert → „Bild unscharf"-Warnung.

### 6.2 Vorverarbeitung
- **OMR-010** Automatisches Erkennen der Blattkanten und perspektivische Entzerrung.
  - *Test:* Synthetisch um 20° gekipptes und perspektivisch verzerrtes Testbild → Ergebnis-Rechteck, Notenlinien weichen < 0.5° von der Horizontalen ab.
- **OMR-011** Drehkorrektur (Deskew) anhand der Notenlinien.
  - *Test:* Um 3° rotiertes Bild → nach Korrektur < 0.2°.
- **OMR-012** Adaptive Binarisierung (z. B. Sauvola) für vergilbtes Papier und ungleichmäßige Beleuchtung.
  - *Test:* Bild mit Helligkeitsverlauf 30 %–90 % → Notenlinien bleiben durchgehend erhalten (Pixel-Vergleich mit Ground Truth ≥ 95 %).
- **OMR-013** Entfernen von Flecken/Rauschen, Wiederherstellen unterbrochener Linien.
- **OMR-014** Manuelle Korrektur der Eckpunkte und des Ausschnitts.
- **OMR-015** Leichte Seitenwölbung (Buchfalz) ausgleichen.

### 6.3 Strukturerkennung
- **OMR-020** Notenlinien erkennen, Linienabstand (Staff Space) bestimmen.
  - *Test:* Testbild mit 10 Systemen à 5 Linien → 50 Linien, 10 Systeme; Linienabstand ±1 px.
- **OMR-021** Systeme zu Akkoladen gruppieren (Klavier: 2 Systeme; Brass-Quintett: 5 Systeme).
  - *Test:* Klavier-Testseite → jede Akkolade hat 2 Systeme.
- **OMR-022** Taktstriche, Doppelstriche, Wiederholungszeichen.
- **OMR-023** Schlüssel, Tonartvorzeichnung, Taktart.
  - *Test:* Violinschlüssel mit 2 Kreuzen → `Clef.treble`, `KeySignature(D-Dur)`; „C"-Symbol → 4/4; „¢" → 2/2.
- **OMR-024** Instrumentennamen am Systemanfang per Texterkennung (OCR) → automatische Instrumentenzuordnung inkl. Transposition.
  - *Test:* Text „Trumpet in B♭" / „Trompete in B" → Profil „Trompete in B".

### 6.4 Symbolerkennung
- **OMR-030** Notenköpfe (gefüllt, hohl, Ganze), Hälse, Fähnchen, Balken, Punkte.
- **OMR-031** Pausen aller Werte, Mehrtaktpausen mit Zahl.
- **OMR-032** Vorzeichen (♯ ♭ ♮ 𝄪 𝄫).
- **OMR-033** Haltebögen/Legatobögen, Triolen-Klammern.
- **OMR-034** Dynamik, Artikulationen, Fermaten.
- **OMR-035** Text: Titel, Komponist, Tempoangaben, Liedtext, Akkordsymbole (OCR).
- **OMR-036** Ausgabe als internes Datenmodell (über MusicXML-Zwischenformat zulässig).
- **OMR-037** Konfidenzwert pro Symbol.

### 6.5 Plausibilisierung & Korrektur
- **OMR-040** Taktsummen-Prüfung: Takte, deren Dauer nicht zur Taktart passt, werden rot markiert.
  - *Test:* Erkennungsergebnis mit fehlendem Punkt → Takt als `underfull` markiert.
- **OMR-041** Automatische Korrekturvorschläge für häufige Fehler (z. B. fehlender Punkt, fehlende Triole).
  - *Test:* Takt 4/4 mit 7/8 Inhalt, in dem eine Viertel steht → Vorschlag „punktierte Viertel".
- **OMR-042** Korrekturansicht: Originalbild und erkannte Noten übereinander (Overlay) oder nebeneinander, synchron gescrollt.
- **OMR-043** Klick auf ein erkanntes Symbol hebt die Bildstelle hervor und umgekehrt.
- **OMR-044** Vergleichsfunktion: Erkanntes Ergebnis abspielen, um Fehler zu hören.

### 6.6 Engine-Strategie
- **OMR-050** OMR-Engine hinter Interface; erste Implementierung mit einer vorhandenen Open-Source-Engine (Kandidaten: Audiveris, oemer, homr), Lizenz vor Integration prüfen.
- **OMR-051** Desktop (Windows) zuerst; auf Mobilgeräten darf OMR zunächst als „auf PC verarbeiten" gelöst sein.
- **OMR-052** Qualitätsziel für **sauber gedruckte** Noten: Symbol-Fehlerrate ≤ 10 % auf dem internen Testkorpus. Für alte Kopien: Fehlerrate wird gemessen und dokumentiert, kein festes Ziel.

---

## 7. Editor

- **EDT-001** Auswahl: einzelne Note, Bereich (Takte), Stimme, ganze Partitur.
- **EDT-002** Noteneingabe per Maus/Touch (Notenwert wählen, Position klicken).
- **EDT-003** Tastatur-Eingabe: Buchstaben A–G für Tonhöhe, Zahlen für Notenwert, Pfeil hoch/runter für Halbton, Strg+Pfeil für Oktave.
  - *Test:* Eingabefolge `5 C D E F` (Viertel) in leeren 4/4-Takt → C4 D4 E4 F4.
- **EDT-004** Step-Eingabe per MIDI-Keyboard (Taste drücken = Note an Cursor-Position).
- **EDT-005** Echtzeit-Eingabe per MIDI mit Metronom (siehe Kap. 5).
- **EDT-006** Kopieren, Ausschneiden, Einfügen, Löschen, Duplizieren von Bereichen, auch zwischen Stimmen.
  - *Test:* Takt 1–2 kopieren, bei Takt 5 einfügen → Takte 5–6 identisch mit 1–2.
- **EDT-007** Undo/Redo unbegrenzt in der Sitzung.
- **EDT-008** Takte einfügen/löschen, Taktart/Tonart/Schlüssel ab Position ändern.
  - *Test:* Taktart von 4/4 auf 3/4 ab Takt 5 ändern → Inhalt wird neu verteilt, keine Note geht verloren (Summe der Dauern gleich).
- **EDT-009** Enharmonische Umdeutung einer ausgewählten Note (F# ↔ Gb).
- **EDT-010** Stimmen tauschen, Noten zwischen Systemen verschieben (Klavier rechte/linke Hand).
- **EDT-011** Instrumente hinzufügen/entfernen/umbenennen, Reihenfolge ändern.
- **EDT-012** Autosave alle 30 s und bei App-Wechsel.
  - *Test:* Simulierter Absturz nach Änderung → beim Neustart wird letzter Autosave angeboten.

---

## 8. Ansichten & Modi

- **VIEW-001** **Partitur** (alle Stimmen).
- **VIEW-002** **Einzelstimmen-Auszug** pro Instrument, automatisch transponiert, mit Mehrtaktpausen.
  - *Test:* Brass-Quintett-Partitur → 5 Auszüge; Hornstimme hat Horn-in-F-Transposition und eigenen Titelkopf.
- **VIEW-003** **Leadsheet** (Melodie + Akkordsymbole + Text).
- **VIEW-004** **Chord Chart** (nur Akkorde + Struktur, Taktstriche, Rhythmus-Slashes).
- **VIEW-005** **Nashville-Ansicht** (Kap. 9).
- **VIEW-006** **Tabulatur** und Noten + Tab kombiniert.
- **VIEW-007** **Pianorolle** (Zeit vs. Tonhöhe), hilfreich zur Kontrolle der Transkription.
- **VIEW-008** **Übungsmodus:** Abschnitt loopen, Tempo 25–150 %, Stimmen stummschalten (z. B. „spiel mit, ohne meine Stimme"), Metronom dazu.
  - *Test:* Loop Takt 9–12 bei 50 % Tempo → Playback-Events liegen nur in Takt 9–12, Zeitabstände verdoppelt.
- **VIEW-009** **Auftrittsmodus:** Vollbild, große Darstellung, Seitenwechsel per Tippen oder Bluetooth-Pedal (Tastatur-Events Bild↑/Bild↓), Halbseiten-Umblättern.
- **VIEW-010** **Setlist:** Songs in Reihenfolge, je Song gewünschte Tonart/Kapo, Export der ganzen Setlist als PDF.
- **VIEW-011** Dunkelmodus und Nachtmodus (rote Schrift) für Bühne.
- **VIEW-012** Zoom und Anpassung an Bildschirmbreite; Querformat auf Tablets mit Doppelseite.

---

## 9. CCM & Gitarre

### 9.1 Akkordsymbole
- **CCM-001** Parser für Akkordsymbole mit Grundton, Qualität, Erweiterungen, Alterationen, Sus, Add, Slash-Bass.
  - *Test:* Folgende Strings werden korrekt geparst: `C`, `Cm`, `C7`, `Cmaj7`, `Cm7`, `C°`/`Cdim7`, `Cø`/`Cm7b5`, `Csus2`, `Gsus4`, `Cadd9`, `C6`, `C9`, `C7#9`, `C7b9`, `D/F#`, `G/B`, `Bb`, `F#m`, `C(no3)`/`C5`.
- **CCM-002** Ungültige Strings werden mit Fehlerposition gemeldet.
  - *Test:* `H7xyz` → Fehler an Position 2 (H wird als B akzeptiert, wenn deutsche Notation aktiv).
- **CCM-003** Deutsche Notation (H statt B, B statt Bb) als Anzeigeoption.
  - *Test:* Einstellung „Deutsch" → `Bb` wird als `B`, `B` als `H` angezeigt.
- **CCM-004** Akkord → Töne (für Playback und Tabs).
  - *Test:* `Cmaj7` → C E G B; `D/F#` → F# im Bass, darüber D F# A.
- **CCM-005** Akkorderkennung aus Noten (umgekehrte Richtung): vertikale Klänge werden benannt.
  - *Test:* Noten G2, B3, D4, F4 → `G7`; Noten F#2, D3, A3, D4 → `D/F#`.
- **CCM-006** Akkordsymbole über Noten positionierbar, auch mitten im Takt.

### 9.2 Nashville Number System (NNS)
- **CCM-010** Umschaltung absolute Akkorde ↔ Stufen.
  - *Test:* In G-Dur: `C` → `4`, `Em` → `6m`, `D/F#` → `5/7`, `Am7` → `2m7`, `Bb` → `b3`.
- **CCM-011** Moll-Schreibweise wählbar (`6m` oder `6-`).
- **CCM-012** NNS-Rhythmuszeichen: Akkord über mehrere Schläge (Unterstrich/Diamant für gehaltene Akkorde, Push-Zeichen „^").
- **CCM-013** NNS-Chart-Ansicht mit Abschnitten (Intro, V1, C, B, Tag, Outro), mehrere Akkorde pro Takt in Klammern.
  - *Test:* Takt mit `G` (2 Schläge) + `C` (2 Schläge) in G-Dur → Darstellung `(1 4)`.
- **CCM-014** NNS bleibt bei Transposition unverändert, nur die Tonartangabe ändert sich.

### 9.3 ChordPro & Songstruktur
- **CCM-020** Import/Export ChordPro (`.cho`, `.chopro`, `.chordpro`) inkl. `{title}`, `{key}`, `{tempo}`, `{capo}`, `{start_of_chorus}` usw.
  - *Test:* ChordPro-Datei mit `[G]Amazing [C]grace` → Liedzeile mit Akkorden über den richtigen Silben; Export → identischer Text.
- **CCM-021** Songstruktur-Editor: Abschnitte definieren und Reihenfolge (Arrangement) festlegen, z. B. `Intro V1 C V2 C B C C Outro`.
  - *Test:* Arrangement-Änderung erzeugt neue Playback-Reihenfolge ohne Kopieren der Noten.
- **CCM-022** Text mit Akkorden (Lyrics-Chart) als eigener Ausgabe-Typ (PDF), schmal für Notenständer/Tablet.
- **CCM-023** Tonart-Vorschlag nach Gesangsumfang: Nutzer gibt tiefsten/höchsten Ton des Sängers an, App schlägt Tonarten vor, in denen die Melodie hineinpasst.
  - *Test:* Melodie-Umfang A3–D5, Sänger G3–C5 → Vorschlag G-Dur (1 Ganzton tiefer).

### 9.4 Kompositionshilfen
- **CCM-030** Diatonische Akkorde der aktuellen Tonart als Palette (1 2m 3m 4 5 6m 7°) plus häufige Leihakkorde (b7, 4m, b6).
  - *Test:* D-Dur → D, Em, F#m, G, A, Bm, C#°.
- **CCM-031** Bibliothek gängiger Akkordfolgen (z. B. 1-5-6-4, 6-4-1-5, 1-4-6-5, 4-1-5-6) – mit einem Klick in beliebiger Tonart einfügen.
  - *Test:* Folge 1-5-6-4 in E-Dur → E, B, C#m, A.
- **CCM-032** Harmonisierungs-Vorschlag: zu einer Melodie passende Akkorde pro Takt (regelbasiert: Melodieton im Akkord bevorzugt).
  - *Test:* Melodie „G A B C" auf den Schlägen eines Taktes in C-Dur → Vorschlag enthält C oder G.
- **CCM-033** Ideen-Recorder: Melodie summen/spielen → monophone Transkription → als Skizze speichern.
- **CCM-034** Rhythmus-Slash-Notation mit Anschlagsmustern für Gitarre.
- **CCM-035** Playback mit einfacher Begleitung (Akkorde als Pad/Klavier, Bassgrundton, einfacher Drum-Groove), um Songideen zu hören.

### 9.5 Gitarren-Tabulaturen
- **TAB-001** Tab-Erzeugung aus Noten: optimale Bund/Saiten-Zuordnung (dynamische Programmierung, minimiert Handbewegung; bevorzugt Lage).
  - *Test:* E-Dur-Tonleiter E2–E3 in Standardstimmung → alles innerhalb der Bünde 0–4.
- **TAB-002** Spielbarkeit: keine zwei gleichzeitigen Töne auf derselben Saite, maximale Spannweite 4–5 Bünde.
  - *Test:* Akkord mit zwei Tönen, die nur auf derselben Saite möglich sind → Warnung „nicht spielbar".
- **TAB-003** Stimmungen: Standard, Drop D, Eb-Standard, DADGAD, Open G, 7-Saiter; 4/5-saitiger Bass.
  - *Test:* Drop D: D2 → 6. Saite Bund 0.
- **TAB-004** Kapo berücksichtigen (Bundzahlen relativ zum Kapo).
- **TAB-005** Techniken: Hammer-on, Pull-off, Slide, Bend (mit Zielton), Palm Mute, Let Ring, Dead Note.
- **TAB-006** Akkorddiagramme (Griffbilder) für Akkordsymbole, mehrere Voicings wählbar.
  - *Test:* `G` → mindestens Voicing 320003; `F` → Barré 133211.
- **TAB-007** Tab-Text-Import (ASCII-Tabs) – einfache Form.

---

## 10. Import & Export

- **IO-001** **MusicXML** 4.0 lesen/schreiben (partwise; `.musicxml`, `.xml`, komprimiert `.mxl`).
  - *Test:* Roundtrip: Datenmodell → MusicXML → Datenmodell → identisch (deep equal) für alle Dateien im Testkorpus.
  - *Test:* Export validiert gegen das offizielle MusicXML-XSD.
- **IO-002** Import von MusicXML-Exporten aus MuseScore, Sibelius, Finale, Dorico (je mindestens 3 Beispieldateien im Testkorpus).
  - *Test:* Jede Beispieldatei importiert ohne Fehler; Anzahl der Takte, Noten und Stimmen stimmt mit Referenzwerten überein.
- **IO-003** **MIDI** (SMF Typ 0 und 1) Import und Export.
  - *Test:* Export eines 4/4-Stücks in 120 BPM → MIDI enthält Tempo-Meta-Event 500 000 µs/Viertel und Taktart-Meta 4/4.
  - *Test:* Import einer MIDI-Datei → Quantisierung (Kap. 5.5) erzeugt gültige Takte.
- **IO-004** **MuseScore** (`.mscz`/`.mscx`) Import (Phase später; Format ist XML in ZIP).
- **IO-005** **ChordPro** Import/Export (siehe CCM-020).
- **IO-006** **ABC-Notation** Import/Export (einfache Melodien, Folk/Hymnen).
  - *Test:* `X:1\nK:G\nGABc|` → 4 Noten G4 A4 B4 C5 in G-Dur.
- **IO-007** **MEI** Export (für Verovio und Musikwissenschaft) – optional.
- **IO-008** **PDF**-Export (Partitur, alle Einzelstimmen in einem Rutsch, Leadsheet, Chord Chart, NNS-Chart, Tabs).
  - *Test:* PDF-Export eines 3-seitigen Stückes → 3 PDF-Seiten, Text „Titel" ist durchsuchbar.
- **IO-009** **SVG/PNG**-Export einzelner Seiten oder Ausschnitte.
- **IO-010** **Audio**-Export (WAV, optional MP3) über interne Wiedergabe.
- **IO-011** Guitar-Pro-Import (`.gp`, `.gp5`) – nur als spätere Option (Format teils proprietär).
- **IO-012** Proprietäre Formate von Finale (`.musx`), Sibelius (`.sib`) und Dorico werden **nicht** direkt unterstützt; Nutzer wird auf MusicXML-Export hingewiesen.
- **IO-013** Teilen-Funktion des Betriebssystems (Android Share-Sheet / Windows Share).
- **IO-014** Robustheit: Fehlerhafte Dateien führen nie zum Absturz, sondern zu einer verständlichen Meldung.
  - *Test:* Fuzzing mit 1 000 zufällig beschädigten MusicXML-Dateien → keine unbehandelte Exception.

---

## 11. Wiedergabe (Playback)

- **PLY-001** Wiedergabe per integriertem Software-Synth mit SoundFont (General MIDI; Klavier, Trompete, Horn, Posaune, Tuba, Gitarre, Bass, Drums).
- **PLY-002** Transponierende Instrumente klingen immer korrekt (klingende Tonhöhe).
  - *Test:* Trompete in B notiert D4 → erzeugtes MIDI-Event Note 60 (C4).
- **PLY-003** Wiederholungen, Volta, D.C./D.S./Coda werden beim Abspielen befolgt.
- **PLY-004** Dynamik → Velocity, Artikulationen → Notenlänge (Staccato 50 %), Fermate verlängert.
  - *Test:* Staccato-Viertel bei 120 BPM → Note-Off nach 250 ms.
- **PLY-005** Mixer: Lautstärke, Mute, Solo pro Stimme; Metronom zuschaltbar.
- **PLY-006** Wiedergabe-Cursor folgt der Note in der Notenansicht, Seite scrollt automatisch.
- **PLY-007** Wiedergabe ab ausgewählter Note/Takt.
- **PLY-008** MIDI-Ausgang an externe Geräte (optional).

---

## 12. Datenhaltung

- **DAT-001** Eigenes Projektformat `.tts` (ZIP mit `score.json`, optional Original-Audio und Scan-Bilder).
- **DAT-002** Versionsnummer im Format; alte Versionen werden automatisch migriert.
  - *Test:* Datei im Format v1 öffnen → nach Migration identischer Inhalt wie v2-Referenz.
- **DAT-003** Bibliothek: Liste aller Stücke mit Suche (Titel, Komponist, Tonart, Tags), Sortierung, Ordner/Tags (z. B. „Quintett", „Gottesdienst", „Uni").
  - *Test:* Suche „Amazing" findet „Amazing Grace" in < 100 ms bei 1 000 Stücken.
- **DAT-004** Versionen eines Stücks speichern (z. B. „Original", „in F für Trompete").
- **DAT-005** Lokale Backup-Datei (alle Projekte als ein ZIP) exportieren/importieren.
- **DAT-006** Keine Daten verlassen das Gerät ohne aktive Nutzeraktion.

---

## 13. Qualität, Testkorpus & Metriken

- **QA-001** Testkorpus im Repo unter `test/fixtures/`:
  - MusicXML: mind. 20 Stücke (Klavier, Brass-Quintett, Leadsheets, Sonderfälle: Triolen, Taktwechsel, Volta, Mehrstimmigkeit).
  - Audio: synthetisch aus MIDI gerendert (exakte Ground Truth) + einige echte Aufnahmen (Klavier, Gitarre, Drums).
  - Bilder: gemeinfreie Notenblätter (z. B. aus IMSLP, nur Public Domain) + eigene Fotos mit manuell erstellter Ground-Truth-MusicXML.
- **QA-002** Metriken werden automatisch berechnet und als Bericht ausgegeben:
  - Transkription: Note-Precision/Recall/F1 (Onset ±50 ms, mit/ohne Offset).
  - Quantisierung: Anteil korrekt notierter Notenwerte.
  - OMR: Symbol-Fehlerrate, Anteil vollständig korrekter Takte.
- **QA-003** Regressionsschutz: Verschlechtert sich eine Metrik um mehr als 2 Prozentpunkte, schlägt CI fehl.
- **QA-004** Golden-Tests für Rendering: SVG-Ausgabe wird mit gespeicherter Referenz verglichen.
- **QA-005** Property-based Tests für Musiktheorie (z. B. „transponieren um +n und −n ergibt Original" für zufällige Stücke).
- **QA-006** Testabdeckung im Core ≥ 90 %, in `io` ≥ 85 %.

---

## 14. Nicht-funktionale Anforderungen

- **NFR-001** App-Start < 3 s auf Mittelklasse-Android-Gerät.
- **NFR-002** Partitur mit 5 Stimmen × 200 Takten: Scrollen flüssig (≥ 50 fps), Transposition < 500 ms.
- **NFR-003** Metronom-Jitter < 1 ms (sample-genau, siehe AUD-002); Audio-Ausgabelatenz auf Android ≤ 50 ms bei unterstützten Geräten.
- **NFR-004** OMR einer A4-Seite auf Windows-PC < 30 s.
- **NFR-005** Speicherverbrauch < 500 MB bei normaler Nutzung (ohne laufende ML-Modelle).
- **NFR-006** Barrierefreiheit: Schriftgrößen skalierbar, Kontrast nach WCAG AA, Bedienung per Tastatur (Desktop).
- **NFR-007** Internationalisierung: alle Texte über Übersetzungsdateien (de, en).
- **NFR-008** Fehlerprotokoll lokal (keine Telemetrie ohne Zustimmung).
- **NFR-009** Läuft vollständig offline; ML-Modelle sind in der App enthalten oder werden einmalig heruntergeladen (optional).

---

## 15. Risiken

| Risiko | Auswirkung | Gegenmaßnahme |
|---|---|---|
| OMR bei alten/vergilbten Kopien ungenau | Viel Handkorrektur | Gute Vorverarbeitung, Overlay-Korrektur-UI, Taktsummen-Prüfung |
| Polyphone Audio-Transkription fehlerhaft | Falsche Noten | MIDI-Eingang als Hauptweg für Klavier, Konfidenzmarkierung, Pianorolle zur Korrektur |
| Quantisierung erzeugt unlesbare Rhythmen | Unbrauchbare Noten | Metronom als Raster, einstellbarer Mindestnotenwert, Notensatz-Regeln |
| Audio-Latenz auf Android | Metronom/Aufnahme versetzt | Oboe/AAudio, Latenz-Kalibrierung |
| Lizenzen (AGPL/GPL von Engines) | Einschränkung der Veröffentlichung | Lizenzen vor Integration prüfen, Projekt als Open Source unter kompatibler Lizenz |
| Plattform-Unterschiede (4 Systeme) | Mehraufwand | Core in reinem Dart, plattformspezifischer Code nur hinter Interfaces |
| Projektumfang | Nie fertig | Klare MVP-Definition, Phasen mit eigenem Nutzen |

---

## 16. Entwicklungsplan (TDD)

**Arbeitsweise in jeder Phase:**
1. Anforderungen der Phase (IDs) auswählen.
2. Tests schreiben (rot).
3. Minimalen Code schreiben, bis Tests grün sind.
4. Refaktorieren, Tests bleiben grün.
5. Phase ist fertig, wenn alle Tests grün sind und die *Definition of Done* erfüllt ist.

Commit-Konvention: Jeder Commit nennt die betroffene(n) ID(s), z. B. `feat(core): TRN-003 Transposition in Zieltonart`.

### Phase 0 – Projektgerüst
- Flutter-Projekt, Core-Package, Ordnerstruktur nach ARC-003.
- CI (GitHub Actions): `dart analyze`, `dart test`, Coverage-Bericht, Lizenzcheck.
- **DoD:** Leerer Test läuft in CI auf Linux-Runner; App startet auf Android und Windows mit Platzhalter-Bildschirm.

### Phase 1 – Musiktheorie-Kern
- MOD-001 … MOD-056 (Pitch, Intervall, Dauer, Taktart, Tonart, Schlüssel, Score-Struktur, Undo).
- **DoD:** Alle MOD-Tests grün, Coverage Core ≥ 90 %.

### Phase 2 – Transposition
- TRN-001 … TRN-012, Instrumentenprofile TRN-020 … TRN-028, Kapo TRN-030 … TRN-031.
- **DoD:** Brass-Quintett-Beispiel lässt sich per Unit-Test in jede Tonart transponieren; Umfangswarnungen korrekt.

### Phase 3 – Notensatz-Logik
- ENG-001 … ENG-043 (Vorzeichen, Balken, Teilung, Pausen, Schlüssel, Klavier-Split).
- **DoD:** Für jeden Testfall ist die vorbereitete Datenstruktur korrekt (ohne Grafik).

### Phase 4 – MusicXML & MIDI
- IO-001 … IO-003, IO-014, Testkorpus QA-001 (MusicXML-Teil).
- **DoD:** Roundtrip-Tests grün, Dateien aus MuseScore importierbar.

### Phase 5 – Rendering & Basis-UI (erstes nutzbares Produkt = **MVP**)
- ARC-004, VIEW-001, VIEW-002, VIEW-012, Bibliothek DAT-001 … DAT-003, PDF-Export IO-008.
- **DoD:** MusicXML öffnen → anzeigen → transponieren → Einzelstimmen als PDF exportieren, auf Android und Windows.
- *Ab hier ist die App für das Brass-Quintett bereits nützlich.*

### Phase 6 – Editor & Playback
- EDT-001 … EDT-012, PLY-001 … PLY-007.
- **DoD:** Stück von Hand eingeben, abspielen, speichern.

### Phase 7 – Metronom & MIDI-Eingang
- AUD-001 … AUD-010, AUD-020 … AUD-021, Quantisierung AUD-050 … AUD-057.
- **DoD:** Digitalpiano per USB anschließen, mit Metronom einspielen → korrekte Noten im Klaviersystem.

### Phase 8 – CCM-Funktionen
- CCM-001 … CCM-035, VIEW-003 … VIEW-005, VIEW-009 … VIEW-010, IO-005.
- **DoD:** ChordPro importieren, NNS-Chart anzeigen, Setlist als PDF.

### Phase 9 – Audio-Transkription (Mikrofon)
- Monophon AUD-030 … AUD-035, dann Polyphon AUD-040 … AUD-044, Stimmgerät AUD-070.
- **DoD:** Metriken nach QA-002 erreicht; Klavier-Aufnahme per Mikrofon → bearbeitbare Noten.

### Phase 10 – OMR
- OMR-001 … OMR-052 (zuerst Windows, gedruckte saubere Noten).
- **DoD:** Symbol-Fehlerrate ≤ 10 % auf sauberem Testkorpus, Korrektur-UI funktionsfähig.

### Phase 11 – Tabs & Schlagzeug
- TAB-001 … TAB-007, AUD-060 … AUD-063.

### Phase 12 – Weitere Formate & Plattformen
- IO-004, IO-006, IO-007, IO-011; macOS- und iOS-Build.

---

## 17. Glossar

- **Klingend (Concert Pitch):** tatsächlich hörbare Tonhöhe.
- **Notiert:** Tonhöhe, wie sie im Stimmauszug eines transponierenden Instruments steht.
- **Akkolade:** mehrere gleichzeitig gelesene Notensysteme (z. B. Klavier rechts/links).
- **Quantisierung:** Einrasten gespielter, ungenauer Zeitpunkte auf ein Notenraster.
- **OMR:** Optical Music Recognition, „Texterkennung für Noten".
- **NNS:** Nashville Number System, Akkorde als Stufenzahlen.
- **Onset:** Anschlagzeitpunkt eines Tons.
