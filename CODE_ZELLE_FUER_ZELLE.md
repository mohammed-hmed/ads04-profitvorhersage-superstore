# Jede Zelle einzeln erklärt

Diese Datei geht das Notebook `notebooks/ADS04_Gesamtanalyse.ipynb` Zelle für Zelle durch: 47 Codezellen, zu jeder steht hier in normaler Sprache, was sie macht und warum. Sie richtet sich bewusst auch an Leserinnen und Leser ohne Programmiererfahrung. Wer bei Null anfängt, liest zuerst den Grundlagenteil.

Eine kürzere Übersicht steht in `CODE_DOKUMENTATION.md`.

---

## Erst mal: sechs Dinge, die immer wiederkommen

**1. Eine Tabelle heißt `df`**

`df` steht für *DataFrame*, das ist einfach das Wort für "Tabelle" in Python. Man kann sich das vorstellen wie ein Excel-Blatt, das im Arbeitsspeicher liegt statt auf dem Bildschirm.

**2. Eckige Klammern heißen "gib mir davon"**

```python
df["Profit"]          # die ganze Spalte Profit
df[df["Profit"] < 0]  # nur die Zeilen, wo Verlust gemacht wurde
```

**3. Der Punkt heißt "mach was damit"**

```python
df["Profit"].mean()   # Durchschnitt
df["Profit"].round(2) # auf zwei Stellen runden
df.shape              # wie viele Zeilen und Spalten
```

**4. Gleichheitszeichen heißt "merk dir das unter diesem Namen"**

```python
train_df = load("train")
```
Lade die Trainingsdaten und merk sie dir als `train_df`.

**5. `X` und `y`**

`y` ist das, was vorhergesagt werden soll, hier der Gewinn. `X` sind alle Angaben, aus denen vorhergesagt wird: Umsatz, Rabatt, Stückzahl und so weiter. Das ist eine Konvention aus der Mathematik, kein Zufall.

**6. Ein Modell lernt und wird dann gefragt**

```python
modell.fit(X, y)        # lerne den Zusammenhang
modell.predict(X_neu)   # sag für neue Fälle vorher
```
`fit` heißt lernen, `predict` heißt vorhersagen. Das ist bei allen sechs Verfahren identisch, nur der Name davor wechselt.

---

# Teil 01 — Datenaufbereitung

### Zelle 1: Vorbereitung

Die längste Zelle im Notebook und die langweiligste. Sie macht drei Dinge.

Sie lädt die Werkzeuge: `pandas` für Tabellen, `numpy` fürs Rechnen, `seaborn` und `matplotlib` für Diagramme. Das ist wie das Öffnen von Programmen, bevor man arbeitet.

Sie legt fest, wo die Dateien liegen: `DATA_PATH`, `PROC_PATH`, `FIG_PATH`, `RES_PATH`. Großbuchstaben bedeuten in Python: das ändert sich nie mehr.

Und sie setzt `RANDOM_STATE = 42`. Das ist wichtig. Viele Verfahren würfeln irgendwo, zum Beispiel beim Aufteilen der Daten. Mit einer festen Zahl würfeln sie jedes Mal gleich, also kommt jedes Mal dasselbe raus. Ohne die Zahl hättest du bei jedem Durchlauf leicht andere Ergebnisse, und dein Bericht wäre nicht mehr überprüfbar. Die 42 ist ein Insider-Witz unter Programmierern, jede andere Zahl täte es genauso.

### Zelle 2: Excel-Datei einlesen

```python
xl = pd.read_excel(RAW_PATH, sheet_name=None)
```
`sheet_name=None` heißt: lies **alle** Blätter, nicht nur das erste. Danach steckt in `xl` ein Verzeichnis mit drei Tabellen: Orders, Returns, People. Die Zeile darunter zeigt, wie groß jede ist.

### Zelle 3: Zwei Blätter herausholen und Rundungsfehler beseitigen

```python
orders, returns = xl["Orders"], xl["Returns"]
```
Zwei Tabellen in einer Zeile herausholen.

Danach werden Geldbeträge gerundet. Grund: Computer speichern Kommazahlen nicht exakt. In der Datei steht mal 0,15 und mal 0,15000000000000002. Für dich ist das dasselbe, für den Computer nicht. Wenn du danach gruppierst, macht er daraus zwei verschiedene Rabattstufen und alles verrutscht. Das Runden verhindert das.

