# Der Code — einfach erklärt

Diese Datei ist für dich geschrieben, nicht für den Prüfer. Sie erklärt in normaler Sprache, was der Code macht — ohne Fachjargon, mit kurzen Sätzen. Am Ende steht, welche Teile direkt aus dem Kursskript stammen und welche darüber hinausgehen.

---

## Die Grundidee in drei Sätzen

Wir haben eine Tabelle mit 51.290 Bestellungen. Für jede Bestellung wissen wir Umsatz, Rabatt, Stückzahl, Produktkategorie und so weiter — und wie viel Gewinn sie gebracht hat. Der Computer soll aus diesen Angaben lernen, den Gewinn vorherzusagen, ohne ihn zu kennen.

---

## Die fünf Bausteine, die immer wiederkommen

**1. Tabelle laden**

```python
df = pd.read_excel("global-superstore.xls")
```

`pd` steht für pandas — das ist das Werkzeug für Tabellen in Python. Der Befehl liest die Excel-Datei ein. Danach liegt sie als `df` im Arbeitsspeicher (df = *DataFrame*, also einfach „Tabelle").

**2. Etwas aus der Tabelle holen**

```python
df["Profit"]              # eine ganze Spalte
df["Profit"].mean()       # der Durchschnitt dieser Spalte
df[df["Profit"] < 0]      # nur die Zeilen, in denen Verlust gemacht wurde
```

Eckige Klammern heißen „gib mir davon". Der Punkt heißt „mach etwas damit".

**3. Neue Spalte ausrechnen**

```python
df["unit_price"] = df["Sales"] / df["Quantity"]
```

Links steht der neue Spaltenname, rechts die Rechnung. Das passiert für alle 51.290 Zeilen gleichzeitig — man muss keine Schleife schreiben.

**4. Gruppieren und zusammenfassen**

```python
df.groupby("Discount")["Profit"].mean()
```

Lies das von links nach rechts: „Sortiere die Zeilen nach Rabattstufe, nimm dann die Profit-Spalte und rechne je Rabattstufe den Durchschnitt." Genau daraus entsteht der zentrale Befund der Arbeit.

**5. Ein Modell lernen lassen und fragen**

```python
modell.fit(X, y)          # lerne den Zusammenhang
modell.predict(X_test)    # sage für neue Fälle vorher
```

`X` sind die Eingaben (Umsatz, Rabatt, …), `y` ist das, was herauskommen soll (Profit). `fit` heißt lernen, `predict` heißt vorhersagen. Das ist bei allen sechs Verfahren gleich — nur der Name davor ändert sich.

---

## Was in jedem Teil des Notebooks passiert

### Teil 01 — Daten vorbereiten

Die Excel-Datei hat drei Blätter. Wir nehmen das Blatt mit den Bestellungen und kleben die Retouren-Information daran (das heißt *join*). Dann werfen wir die Spalte `Postal Code` weg, weil sie bei allen Bestellungen außerhalb der USA leer ist.

Danach rechnen wir fünf neue Spalten aus, zum Beispiel den Stückpreis. Und dann kommt der wichtigste Schritt:

```python
train_df, test_df = train_test_split(df, test_size=0.2, random_state=42)
```

Wir teilen die Daten in zwei Teile: 80 % zum Lernen, 20 % zum Prüfen. Die 20 % werden weggeschlossen und erst ganz am Ende angeschaut. Sonst wäre es wie eine Klausur, bei der man die Lösungen vorher kennt.

`random_state=42` sorgt dafür, dass immer dieselben Zeilen im Testteil landen. Sonst käme bei jedem Durchlauf ein anderes Ergebnis heraus.

### Teil 02 — Daten anschauen

Hier wird nur geguckt, noch nicht gerechnet: Wie verteilt sich der Gewinn? Welche Spalten hängen zusammen? Was passiert bei welchem Rabatt?

Ergebnis: Ab etwa 25 % Rabatt wird das Geschäft im Schnitt zum Verlustgeschäft.

### Teil 03 — Sechs Verfahren vergleichen

Ein Computer kann nicht mit dem Wort „Europa" rechnen. Und er verwirrt sich, wenn eine Spalte in Tausendern steht (Umsatz) und eine andere in Einern (Stückzahl). Deshalb müssen die Daten erst umgeformt werden. Diese Umformung nennt man **Pipeline** — wie ein Fließband: vorne kommt die Rohtabelle rein, hinten kommen Zahlen raus, mit denen das Modell arbeiten kann.

Wir bauen drei verschiedene Fließbänder und schicken sechs verschiedene Verfahren durch. Das ergibt 18 Kombinationen.

Der wichtigste Trick steckt im dritten Fließband: Es rechnet zusätzlich **Umsatz × Rabatt** aus. Warum? Weil der Gewinn ungefähr so entsteht: „Marge mal Umsatz, minus Rabatt mal Umsatz." Der Rabatt wirkt also nicht für sich allein, sondern zusammen mit dem Umsatz. Gibt man dem Modell dieses Produkt als eigene Spalte, springt die Trefferquote von 34 % auf 71 %.

### Teil 04 — Stellschrauben einstellen

Jedes Verfahren hat Einstellungen, die man vorher festlegen muss — zum Beispiel: Wie viele Nachbarn soll das Nachbarschaftsverfahren anschauen? Der Computer probiert alle vorgegebenen Werte systematisch durch und behält den besten. Das heißt Rastersuche.

### Teil 05 — Auswendiglernen sichtbar machen

Hier wird gezeigt, was passiert, wenn man ein Modell zu kompliziert macht: Es merkt sich die Trainingsdaten auswendig und wird bei neuen Daten schlechter. Die Grafik zeigt beide Kurven — die eine steigt immer weiter, die andere kippt ab. Das ist die wichtigste Lehre aus dem ganzen Modul.

### Teil 06 — Der ernste Test

Jetzt werden die weggeschlossenen 20 % ausgepackt. Jedes Modell darf einmal ran, danach wird nichts mehr geändert.

Zusätzlich zwei ehrliche Vergleiche: Was schafft ein Modell, das immer nur den Durchschnitt rät? Und was schafft ein Modell, das nur Umsatz und Rabatt kennt? Damit sieht man, wie viel die ganze Arbeit wirklich bringt.

### Teil 07 — Die Entdeckung

Hier kommt heraus, dass der Gewinn im Datensatz gar nicht „geschätzt" werden muss — er lässt sich ausrechnen: `Gewinn = Umsatz × (Marge − Rabatt) ÷ (1 − Rabatt)`. Die Marge ist je Produkt fast immer gleich.

Eine Formel ganz ohne Modell trifft damit besser als alle sechs Verfahren. Das klingt erst mal ernüchternd, ist aber der stärkste Teil der Arbeit: Sie zeigt, dass die Struktur des Problems verstanden wurde.

### Teil 08 — Ist der Unterschied echt?

Modell A ist 0,01 besser als Modell B — ist das ein echter Vorsprung oder Zufall? Hier wird nachgerechnet. Ergebnis: Es ist Zufall. Deshalb steht im Bericht nicht „Modell A gewinnt", sondern „beide sind gleich gut".

### Teil 09 — Wo liegt das Modell daneben?

Zum Schluss wird geprüft, bei welchen Bestellungen die Fehler groß sind (bei teuren) und ob das Modell für den vorgeschlagenen Einsatzzweck überhaupt taugt (tut es nicht — eine einfache Regel ist besser).

---

## Was aus dem Kursskript kommt und was darüber hinausgeht

Du hattest mir das Skript zu Modul ADS-04 gegeben. Hier steht ehrlich, was von dort stammt.

### Direkt aus dem Kursskript

| Was | Wo im Skript |
|---|---|
| Zuerst Train/Test-Split, Testdaten wegsperren | Kap. 1.8 und 1.9, Titanic-Beispiel |
| Aufbau als Pipeline und ColumnTransformer | Kap. 1.8 und 2.8 |
| Median-Imputation, StandardScaler, OneHotEncoder, PolynomialFeatures, log1p | Kap. 2.7, Feature Engineering |
| Cross-Validation statt einzelnem Split | Kap. 1.8, „Preview: Cross Validation" |
| Overfitting über die Baumtiefe zeigen | Kap. 1.8 und 3.6 |
| Die sechs Verfahren (kNN, lineare Regression, Ridge, Lasso, Baum, Random Forest) | Kap. 3.2 bis 3.6 |
| Metriken R², RMSE, MAE | Kap. 1.9 |
| Datenqualität nach Validity, Completeness, Consistency, Uniformity, Accuracy | Kap. 2.9 |
| ETL+-Struktur der Notebooks (Import, Inspect, Transform, Analysis, Review, Export) | Kap. 2.2 |
| Die Machine-Learning-Checkliste als roter Faden | Kap. 1.10 |
| Rastersuche für Hyperparameter | Kap. 1.10, Schritt 6, und Kap. 6.2 |

Das ist der Kern der Arbeit — er folgt dem Kurs Schritt für Schritt.

### Zusätzlich, nicht aus dem Skript

| Was | Warum ergänzt |
|---|---|
| Die Rechenidentität `Gewinn = Umsatz × (Marge − Rabatt) ÷ (1 − Rabatt)` | Weil sonst ein Prüfer in 30 Sekunden zeigen kann, dass eine Formel besser ist als das Modell |
| Vergleichsmaßstäbe (Durchschnitts-Baseline, Zwei-Merkmals-Modell) | Um zu zeigen, wie viel das Modell wirklich leistet |
| Gepaarter t-Test und Bootstrap-Konfidenzintervall | Um zu belegen, dass ein Vorsprung von 0,01 kein echter Vorsprung ist |
| Gruppierter Split nach Order ID und zeitlicher Split | Weil Positionen derselben Bestellung sonst in beiden Hälften landen |
| Permutation Importance | Um die dritte Forschungsfrage aus dem Modell heraus zu beantworten, nicht nur aus der Beschreibung |
| Residuendiagnostik und Marge als Zielgröße | Weil die Fehler mit dem Umsatz wachsen — das erkennt man nur, wenn man hinschaut |
| Fehleranalyse nach Sortimentsbereich | Damit klar ist, wo das Modell zuverlässig ist und wo nicht |
| Ein-Standardfehler-Regel bei der Baumtiefe | Weil das Maximum der Kurve innerhalb der Streuung liegt |

Diese Ergänzungen sind der Grund, warum die Arbeit über eine reine Nacharbeit des Kurses hinausgeht. Sie bauen aber alle auf dem auf, was im Skript steht — sie ersetzen nichts davon.

---

## Wenn dich jemand nach dem Code fragt

Drei Sätze, die immer passen:

1. „Ich habe die Daten zuerst in Trainings- und Testdaten geteilt und die Testdaten bis zum Schluss nicht angefasst."
2. „Die Vorverarbeitung steckt in einer Pipeline zusammen mit dem Modell — dadurch kann bei der Kreuzvalidierung kein Wissen aus den Prüfdaten hineinsickern."
3. „Der entscheidende Hebel war nicht die Wahl des Verfahrens, sondern das Merkmal Umsatz × Rabatt — das folgt direkt daraus, wie ein Gewinn kaufmännisch zustande kommt."
