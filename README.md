# Profit-Vorhersage im Global-Superstore-Datensatz

**Projektarbeit im Modul (4) Machine Learning (ADS-04) · Applied Data Science · Digital Business University of Applied Sciences**

## Gegenstand des Projekts

Der im Kurs bereitgestellte Global-Superstore-Datensatz (Tableau) umfasst 51.290 Bestellpositionen eines fiktiven Einzelhändlers aus sieben Weltmärkten (2011–2014) inklusive Retouren. 24,46 % der Positionen sind Verlustgeschäfte. Ziel ist die **Vorhersage des Profits je Bestellposition** (Regression).

**Forschungsfragen**

* **RQ1:** Wie genau lässt sich der Profit vorhersagen, und wie viel leistet ein Modell gegenüber einfachen Vergleichsmaßstäben?
* **RQ2:** Welche Kombination aus Transformationspipeline und Algorithmus generalisiert am besten, und wie belastbar ist dieser Vergleich?
* **RQ3:** Welche Merkmale treiben Verlustgeschäfte, und welche Handlungsempfehlungen folgen daraus?

## Zentrale Ergebnisse

* **Merkmalskonstruktion schlägt Algorithmenwahl.** Interaktionsterme (`Sales × Discount`, Pipeline C) heben die linearen Modelle von R² = 0,339 auf 0,714. Polynomgrad drei verbessert das nochmals signifikant auf 0,748 (gepaarter t-Test, p = 0,006) — der einzige statistisch abgesicherte Modellunterschied der Arbeit.
* **Der Unterschied zwischen den Algorithmen ist nicht belastbar.** Lineares Interaktionsmodell 0,663 gegenüber Random Forest 0,653 auf den Testdaten; p = 0,544 in der Cross-Validation, Bootstrap-Konfidenzintervall für die R²-Differenz [−0,032; +0,068]. Unter gruppiertem und zeitlichem Split kehrt sich die Rangfolge um. Belastbar ist nur der Vorsprung des Random Forest beim MAE (Differenz 5,28 USD, KI [+4,31; +6,00]).
* **Der Profit folgt einer Rechenidentität.** `Profit = Sales · (m − d)/(1 − d)`; die Rohmarge *m* ist mit einer Standardabweichung von 0,0040 innerhalb eines Produkts (gegenüber 0,1490 insgesamt) praktisch eine Produktkonstante. Eine Formel **ohne jeden gefitteten Parameter** erreicht R² = 0,736, mit produktspezifischen Margen 0,974 — mehr als jedes trainierte Modell.
* **Rabatt ist der dominierende Verlusttreiber.** Umschlagpunkt zwischen 25 und 27 Prozent Rabatt (bei 25 %: +9,62 USD; bei 27 %: −2,75 USD). Die Permutation Importance weist `Discount` (1,447) und `Sales` (1,353) mit weitem Abstand die höchsten Werte zu.
* **Gegenbefund zum Sortiment:** Die Unterkategorie *tables* ist im Mittel defizitär (−80,92 USD), ohne Rabatt aber der **profitabelste** Bereich überhaupt (+292,86 USD). Der Verlust folgt aus der Rabattvergabe, nicht aus der Kalkulation.
* **Datenqualitätsbefund:** `Profit` ist nicht um Retouren korrigiert — retournierte Positionen weisen im Mittel höheren Profit aus (42,07 gegenüber 27,82 USD).

## Struktur des Repositories

```
├── README.md                    <- Ausgangspunkt der Dokumentation (diese Datei)
├── CODE_DOKUMENTATION.md        <- der Code Schritt für Schritt erklärt
├── requirements.txt             <- exakte Paketversionen
├── .gitignore
├── bericht/                     <- Bericht (docx + pdf), inkl. Eigenständigkeitserklärung
├── data/
│   ├── raw/global-superstore.xls    <- Rohdaten, unverändert (3 Blätter)
│   ├── raw/global-superstore.xlsx   <- formatgleiche Kopie (falls xlrd nicht installiert ist)
│   └── processed/*.csv.gz           <- aufbereitete Daten (von Notebook 01 als Pickle erzeugt,
│                                       hier zusätzlich komprimiert als CSV abgelegt)
├── notebooks/
│   ├── ADS04_Gesamtanalyse.ipynb          <- Referenzfassung: die vollständige Analyse in EINEM
│   │                                         durchgehenden Notebook, mit Erläuterungen und Befunden
│   └── ADS04_Gesamtanalyse_nur_Code.ipynb <- inhaltsgleiche Fassung ohne Fließtext, nur Code
│                                             (beide vollständig ausgeführt, Zähler 1…47)
├── figures/                     <- Abbildungen (vom Notebook erzeugt)
└── results/                     <- cv_vergleich.csv, tuned_params.json, test_scores.csv,
                                    robustheit_splits.csv, overfitting_baumtiefe.csv,
                                    marge_modellierung.csv, fehler_je_segment.csv,
                                    warnsystem_vergleich.csv
```