### Zelle 4: Wo fehlen Werte?

```python
na = orders.isnull().sum()
na[na > 0]
```
`isnull()` fragt jede einzelne Zelle: bist du leer? `sum()` zählt die Ja-Antworten je Spalte. Die zweite Zeile zeigt nur die Spalten, wo tatsächlich was fehlt. Ergebnis: nur `Postal Code`, und zwar bei allen Bestellungen außerhalb der USA.

### Zelle 5: Gibt es doppelte Zeilen?

Die Spalte `Row ID` wird vorher weggelassen, weil sie in jeder Zeile anders ist. Sonst wären zwei identische Bestellungen nie "doppelt". Ergebnis: keine Duplikate.

### Zelle 6: Retouren dazuholen

```python
df = orders.merge(returns[["Order ID"]].assign(returned=1).drop_duplicates(),
                  on="Order ID", how="left")
```
Von hinten gelesen: Nimm aus der Retouren-Tabelle nur die Bestellnummern, häng eine Spalte `returned = 1` dran, wirf Doppelte weg. Dann klebe das über die Bestellnummer an die Bestellungen. `how="left"` heißt: alle Bestellungen bleiben erhalten, auch die ohne Retoure. Bei denen steht danach nichts, und `fillna(0)` macht daraus eine 0.

Das Zusammenkleben von zwei Tabellen über eine gemeinsame Spalte heißt *join*. Wie SVERWEIS in Excel, nur für die ganze Tabelle auf einmal.

### Zelle 7: Aufräumen und neue Spalten bauen

`Postal Code` fliegt raus. Dann werden fünf neue Spalten berechnet:

```python
df["unit_price"] = df["Sales"] / df["Quantity"]
```
Links der neue Name, rechts die Rechnung. Das passiert für alle 51.290 Zeilen gleichzeitig, ganz ohne Schleife. Das ist der Grund, warum man pandas benutzt und nicht Excel.

### Zelle 8: Die wichtigste Zeile im ganzen Notebook

```python
train_df, test_df = train_test_split(df, test_size=0.2, random_state=RANDOM_STATE)
```
80 Prozent der Zeilen zum Lernen, 20 Prozent zum Prüfen. Die 20 Prozent werden weggeschlossen und erst ganz am Ende angefasst.

Warum so streng? Weil ein Modell sonst betrügen kann, ohne dass es jemand merkt. Es lernt die Antworten auswendig statt den Zusammenhang. Bei einer Klausur würdest du auch nicht vorher die Lösungen austeilen.

### Zelle 9: Speichern

Die drei Tabellen werden auf die Festplatte geschrieben, zweimal: als `.pkl` (schnell, behält Datentypen wie Datumsangaben) und als `.csv.gz` (langsamer, dafür überall lesbar).

---

# Teil 02 — Die Daten anschauen

### Zelle 10: Hilfsfunktionen und erste Übersicht

Hier werden zwei kleine Werkzeuge gebaut. `def` heißt "definiere eine Funktion", also einen Handgriff, den man später mit einem Wort aufrufen kann.

`load("train")` holt eine gespeicherte Tabelle zurück. `print_scores(...)` gibt Mittelwert und Streuung von Ergebnissen aus.

Danach `describe()`: Anzahl, Mittelwert, Minimum, Maximum und die Quartile für jede Zahlenspalte. Der erste echte Blick in die Daten.

### Zelle 11: Wie ist der Gewinn verteilt?

Zwei Diagramme nebeneinander. Links ein Histogramm, also Balken, die zeigen, wie oft welcher Gewinn vorkommt. Rechts ein Boxplot, der Ausreißer sichtbar macht.

Befund: Der Durchschnitt liegt bei 28,66 Dollar, der Median aber nur bei 9,33. Wenn Durchschnitt und Median so weit auseinanderliegen, ziehen wenige sehr große Werte den Durchschnitt nach oben. Genau das passiert hier.

### Zelle 12: Was hängt womit zusammen?

```python
sns.heatmap(train_df[corr_attributes].corr(), vmin=-1, vmax=1, cmap="RdBu", annot=True)
```
`corr()` rechnet für jedes Spaltenpaar aus, wie stark sie zusammenhängen. Der Wert liegt zwischen −1 und +1. Bei +1 steigen beide immer gemeinsam, bei −1 geht das eine hoch, wenn das andere runtergeht, bei 0 gibt es keinen Zusammenhang.

