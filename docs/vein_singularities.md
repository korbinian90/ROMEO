# Phasen-Singularitäten in Venen — Analyse, Lösungsansätze und Evaluationsplan

Status: Design-Dokument / Diskussionsgrundlage. Noch keine Implementierung.
Betrifft: `ROMEO.jl` (Unwrapping-Kern), `MriResearchTools.jl` (Pipeline), `romeo` CLI.

---

## 1. Warum treten in Venen Singularitäten auf?

### 1.1 Was eine Singularität formal ist

Eine Phasen-Singularität (Residuum, Vortex) ist ein Ort, an dem das Umlaufintegral der
gewrappten Phasendifferenzen um eine geschlossene Schleife nicht verschwindet. Diskret,
für eine elementare 2×2-Plakette mit Ecken `a,b,c,d`:

```
q = [ W(φb−φa) + W(φc−φb) + W(φd−φc) + W(φa−φd) ] / 2π  ∈ {−1, 0, +1}
W(x) = mod(x + π, 2π) − π
```

`q ≠ 0` heißt: das Feld der gewrappten Gradienten ist **nicht rotationsfrei**. Damit
existiert *kein* ganzzahliges Feld `n(x)`, sodass `φ_unwrapped = φ_wrapped + 2π·n`
überall stetig ist. Das ist eine **topologische Obstruktion**, kein Implementierungsfehler.

In 3D ist das Residuenfeld `q` divergenzfrei (die Summe von `q` über die 6 Flächen eines
Würfels ist 0). Residuen sind daher in 3D **keine Punkte, sondern geschlossene Linien
(Loops)**, die entweder in sich geschlossen sind oder am Volumenrand enden. Um eine Vene
herum bilden sie typischerweise Ringe, die das Gefäß umschließen.

### 1.2 Der physikalische Mechanismus in Venen

Eine Singularität liegt genau dort, wo das komplexe Signal verschwindet: `Re(S) = 0` **und**
`Im(S) = 0`. Zwei Bedingungen in 3D ⇒ generisch eine 1D-Nullstellenlinie — genau die
Vortex-Linie von oben. Es braucht also nur eine Nullstelle des komplexen Signals, um
eine Singularität zu erzeugen. In Venen passiert das aus mehreren, sich verstärkenden Gründen:

1. **Suszeptibilitätssprung venöses Blut ↔ Gewebe.**
   Δχ ≈ 0.3–0.45 ppm (deoxygeniert, Hct-abhängig). Das erzeugt ein Dipolfeld mit sehr
   steilen Gradienten unmittelbar an der Gefäßwand. Bei 7 T sind Frequenz-Offsets von
   einigen 10 Hz über Sub-Millimeter-Distanzen normal.

2. **Partialvolumen an der Gefäßwand — der Hauptmechanismus.**
   Ein Randvoxel enthält zwei Spin-Populationen: `S = a·e^{iφ₁} + b·e^{iφ₂}`.
   Wenn `a ≈ b` und `φ₁ − φ₂ ≈ π`, gilt `S ≈ 0`. Da sich sowohl das Mischungsverhältnis
   `a/b` (räumlich) als auch die Phasendifferenz (räumlich und mit TE) kontinuierlich
   ändern, wird die Nullstellenbedingung an einer 1D-Menge exakt erfüllt. Bei einer
   Frequenzdifferenz von ~25 Hz ist `φ₁ − φ₂ = π` bereits bei TE = 20 ms erreicht — also
   genau in dem TE-Bereich, den man für SWI/QSM verwendet.

3. **Intravoxel-Dephasierung im Gefäßinneren und in der Umgebung.**
   Die Feldverteilung innerhalb eines Voxels ist breit; die Vektorsumme über das Voxel
   läuft durch Betragsminima. Das ist derselbe Mechanismus, der Venen im SWI-Magnitudenbild
   dunkel macht — nur dass er, wenn er bis auf ~0 durchgeht, zusätzlich eine Singularität setzt.