## Reproduktion

```bash
pip install -r requirements.txt
# notebooks/ADS04_Gesamtanalyse.ipynb öffnen und
# "Kernel → Restart Kernel and Run All Cells" ausführen
```

Die gesamte Analyse liegt in **einem einzigen, durchgehend ausführbaren Notebook**. Die neun Teile (01 Datenaufbereitung bis 09 Diagnostik) entsprechen der Reihenfolge der Ausführung und sind über die Überschriften navigierbar; das Inhaltsverzeichnis steht in der ersten Zelle. Das Notebook liegt **vollständig ausgeführt mit allen Ausgaben und Abbildungen** im Repository, die Ausführungszähler laufen lückenlos von 1 bis 47.

Daneben liegt `ADS04_Gesamtanalyse_nur_Code.ipynb`: dieselbe Analyse, dieselben Zellen und dieselben Ergebnisse, aber ohne Fließtext — nur Code, gegliedert durch eine Kommentarzeile je Teil. Wer ausschließlich den Code lesen will, nimmt diese Datei; die Erläuterungen stehen dann in `CODE_DOKUMENTATION.md` und im Bericht.

Es läuft unverändert lokal in Anaconda/Jupyter aus dem Ordner `notebooks/` sowie in Google Colab (Projektordner in Google Drive; die erste Zelle bindet Drive ein). Fehlt das Paket `xlrd`, wechselt die erste Zelle automatisch auf die mitgelieferte `.xlsx`-Kopie der Rohdaten.

Reproduzierbarkeit: Alle Zufallsprozesse sind mit `random_state=42` fixiert; das Validierungsschema ist überall `KFold(n_splits=5, shuffle=True, random_state=42)`. Gesamtlaufzeit rund drei Minuten auf einem Rechner mit vier Kernen; die verwendeten Paketversionen stehen in `requirements.txt`.

## Datenherkunft, Aufbereitung und Datenqualität

**Herkunft:** `global-superstore.xls` (Tableau), bereitgestellt über das Kursmaterial ADS-04; Blätter *Orders* (51.290 × 24), *Returns* (1.173 Zeilen mit 1.172 eindeutigen Bestellnummern) und *People* (13 × 2, nicht verwendet).

**Integration:** Die Retouren werden über `Order ID` an die Bestellpositionen gejoint (3.050 betroffene Positionen). Das Flag `returned` wird bewusst **nicht** als Modellmerkmal verwendet, da eine Retoure zum Vorhersagezeitpunkt unbekannt ist (Vermeidung von Datenleckage).

**Aufbereitung:** `Postal Code` entfernt (fehlt systematisch bei 41.296 Nicht-US-Bestellungen); `Discount` auf drei Nachkommastellen gerundet, da die Rohdaten Gleitkomma-Dubletten enthalten (z. B. 0,15 und 0,15000000000000002); fünf Merkmale abgeleitet (`shipping_days`, `order_month`, `order_year`, `unit_price`, `ship_cost_ratio`), die geprüft, aber im Endmodell nicht verwendet werden; Median-Imputation, Standardisierung und One-Hot-Encoding erfolgen innerhalb der Pipelines und damit je Validierungsfold neu.

**Qualitätsbeurteilung:** Validity (Wertebereiche plausibel), Completeness (nur `Postal Code` lückenhaft), Consistency (keine inhaltlichen Duplikate) und Uniformity (durchgängig USD) sind erfüllt. Eingeschränkt ist die Accuracy der Zielgröße, da `Profit` Retouren nicht verrechnet. Details in Teil 01 des Notebooks (Abschnitt Review) und im Bericht, Kapitel 2.1.

## Verwendete Quellen und übernommene Ansätze

Der methodische Aufbau (Pipeline-/ColumnTransformer-Muster, Split vor jeder Analyse, Cross-Validation, Rastersuche) folgt dem Kursskript ADS-04 sowie Géron (2019), Kapitel 2; das Feature Engineering orientiert sich an Zheng und Casari (2018). Es wurde **kein** Code aus fremden Notebooks (etwa von Kaggle) übernommen. Die eingesetzten Hilfsmittel einschließlich KI-Unterstützung sind im Bericht im Abschnitt „Verwendete Hilfsmittel" vollständig offengelegt.
