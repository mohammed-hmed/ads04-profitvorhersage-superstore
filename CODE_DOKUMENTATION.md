# Code-Dokumentation — Schritt für Schritt erklärt

Diese Datei erklärt den gesamten Analyse-Code in verständlicher Sprache: **was** jeder Schritt macht, **warum** er so gemacht wird und **was dabei herauskommt**. Sie richtet sich an Leserinnen und Leser, die den Code nachvollziehen wollen, ohne jede scikit-learn-Funktion zu kennen.

Der gesamte Code liegt in **einem** Notebook: `notebooks/ADS04_Gesamtanalyse.ipynb`. Es ist von oben nach unten durchgehend ausführbar; die neun Teile (01 Datenaufbereitung bis 09 Diagnostik) entsprechen der Reihenfolge der Ausführung.

---

## Das Grundprinzip in einem Satz

Wir bauen eine „Fließband-Maschine" (Pipeline), die rohe Bestelldaten automatisch in Zahlen umwandelt, die ein Algorithmus versteht, und daran anschließend ein Vorhersagemodell trainiert — so, dass in jedem Prüfschritt garantiert kein Wissen über die Testdaten hineinsickert.

---

## Teil 01 — Datenaufbereitung

### Schritt 1: Daten einlesen

```python
xl = pd.read_excel(RAW, sheet_name=None)
```

`sheet_name=None` bedeutet: **alle** Excel-Blätter einlesen. Man bekommt kein einzelnes Tabellenblatt, sondern ein Wörterbuch `{"Orders": Tabelle, "Returns": Tabelle, "People": Tabelle}`. Nötig, weil unsere Datei drei Blätter hat und wir zwei davon brauchen.

**Ergebnis:** Orders 51.290 × 24, Returns 1.173 × 3, People 13 × 2.

### Schritt 2: Auf Lücken und Duplikate prüfen

```python
na = orders.isnull().sum()
na[na > 0]
```