4. **T2*-Zerfall + Rauschen.**
   Venöses Blut hat kurzes T2* (bei 7 T grob 5–15 ms). Bei langem TE liegt `|S|` im
   Rauschbereich; dort ist die Phase quasi uniform verteilt und Residuen entstehen mit
   hoher Wahrscheinlichkeit rein stochastisch.

5. **Flow/Pulsatilität** (kleinerer Beitrag): Einströmendes Blut und pulsatile
   Geschwindigkeitsphase brechen die Annahme eines statischen Feldes, besonders in
   größeren Venen und Sinus.

6. **Undersampling des Feldgradienten.** Wenn das *wahre* Feld pro Voxel um mehr als π
   springt, ist der gewrappte Gradient aliasiert und das Umlaufintegral wird falsch. Das
   ist an großen Gefäßen senkrecht zu B0 möglich, spielt aber bei hoher Auflösung eine
   kleinere Rolle als (2).

**Warum gerade hochaufgelöste Daten betroffen sind:** höhere Auflösung mittelt die
Feldvariation *weniger* weg, d.h. die Zweikompartiment-Situation (2) wird an der Gefäßwand
sauber aufgelöst statt verschmiert — die exakte Auslöschung wird wahrscheinlicher, nicht
seltener. Zusätzlich sinkt die SNR pro Voxel. Umgekehrt: bei 1 mm/3 T/kurzem TE gibt es
das Problem kaum, deswegen ist es lange nicht aufgefallen.

### 1.3 Warum ROMEO das nicht auflösen kann — und warum das gut ist

ROMEO ist **kongruent**: die Ausgabe unterscheidet sich von der Eingabe nur um
ganzzahlige Vielfache von 2π. Das ist die zentrale Stärke — die gemessene Phase wird
nirgends verfälscht, und es gibt keine globale Glättung/Bias.

Genau deshalb kann ROMEO Residuen nicht entfernen: Der Region-Growing/MST-Pfad legt den
unvermeidlichen 2π-Sprung dorthin, wo die Kantengewichte am schlechtesten sind (üblicherweise
ins Gefäßinnere) — aber ein Sprung *muss* übrig bleiben. Er bildet eine **Branch-Cut-Fläche**,
deren Rand exakt der Residuen-Loop ist. Wenn der Loop das Gefäß umschließt, muss die
Schnittfläche das Gefäß irgendwo verlassen und läuft dann durch Gewebe mit gutem Signal.

Least-Squares-/Laplace-basierte Unwrapper (DCT-Poisson, Fourier-Laplacian) haben das
Problem nicht, weil sie über die Helmholtz-Zerlegung den rotationsbehafteten Anteil
einfach wegwerfen. Sie sind dafür überall nicht-kongruent, glätten global und verlieren
echten Kontrast. Der Kompromiss, den wir suchen, ist also:
**kongruent überall — bis auf kleine, explizit markierte Patches um die Residuen-Loops.**

### 1.4 Warum das QSM kaputt macht

Ein 2π-Sprung über eine Voxelfläche ist im Feldkarten-Bild eine Stufe mit unendlicher
Steigung. Nachgelagerte Schritte, die Ableitungen bilden, explodieren daran:

* Laplace-basierte Hintergrundfeldentfernung (LBV, V-SHARP mit Laplace-Kern, SHARP)
  bildet `∇²φ` — eine Stufe wird zu einem Dipol-Doppelschicht-Term.
* Die Dipolinversion (TKD, closed-form L2, MEDI, TGV) verstärkt Kanten in der
  Magic-Angle-Richtung ⇒ Streaking, das weit über die Vene hinausreicht.
* SWI-Phasenmasken reagieren direkt auf den Sprung.
* Bei Multi-Echo B0-Fits erzeugt ein echo-abhängiger Sprung Ausreißer im Fit.

Wichtig für die Bewertung: der Schaden ist **nicht lokal**. Deshalb lohnt sich der Aufwand,
obwohl nur ein winziger Bruchteil der Voxel betroffen ist.

