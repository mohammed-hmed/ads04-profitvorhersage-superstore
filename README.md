# Profit-Vorhersage im Global-Superstore-Datensatz

Projektarbeit im Modul Machine Learning (ADS-04), Applied Data Science, Digital Business University of Applied Sciences.

## Worum geht's

Der Global-Superstore-Datensatz enthält 51.290 Bestellpositionen eines Einzelhändlers aus sieben Märkten, 2011 bis 2014. Knapp ein Viertel davon sind Verlustgeschäfte (24,46 %). Ich versuche vorherzusagen, wie viel Gewinn eine einzelne Position bringt.

Drei Fragen:

1. Wie genau geht das, und was bringt ein Modell gegenüber ganz simplen Vergleichswerten?
2. Welche Datenaufbereitung und welcher Algorithmus funktionieren am besten?
3. Woran liegt es, wenn eine Position Verlust macht?

## Was rausgekommen ist

**Die Aufbereitung bringt mehr als der Algorithmus.** Sobald das Modell das Produkt aus Umsatz und Rabatt als eigene Spalte bekommt, springt die lineare Regression von R² 0,34 auf 0,71. Mit Polynomgrad 3 sogar auf 0,75.

**Zwischen den Algorithmen ist dagegen kaum ein Unterschied.** Auf den Testdaten: lineares Modell 0,663, Random Forest 0,653. Sieht nach einem Vorsprung aus, ist aber keiner. Der gepaarte t-Test liefert p = 0,54, und je nach Split-Verfahren dreht sich die Reihenfolge um. Nur beim MAE ist der Random Forest wirklich besser (5,28 USD Unterschied).

**Der Profit muss eigentlich gar nicht geschätzt werden.** Er folgt einer Rechenformel: `Profit = Sales · (Marge − Rabatt) / (1 − Rabatt)`. Die Marge ist pro Produkt fast konstant. Die Formel allein, ohne ein einziges trainiertes Modell, kommt auf R² 0,736; mit produktspezifischen Margen auf 0,974. Damit schlägt sie jedes Modell in diesem Projekt.

**Rabatt ist der Haupttreiber für Verluste.** Der Kipppunkt liegt zwischen 25 und 27 Prozent. Bei 25 % Rabatt bleiben im Schnitt noch +9,62 USD übrig, bei 27 % sind es −2,75 USD.

**Ein Gegenbefund:** Die Unterkategorie *Tables* macht im Mittel Verlust (−80,92 USD). Ohne Rabatt ist sie aber der profitabelste Bereich überhaupt (+292,86 USD). Das Problem ist also die Rabattvergabe, nicht die Kalkulation.

**Und ein Datenproblem:** Die Spalte `Profit` ist nicht um Retouren bereinigt. Zurückgeschickte Positionen zeigen sogar einen höheren Profit als der Rest (42,07 gegenüber 27,82 USD). Steht im Bericht als Limitation drin.

## Was wo liegt

```
notebooks/   ADS04_Gesamtanalyse.ipynb           die ganze Analyse, mit Erklärungen
             ADS04_Gesamtanalyse_nur_Code.ipynb  dasselbe, nur der Code
data/raw/    Rohdaten (.xls und eine .xlsx-Kopie)
data/processed/  die aufbereiteten Tabellen
figures/     die Abbildungen
results/     alle Kennzahlen als csv und json
bericht/     der Bericht als Word und PDF
CODE_DOKUMENTATION.md      Code Schritt für Schritt erklärt
CODE_EINFACH_ERKLAERT.md   dasselbe für Leute ohne Programmiererfahrung
```

## Selbst ausführen

```bash
pip install -r requirements.txt
```

Dann `notebooks/ADS04_Gesamtanalyse.ipynb` öffnen und *Kernel → Restart Kernel and Run All Cells*. Läuft in etwa drei Minuten durch.

Das Notebook geht von oben nach unten durch, in neun Teilen: Datenaufbereitung, explorative Analyse, Modellvergleich, Tuning, Overfitting, Test, Struktur der Zielgröße, Signifikanz, Diagnostik. Die Teile bauen aufeinander auf, umsortieren funktioniert nicht.

Es liegt fertig ausgeführt im Repo, mit allen Ausgaben und Grafiken. Man muss also nichts starten, um die Ergebnisse zu sehen. Alle Zufallszahlen sind mit `random_state=42` festgenagelt, validiert wird durchgehend mit `KFold(5, shuffle=True, random_state=42)`.

Falls das Paket `xlrd` fehlt, wechselt die erste Zelle automatisch auf die `.xlsx`-Kopie.

## Zu den Daten

Die Datei `global-superstore.xls` kommt aus dem Kursmaterial (ursprünglich von Tableau). Sie hat drei Blätter: *Orders* mit 51.290 Zeilen, *Returns* mit 1.173 Zeilen zu 1.172 Bestellungen, und *People*, das ich nicht benutze.

Die Retouren habe ich über die `Order ID` an die Bestellungen gehängt, davon sind 3.050 Positionen betroffen. Die Information, ob etwas zurückkam, benutze ich aber **nicht** als Merkmal. Zum Zeitpunkt der Bestellung weiß man das schließlich noch nicht, und sonst hätte ich mir Datenleckage eingebaut.

`Postal Code` ist rausgeflogen, die Spalte fehlt bei allen 41.296 Bestellungen außerhalb der USA. `Discount` habe ich auf drei Nachkommastellen gerundet, weil in den Rohdaten sowohl 0,15 als auch 0,15000000000000002 vorkommt und das jede Gruppierung zerlegt. Fünf zusätzliche Merkmale habe ich abgeleitet (Versanddauer, Bestellmonat, Bestelljahr, Stückpreis, Versandkostenanteil), im Endmodell brauche ich sie aber nicht.

Imputation, Skalierung und One-Hot-Encoding passieren innerhalb der Pipelines, also bei jeder Kreuzvalidierung neu. Damit sickert nichts aus den Prüfdaten ins Training.

Zur Datenqualität: Wertebereiche sind plausibel, Lücken gibt es nur bei `Postal Code`, inhaltliche Duplikate keine, Währung durchgehend USD. Der Schwachpunkt ist die Zielgröße selbst, weil Retouren nicht verrechnet sind. Details stehen in Teil 01 des Notebooks und in Kapitel 2.1 des Berichts.

## Quellen

Der methodische Aufbau folgt dem Kursskript ADS-04 und Géron (2019), Kapitel 2. Beim Feature Engineering habe ich mich an Zheng und Casari (2018) orientiert. Fremden Notebook-Code, etwa von Kaggle, habe ich nicht übernommen. Welche Hilfsmittel ich benutzt habe, einschließlich KI, steht im Bericht im Abschnitt "Verwendete Hilfsmittel".