Die Heatmap färbt das ein. Ergebnis: Umsatz hängt am stärksten mit dem Gewinn zusammen (0,50), Rabatt deutlich negativ (−0,32).

### Zelle 13: Gewinn je Rabattstufe

```python
stufen = train_df.groupby("Discount")["Profit"].agg(["mean", "count"]).round(2)
```
Von links nach rechts gelesen: Sortiere die Zeilen nach Rabattstufe, nimm dann die Gewinnspalte, und rechne je Stufe Durchschnitt und Anzahl aus.

Danach werden Stufen mit weniger als 100 Fällen weggelassen, weil Durchschnitte aus zehn Zeilen nichts aussagen.

Daraus kommt der wichtigste Befund der Arbeit: bis 25 Prozent Rabatt bleibt im Schnitt Gewinn übrig, ab 27 Prozent nicht mehr.

### Zelle 14: Abbildung 1

Das ist das Diagramm, das auch im Bericht steht. Drei Ebenen übereinander: hellblaue Punkte für jede einzelne Bestellposition, eine graue Nulllinie, und rote Punkte für den Durchschnitt je Rabattstufe. Die roten Punkte sind unterschiedlich groß, je nachdem wie viele Bestellungen dahinterstehen.

Der Zufallszahlen-Zusatz bei den blauen Punkten ist Absicht. Rabatte gibt es nur in festen Stufen, sonst würden alle Punkte exakt aufeinander liegen und man sähe eine Linie statt einer Wolke.

### Zelle 15: Welcher Sortimentsbereich lohnt sich?

Ein Balkendiagramm, sortiert. Rot für Verlust, grün für Gewinn. Tables sticht negativ heraus.

### Zelle 16: Nachhaken bei Tables

Hier wird geprüft, ob Tables wirklich generell Verlust macht oder nur wegen der Rabatte.

```python
klassen = pd.cut(t["Discount"], [-0.01, 0.0, 0.2, 0.4, 0.6, 0.85])
```
`cut` teilt den Rabatt in Bereiche ein, also 0 Prozent, bis 20, bis 40 und so weiter. Danach wird je Bereich der Durchschnitt gerechnet.

Ergebnis: Ohne Rabatt bringt Tables +292,86 Dollar und ist damit der beste Bereich überhaupt. Das Problem sind die Rabatte, nicht das Produkt. Genau solche Nachfragen unterscheiden eine gute Arbeit von einer, die den ersten Eindruck einfach übernimmt.

### Zelle 17: Zwei letzte Kontrollen

Gewinn je Markt, und Gewinn nach Retouren-Status. Der zweite Punkt liefert einen unangenehmen Befund: Zurückgeschickte Positionen zeigen einen **höheren** Gewinn als der Rest. Das kann nicht stimmen und heißt: Die Spalte `Profit` verrechnet Retouren nicht. Steht im Bericht als Einschränkung.

---

# Teil 03 — Sechs Verfahren vergleichen

### Zelle 18: Vorbereitung

Die Werkzeuge für das Maschinelle Lernen werden geladen, dann:

```python
X_train = train_df.drop(columns=["Profit", "returned"])
y_train = train_df["Profit"].values
```
`X` bekommt alles außer dem Gewinn (den soll das Modell ja nicht kennen) und außer `returned`. Warum `returned` raus muss: Bei der Bestellannahme weiß niemand, ob die Ware zurückkommt. Wenn das Modell diese Information bekäme, hätte es Wissen aus der Zukunft. Das nennt man Datenleckage und ist einer der häufigsten Fehler überhaupt.

```python
kf = KFold(n_splits=5, shuffle=True, random_state=RANDOM_STATE)
```
Das ist die Kreuzvalidierung. Die Trainingsdaten werden in fünf Teile geschnitten. Dann wird fünfmal trainiert: jedes Mal vier Teile zum Lernen, einer zum Prüfen, reihum. So bekommt man fünf Bewertungen statt einer und sieht, wie stark das Ergebnis schwankt.

### Zelle 19: Die drei Pipelines

Ein Computer kann nicht mit dem Wort "Europa" rechnen. Und er kommt durcheinander, wenn eine Spalte in Tausendern steht und die nächste in Einern. Also müssen die Daten vorher umgeformt werden.

Diese Umformung heißt **Pipeline**, wie ein Fließband: vorne die Rohtabelle rein, hinten Zahlen raus.

