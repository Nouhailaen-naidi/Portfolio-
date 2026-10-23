# Analyse quantitative et fondamentale de Microsoft (MSFT)

Du traitement des données avec Python à la restitution dans Power BI

**Outils** : Python (pandas, NumPy, yfinance, matplotlib, openpyxl), Ubuntu sous WSL, VS Code, Power BI Desktop, DAX
**Périmètre** : cinq ans de données boursières quotidiennes, comparaison avec deux indices de référence, analyse de rentabilité sur les derniers exercices fiscaux

---

## 1. Objectif et démarche

Ce projet mesure le couple rendement/risque de l'action Microsoft, la compare au marché, puis rapproche cette performance boursière des résultats réels de l'entreprise. L'objectif est de vérifier si la hausse du cours s'explique par la performance financière ou par d'autres facteurs.

La démarche suit quatre étapes :

1. Collecte et nettoyage automatisés des données avec Python.
2. Calcul des indicateurs de rendement, de risque et de rentabilité.
3. Export de tables structurées prêtes à être exploitées.
4. Modélisation et visualisation dans Power BI.

Il s'agit d'une analyse rétrospective, qui ne constitue pas une recommandation d'investissement.


## 2. Environnement de travail

Le projet est isolé dans un environnement virtuel Python sous Ubuntu (WSL), ce qui garantit la reproductibilité de l'installation et évite tout conflit avec les paquets système.

```bash
mkdir -p ~/microsoft_analysis && cd ~/microsoft_analysis
python3 -m venv .venv
source .venv/bin/activate
pip install yfinance pandas numpy matplotlib openpyxl
```

Le développement et l'exécution se font depuis VS Code avec la commande `python msft_pipeline.py`.


## 3. Collecte et nettoyage des données

Les cours (ouverture, plus haut, plus bas, clôture, clôture ajustée, volume, dividendes) sont récupérés via `yfinance`. Les indices S&P 500 et Nasdaq 100 servent de références, et le bon du Trésor américain à 13 semaines (`^IRX`) de taux sans risque.

```python
h = yf.Ticker("MSFT").history(period="5y", auto_adjust=False, actions=True)
```

Le nettoyage porte sur quatre points : normalisation de l'index de dates (suppression du fuseau horaire), élimination des doublons, suppression des valeurs manquantes ou négatives, et contrôle de cohérence entre les prix les plus hauts et les plus bas. Une fonction dédiée aplatit également les colonnes à plusieurs niveaux renvoyées par `yfinance`, source d'erreurs récurrentes.

```python
df = df[~df.index.duplicated(keep="last")].sort_index()
df = df.dropna(subset=["close", "adj_close"])
df = df[(df[["open", "high", "low", "close", "adj_close"]] > 0).all(axis=1)]
```


## 4. Indicateurs de rendement et de risque

Tous les calculs sont vectorisés avec pandas et NumPy et reposent sur le prix ajusté des dividendes et des divisions d'actions.

```python
a = df["adj_close"]
df["simple_return"] = a.pct_change()
df["log_return"]    = np.log(a / a.shift(1))
df["cum_return"]    = a / a.iloc[0] - 1
df["drawdown"]      = a / a.cummax() - 1
df["vol_30d_ann"]   = df["log_return"].rolling(30).std() * np.sqrt(252)
df["ma_50"], df["ma_200"] = a.rolling(50).mean(), a.rolling(200).mean()
```

Les statistiques globales couvrent le rendement annualisé, la volatilité, l'asymétrie et l'aplatissement de la distribution, le drawdown maximal, la VaR et la CVaR à 95 %, ainsi que les ratios de Sharpe, de Sortino et de Calmar.

```python
var95  = r.quantile(0.05)
cvar95 = r[r <= var95].mean()
sharpe = ex.mean() * 252 / (ex.std() * np.sqrt(252))   # ex = rendement excédentaire
```

Par rapport à chaque indice, une régression sur les rendements excédentaires donne le bêta et l'alpha annualisé, complétés par la corrélation, la tracking error et l'information ratio.

```python
beta  = y.cov(x) / x.var()
alpha = (y.mean() - beta * x.mean()) * 252
```

Les rendements sont également agrégés par mois et par année. Les périodes incomplètes sont signalées par un indicateur afin de ne pas fausser les moyennes.


## 5. Analyse fondamentale de la rentabilité

Le compte de résultat, le bilan et le tableau de flux de trésorerie sont extraits pour chaque exercice fiscal (clôture au 30 juin). Les ratios suivants sont calculés : croissance du chiffre d'affaires, marges brute, opérationnelle et nette, ROE, ROA, free cash flow et taux de conversion de trésorerie. Les multiples de valorisation (PER, PER prévisionnel, PEG, price-to-book) complètent l'analyse.

```python
f["operating_margin"] = f["operating_income"] / f["revenue"]
f["roe"]              = f["net_income"] / f["equity"]
f["free_cash_flow"]   = f["operating_cash_flow"] + f["capex"]
f["fcf_margin"]       = f["free_cash_flow"] / f["revenue"]
```