---

## 2. Erkennung — Optionen

### A. Residuen-/Curl-Detektion auf der gewrappten Phase (Goldstein)
Pro Voxel 3 Plaketten (xy, xz, yz), `q ∈ {−1,0,+1}`. Anschließend Zusammenhangskomponenten
der Vortex-Linien bilden (das Feld ist divergenzfrei ⇒ Loops).

* **Pro:** Das ist die *Definition* des Problems, exakt, parameterfrei, O(N), extrem billig.
  Liefert direkt die Loops, die man später als Patch-Rand braucht.
* **Contra:** Rauschen erzeugt außerhalb des Objekts und in Low-SNR-Regionen sehr viele
  Residuen. Viele davon sind harmlose ±-Paare in direkter Nachbarschaft (Loop-Länge 4),
  die nur einen 1-Voxel-Cut erzeugen. ⇒ Filterung über Loop-Länge / eingeschlossene Fläche
  und über die Maske ist zwingend.

### B. Detektion auf dem *Ergebnis*: verbleibende Sprünge
Nach dem Unwrapping alle Voxelflächen suchen mit `|φ(x) − φ(x+e)| > π` (bzw. > Schwelle).
In einer kongruenten Lösung sind das exakt die Branch-Cut-Flächen.

* **Pro:** Findet genau das, was QSM stört — die Schnittfläche, nicht nur den Rand.
  Nutzt implizit, dass ROMEO den Cut bereits optimal platziert hat: was übrig bleibt, ist
  das, was wirklich weh tut. Sehr einfach zu implementieren.
* **Contra:** Markiert auch echte, große Feldsprünge (Luft-Gewebe-Grenzen), wo der Cut
  eventuell "richtig" ist. ⇒ muss mit Magnitude/Weights gegatet werden.

### C. Magnitude-/Dephasierungs-basiert
Schwellwert auf `|S|` relativ zum lokalen Median, oder auf R2*, oder auf den Abfall
`|S(TE_n)|/|S(TE_1)|`, oder auf die lokale Varianz des Feldes (Intravoxel-Dephasierungsmaß).

* **Pro:** Physikalisch motiviert, robust, direkt an die Forderung gekoppelt
  "Phase nicht verfälschen, wo Magnitude vorhanden ist".
* **Contra:** Völlig unspezifisch — die meisten Low-Magnitude-Voxel haben keine Singularität,
  und manche Singularität sitzt am Gefäßrand bei noch brauchbarer Magnitude.
  ⇒ **Als Gate/Erlaubnismaske verwenden, nicht als Detektor.**

### D. ROMEO-eigene Gewichte / Qualitätsmaß
Die vorhandenen Kantengewichte (phasecoherence, phasegradientcoherence, phaselinearity,
magcoherence, magweight) bzw. die Reihenfolge im Region Growing (spät unwrappte Voxel =
unzuverlässig).

* **Pro:** Kostenlos, bereits vorhanden, konsistent mit dem eigenen Zuverlässigkeitsmodell.
* **Contra:** Heuristisch, kein topologischer Bezug. Gut als zusätzliches Gewicht in der
  Korrektur, schwach als Detektor.

### E. Nullstellen des komplexen Signals (Sub-Voxel)
Schnitt der Isoflächen `Re(S)=0` und `Im(S)=0` in einem Voxel-Nachbarschafts-Interpolant
(Marching-Cubes-artig) ⇒ Sub-Voxel-Lokalisierung der Vortex-Linie. `|S| = 0` ist invariant
gegen globale Phasenrotation, also wohldefiniert.

* **Pro:** Elegant, sub-voxel-genau, trennt "echte Nullstelle des kontinuierlichen Feldes"
  von reinem Rauschen (über die Persistenz unter leichter Glättung).
