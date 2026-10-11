# us_quant_lab — Laboratoire de recherche quantitative (actions/ETFs US)

Backtest de portefeuille, mean-reversion de Connors (RSI(2)), univers d'ETFs US.

> **Projet de RECHERCHE UNIQUEMENT.** Aucun code de trading réel, aucune connexion à un
> courtier, aucune clé d'API de courtier. Tous les résultats sont **rétrospectifs** et ne
> constituent en aucun cas une prédiction de performance future.

État du document : **BROUILLON — en attente de validation** (voir § 7, journal de scellement).

---

## 1. Pourquoi ce protocole est strict

Objectif : ne pas se tromper soi-même. Chaque règle ci-dessous répond à un risque identifié.

| Risque | Règle qui y répond |
|---|---|
| Biais du survivant | Univers d'ETFs larges/sectoriels (pas de stock-picking) ; univers S&P 100 optionnel marqué « biais majeur » |
| Lookahead | Signaux calculés au Close de T, exécution à l'Open de T+1 ; test dédié |
| Résultats flatteurs par hasard (multiplicité) | **Un seul** run principal, défini à l'avance ; runs de sensibilité explicitement non décisionnels |
| Critères fixés après coup | README commité **avant** tout run ; aucun seuil modifié après un résultat |
| Coûts sous-estimés (capital 1 000 USD) | Actions entières, commissions minimum par ordre, taxes datées, retenue à la source, slippage défavorable |
| Fausse indépendance statistique | Significativité évaluée sur les **épisodes** de signaux, pas sur les trades |
| Sur-ajustement OOS | Paramètres fixes issus de la littérature ; OOS exécuté **une seule fois**, verrou logiciel |

---

## 2. Protocole pré-engagement

### 2.1 Run principal (unique, défini à l'avance)

| Paramètre | Valeur |
|---|---|
| Stratégie | **Variante C** (Connors Pullback) : achat si Close > SMA200 **et** RSI(2) < 10 ; vente si Close > SMA10 |
| Univers | SPY, QQQ, IWM, XLF, XLE, XLK, XLV, XLY, XLP, XLI, XLU |
| Capital initial | 1 000 USD |
| Positions max | 4 (~25 % chacune) |
| Grille de commission | IBKR Pro **Fixe** |
| Actions | **Entières** (`allow_fractional=False`) |
| Retenue à la source sur dividendes | **30 %** (benchmark SPY : même retenue) |
| Slippage | 0,05 % par exécution, sens défavorable |
| Conversion de devises | 2 USD, une seule fois, sur le capital initial |
| Rendement du cash | 0 % |