Une table de dates intègre le calendrier fiscal de Microsoft (juillet à juin), ce qui permet de relier les données de marché et les données comptables dans un même modèle.


## 6. Export des données

Le script produit un classeur Excel multi-feuilles et un fichier CSV par table :

| Table | Contenu |
|---|---|
| Daily | Prix, rendements, drawdown, volatilité et moyennes mobiles |
| Monthly, Annual | Rendements et volatilité agrégés |
| Statistics | Indicateurs globaux de rendement et de risque |
| Benchmark_Stats | Bêta, alpha, corrélation par indice |
| Compare_Base100 | Performance relative en base 100 |
| Fundamentals, Valuation | Ratios de rentabilité et multiples |
| Dividends, Dim_Date | Dividendes versés et table de dates |

Un graphique de synthèse à six panneaux (prix et moyennes mobiles, performance relative, drawdown, distribution des rendements, volatilité glissante, rendements annuels) est aussi généré pour contrôler les résultats avant leur import.


## 7. Modélisation dans Power BI

**Import et préparation.** Les feuilles du classeur sont chargées puis typées dans Power Query : dates au format Date, indicateurs en nombre décimal, indicateur de période partielle en booléen.

**Schéma en étoile.** La table `Dim_Date`, marquée comme table de dates, est reliée en un-à-plusieurs aux tables `Daily`, `Monthly`, `Annual`, `Compare_Base100`, `Dividends` et `Fundamentals`. Les tables de statistiques restent autonomes. Un segment d'année agit ainsi sur l'ensemble des visuels.

**Mesures DAX.** Les mesures sont recalculées dans Power BI et recoupées avec les valeurs obtenues en Python, ce qui constitue un contrôle de cohérence entre les deux outils.

```dax
Rendement Periode =
VAR p0 = CALCULATE(MAX(Daily[adj_close]), Daily[Date] = MIN(Daily[Date]))
VAR p1 = CALCULATE(MAX(Daily[adj_close]), Daily[Date] = MAX(Daily[Date]))
RETURN DIVIDE(p1, p0) - 1

Volatilite Annualisee = STDEV.S(Daily[log_return]) * SQRT(252)

Ratio Sharpe =
VAR rm  = AVERAGE(Daily[simple_return]) * 252
VAR vol = STDEV.S(Daily[simple_return]) * SQRT(252)
RETURN DIVIDE(rm - [Taux Sans Risque], vol)

VaR 95 = PERCENTILE.INC(Daily[simple_return], 0.05)
```

Plusieurs écarts entre Python et DAX ont été identifiés et corrigés : le rendement cumulé doit s'appuyer sur le premier et le dernier prix, et non sur les extrêmes ; le Sharpe et la VaR doivent utiliser les rendements simples pour rester cohérents avec le script ; les années partielles sont exclues des moyennes annuelles.


## 8. Tableau de bord

Le rapport comporte trois pages denses, construites autour d'une palette cohérente (bleu pour MSFT, orange pour le S&P 500, vert pour le Nasdaq 100, rouge pour le risque).

| Page | Contenu |
|---|---|
| Synthèse | Indicateurs clés (prix, rendement, rendement annualisé, volatilité, Sharpe, bêta), évolution du prix, segment d'année |
| Risque | Drawdown, volatilité glissante, distribution des rendements, VaR et CVaR |
| Comparaison et fondamentaux | Performance en base 100 face aux indices, marges, flux de trésorerie opérationnels et investissements, ROE et ROA, valorisation |

Chaque visuel est titré explicitement, les pourcentages et ratios sont formatés correctement et la source des données est mentionnée en pied de page.


## 9. Principaux enseignements

- Sur cinq ans, l'action affiche un rendement cumulé d'environ 78 %, pour une volatilité annualisée de 28 % et un drawdown maximal de 37 %. Le ratio de Sharpe de 0,42 indique un rendement correct au regard d'un risque élevé.
- Microsoft surperforme le S&P 500 mais pas le Nasdaq 100.
- Le bêta de 1,15 et un alpha annualisé limité (environ 2 %) montrent que la performance provient surtout de l'exposition au marché.
- Côté fondamentaux, la croissance du chiffre d'affaires s'accompagne d'une hausse des marges. Les investissements ont toutefois fortement augmenté, ce qui pèse sur la marge de free cash flow et sur le ROE.
- Le rapprochement entre la hausse du cours et la progression du chiffre d'affaires pose la question de la part due à l'expansion des multiples de valorisation.


## 10. Limites et prolongements

Les données proviennent de Yahoo Finance, source non auditée : les chiffres comptables doivent être confrontés aux documents officiels (rapport annuel et formulaire 10-K de la SEC). L'historique fondamental se limite à quelques exercices, et l'analyse est rétrospective.

Prolongements envisagés : intégration de l'analyse du risque de crédit, valorisation par actualisation des flux, et extension à un panier de valeurs comparables.