* **Contra:** Interpolationsmodell-abhängig, teurer, für den eigentlichen Fix (der auf dem
  Gitter passiert) bringt die Sub-Voxel-Genauigkeit wenig. Eher Analyse- als Produktionswerkzeug.

### F. Multi-Echo-Konsistenz
Residuum des linearen Fits `φ(TE) = φ₀ + 2π·f·TE` bzw. Phasenkohärenz zwischen Echos.
Eine echte Dephasierungs-Nullstelle wandert mit TE und erzeugt einen π-Sprung im
Echo-Verlauf.

* **Pro:** Sehr spezifisch, unterscheidet systematische Effekte von Rauschen; passt gut zu
  ROMEOs bestehendem `phaselinearity`-Gewicht.
* **Contra:** Nur bei Multi-Echo; TE-abhängig (Singularität kann bei TE4 existieren und bei
  TE1 nicht — das ist aber korrekt so, wir behandeln pro Echo).

### G. Persistenz unter Glättung (Skalenraum)
Ein Residuum, das nach Glättung des *komplexen* Bildes mit σ = 1–2 Voxel verschwindet, war
ein enges ±-Paar (harmlos und leicht zu fixen). Eines, das überlebt, umschließt eine echte
Struktur.

* **Pro:** Sehr guter Klassifikator "harmlos vs. relevant", und sagt gleichzeitig die
  nötige Kernelgröße für die Korrektur voraus.
* **Contra:** Zusätzlicher Rechenschritt, ein Parameter mehr.

### Empfehlung für die Erkennung

```
Kandidat = (A: Residuen-Loops, gefiltert nach Loop-Länge)
           ∪ (B: verbleibende |Δφ| > π im Ergebnis)
           ∩ (Maske ∩ C/D: niedrige Magnitude bzw. niedriges ROMEO-Gewicht)
```

A/B definieren *was* das Problem ist, C/D definieren *wo wir überhaupt anfassen dürfen*.
Diese Verknüpfung erfüllt die Anforderung "Phase nicht verfälschen, wo Magnitude da ist"
per Konstruktion. F und G als optionale Verfeinerungen, E als Analysewerkzeug.

Ausgabe in jedem Fall: `singularities.nii` (Loop-Label + Ladung) und `branchcuts.nii`
(Cut-Flächen), auch wenn keine Korrektur aktiviert wird.

---

## 3. Beseitigung — Optionen

### 0. Nur markieren, nichts ändern (Baseline / Kontrollarm)
Singularitätsmaske ausgeben und als Datenterm-Gewicht `W ≈ 0` an QSM weiterreichen.
QSM-Dipolinversionen sind gewichtete Least-Squares-Probleme; ein Gewicht von 0 an den
Cut-Voxeln entfernt den Einfluss der scharfen Kante vollständig — ohne die Phase anzufassen.

* **Pro:** Null Risiko, trivial zu implementieren, konzeptionell sauber.
* **Contra:** Hintergrundfeldentfernung (V-SHARP/LBV) nimmt oft keine Gewichte entgegen,
  und die Kante wirkt dort schon. Viele Pipelines ignorieren Gewichte.
* **Muss als Vergleichsarm mitlaufen** — wenn das reicht, brauchen wir den Rest nicht.

### 1. Gewichtete lokale Glättung der unwrappten Phase
`φ'(x) = Σ_y w(y)·φ(y) / Σ_y w(y)`, mit `w = mag^p · G_σ(|x−y|)`, angewandt nur in der
Maske, mit weichem Blend `α(x) ∈ [0,1]` (geglättete Maske), damit keine neue Kante entsteht.

* **⚠ Fallstrick:** Über einen Branch-Cut hinweg zu mitteln ist Unsinn — man mittelt Werte,
  die sich um 2π unterscheiden, und erzeugt einen Wert, der in *keiner* der beiden Regionen
  stimmt. Naive Glättung der unwrappten Phase verschlimmert das Problem.
  ⇒ Entweder vorher den Sprung entfernen, oder im Gradientenraum arbeiten, oder (siehe 2/3)
  ein Randwertproblem lösen.