Le verdict (abandon / passage à l'OOS / réussite OOS) est rendu **uniquement** sur ce run.

### 2.2 Runs de sensibilité (exploratoires)

Ils ne modifient **jamais** le verdict du run principal. Ils servent à comprendre d'où vient le résultat.

| Run | Ce qui change par rapport au run principal |
|---|---|
| S1 | Variante **A** (RSI(2) < 10 / RSI(2) > 60, sans filtre SMA200) |
| S2 | Variante **B** (Close > SMA200 et RSI(2) < 10 / RSI(2) > 60) |
| S3 | Commission IBKR Pro **Dégressive** (présentée comme **borne basse** : pass-through exchange/clearing non inclus) |
| S4 | Actions **fractionnées** (`allow_fractional=True`) — sert aussi à interpréter un OOS « non concluant », voir § 2.5 |

Les runs de sensibilité sont lancés sur l'IS en même temps que le run principal. Sur l'OOS, ils
sont lancés dans le **même** lancement scellé que le run principal (un seul verrou, voir § 2.6),
jamais après coup.

### 2.3 Critères d'abandon — évalués sur l'IN-SAMPLE (2004-01-01 → 2015-12-31), run principal

La stratégie est **abandonnée** (pas d'OOS) si **au moins un** des critères suivants est vrai :

1. Nombre d'épisodes fusionnés IS **< 30**
2. CAGR net **≤ 0 %** **ou** CAGR net **< CAGR de SPY (net, avec retenue) − 3 points**
3. Drawdown maximal **> 30 %**
4. Profit factor net **< 1,25**

Si aucun critère n'est déclenché, la stratégie **passe** et l'OOS attend un feu vert explicite.

### 2.4 Critères de réussite OOS — **PROPOSITION À VALIDER AVANT LE PREMIER RUN**

Trois jeux de seuils sont proposés. **Un seul** sera retenu et inscrit ici ; les deux autres seront supprimés
du README avant le commit de scellement. Tous les seuils s'appliquent au run principal sur l'OOS
(2016-01-01 → date de scellement), sauf la règle « non concluant » du § 2.5 qui est évaluée en premier.

| Critère | Option 1 « Standard » (recommandée) | Option 2 « Stricte » | Option 3 « Souple » |
|---|---|---|---|
| Épisodes fusionnés OOS (minimum, sinon OOS **non concluant**) | ≥ 25 | ≥ 30 | ≥ 20 |
| CAGR net | > 0 % | > 0 % **et** ≥ CAGR SPY net − 3 pts | > 0 % |
| CAGR / drawdown max vs SPY net (même période) | ≥ 1,0 × celui de SPY | ≥ 1,5 × celui de SPY | ≥ 0,8 × celui de SPY |
| Drawdown maximal | ≤ 30 % | ≤ 25 % | ≤ 30 % |
| Profit factor net | ≥ 1,25 | ≥ 1,40 | ≥ 1,15 |
| Dégradation IS → OOS (optionnel) | PF net OOS ≥ 0,6 × PF net IS | PF net OOS ≥ 0,75 × PF net IS | non évalué |

Verdict : **réussite** si **tous** les critères retenus sont vrais ; **échec** sinon ; **non concluant** si
le minimum d'épisodes n'est pas atteint ou si la règle du § 2.5 se déclenche.

Justification du seuil d'épisodes : le seuil IS de 30 épisodes porte sur 12 ans (2,5/an). L'OOS couvre
~10,75 ans ; la proportionnalité donne ~27, arrondi à 25 pour l'option Standard.

### 2.5 Règle « OOS non concluant » (budget insuffisant)

Si, sur l'OOS, **plus de 30 %** des **ordres d'achat tentés** par le run principal sont **rejetés faute de
budget** (quantité entière = 0 car l'ETF est devenu trop cher pour ~250 USD), l'OOS est déclaré
**non concluant** — et non raté. Il est alors interprété avec le run S4 (actions fractionnées), à titre
indicatif uniquement.

Définition d'un « ordre d'achat tenté » : un signal d'achat qui, après classement par RSI(2) croissant,
obtient une place parmi les positions libres à T+1. Les signaux non retenus faute de place ne comptent
pas comme rejets.

### 2.6 Verrou OOS : exécution unique, code et paramètres scellés

- L'OOS est exécuté **une seule fois**, par `python main.py --oos`.
- Avant d'exécuter, `main.py` :
  1. refuse si `data/oos_lock.json` existe déjà (**aucune option de contournement** dans le code) ;
  2. refuse si l'arbre de travail Git n'est pas propre (fichiers suivis modifiés non commités) ;
  3. enregistre dans `data/oos_lock.json` : horodatage UTC, hash du commit Git, SHA-256 de chaque fichier
     `.py` et du `README.md`, dernière date de données utilisée, paramètres complets du run.
- Si la stratégie échoue en OOS, **aucun réajustement** (paramètres, règles, univers, coûts) n'est toléré.
  Une nouvelle idée = un nouveau projet, un nouveau README, un nouveau pré-engagement.

### 2.7 Immutabilité

Aucun seuil, paramètre ou définition de ce README n'est modifié après avoir vu un résultat (IS ou OOS).
Les seules modifications autorisées après scellement sont des corrections de **bugs manifestes** du code,
documentées dans le § 7, et qui déclenchent alors une ré-exécution de l'**IS uniquement** ; l'OOS reste
soumis au verrou (si déjà exécuté, il n'est pas relancé).

---

## 3. Paramètres figés

### 3.1 Données (`data_loader.py`)

- Source : **yfinance** (outil non officiel, acceptable pour le prototypage uniquement). Cache local en
  **parquet** dans `data/` ; pas de retéléchargement à chaque run (rafraîchissement explicite par option).
- Téléchargement à partir du **2003-01-01**. Les ~200 premières séances servent uniquement de chauffe pour
  la SMA200. Le backtest commence au **premier jour ouvré de 2004** ; le chargeur vérifie que la SMA200 est
  valide pour tous les titres de l'univers à cette date (sinon erreur).
- Deux jeux de données :
  - **(a) BRUTS** : Open, Close non ajustés des dividendes (`auto_adjust=False`), dividendes bruts par
    action (à la date de détachement), splits ;
  - **(b) AJUSTÉS** : Close ajusté (dividendes + splits), Open ajusté par le même facteur.
- **Signaux** : calculés sur les prix **ajustés** (continuité des indicateurs).
- **Quantités, frais, trésorerie, valorisation** : calculés sur les prix **BRUTS**.
- Splits : aucun titre de l'univers n'a de split sur 2003 → aujourd'hui (vérifié par le chargeur ; si un
  split apparaît, les quantités détenues sont ajustées le jour du split et un avertissement est loggé).
- **Test de cohérence obligatoire** avant tout backtest : le rendement total de SPY reconstruit avec
  prix bruts + dividendes bruts réinvestis au Close du jour de détachement doit correspondre au rendement
  total calculé avec l'Adj Close. **Seuil : écart de CAGR > 0,1 point/an → `DataIntegrityWarning`**
  explicite avec le détail (CAGR des deux méthodes, écart, périodes). Les petits écarts sont normaux
  (moment de réinvestissement différent).
- **Détection d'anomalies de distributions** : toute distribution > 5 % du Close de la veille est listée
  dans le rapport d'intégrité (exemple connu : XLF, septembre 2016, distribution en nature d'XLRE
  enregistrée par Yahoo comme un dividende d'environ 4,44 USD). Le traitement retenu est **à décider à la
  lecture du rapport d'intégrité, avant tout backtest** (voir § 5, point 6).

### 3.2 Périodes

| Période | Début | Fin |
|---|---|---|
| Chauffe | 2003-01-01 | dernier jour ouvré de 2003 |
| In-sample (IS) | 2004-01-01 (premier jour ouvré) | 2015-12-31 |
| Out-of-sample (OOS) | 2016-01-01 | dernière séance complète disponible à la date du run scellé (enregistrée dans le verrou) |

Aucun paramètre n'est ajusté sur l'OOS. Les paramètres **(10, 60, 200, 10, 2)** sont fixes, issus de la
littérature publiée (Connors & Alvarez).

### 3.3 Stratégies (`strategy.py`) — signaux calculés au Close de T

- RSI(2) = RSI de **Wilder**, période 2 (lissage exponentiel α = 1/2, amorçage par moyenne simple).
- SMA200 et SMA10 : moyennes simples du **Close ajusté**.

| Variante | Entrée (signal d'achat à T) | Sortie (signal de vente à T) |
|---|---|---|
| A | RSI(2) < 10 | RSI(2) > 60 |
| B | Close > SMA200 **et** RSI(2) < 10 | RSI(2) > 60 |
| **C** (principal) | Close > SMA200 **et** RSI(2) < 10 | Close > SMA10 |

Toutes les comparaisons sont strictes. Aucun signal n'utilise de donnée postérieure à T (test dédié).

### 3.4 Portefeuille et exécution (`portfolio_backtester.py`)

- Exécution à l'**Open de T+1** (Market on Open), **jamais** au Close de T.
- Ordre des opérations à T+1 :
  1. Crédit des dividendes détachés à T+1 (voir ci-dessous).
  2. **Ventes** des positions ayant un signal de sortie à T → cash libéré.
  3. **Classement** des candidats à l'achat (signal d'achat à T, titre non détenu, titre non vendu à cette
     même ouverture) par **RSI(2) croissant** ; on retient autant de candidats que de places libres.
  4. **Budget cible par ligne** = valeur totale du portefeuille à l'Open de T+1 (cash après ventes et
     dividendes + positions restantes valorisées à l'Open brut) / 4 ; **budget effectif** =
     min(budget cible, cash disponible restant), les candidats étant servis dans l'ordre du classement.
  5. **Quantité** = plancher(budget effectif / (Open brut × (1 + slippage))). Actions **entières** par
     défaut. Si quantité = 0 → **ordre rejeté**, log explicite (date, titre, Open, budget effectif).
     Avec `allow_fractional=True`, quantité = budget effectif / (Open brut × (1 + slippage)) sans arrondi.
  6. Prix exécutés : achat à Open × (1 + slippage), vente à Open × (1 − slippage).
- Pas de pyramidage : un titre déjà détenu ne reçoit pas d'achat supplémentaire.
- **Dividendes** : à la date de détachement D, si le portefeuille détient Q actions à la clôture de D−1,
  crédit de cash le matin de D : **Q × dividende brut × (1 − retenue)**. Retenue paramétrable, défaut
  **30 %** (le taux réel dépend de la convention fiscale et du formulaire W-8BEN — à vérifier par le
  propriétaire du projet). Le benchmark SPY subit la **même** retenue.
- Rendement du cash inactif : **0 %**.
- Valeur liquidative quotidienne (pour les métriques) : cash + Σ Q × Close brut.

### 3.5 Coûts — paramètres datés et sourcés (vérifiés le 2026-10-11)

**Slippage** : 0,05 % par exécution, sens défavorable (paramétrable).

**Commissions IBKR Pro** (actions/ETFs US, source : interactivebrokers.com, « Commissions — Stocks », consulté
le 2026-10-11) :

| Grille | Par action | Minimum par ordre | Maximum par ordre | Remarque |
|---|---|---|---|---|
| Fixe | 0,005 USD | 1,00 USD | 1 % de la valeur de l'ordre | Inclut les frais d'exchange/clearing ; frais réglementaires en sus |
| Dégressive (palier ≤ 300 000 actions/mois) | 0,0035 USD | 0,35 USD | 1 % de la valeur de l'ordre | **Borne basse** : pass-through exchange, clearing et autres non inclus |

Règle IBKR reprise telle quelle : si le maximum calculé (1 %) est inférieur au minimum, c'est le maximum
qui s'applique. Commission = min(max(minimum, taux × Q), 1 % × Q × prix exécuté). Mêmes commissions pour
les actions fractionnées.

**Taxe SEC Section 31** (ventes uniquement) — table **datée** (sources : SEC Fee Rate Advisories et
Federal Register ; « charge date » = date de transaction) :

| Du (inclus) | Au (inclus) | Taux (USD par million de valeur vendue) | Statut |
|---|---|---|---|
| 2004-01-01 | 2019-04-15 | **20,00 (approximation documentée)** | Approximation : les taux réels ont oscillé entre ~5 et ~30 USD/M sur la période ; impact ≈ 0,5 cent sur une vente de 250 USD, non significatif |
| 2019-04-16 | 2020-02-17 | 20,70 | Sourcé |
| 2020-02-18 | 2021-02-24 | 22,10 | Sourcé |
| 2021-02-25 | 2022-05-13 | 5,10 | Sourcé |
| 2022-05-14 | 2023-02-26 | 22,90 | Sourcé |
| 2023-02-27 | 2024-05-21 | 8,00 | Sourcé |
| 2024-05-22 | 2025-05-13 | 27,80 | Sourcé |
| 2025-05-14 | 2026-04-03 | 0,00 | Sourcé |
| 2026-04-04 | — | 20,60 | Sourcé (Release 34-104909) |

**FINRA Trading Activity Fee** (ventes uniquement) — table **datée** (source : FINRA, Fee Adjustment
Schedule SR-FINRA-2024-019 et SR-FINRA-2026-021) :

| Du (inclus) | Au (inclus) | Par action (USD) | Plafond par ordre (USD) | Statut |
|---|---|---|---|---|
| 2004-01-01 | 2023-12-31 | **0,000166 (approximation documentée)** | 8,30 | Approximation : taux réels inférieurs sur la période (légèrement conservateur) ; impact < 0,1 cent par ordre de quelques actions |
| 2024-01-01 | 2025-12-31 | 0,000166 | 8,30 | Sourcé |
| 2026-01-01 | 2026-09-30 | 0,000195 | 9,79 | Sourcé |
| 2026-10-01 | 2026-12-31 | 0,00 | 0,00 | Sourcé — suspension temporaire (SR-FINRA-2026-021) |
| 2027-01-01 | — | 0,000232 | 11,61 | Sourcé (programmé) |

> Note : le cahier des charges initial indiquait 0,000166 USD / 8,30 USD « à vérifier ». Vérification faite :
> c'est le taux 2024-2025 ; le taux 2026 est différent, d'où la table ci-dessus.

**Conversion de devises** : forfait **2 USD**, appliqué **une seule fois** au capital initial (paramétrable).

---

## 4. Métriques et définitions (`reporting.py`)

Pour chaque run, trois colonnes : **IS**, **OOS**, **Total**. Les métriques sur une période sont calculées
sur la trajectoire de valeur liquidative restreinte à cette période (la valeur liquidative n'est pas remise
à 1 000 USD au début de l'OOS dans la colonne Total ; dans la colonne OOS, la base est la valeur liquidative
au dernier jour de l'IS).

| Métrique | Définition |
|---|---|
| CAGR | (VL fin / VL début)^(1 / années) − 1, années = jours calendaires / 365,25 |
| Drawdown maximal | max sur le temps de (1 − VL / plus haut historique de VL) |
| CAGR / drawdown max | ratio simple (non défini si drawdown = 0) |
| Exposition moyenne | **moyenne dans le temps** de (Σ Q × Close brut) / VL, calculée chaque séance (pas un simple « au moins une position ») |
| Rendement par unité d'exposition | CAGR / exposition moyenne |
| Win rate net | part des trades clôturés dont le P&L **net** est > 0 |
| Profit factor net | Σ P&L nets positifs / |Σ P&L nets négatifs| des trades clôturés |
| Nombre de trades | trades clôturés (aller-retour) dans la période (date de sortie) ; les positions ouvertes en fin de période sont comptées à part |
| Nombre d'épisodes | voir ci-dessous |
| Durée moyenne de détention | moyenne du nombre de séances entre l'Open d'entrée et l'Open de sortie |
| Ordres rejetés | nombre d'ordres d'achat tentés avec quantité = 0, **par ETF et par période**, et en % des ordres tentés |
| Benchmark SPY | indice de rendement total **net** : SPY avec dividendes nets de retenue (30 %) réinvestis au Close du jour de détachement, sans contrainte d'actions entières ni commission (indice de référence, pas un portefeuille simulé) |

**P&L net d'un trade** = produit de vente net (après slippage, commission, SEC, TAF) − coût d'achat net
(après slippage, commission) + dividendes nets perçus pendant la détention.

**Épisode** : on prend l'union, sur tout l'univers, des séances où **au moins un** signal d'achat de la
variante est émis (que l'ordre soit exécuté, rejeté ou non retenu faute de place). Deux séances de signal
séparées de **moins de 5 séances** (`episode_merge_gap = 5`, c'est-à-dire ≤ 4 séances de bourse d'écart)
appartiennent au **même** épisode. Les ETFs étant très corrélés, un épisode = un choc de marché distinct,
pas un trade indépendant. **Les seuils de significativité s'appliquent aux épisodes, pas aux trades bruts.**

**Alerte en rouge** dans le rapport si le nombre d'épisodes d'une période < 30.

Avertissement obligatoire dans tout rapport utilisant l'univers S&P 100 : « **Biais du survivant majeur :
composants actuels, pas historiques.** »

---

## 5. Décisions d'implémentation (ambiguïtés levées à l'avance)

1. **Vente et achat du même titre à la même ouverture** : un titre vendu à l'Open de T+1 n'est pas éligible
   à un rachat à cette même ouverture (il redevient éligible dès T+2).
2. **Ordre de service des achats** : les candidats sont servis dans l'ordre RSI(2) croissant ; le budget
   cible est recalculé une seule fois (VL à l'Open de T+1 / 4) pour tous les candidats du jour ; le budget
   effectif est borné par le cash restant après chaque achat.
3. **Dernier jour de données** : les signaux de la dernière séance ne peuvent pas être exécutés (pas de T+1) ;
   ils sont comptés dans les épisodes mais pas dans les ordres.
4. **Fin de période** : les positions ouvertes à la fin de l'IS sont conservées et continuent dans l'OOS
   (un seul portefeuille continu 2004 → fin) ; les métriques par période utilisent la VL marquée au marché.
5. **Volume** : non utilisé (pas de filtre de liquidité ; les ETFs de l'univers sont très liquides).
6. **Distributions anormales** (> 5 % du prix) : traitement décidé **avant tout backtest** sur la base du
   rapport d'intégrité. Options possibles : (a) traiter comme dividende cash (surestime le cash si c'est une
   distribution en nature) ; (b) plafonner / exclure la distribution et logger. Choix à inscrire ici.
7. **Arrondi monétaire** : cash et frais en float64, sans arrondi au cent (simplicité ; l'écart est
   négligeable).
8. **Jours fériés / séances manquantes** : le calendrier est celui des données SPY ; un titre sans cotation un
   jour où SPY cote est exclu des signaux et des ordres ce jour-là, avec avertissement.

---

## 6. Structure du projet

```
us_quant_lab/
├── README.md                  # ce document (protocole pré-engagement)
├── data/                      # cache parquet, rapport d'intégrité, oos_lock.json
├── data_loader.py             # téléchargement, cache, bruts vs ajustés, test de cohérence SPY
├── strategy.py                # RSI(2) Wilder, SMA, signaux A/B/C, épisodes
├── portfolio_backtester.py    # moteur événementiel, sizing, coûts datés, dividendes
├── reporting.py               # métriques IS/OOS/Total, rapport, alertes, critères
├── main.py                    # orchestration, runs IS, verrou OOS
└── tests/                     # pytest
```

---

## 7. Journal de scellement

| Date | Événement | Commit |
|---|---|---|
| 2026-10-11 | Rédaction du README (brouillon) | — |
| — | Validation des critères OOS (§ 2.4) et des décisions § 5 par le propriétaire du projet | — |
| — | Commit de scellement du README (avant tout run) | — |
| — | Rapport d'intégrité des données validé | — |
| — | Run principal IS exécuté ; critères d'abandon appliqués | — |
| — | Feu vert explicite OOS | — |
| — | OOS exécuté (verrou créé) | — |

---

## Annexe A — Plan technique (pour validation avant codage)

### A.1 Environnement

- Python 3.14 (installé : 3.14.6). Dépendances : `pandas`, `numpy`, `pyarrow`, `yfinance`, `pytest`.
  Aucune bibliothèque de backtest tierce : moteur événementiel écrit à la main pour être auditable.
- Environnement virtuel `.venv` dans `us_quant_lab/`, `requirements.txt` avec versions épinglées au moment
  de l'installation.

### A.2 `data_loader.py`

- `UNIVERSE_ETF`, `UNIVERSE_SP100` (liste statique des composants actuels + constante d'avertissement).
- `load_universe(tickers, start="2003-01-01", refresh=False) -> MarketData` : télécharge via
  `yf.download(..., auto_adjust=False, actions=True)`, met en cache `data/raw_{ticker}.parquet`.
- `MarketData` : `open_raw`, `close_raw`, `open_adj`, `close_adj`, `dividends`, `splits` (DataFrames
  date × ticker), `calendar` (index des séances SPY).
- `warmup_check(md, start="2004-01-01")` : premier jour ouvré de 2004, SMA200 valide pour tous.
- `spy_integrity_report(md, threshold_pp=0.1) -> IntegrityReport` : CAGR « bruts + dividendes réinvestis »
  vs CAGR « Adj Close », écart, `DataIntegrityWarning` si écart > seuil ; liste des distributions > 5 % ;
  liste des splits ; séances manquantes par ticker. Écrit `data/integrity_report.md`.

### A.3 `strategy.py`

- `rsi_wilder(close, period=2)`, `sma(close, n)`.
- `compute_indicators(close_adj) -> dict[str, DataFrame]` (rsi2, sma10, sma200).
- `signals(close_adj, variant) -> Signals` avec `buy` (bool, date × ticker), `sell` (bool), `rank_key`
  (RSI(2)). Tout est indexé à **T** (date du Close utilisé).
- `episodes(buy_signals, merge_gap=5) -> list[(start, end)]` ; `count_episodes(..., period)`.

### A.4 `portfolio_backtester.py`

- `CostModel` : `commission(q, price, side)`, `sec_fee(value, date)`, `finra_taf(q, date)`, tables datées en
  constantes de module (reprises du § 3.5), `slippage`.
- `BacktestConfig` : capital, max_positions, allow_fractional, withholding, commission_plan ("fixed" |
  "tiered"), slippage, fx_fee, variant, start, end.
- `run_backtest(md, signals, cfg) -> BacktestResult` : boucle séance par séance ; à chaque T+1 : dividendes
  → ventes → classement → achats ; journal des trades (`trades`), des ordres rejetés (`rejected`), VL
  quotidienne (`nav`), exposition quotidienne (`exposure`), cash.
- Un seul run continu 2004 → fin ; le découpage IS/OOS/Total est fait dans `reporting.py`. Le mode IS seul
  arrête simplement la boucle au 2015-12-31.

### A.5 `reporting.py`

- `metrics(result, period) -> dict` ; `benchmark_spy(md, withholding, period)`.
- `report(results: dict[name, BacktestResult], periods) -> str` (Markdown, tableau IS/OOS/Total par run,
  tableau des rejets par ETF × période, alertes rouges, verdict des critères § 2.3 pour l'IS).
- `evaluate_abandon_criteria(metrics_is, spy_is) -> Verdict` et `evaluate_oos_criteria(...)` (seuils lus
  depuis des constantes qui recopient le README, avec le texte du critère dans la sortie).

### A.6 `main.py`

- `python main.py integrity` : rapport d'intégrité uniquement.
- `python main.py is` : run principal + S1–S4 sur l'IS ; écrit `data/reports/is_<date>.md`. Interdit si le
  rapport d'intégrité n'a pas été produit.
- `python main.py --oos` : vérifie l'absence de `data/oos_lock.json`, l'arbre Git propre, écrit le verrou
  (horodatage, hash du commit, SHA-256 des `.py` et du README, dernière date de données, config), puis lance
  le run principal + S1–S4 sur 2004 → fin et produit le rapport IS/OOS/Total. Pas d'option de contournement.

### A.7 Tests (`tests/`)

| Fichier | Ce qui est testé |
|---|---|
| `test_sizing.py` | budget cible/effectif, plancher entier, rejet si quantité 0, mode fractionné |
| `test_costs.py` | commissions : minimum, plafond 1 %, cas « plafond < minimum » ; SEC et TAF par date (chaque borne de la table, y compris 0 USD) |
| `test_dividends.py` | crédit Q × div × (1 − retenue) à la date de détachement, Q = détention à la veille, aucune détention → 0 |
| `test_no_lookahead.py` | signaux calculés sur les données tronquées à T identiques aux signaux à T sur l'historique complet ; exécution datée strictement à T+1 ; test de perturbation (modifier le Close de T+1 ne change pas le signal de T) |
| `test_episodes.py` | fusion avec écart ≤ 4 séances, non-fusion à 5, plusieurs tickers le même jour = un seul épisode |
| `test_warmup.py` | début de backtest = premier jour ouvré de 2004 ; SMA200 valide ; aucune position avant |
| `test_oos_lock.py` | second lancement refusé ; contenu du verrou (hash, date) ; refus si arbre Git sale |
| `test_integrity.py` | `DataIntegrityWarning` déclenché au-dessus du seuil sur des données synthétiques, pas en dessous |

### A.8 Ordre de travail (rappel, respecté strictement)

1. README (ce document) → validation → **commit Git de scellement par le propriétaire**.
2. `data_loader.py` + test de cohérence SPY → rapport d'intégrité présenté → décision § 5.6.
3. `strategy.py` + `portfolio_backtester.py` + `reporting.py` + tests.
4. Run principal (+ S1–S4) sur l'**IS uniquement** → rapport présenté. **Pas d'OOS.**
5. Application des critères d'abandon. Si passage : attente du feu vert explicite, puis OOS **une seule fois**.