Drei Fließbänder werden gebaut:

**A, die Basis.** Fehlende Werte mit dem Median auffüllen (`SimpleImputer`), alle Zahlen auf dieselbe Skala bringen (`StandardScaler`), Textspalten in Nullen und Einsen umwandeln (`OneHotEncoder`).

Zu One-Hot: Aus einer Spalte "Markt" mit sieben Werten werden sieben Spalten. Für jede Zeile steht in genau einer davon eine 1, im Rest eine 0. Das klingt umständlich, ist aber die einzige ehrliche Art, Kategorien in Zahlen zu übersetzen. Würde man einfach 1 bis 7 durchnummerieren, würde das Modell glauben, Markt 7 sei siebenmal so viel wie Markt 1.

**B, mit zusätzlichen Merkmalen.** Wie A, aber der Umsatz wird logarithmiert und die fünf abgeleiteten Spalten kommen dazu. Spoiler: Das macht es schlechter, nicht besser.

**C, mit Interaktionstermen.** Hier steckt der entscheidende Trick. `PolynomialFeatures(degree=2)` baut aus den Spalten auch alle Produkte, insbesondere Umsatz mal Rabatt.

Warum das hilft: Gewinn entsteht ungefähr so, dass man Marge mal Umsatz rechnet und davon Rabatt mal Umsatz abzieht. Der Rabatt wirkt also nicht für sich allein, sondern immer zusammen mit dem Umsatz. Gibt man dem Modell dieses Produkt als eigene Spalte, springt die Trefferquote von 34 auf 71 Prozent.

### Zelle 20: 18 Kombinationen durchrechnen

Sechs Verfahren mal drei Fließbänder. Zwei ineinander verschachtelte Schleifen probieren jede Kombination und bewerten sie mit der Kreuzvalidierung.

Die sechs Verfahren kurz:

- **Lineare Regression** legt eine Gerade durch die Daten
- **Ridge** und **Lasso** sind lineare Regression mit Bremse, damit sie nicht übertreibt
- **kNN** schaut, was bei ähnlichen Bestellungen herauskam, und mittelt
- **Entscheidungsbaum** stellt Ja/Nein-Fragen: Rabatt über 20 Prozent? Dann links, sonst rechts
- **Random Forest** baut viele solcher Bäume und lässt sie abstimmen

Am Ende landet alles in einer Tabelle, sortiert nach Güte, und wird gespeichert.

Zu R²: Der Wert sagt, welchen Anteil der Schwankung das Modell erklärt. 1,0 wäre perfekt, 0 wäre so gut wie raten.

---

# Teil 04 — Stellschrauben einstellen

### Zellen 21 und 22: Rastersuche

Jedes Verfahren hat Einstellungen, die man vorher festlegen muss. Wie viele Nachbarn soll kNN anschauen? Wie tief darf der Baum werden? Diese Einstellungen heißen Hyperparameter.

```python
gs = GridSearchCV(Pipeline([...]), grid, cv=kf, scoring="r2")
```
`GridSearchCV` probiert stur alle vorgegebenen Werte durch und merkt sich den besten. Rastersuche eben.

Das Raster für Ridge geht absichtlich bis 100.000. Bei einem engen Raster käme nur heraus, dass alle Werte ähnlich gut sind. Erst ein weites zeigt, ab wann die Bremse überhaupt greift.

### Zelle 23: Was hat die Bremse gebracht?

Eine kleine Tabelle. Antwort: nichts. Bei 41.000 Zeilen und 47 Spalten hat das Modell kein Problem mit Übertreiben, dafür sind es viel zu viele Beobachtungen. Auch das ist ein Ergebnis, kein Fehler.

---

# Teil 05 — Auswendiglernen sichtbar machen

### Zellen 24 bis 26

Hier wird der wichtigste Effekt aus dem ganzen Modul gezeigt. Der Baum wird mit verschiedenen Tiefen trainiert, von 2 bis 20, und jedes Mal doppelt bewertet: einmal auf den Daten, die er gesehen hat, einmal auf Daten, die er nicht kennt.

Ergebnis: Auf den bekannten Daten wird er immer besser, bis fast 100 Prozent. Auf den unbekannten wird er erst besser, dann wieder schlechter. Ab einem bestimmten Punkt lernt er auswendig statt zu verstehen. Das heißt **Overfitting**.