* **Pro:** Trivial, schnell, magnitudengewichtet.
* **Contra:** Ohne Behandlung des Cuts falsch; entfernt das Residuum topologisch nicht
  (siehe 2).

### 2. Glättung im komplexen Raum + lokales Re-Unwrapping
Im Umfeld der Singularität das komplexe Signal tiefpassfiltern (magnitudengewichtet per
Konstruktion, stetig über Wraps hinweg), dann lokal neu unwrappen.

* **Pro:** Physikalisch natürlich, keine Wrap-Probleme, Magnitude-Gewichtung gratis.
* **⚠ Wichtig:** Glättung entfernt ein Residuum **nur**, wenn sie ein ±-Paar innerhalb der
  Kernelbreite annihiliert. Ein großer Loop wird durch Faltung nur verschoben/verkleinert,
  nicht aufgelöst — die Windungszahl ist topologisch geschützt, solange das gefaltete Feld
  nicht selbst durch null geht. ⇒ Kernelgröße muss ≥ Loop-Ausdehnung sein, was wiederum
  bedeutet, dass die Glättung in gutes Gewebe hineinreicht. Deswegen ist das für kleine
  Loops sehr gut und für große Loops unzureichend.

### 3. Lokales gewichtetes Least-Squares-Unwrapping (Patch-Laplacian) — **Favorit**
Box um den Residuen-Loop, ROMEOs Ergebnis auf dem Patch-Rand als Dirichlet-Randbedingung,
im Inneren lösen:

```
min_φ  Σ_(i,j)  w_ij · ( φ_i − φ_j − W(φ_i^w − φ_j^w) )²
```

mit `w_ij` aus Magnitude und/oder ROMEO-Gewichten. Normalengleichungen = gewichtete
Poisson-Gleichung ⇒ CG (mit DCT-Poisson-Vorkonditionierer) oder direkter Cholesky auf dem
kleinen Patch.

* **Pro:** Mathematisch die *richtige* Antwort — die LS-Lösung behält nur den rotationsfreien
  Anteil der Helmholtz-Zerlegung, also existiert kein Branch-Cut mehr. Außerhalb des Patches
  ändert sich **exakt nichts** (Dirichlet), Grundfunktionalität ist per Konstruktion
  unangetastet. Über die Gewichte bleibt die Phase in Voxeln mit guter Magnitude praktisch
  unverändert. Skaliert: Patches sind winzig.
* **⚠ Wichtig:** Der Patch muss den **kompletten Residuen-Loop enthalten**, d.h. die
  Netto-Residuenladung über den Patchrand muss 0 sein. Sonst ist die Dirichlet-Randbedingung
  mit der inneren Lösung inkonsistent und der Fehler wird über den ganzen Patch verschmiert.
  ⇒ Patch = Dilatation der Zusammenhangskomponente des Loops, mit Konsistenzcheck; wenn ein
  Loop zu groß wird (z.B. > N Voxel), abbrechen und auf Variante 0 zurückfallen.
* **Varianten zum Ausprobieren:** (a) ungewichtet, DCT-Poisson, sehr schnell;
  (b) magnitudengewichtet, CG; (c) gewichtet mit `w = mag²` vs. `w = ROMEO-Kantengewicht`.

### 4. Harmonisches / biharmonisches Inpainting
Singuläre Voxel als "unbekannt" markieren und die Phase aus dem Rand harmonisch
(`∇²φ = 0`) oder biharmonisch (`∇⁴φ = 0`) fortsetzen. Spezialfall von 3 mit `w = 0` innen.

* **Pro:** Maximal einfach und vorhersagbar. Genau das Richtige dort, wo die Phase
  ohnehin nur Rauschen ist. Biharmonisch liefert C¹-Stetigkeit — deutlich freundlicher für
  alles, was danach einen Laplacian bildet.