`isnull()` macht aus jeder Zelle ein Ja/Nein („ist leer?"), `sum()` zählt die Ja pro Spalte. Der zweite Teil filtert auf Spalten, die überhaupt Lücken haben — sonst müsste man 24 Nullen durchlesen.

```python
orders.drop(columns=["Row ID"]).duplicated().sum()
```

`duplicated()` markiert Zeilen, die schon einmal vorkamen. **Wichtig:** Die Spalte `Row ID` wird vorher entfernt, weil sie in jeder Zeile eine andere Nummer hat — sonst wären zwei inhaltlich identische Zeilen niemals „doppelt".

**Ergebnis:** Nur `Postal Code` hat Lücken (41.296 Stück, alle außerhalb der USA), keine inhaltlichen Duplikate.

### Schritt 3: Retouren anfügen (Datenintegration)

```python
df = orders.merge(returns[["Order ID"]].assign(returned=1).drop_duplicates(),
                  on="Order ID", how="left")
df["returned"] = df["returned"].fillna(0).astype(int)
```

Zeile für Zeile:

* `returns[["Order ID"]]` — nur die Bestellnummern der Retouren.
* `.assign(returned=1)` — eine Hilfsspalte anhängen, die überall 1 ist („diese Bestellung kam zurück").
* `.drop_duplicates()` — jede Bestellnummer nur einmal, sonst würden Zeilen beim Zusammenführen vervielfacht.
* `merge(..., how="left")` — an jede Bestellposition die passende Retoureninfo kleben. „left" heißt: Alle Bestellpositionen bleiben erhalten, auch die ohne Retoure (die bekommen einen leeren Wert).
* `fillna(0)` — leer bedeutet „keine Retoure", also 0.

**Wichtig für die Bewertung:** Dieses Flag wird **nicht** als Merkmal fürs Modell benutzt. Ob etwas zurückgeschickt wird, weiß man beim Bestelleingang noch nicht — es wäre Wissen aus der Zukunft (*data leakage*) und würde das Modell unrealistisch gut aussehen lassen.

### Schritt 4: Neue Merkmale bilden (Feature Engineering)

```python
df["shipping_days"]   = (df["Ship Date"] - df["Order Date"]).dt.days
df["order_month"]     = df["Order Date"].dt.month
df["order_year"]      = df["Order Date"].dt.year
df["unit_price"]      = df["Sales"] / df["Quantity"]
df["ship_cost_ratio"] = df["Shipping Cost"] / df["Sales"]
```

Ein Datum als solches kann ein Algorithmus nicht verwerten. Also wird daraus herausgerechnet, was inhaltlich interessant ist: Wie lange dauerte die Lieferung? In welchem Monat und Jahr wurde bestellt? Was kostet ein einzelnes Stück? Welchen Anteil am Umsatz frisst der Versand?

### Schritt 5: Train/Test-Split — der wichtigste Schritt

```python
train_df, test_df = train_test_split(df, test_size=0.2, random_state=42)
```

20 % der Daten werden weggeschlossen und bis Teil 06 nicht angefasst. Grund: Ein Modell an denselben Daten zu bewerten, mit denen es gelernt hat, ist wie eine Klausur mit vorher bekannten Lösungen. Nur ungesehene Daten zeigen, ob es wirklich etwas kann.

`random_state=42` fixiert den Zufall, damit bei jedem Durchlauf exakt dieselben Zeilen im Testset landen — sonst wären die Ergebnisse nicht reproduzierbar.

**Ergebnis:** 41.032 Trainings- und 10.258 Testzeilen.

### Schritt 6: Speichern

```python
df.to_pickle(f"{PROC}/gs_clean.pkl")
```

Pickle statt CSV, weil Pickle die Datentypen erhält — ein Datum bleibt ein Datum und wird nicht beim nächsten Einlesen zu Text.

---

## Teil 02 — Explorative Datenanalyse

Alles hier läuft **nur auf den Trainingsdaten**.

### Korrelationen

```python
train_df[[...]].corr()
```

Die Korrelation misst auf einer Skala von −1 bis +1, wie stark zwei Größen gemeinsam steigen oder fallen. `Sales` und `Profit`: 0,50 (mehr Umsatz, mehr Gewinn). `Discount` und `Profit`: −0,32 (mehr Rabatt, weniger Gewinn).

### Rabattklassen bilden

```python
bins = pd.cut(train_df["Discount"], [-0.01, 0.0, 0.2, 0.4, 0.6, 0.85])
train_df.groupby(bins, observed=False)["Profit"].agg(["mean", "count"])
```

`pd.cut` steckt jede Bestellung in eine Rabatt-Schublade (0 %, bis 20 %, bis 40 %, …). `groupby` rechnet dann je Schublade den mittleren Profit aus. Genau daraus entsteht der Kernbefund: Ab mehr als 20 % Rabatt wird der Mittelwert negativ.

### Abbildung mit Mittelwertlinie

Das reine Streudiagramm zeigt nur eine Punktwolke. Deshalb werden die Klassenmittelwerte als rote Linie darübergelegt — erst dadurch wird der Kipppunkt sichtbar. (Ein Diagramm muss die Aussage tragen, die im Text steht.)

---

## Teil 03 — Die drei Transformationspipelines

### Warum überhaupt Pipelines?

Ein Algorithmus kann nicht mit dem Wort „Nordamerika" rechnen und verwirrt sich, wenn ein Merkmal in Tausendern (Umsatz) und ein anderes in Einern (Stückzahl) vorliegt. Also müssen die Daten umgeformt werden. Der entscheidende Punkt: Diese Umformung muss **innerhalb** der Kreuzvalidierung passieren, sonst fließt Information aus den Validierungsdaten in die Umformung ein — wieder Leckage. Eine `Pipeline` verbindet Umformung und Modell zu **einem** Objekt und löst das automatisch.

### Die Bausteine

| Baustein | Was er macht | Warum |
|---|---|---|
| `SimpleImputer(strategy="median")` | füllt fehlende Zahlen mit dem Median | Median statt Mittelwert, weil er robust gegen Ausreißer ist |
| `StandardScaler()` | rechnet jedes Merkmal auf Mittelwert 0 und Streuung 1 um | damit Umsatz (Tausender) und Stückzahl (Einer) gleiches Gewicht bekommen |
| `OneHotEncoder(sparse_output=False)` | macht aus „Markt = EU" eine eigene 0/1-Spalte je Ausprägung | erzeugt keine künstliche Rangfolge (anders als 1, 2, 3 …); `sparse_output=False` liefert eine dichte Matrix, sonst wird kNN extrem langsam |
| `PolynomialFeatures(degree=2)` | ergänzt Quadrate und Produkte, z. B. `Sales × Discount` | bildet ab, dass Umsatz und Rabatt **gemeinsam** wirken |
| `FunctionTransformer(np.log1p)` | staucht große Werte, `log1p(x) = log(1+x)` | die „+1" verhindert `log(0)`, was minus unendlich wäre |
| `ColumnTransformer` | wendet verschiedene Bausteine auf verschiedene Spaltengruppen an | Zahlen brauchen Skalierung, Kategorien brauchen Kodierung |

### Die drei Varianten

* **A (Basis):** Zahlen skalieren, Kategorien kodieren. Der einfachste sinnvolle Aufbau.
* **B (Feature Engineering):** zusätzlich `log1p(Sales)` und die fünf abgeleiteten Merkmale.
* **C (Interaktionen):** Produkte der Zahlen, insbesondere `Sales × Discount`.

Warum C? Weil der Gewinn ökonomisch etwa so entsteht: `Profit ≈ Marge · Umsatz − Rabattanteil · Umsatz`. Der Rabatt wirkt also nicht additiv, sondern **multipliziert** mit dem Umsatz. Genau dieses Produkt kann ein lineares Modell nur sehen, wenn man es ihm als Merkmal gibt.

**Ergebnis:** Die linearen Modelle springen von R² 0,34 (A) auf 0,71 (C) — die Vermutung war richtig.

### Der Vergleich

```python
s = cross_val_score(Pipeline([("prep", prep), ("model", model)]),
                    X_train, y_train, cv=kf, scoring="r2")
```

`cross_val_score` teilt die Trainingsdaten in fünf Teile, trainiert fünfmal auf vier Teilen und prüft am fünften. Man erhält fünf Werte; berichtet werden Mittelwert **und Streuung**. Die Streuung ist wichtig: Liegen zwei Modelle näher beieinander als die Streuung groß ist, ist der Unterschied nicht belastbar.

**R² erklärt:** 1,0 = perfekt, 0 = so gut wie immer den Mittelwert raten, negativ = schlechter als raten.

---

## Teil 04 — Hyperparameter-Tuning

**Parameter** lernt das Modell selbst aus den Daten. **Hyperparameter** legt man vorher fest — z. B. wie viele Nachbarn kNN betrachtet oder wie stark Ridge bestraft wird. Die Rastersuche probiert systematisch alle vorgegebenen Werte durch:

```python
gs = GridSearchCV(Pipeline([("prep", prep_C), ("model", model)]),
                  grid, cv=kf, scoring="r2")
gs.fit(X_train, y_train)
```

Für jede Kombination wird eine vollständige 5-fache Kreuzvalidierung gerechnet und am Ende die beste ausgewählt. Wichtig: Das passiert **nur** auf den Trainingsdaten — die Testdaten bleiben unberührt.

**Ergebnis:** Ridge alpha = 0,01; kNN k = 10; Random Forest 100 Bäume, Tiefe 12.

Das alpha-Raster reicht bis 100.000, um zu zeigen, *ab wann* Regularisierung überhaupt wirkt: Bis alpha = 100 ändert sich nichts, erst bei 10.000 fällt die Güte ab. Das ist der Befund „bei 41.000 Zeilen und 40 Merkmalen gibt es kein Overfitting-Problem" — und erklärt, warum Ridge und die einfache lineare Regression dieselben Zahlen liefern.

---

## Teil 05 — Overfitting sichtbar machen

```python
for d in [2, 4, 6, 8, 10, 14, 20]:
    cvs = cross_val_score(...)     # Güte auf ungesehenen Daten
    p.fit(X_train, y_train)        # Güte auf den Trainingsdaten
```

Für jede Baumtiefe werden zwei Werte gemessen: wie gut das Modell die Trainingsdaten trifft und wie gut es auf ungesehenen Daten abschneidet. Ergebnis: Der Trainingswert steigt immer weiter (bis 0,977 bei Tiefe 20), der Validierungswert erreicht bei Tiefe 8 sein Maximum (0,569) und fällt dann ab (0,449).

**Die Lehre:** Ein Modell, das die Trainingsdaten fast perfekt trifft, hat sie oft nur auswendig gelernt. Deshalb wird die beste Tiefe aus der **Validierungs**-Kurve abgelesen, nicht aus dem Trainingswert.

---

## Teil 06 — Finale Bewertung

### Schritt 1: Testdaten (einmalig!)

```python
p = Pipeline([("prep", make_prep_C()), ("model", model)]).fit(X_train, y_train)
pred = p.predict(X_test)
r2_score(y_test, pred), np.sqrt(mean_squared_error(y_test, pred)), mean_absolute_error(y_test, pred)
```

Drei Maße, weil sie Unterschiedliches messen:

* **R²** — Anteil der erklärten Streuung. Quadriert die Fehler, große Ausreißer zählen also überproportional.
* **RMSE** — mittlerer Fehler in USD, ebenfalls ausreißerempfindlich.
* **MAE** — durchschnittlicher Fehler in USD ohne Quadrierung, zeigt den *typischen* Fehler.

Deshalb kann ein Modell beim R² gewinnen und beim MAE verlieren — genau das passiert hier zwischen linearem Modell und Random Forest.

### Schritt 2: Baselines — die ehrliche Frage „bringt das Modell überhaupt was?"

```python
DummyRegressor(strategy="mean")     # rät immer den Mittelwert
```

Ohne Vergleichsmaßstab sagt R² = 0,66 wenig. Die Mittelwert-Baseline liefert per Definition R² = 0 und einen MAE von 67,52 USD. Unser Modell kommt auf 35,80 bis 41,03 USD — also rund 40 % weniger Fehler.

Die zweite Baseline nutzt nur `Sales` und `Discount` und erreicht bereits R² = 0,660. Der Zugewinn durch alle übrigen Merkmale beträgt also nur 0,003. Dieses unbequeme Ergebnis wird bewusst berichtet, weil es die ehrliche Einordnung der Modellgüte ist.

### Schritt 3: Robustheitsprüfung

```python
GroupShuffleSplit(...).split(Xf, yf, groups=full["Order ID"])
```

Beim zufälligen Split können zwei Positionen derselben Bestellung in Training und Test landen — sie teilen Kunde, Datum, Markt. Der **gruppierte** Split verhindert das, der **zeitliche** Split (2011–2013 → 2014) simuliert die echte Vorhersagesituation.

Ergebnis: Die Rangfolge der Modelle dreht sich. Das ist eine wichtige Erkenntnis — sie verhindert eine überzogene Behauptung im Bericht.

### Schritt 4: Merkmalswichtigkeit

```python
permutation_importance(pipe_best, X_test, y_test, n_repeats=5, scoring="r2")
```

Idee: Man mischt die Werte eines Merkmals zufällig durch und misst, wie stark die Güte einbricht. Bricht sie stark ein, war das Merkmal wichtig. Ergebnis: `Discount` (1,447) und `Sales` (1,353) dominieren; alles andere ist nahezu bedeutungslos.

---

## Wiederkehrende Muster im Code

| Muster | Bedeutung |
|---|---|
| `random_state=42` | fixiert den Zufall, damit Ergebnisse reproduzierbar sind |
| `X = df.drop(columns=["Profit", "returned"])` | Merkmale ohne Zielgröße und ohne Zukunftswissen |
| `Pipeline([...])` | Vorverarbeitung und Modell als ein Objekt — leckagefrei |
| `cv=KFold(5, shuffle=True, random_state=42)` | überall dasselbe Validierungsschema, damit Zahlen vergleichbar sind |
| `.fit(...)` / `.predict(...)` / `.score(...)` | lernen / vorhersagen / bewerten — die drei Grundoperationen in scikit-learn |

---

## Häufige Fragen

**Warum nicht einfach das genaueste Modell nehmen?**
Weil der Unterschied (0,663 vs. 0,653) kleiner ist als die Schwankung zwischen den Validierungsdurchläufen (± 0,045) und sich bei anderem Split umdreht. Bei gleicher Güte entscheidet man nach Interpretierbarkeit und Laufzeit.

**Warum wurden die Ausreißer nicht entfernt?**
Weil sie keine Fehler sind, sondern die betriebswirtschaftlich wichtigsten Fälle: Großaufträge und teure Fehlgeschäfte. Sie zu löschen würde die Kennzahlen schöner machen und das Modell unbrauchbarer.

**Warum ist `Postal Code` gelöscht und nicht aufgefüllt worden?**
Weil die Lücke systematisch ist: Alle Nicht-US-Bestellungen haben keine Postleitzahl in diesem Format. Ein erfundener Wert wäre schlicht falsch.