Die Lehre daraus: Ein hoher Wert auf den Trainingsdaten sagt gar nichts. Nur die Prüfung auf ungesehenen Daten zählt.

Am Schluss noch die Ein-Standardfehler-Regel: Statt einfach das Maximum zu nehmen, nimmt man das einfachste Modell, das noch innerhalb der Messungenauigkeit des Maximums liegt. Einfacher ist bei gleicher Güte immer besser.

---

# Teil 06 — Der ernste Test

### Zelle 27: Vorbereitung

Jetzt werden zum ersten Mal die weggeschlossenen Testdaten geladen. Ab hier wird nichts mehr geändert.

`make_prep_C()` ist eine Funktion, die eine frische Kopie von Pipeline C zurückgibt. Der Grund ist technisch: Jedes Modell braucht sein eigenes, unbenutztes Fließband, sonst überschreiben sie sich gegenseitig.

### Zelle 28: Alle Modelle einmal auf den Testdaten

Sieben Zeilen in der Ergebnistabelle, darunter zwei Vergleichsmaßstäbe:

Die **Mittelwert-Baseline** sagt immer denselben Wert vorher, nämlich den Durchschnitt. Dümmer geht es nicht. Wenn ein Modell das nicht schlägt, ist es wertlos.

Das **Modell nur mit Umsatz und Rabatt** kennt keine Kategorien. Daran sieht man, was Sortiment, Markt und Segment überhaupt beitragen. Antwort: erstaunlich wenig.

Drei Maßzahlen werden ausgegeben. R² ist der erklärte Anteil. RMSE ist der typische Fehler in Dollar, bestraft große Ausreißer stark. MAE ist der durchschnittliche Fehler in Dollar und lässt sich am leichtesten erklären.

### Zelle 29: Hält das auch bei anderer Aufteilung?

Der Zufalls-Split hat eine Schwäche. Mehrere Positionen können zur selben Bestellung gehören, und die landen dann teils im Training, teils im Test. Sie teilen sich Kunde, Datum und Versandart, das Modell hat also indirekt Vorwissen.

Deshalb werden zwei strengere Varianten geprüft. Beim gruppierten Split ist keine Bestellung in beiden Hälften. Beim zeitlichen Split wird auf 2011 bis 2013 gelernt und auf 2014 geprüft, so wie es in echt wäre.

Ergebnis: Die Modelle liegen überall eng zusammen, aber die Reihenfolge kippt. Also kann man nicht behaupten, ein Modell sei besser.

### Zellen 30 und 31: Welche Merkmale zählen?

```python
permutation_importance(...)
```
Die Idee ist einfach und ziemlich elegant: Man nimmt eine Spalte und mischt sie zufällig durch. Wird das Modell danach deutlich schlechter, war die Spalte wichtig. Ändert sich nichts, war sie egal.

Ergebnis: Rabatt und Umsatz weit vorn, alles andere weit dahinter.

### Zelle 32: Was wäre ohne Umsatz?

Der Umsatz steht bei einer Anfrage vielleicht noch gar nicht fest. Ohne ihn fällt die Güte von 0,66 auf 0,49. Der Umsatz trägt also fast alles.

### Zelle 33: Vorhersage gegen Wirklichkeit

Ein Streudiagramm. Jeder Punkt ist eine Bestellung, waagerecht der echte Gewinn, senkrecht der vorhergesagte. Läge alles perfekt, wären alle Punkte auf einer Diagonalen.

---

# Teil 07 — Die eigentliche Entdeckung

### Zellen 34 bis 36: Der Gewinn ist gar nicht geschätzt, er ist gerechnet

Das ist der stärkste Teil der Arbeit.

Überlegung: Ein Händler hat einen Listenpreis und eine Marge darauf. Gibt er Rabatt, sinkt beides. Daraus lässt sich hinschreiben:

```
Gewinn = Umsatz × (Marge − Rabatt) ÷ (1 − Rabatt)
```

Wenn das stimmt, müsste sich die Marge aus den Daten zurückrechnen lassen, und sie müsste für dasselbe Produkt immer gleich sein. Genau das wird geprüft. Ergebnis: Die Schwankung innerhalb eines Produkts ist 37-mal kleiner als insgesamt. Die Marge ist also praktisch eine Produkteigenschaft.