* **Contra:** Wirft echte Information im Gefäßinneren weg (der intravaskuläre
  Suszeptibilitätswert geht verloren) ⇒ QSM-Werte *in* der Vene werden unbrauchbar,
  während die Umgebung besser wird. Muss in der Evaluation explizit gemessen werden.

### 5. Optimale Cut-Platzierung statt Entfernung (Min-Cut / diskrete Minimalfläche)
Der Branch-Cut ist eine Fläche mit dem Residuen-Loop als Rand. Die "am wenigsten
schädliche" Fläche = minimale magnitudengewichtete Fläche ⇒ diskretes Plateau-Problem,
exakt lösbar als Max-Flow/Min-Cut.

* **Pro:** Bleibt **vollständig kongruent**, ändert keine Phase, nur die Position des
  Sprungs — Null-Risiko-Verbesserung gegenüber dem MST-Ergebnis.
* **Contra:** Die scharfe Kante bleibt, nur kleiner und besser versteckt. Für QSM
  wahrscheinlich nicht ausreichend, aber ein guter *zusätzlicher* Schritt vor 3/4 (kleinerer
  Cut ⇒ kleinerer Patch).

### 6. Korrektur schon in den komplexen Daten (vor dem Unwrapping)
Nullstellen detektieren und das komplexe Signal dort lokal regularisieren (z.B. Beimischung
des geglätteten komplexen Feldes, oder Prädiktion aus Nachbarechos), dann normal unwrappen.

* **Pro:** Ein einziger Unwrapping-Durchlauf; ROMEOs Kongruenz bleibt für alles andere gültig.
* **Contra:** Verändert die Rohdaten, TE-abhängig, schwerer zu auditieren ("was wurde
  geändert?").

### 7. Multi-Echo-gestützter Ersatz
Aus den Echos, in denen das Voxel nicht singulär ist, plus dem B0-Fit die Phase für das
betroffene Echo prädizieren und ersetzen.

* **Pro:** Sehr gezielt, nutzt echte Information statt Interpolation, kein räumliches
  Verschmieren.
* **Contra:** Nur Multi-Echo; scheitert, wenn das Voxel in *allen* langen Echos singulär ist
  (bei starker Dephasierung realistisch).

### 8. Globales gewichtetes LS (nur als Referenzarm)
Nicht als Lösung gedacht, sondern als oberes/unteres Vergleichsniveau: zeigt, wie viel
Kontrast man verliert, wenn man das Problem "global" löst.

### Empfehlung für die Korrektur

Kaskade, jeweils mit Fallback:

1. Loops klassifizieren (Länge / Persistenz unter Glättung).
2. Kleine Loops (≤ ~6 Kanten, ±-Paare): komplexe Glättung mit σ ≈ 1 Voxel (Variante 2) —
   annihiliert sie sauber.
3. Große Loops: Patch um den vollständigen Loop, **gewichtetes LS mit Dirichlet-Rand**
   (Variante 3b) — mit `w = mag²`, sodass gut-Signal-Voxel ihre Phase behalten.
4. Wo `w` praktisch 0 ist (echte Signalauslöschung): entartet das automatisch zu
   biharmonischem Inpainting (Variante 4) — kein separater Codepfad nötig.
5. Wenn ein Loop zu groß/der Patch inkonsistent ist: nicht anfassen, nur markieren
   (Variante 0) und warnen.
6. Immer die Differenzkarte `Δ = φ_korrigiert − φ_ROMEO` als optionalen Output schreiben —
   auditierbar und rückgängig machbar.

Optional vorgeschaltet: Variante 5 (Min-Cut) zur Verkleinerung der Patches.

---

## 4. Experimentmatrix

| Achse | Werte |
|---|---|
| Detektion | (A) Residuen-Curl · (B) Rest-Sprünge · (A∩C) gegatet · (A∪B)∩C∩D · (G) Persistenz |
| Gating | keins · Magnitude-Perzentil · ROMEO-Gewicht · R2* |
| Korrektur | (0) nur Maske · (2) komplexe Glättung · (3a) DCT-Poisson-Patch · (3b) gewichtetes LS-Patch · (4) biharmonisches Inpainting · (5) Min-Cut · (7) Multi-Echo-Ersatz |
| Patchgröße | Loop-Dilatation +1/+2/+4 Voxel |
| Gewicht | `mag` · `mag²` · ROMEO-Kantengewicht · binär |

Realistisch: nicht das volle Kreuzprodukt, sondern erst Detektion fixieren (die ist billig
zu bewerten: Precision/Recall gegen die Simulation), dann die Korrekturen auf der besten
Detektion vergleichen.

---

## 5. Metriken

### Schutz-Metriken (dürfen sich *nicht* verschlechtern)
* **Anteil geänderter Voxel** innerhalb der Maske — Ziel < 0.1 %.
* **Identität außerhalb der Maske**: `φ' == φ_ROMEO` bitgenau. Als Assertion im Test.
* **Kongruenzverletzung nach Magnitude-Dezil**: `|φ' − φ_wrapped mod 2π|`, Median/95./Max
  pro Dezil. Im obersten Dezil muss das ≈ 0 sein. Das ist die direkte Operationalisierung
  von "Phase nicht verfälschen, wo Magnitude vorhanden ist".
* **Bestehende ROMEO-Testdaten**: alle aktuellen Regressionstests unverändert grün, mit
  neuem Schritt aktiviert *und* deaktiviert.
* **Temporale Stabilität** (EPI-Zeitserie): Standardabweichung der Feldkarte über die Zeit
  darf nicht steigen.
* **Laufzeit**: Overhead < 10 % (Detektion ist O(N), Patches sind winzig).

### Ziel-Metriken
* Anzahl verbleibender Residuen und **totale Residuen-Loop-Länge** innerhalb der Maske.
* Anzahl Voxelflächen mit `|Δφ| > π` im Ergebnis (= Branch-Cut-Fläche).
* Total Variation / Laplace-Energie der Feldkarte in der Umgebung der Venen.
* **QSM-Downstream** (der eigentliche Test):
  * RMSE/NRMSE der Suszeptibilität gegen Ground Truth (nur Simulation);
  * Streaking-Maß: Std in einer venenfernen ROI in weißer Substanz, bzw. Energie entlang
    der Magic-Angle-Richtung;
  * Vene-zu-Gewebe-Kontrast und mittleres χ *im* Gefäß (prüft, ob Inpainting zu viel
    wegnimmt);
  * mehrere Rekonstruktionen, gezielt kantensensitive: Laplace-Hintergrundentfernung
    (LBV/V-SHARP) + TKD/closed-form L2 — die reagieren am stärksten. TGV/MEDI zusätzlich
    als robustere Gegenprobe.
* **SWI-Phasenmaske**: sehr sensitiv auf Phasenkanten, gutes Zusatz-Bild.
* **Cross-Check gegen andere Unwrapper** (PRELUDE/SEGUE/best-path) an denselben Stellen.
* **Visuell**: Feldkarten-Ausschnitte um Venen, vorher/nachher/Differenz, plus QSM-Schnitte;
  ein festes Montage-Skript, damit alle Varianten identisch dargestellt werden.

---

## 6. Datensätze

Ohne Ground Truth ist die Bewertung Geschmackssache — deshalb ist die Simulation Pflicht.

1. **Numerisches Venen-Phantom (wichtigster Datensatz, selbst zu bauen).**
   Gefäßbaum (Zylinder/röhrenförmige Strukturen) mit χ_Vene vs. Gewebe ⇒ Dipolfeld per
   k-Raum-Kernel. Entscheidend: das komplexe Signal auf einem **feinen Untergitter**
   erzeugen (z.B. 4×4×4 Subvoxel), mit T2*-Zerfall gewichten und dann **komplex mitteln**.
   Dadurch entstehen Intravoxel-Dephasierung und Nullstellen ganz von selbst — man bekommt
   echte Singularitäten *und* die wahre Feldkarte. Rauschen additiv komplex.
   Parameter-Sweep: Gefäßdurchmesser (0.2–2 mm), Winkel zu B0, Δχ/Oxygenierung, TE, SNR,
   Voxelgröße ⇒ liefert nebenbei eine Karte "ab wann tritt das Problem auf".
2. **In-vivo hochaufgelöst 7 T ME-GRE**, 0.3–0.6 mm, TE bis ≥ 20–25 ms — der eigentliche
   Zielfall. Vermutlich in-house am schnellsten verfügbar; hier ist am ehesten mit sichtbaren
   Venen-Singularitäten zu rechnen.
3. **QSM Reconstruction Challenge 1.0 / 2.0** — etabliert, teils mit Ground Truth
   (Challenge 2.0 enthält simulierte Daten), gut für Vergleichbarkeit und als Benchmark
   gegenüber Reviewern.
4. **OpenNeuro 7 T Multi-Echo-GRE/QSM-Datensätze** — für Generalisierung über Scanner und
   Protokolle; Accession-IDs vor Verwendung prüfen.
5. **Negativkontrollen (dürfen sich nicht ändern):**
   * 3 T, 1 mm, kurze TEs — hier sollte der neue Schritt praktisch ein No-Op sein;
   * EPI-fMRI-Zeitserie (`-t epi`) — temporale Stabilität;
   * Datensatz mit starken Luft-Gewebe-Übergängen (Sinus/Ohr) — dort gibt es echte große
     Feldsprünge, die *nicht* geglättet werden dürfen.
6. **Stresstest:** Mikroblutungen/Hämorrhagie oder Implantat/DBS — massive Residuen, bei
   denen ein zu aggressiver Fix echte Pathologie zerstören würde. Wichtig für die
   Fallback-Regel "zu großer Loop ⇒ nicht anfassen".

---

## 7. Implementierung & API-Skizze

Getrennter, nachgelagerter Schritt — nichts am MST/Region-Growing anfassen:

```julia
# ROMEO.jl
detect_singularities(phase_wrapped; mag=nothing, weights=nothing, mask=nothing)
    -> (loops, charges, branchcuts)

fix_singularities!(unwrapped, phase_wrapped, loops;
                   mag, weights, method=:lsq, patch_dilation=2, max_loop_size=…)
    -> correction_map
```

CLI:

```
--fix-singularities [off|mask|smooth|lsq|inpaint]   # default: off
--write-singularities                                # singularities.nii, branchcuts.nii
--write-singularity-correction                       # Δ-Karte
```

**Default aus.** Zuerst nur die Detektion ausliefern (Risiko null, liefert sofort Daten
darüber, wie häufig und wo das Problem real auftritt), Korrektur danach hinter einem Flag.

Reihenfolge im Pipeline: nach dem räumlichen Unwrapping, vor der B0-Berechnung. Zu prüfen:
ob die Korrektur besser pro Echo oder erst auf der finalen B0-Karte wirkt (pro Echo ist
sauberer, weil die Singularität echo-abhängig ist; auf der B0-Karte ist es billiger).

---

## 8. Vorgeschlagene Reihenfolge

1. Simulationsphantom bauen (Ground Truth + echte Singularitäten). Ohne das kein
   belastbarer Vergleich.
2. Detektion A + B implementieren, Maps ausgeben, auf allen Datensätzen zählen —
   erst dann wissen wir, wie groß das Problem quantitativ ist.
3. Detektionsvarianten gegen die Simulation bewerten (Precision/Recall), eine festlegen.
4. Korrekturvarianten 0 / 2 / 3b / 4 auf der Simulation vergleichen, inkl. QSM-Downstream.
5. Beste 2 Varianten in vivo, visuell + Metriken; Schutz-Metriken bei jedem Schritt.
6. Hinter Flag mergen, Default aus; nach Erfahrung ggf. Default an.