Dann wird die Formel als Vergleich benutzt, ganz ohne trainiertes Modell. Sie kommt auf 0,736. Mit den Margen je Produkt auf 0,974. Beides besser als jedes Modell in der Arbeit.

Das klingt erst mal ernüchternd. Es ist aber genau umgekehrt: Es zeigt, dass das Problem wirklich durchdrungen wurde und nicht nur Verfahren durchprobiert wurden. Die Grenze wird ehrlich dazugesagt: Bei neuen Produkten kennt man die Marge nicht, da funktioniert es nicht.

### Zellen 37 und 38: Der Polynomgrad war nie geprüft

In Pipeline C steht `degree=2` fest. Aber das ist selbst eine Einstellung, die man prüfen müsste. Hier passiert das: Grad 3 statt 2 hebt die Güte von 0,714 auf 0,748, und das ist statistisch abgesichert (p = 0,006).

Das ist der einzige belastbare Modellunterschied der ganzen Arbeit, und er wäre nie aufgefallen.

### Zellen 39 und 40: Lieber die Marge vorhersagen

Wenn der Gewinn multiplikativ vom Umsatz abhängt, liegt es nahe, statt des Gewinns die Marge vorherzusagen und danach mit dem Umsatz zu multiplizieren. Damit ist die Formel herausgekürzt.

Bringt mehr als jede Modellwahl: von 0,654 auf 0,749.

---

# Teil 08 — Ist der Unterschied echt?

### Zellen 41 bis 43

Modell A ist 0,01 besser als Modell B. Echter Vorsprung oder Messrauschen?

Zwei Verfahren beantworten das. Der **gepaarte t-Test** vergleicht die fünf Fold-Ergebnisse paarweise und liefert einen p-Wert. Klein bedeutet: unwahrscheinlich, dass das Zufall ist. Hier kommt 0,54 heraus, also klar Zufall.

Das **Bootstrap-Konfidenzintervall** zieht tausendmal zufällig Stichproben aus den Testdaten und schaut, wie stark der Unterschied schwankt. Das Intervall reicht von −0,032 bis +0,068, schließt die Null also ein. Auch das heißt: kein belastbarer Unterschied.

Deshalb steht im Bericht nicht "Modell A gewinnt", sondern "beide sind gleich gut". Genau das erwartet ein Prüfer.

---

# Teil 09 — Wo liegt das Modell daneben?

### Zellen 44 und 45: Residuen

Ein Residuum ist die Differenz zwischen echtem und vorhergesagtem Wert, also der Fehler. Ein lineares Modell unterstellt, dass dieser Fehler überall ungefähr gleich groß streut.

Hier stimmt das nicht: Der Fehler wächst mit dem Umsatz (r = 0,74). Bei kleinen Bestellungen liegt das Modell nah dran, bei großen weit daneben. Das ist der typische Fingerabdruck eines multiplikativen Zusammenhangs und genau der Grund für den Wechsel der Zielgröße in Teil 07.

### Zelle 46: Fehler nach Bereich

Ein Modell muss nicht überall gleich gut sein. Man muss aber wissen, wo es schwächelt, bevor man es einsetzt.

### Zelle 47: Taugt das Modell für das, was der Bericht vorschlägt?

Der Bericht schlägt vor, verlustträchtige Positionen bei der Auftragsannahme zu markieren. Das ist aber keine Frage nach einer Zahl, sondern nach ja oder nein. Also eine Klassifikation, keine Regression.

Vier Varianten werden verglichen, unter anderem eine einzelne Faustregel: Rabatt über 20 Prozent, fertig. Diese Regel gewinnt (F1 = 0,849), das empfohlene Regressionsmodell ist die schlechteste Option (0,717).

Das ist unbequem, aber der ehrliche Befund. Der Bericht wurde deshalb umgeschrieben: Faustregel fürs Markieren, Modell für die Höhe des erwarteten Gewinns.

---

## Drei Sätze, die immer passen, falls dich jemand fragt

1. "Ich habe die Daten zuerst geteilt und die Testdaten bis zum Schluss nicht angefasst."
2. "Die Vorverarbeitung steckt zusammen mit dem Modell in einer Pipeline, deshalb kann bei der Kreuzvalidierung nichts aus den Prüfdaten hineinsickern."
3. "Der entscheidende Hebel war nicht die Wahl des Verfahrens, sondern das Merkmal Umsatz mal Rabatt. Das folgt direkt daraus, wie ein Gewinn kaufmännisch entsteht."
