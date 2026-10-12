# us_quant_lab — Laboratoire de recherche quantitative (actions/ETFs US)

Backtest de portefeuille, mean-reversion de Connors (RSI(2)), univers d'ETFs US.

> **Projet de RECHERCHE UNIQUEMENT.** Aucun code de trading réel, aucune connexion à un
> courtier, aucune clé d'API de courtier. Tous les résultats sont **rétrospectifs** et ne
> constituent en aucun cas une prédiction de performance future.

État du document : **BROUILLON — en attente de validation** (voir § 7, journal du protocole et du verrou OOS).

**Définitions temporelles :**

- **Date du protocole** : horodatage UTC du commit Git qui contient la version finale validée de ce
  README, avant tout téléchargement de données et tout run.
- **Date du verrou OOS** : horodatage UTC de la création de `data/oos_lock.json`, immédiatement avant
  l'unique lancement OOS. Le verrou enregistre aussi la dernière séance complète présente dans le cache.

**Limite d'aveuglement :** l'OOS est pré-enregistré, mais il n'est pas aveugle. Les paramètres de la
stratégie sont issus de travaux publiés et le propriétaire possède une connaissance générale de
l'évolution passée du marché. Le protocole réduit les degrés de liberté après coup ; il ne supprime pas
cette connaissance préalable.

---

## 1. Pourquoi ce protocole est strict

Objectif : ne pas se tromper soi-même. Chaque règle ci-dessous répond à un risque identifié.

| Risque | Règle qui y répond |
|---|---|
| Biais du survivant | Univers d'ETFs larges/sectoriels (pas de stock-picking) ; univers S&P 100 optionnel marqué « biais majeur » |
| Lookahead | Signaux calculés au Close de T, exécution à l'Open de T+1 ; test dédié |
| Résultats flatteurs par hasard (multiplicité) | **Un seul** run principal, défini à l'avance ; runs de sensibilité explicitement non décisionnels |
| Critères fixés après coup | README commité **avant** tout run ; aucun seuil modifié après un résultat |
| Coûts et contraintes sous-estimés (capital 1 000 USD) | Run principal fractionné avec commissions minimum, taxes datées, retenue et slippage ; run S4 en actions entières pour diagnostiquer le réalisme opérationnel |
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
| Actions | **Fractionnées** (`allow_fractional=True`) |
| Retenue à la source sur dividendes | **30 %** (benchmark SPY : même retenue) |
| Slippage | 0,05 % par exécution, sens défavorable |
| Conversion de devises | 2 USD, une seule fois, sur le capital initial |
| Rendement du cash | 0 % |

Le verdict (abandon / passage à l'OOS / réussite OOS) est rendu **uniquement** sur ce run.

Le fractionnement évite les rejets dus au seul fait que le prix d'une action dépasse le budget de la ligne.
**Hypothèse opérationnelle : l'éligibilité aux fractions IBKR est à confirmer à l'ouverture du compte.**
Cette hypothèse ne transforme pas le projet en système de trading réel ; elle doit être rappelée dans les
rapports. Le verdict du run principal vaut sous cette hypothèse d'éligibilité ; S4 décrit la situation
sans fractions et mesure séparément le réalisme de cette contrainte.

### 2.2 Runs de sensibilité (exploratoires)

Ils ne modifient **jamais** le verdict du run principal. Ils servent à comprendre d'où vient le résultat.

| Run | Ce qui change par rapport au run principal |
|---|---|
| S1 | Variante **A** (RSI(2) < 10 / RSI(2) > 60, sans filtre SMA200) |
| S2 | Variante **B** (Close > SMA200 et RSI(2) < 10 / RSI(2) > 60) |
| S3 | Commission IBKR Pro **Dégressive** (présentée comme **borne basse** : pass-through exchange/clearing non inclus) |
| S4 | Actions **entières** (`allow_fractional=False`) — diagnostic de réalisme opérationnel ; la part de rejets faute de budget est rapportée, sans effet sur le verdict |

Les runs de sensibilité sont lancés sur l'IS en même temps que le run principal. Sur l'OOS, ils
sont lancés dans le **même lancement OOS unique** que le run principal (un seul verrou, voir § 2.7),
jamais après coup.

### 2.3 Critères d'abandon — évalués sur l'IN-SAMPLE (2004-01-01 → 2015-12-31), run principal

La stratégie est **abandonnée** (pas d'OOS) si **au moins un** des critères suivants est vrai :

1. Nombre d'épisodes fusionnés IS **< 30**
2. CAGR net **≤ 0 %** **ou** CAGR net **< CAGR de SPY (net, avec retenue) − 3 points**
3. Drawdown maximal **> 30 %**
4. Profit factor net **< 1,25**

Si aucun critère n'est déclenché, la stratégie **passe** et l'OOS attend un feu vert explicite.

### 2.4 Critères de réussite OOS — seuils Standard retenus

Les seuils suivants ont été choisis **avant le premier run**. Ils s'appliquent au run principal sur l'OOS
(2016-01-01 → dernière séance complète enregistrée à la date du verrou OOS), sauf les règles
« non concluant », qui sont évaluées en premier.

| Critère | Seuil Standard retenu | Effet |
|---|---|---|
| Épisodes fusionnés OOS | **≥ 25** | Sinon : OOS **non concluant** |
| CAGR net | **> 0 %** | Sinon : échec |
| CAGR net relatif au SPY | **≥ CAGR net du SPY − 3 points de pourcentage par an** | Sinon : échec |
| Ratio CAGR / drawdown maximal vs SPY net | **≥ 1,0 ×** le ratio du SPY net sur la même période | Sinon : échec |
| Drawdown maximal | **≤ 30 %** | Sinon : échec |
| Profit factor net | **≥ 1,25** | Sinon : échec |

**Le CAGR net, le drawdown maximal et le ratio CAGR / drawdown maximal du SPY net sur 2016 → dernière
séance complète enregistrée à la date du verrou OOS restent volontairement inconnus avant le commit du
protocole.** Les calculer ou les consulter avant ce commit révélerait le benchmark OOS et permettrait de
choisir le critère après avoir vu le résultat. Ce serait une contamination de l'OOS contraire au présent
protocole.

Le verdict impose **simultanément** :

- un rendement absolu proche du SPY net (`CAGR stratégie ≥ CAGR SPY net − 3 points/an`) ;
- un rendement ajusté du risque au moins égal à celui du SPY net
  (`CAGR/MaxDD stratégie ≥ 1,0 × CAGR/MaxDD SPY net`).

Le ratio CAGR / drawdown maximal de la stratégie et celui du SPY net sont tous deux affichés dans le
rapport. La condition de rendement relatif a été ajoutée pour éviter qu'un CAGR très faible ne passe
uniquement grâce à un faible drawdown. Le plancher **SPY net − 3 points/an** est retenu pour assurer la
cohérence avec le critère d'abandon in-sample, qui emploie le même écart.

Verdict : **réussite** si tous les critères de réussite sont vrais ; **échec** sinon ; **non concluant**
uniquement si le minimum de 25 épisodes n'est pas atteint.

La dégradation du profit factor IS → OOS est **informative uniquement** : le rapport affiche le ratio
`PF net OOS / PF net IS` et indique s'il est inférieur à **0,60**, mais cette information ne modifie
jamais le verdict OOS.

Justification du seuil d'épisodes : le seuil IS de 30 épisodes porte sur 12 ans (2,5/an). L'OOS couvre
environ 10,75 ans à la date de rédaction ; la proportionnalité donne environ 27, arrondi à 25.

### 2.5 Diagnostic indicatif des rejets — S4 en actions entières

Si, sur l'OOS, **plus de 30 %** des **ordres d'achat tentés** par S4 sont **rejetés faute de budget**
(quantité entière = 0), le rapport émet une alerte de réalisme opérationnel. Cette règle est
**indicative uniquement** : elle ne rend pas l'OOS non concluant et ne modifie jamais le verdict du run
principal fractionné.

Définition d'un « ordre d'achat tenté » : un signal d'achat qui, après classement par RSI(2) croissant,
obtient une place parmi les positions libres à T+1. Les signaux non retenus faute de place ne comptent
pas comme rejets.

### 2.6 Conséquence d'un OOS non concluant

Si l'OOS est non concluant parce que le nombre d'épisodes fusionnés est **< 25** :

1. l'OOS n'est **jamais relancé**, même avec davantage de données ;
2. la stratégie est déclarée **non tranchée** : ni réussite, ni échec ;
3. le run S4 en actions entières déjà inclus dans le lancement OOS unique est présenté comme diagnostic
   de réalisme, sans modifier ce verdict ;
4. toute évaluation ultérieure se fait uniquement par **suivi prospectif** sur des données dont la date est
   strictement postérieure à la **dernière séance incluse dans l'OOS**, enregistrée dans
   `data/oos_lock.json` ;
5. ce suivi prospectif est rapporté séparément et n'est jamais fusionné avec l'OOS historique pour
   requalifier rétroactivement son verdict.

### 2.7 Verrou OOS : exécution unique, code et paramètres verrouillés

- L'OOS est exécuté **une seule fois**, par `python main.py --oos`.
- Avant d'exécuter, `main.py` :
  1. refuse si `data/oos_lock.json` existe déjà (**aucune option de contournement** dans le code) ;
  2. refuse si l'arbre de travail Git n'est pas propre (fichiers suivis modifiés non commités) ;
  3. refuse si le cache diffère du manifeste SHA-256 enregistré lors du dernier run IS réussi ; tout
     rafraîchissement, ajout, suppression ou modification du cache entre ce run IS et l'OOS est interdit ;
  4. enregistre dans `data/oos_lock.json` : date du verrou OOS, hash du commit Git, SHA-256 de chaque
     fichier `.py`, du `README.md` et de chaque fichier du cache de données, dernière séance complète
     utilisée, paramètres complets du run et hash du manifeste du dernier run IS.
- `data/oos_lock.json` et le rapport OOS sont ajoutés et commités dans Git **immédiatement après** la fin
  du run OOS.
- **Exception unique pour crash technique sans divulgation** : si le processus s'interrompt avant qu'un
  quelconque chiffre dérivé de l'OOS soit affiché à l'écran, écrit dans un rapport, un log ou tout autre
  fichier, une seule reprise est autorisée. La cause, la correction et le hash du commit correctif sont
  documentés au § 7 avant la reprise ; le verrou conserve les deux tentatives et `retry_count=1`.
  Dès qu'au moins un chiffre dérivé de l'OOS a été affiché ou écrit, le verrou est définitif et aucune
  reprise n'est autorisée, y compris après un crash ultérieur.
- Si la stratégie échoue en OOS, **aucun réajustement** (paramètres, règles, univers, coûts) n'est toléré.
  Une nouvelle idée = un nouveau projet, un nouveau README, un nouveau pré-engagement.

### 2.8 Immutabilité

Aucun seuil, paramètre ou définition de ce README n'est modifié après avoir vu un résultat (IS ou OOS).
Les seules modifications autorisées après le commit du protocole sont des corrections de **bugs manifestes**
du code, documentées dans le § 7, et qui déclenchent alors une ré-exécution de l'**IS uniquement** ; l'OOS
reste soumis au verrou. La seule exception est la reprise technique unique définie au § 2.7, exclusivement
si aucun chiffre OOS n'a été affiché ou écrit.

---

## 3. Paramètres figés

### 3.1 Données (`data_loader.py`)

- Source : **yfinance** (outil non officiel, acceptable pour le prototypage uniquement). Cache local en
  **parquet** dans `data/` ; pas de retéléchargement à chaque run. Le rafraîchissement explicite reste
  possible jusqu'au dernier run IS réussi, puis devient interdit jusqu'à l'OOS.
- Téléchargement à partir du **2003-01-01**. Les ~200 premières séances servent uniquement de chauffe pour
  la SMA200. Le backtest commence au **premier jour ouvré de 2004** ; le chargeur vérifie que la SMA200 est
  valide pour tous les titres de l'univers à cette date (sinon erreur).
- Deux jeux de données :
  - **(a) BRUTS** : Open, Close non ajustés des dividendes (`auto_adjust=False`), mais normalisés par
    Yahoo/yfinance pour les splits, dividendes bruts par action (à la date de détachement), événements
    de split conservés pour audit ;
  - **(b) AJUSTÉS** : Close ajusté (dividendes + splits), Open ajusté par le même facteur.
- **Signaux** : calculés sur les prix **ajustés** (continuité des indicateurs).
- **Quantités, frais, trésorerie, valorisation** : calculés sur les prix **BRUTS**.
- **Convention splits** : les OHLC fournis par yfinance avec `auto_adjust=False` sont utilisés tels quels ;
  ils sont déjà normalisés des splits par la source. Les quantités sont exprimées dans la même unité
  d'action normalisée et ne sont donc **pas** multipliées ou divisées une seconde fois à la date d'un
  split. Les événements de split sont néanmoins conservés et listés dans le rapport pour contrôler
  l'absence de double ajustement.
- **Test de cohérence obligatoire** avant tout backtest : le rendement total de SPY reconstruit avec
  prix bruts + dividendes bruts réinvestis au Close du jour de détachement doit correspondre au rendement
  total calculé avec l'Adj Close, **uniquement sur 2003-01-01 → 2015-12-31**. Le rapport n'affiche que
  l'**écart absolu de CAGR en point par an**, jamais les deux niveaux de CAGR. **Seuil : écart > 0,1
  point/an → `DataIntegrityWarning`** explicite avec la période et l'écart. Aucun niveau de performance
  SPY sur l'OOS n'est calculé ni affiché avant le commit du protocole et l'unique lancement OOS. Les petits
  écarts sont normaux (moment de réinvestissement différent).
- **Distributions anormales** : toute distribution strictement supérieure à **5 % du Close de la veille**
  est exclue du cash crédité au portefeuille et au benchmark, et listée dans le rapport d'intégrité avec
  date, ticker, montant et ratio. Avant le backtest, les colonnes « détenu » et « effet sur la VL » sont
  marquées **non encore déterminable**. Après chaque run autorisé, le rapport d'intégrité correspondant est
  complété avec la quantité détenue à la clôture de la veille, `détenu = oui/non`, le cash brut exclu,
  le cash net exclu après retenue et l'effet en USD et en pourcentage sur la VL du portefeuille. Pour Q
  actions détenues, l'effet de l'exclusion est
  `−Q × distribution brute × (1 − retenue)` USD, puis ce montant divisé par la VL de la veille pour le
  pourcentage ; sans détention, l'effet est nul. Cette règle est fixée avant tout run et s'applique sans
  exception.

### 3.2 Périodes

| Période | Début | Fin |
|---|---|---|
| Chauffe | 2003-01-01 | dernier jour ouvré de 2003 |
| In-sample (IS) | 2004-01-01 (premier jour ouvré) | 2015-12-31 |
| Out-of-sample (OOS) | 2016-01-01 | dernière séance complète disponible à la date du verrou OOS (enregistrée dans le verrou) |

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
  5. **Quantité du run principal** = budget effectif / (Open brut × (1 + slippage)), sans arrondi
     (`allow_fractional=True`). Pour S4 seulement, la quantité est arrondie à l'entier inférieur ; si elle
     vaut 0, l'ordre est rejeté avec un log explicite (date, titre, Open, budget effectif).
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
| 2004-01-01 | 2019-04-15 | **20,00 (approximation documentée)** | **Non vérifié à la source le 2026-10-11** ; approximation : les taux réels ont oscillé entre ~5 et ~30 USD/M ; impact ≈ 0,5 cent sur une vente de 250 USD |
| 2019-04-16 | 2020-02-17 | 20,70 | **Non vérifié à la source le 2026-10-11** |
| 2020-02-18 | 2021-02-24 | 22,10 | Vérifié à la source le 2026-10-11 (Federal Register) |
| 2021-02-25 | 2022-05-13 | 5,10 | Vérifié à la source le 2026-10-11 (SEC) |
| 2022-05-14 | 2023-02-26 | 22,90 | Vérifié à la source le 2026-10-11 (SEC) |
| 2023-02-27 | 2024-05-21 | 8,00 | **Non vérifié à la source le 2026-10-11** pour la borne de fin ; taux et date de début confirmés par la SEC |
| 2024-05-22 | 2025-05-13 | 27,80 | **Non vérifié à la source le 2026-10-11** |
| 2025-05-14 | 2026-04-03 | 0,00 | **Non vérifié à la source le 2026-10-11** pour la borne de début ; taux nul confirmé par la SEC jusqu'au 2026-04-03 |
| 2026-04-04 | — | 20,60 | Vérifié à la source le 2026-10-11 (SEC Release 34-104909) |

**FINRA Trading Activity Fee** (ventes uniquement) — table **datée** (source : FINRA, Fee Adjustment
Schedule SR-FINRA-2024-019 et SR-FINRA-2026-021) :

| Du (inclus) | Au (inclus) | Par action (USD) | Plafond par ordre (USD) | Statut |
|---|---|---|---|---|
| 2004-01-01 | 2023-12-31 | **0,000166 (approximation documentée)** | 8,30 | **Non vérifié à la source le 2026-10-11** ; approximation conservatrice ; impact < 0,1 cent par ordre de quelques actions |
| 2024-01-01 | 2025-12-31 | 0,000166 | 8,30 | Vérifié à la source le 2026-10-11 (FINRA Fee Adjustment Schedule) |
| 2026-01-01 | 2026-09-30 | 0,000195 | 9,79 | Vérifié à la source le 2026-10-11 (FINRA Fee Adjustment Schedule) |
| 2026-10-01 | 2026-12-31 | 0,00 | 0,00 | Vérifié à la source le 2026-10-11 (SR-FINRA-2026-021) |
| 2027-01-01 | — | 0,000232 | 11,61 | Vérifié à la source le 2026-10-11 (FINRA, taux programmé) |

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
| Profit factor net | somme des P&L nets strictement positifs des trades clôturés, divisée par la valeur absolue de la somme de leurs P&L nets strictement négatifs ; `+∞` si des gains existent sans aucune perte, non défini s'il n'existe aucun gain ni aucune perte |
| Nombre de trades | trades clôturés (aller-retour) dans la période (date de sortie) ; les positions ouvertes en fin de période sont comptées à part |
| Nombre d'épisodes | voir ci-dessous |
| Durée moyenne de détention | moyenne du nombre de séances entre l'Open d'entrée et l'Open de sortie |
| Ordres rejetés (S4 uniquement) | nombre d'ordres d'achat tentés par S4 avec quantité entière = 0, **par ETF et par période**, et en % des ordres tentés de S4 ; métrique indicative sans effet sur le verdict principal |
| Benchmark SPY principal | indice de rendement total **net** : SPY avec dividendes nets de retenue (30 %) réinvestis au Close du jour de détachement, sans contrainte d'actions entières ni commission ; seul ce benchmark intervient dans les critères |
| Buy & hold SPY simulé (informatif) | 1 000 USD moins le forfait de conversion de 2 USD ; achat d'actions entières de SPY à l'Open brut du premier jour de la période, avec slippage et commission fixe ; dividendes nets crédités en cash sans réinvestissement ; valorisation au Close brut ; rapporté séparément et sans effet sur le verdict |

**P&L net d'un trade** = produit de vente net (après slippage, commission, SEC, TAF) − coût d'achat net
(après slippage, commission) + dividendes nets perçus pendant la détention.

**Épisode** : on prend l'union, sur tout l'univers, des séances où **au moins un** signal d'achat de la
variante est émis (que l'ordre soit exécuté, rejeté ou non retenu faute de place). Deux séances de signal
séparées de **moins de 5 séances** (`episode_merge_gap = 5`, c'est-à-dire ≤ 4 séances de bourse d'écart)
appartiennent au **même** épisode. Les ETFs étant très corrélés, un épisode = un choc de marché distinct,
pas un trade indépendant. **Les seuils de significativité s'appliquent aux épisodes, pas aux trades bruts.**

**Alerte visuelle en rouge** dans le rapport si le nombre d'épisodes d'une période < 30. Cette alerte
n'ajoute aucun critère de verdict et ne le modifie pas : le seuil OOS pré-enregistré reste **25 épisodes**.

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
6. **Distributions anormales** : toute distribution strictement supérieure à **5 % du Close brut de la
   veille** est exclue intégralement du cash crédité au portefeuille et au benchmark. Elle est listée dans
   le rapport d'intégrité (date, ticker, montant, Close de la veille et ratio), puis annotée après chaque
   run avec la détention à la veille et l'effet de l'exclusion sur la VL. Aucun choix discrétionnaire n'est
   effectué après lecture du rapport.
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

## 7. Journal du protocole et du verrou OOS

| Date | Événement | Commit |
|---|---|---|
| 2026-10-11 | Rédaction du README (brouillon) | — |
| 2026-10-11 | Choix préliminaires : seuils OOS Standard ; dégradation PF informative ; benchmark principal SPY total return net | — |
| 2026-10-11 | Critères OOS relatifs retenus conjointement : CAGR ≥ SPY net − 3 points/an et ratio CAGR/MaxDD ≥ 1× SPY net | — |
| — | Validation finale du README, de l'annexe A et des décisions § 5 par le propriétaire du projet | — |
| — | Commit du protocole (définit la date du protocole, avant tout téléchargement et tout run) | — |
| — | Rapport d'intégrité des données validé | — |
| — | Run principal IS exécuté ; critères d'abandon appliqués | — |
| — | Feu vert explicite OOS | — |
| — | OOS exécuté (date du verrou OOS enregistrée), verrou et rapport immédiatement commités | — |
| — | Le cas échéant : crash OOS sans chiffre divulgué, cause et correction, commit correctif, reprise unique | — |

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
  `yf.download(..., auto_adjust=False, actions=True)`, met en cache `data/raw_{ticker}.parquet`. Tout
  `refresh=True` est refusé après le dernier run IS réussi et avant l'OOS.
- `MarketData` : `open_raw`, `close_raw`, `open_adj`, `close_adj`, `dividends`, `splits` (DataFrames
  date × ticker), `calendar` (index des séances SPY). Les OHLC `auto_adjust=False` sont traités comme
  non ajustés des dividendes mais déjà normalisés des splits ; les événements de split sont audités sans
  réajuster les quantités.
- `warmup_check(md, start="2004-01-01")` : premier jour ouvré de 2004, SMA200 valide pour tous.
- `spy_integrity_report(md, end="2015-12-31", threshold_pp=0.1) -> IntegrityReport` : compare sur
  2003-01-01 → 2015-12-31 le CAGR « bruts + dividendes réinvestis » au CAGR « Adj Close » ; le rapport
  n'expose que leur écart absolu, avec `DataIntegrityWarning` s'il dépasse le seuil. Il liste les
  distributions > 5 % exclues du cash, les événements de split et les séances manquantes par ticker,
  sans niveau de performance SPY OOS. Écrit `data/integrity_report.md`.
- `annotate_distribution_impact(report, result)` complète, après chaque run autorisé, chaque distribution
  exclue avec la quantité détenue à la veille, l'indicateur détenu oui/non et l'effet de l'exclusion en
  USD et en pourcentage de VL ; avant le run, ces champs portent « non encore déterminable ».
- `write_cache_manifest(...)` enregistre le SHA-256 de chaque parquet lors du dernier run IS réussi ;
  `verify_cache_manifest(...)` interdit l'OOS si un fichier a changé.

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

- `metrics(result, period) -> dict` ; `benchmark_spy(md, withholding, period)` pour l'indice SPY total
  return net principal ; `simulated_spy_buy_hold(...)` pour le comparatif informatif.
- `report(results: dict[name, BacktestResult], periods) -> str` (Markdown, tableau IS/OOS/Total par run,
  tableau des rejets de S4 uniquement par ETF × période, benchmark principal, buy & hold simulé
  informatif, dégradation du profit factor IS → OOS informative, ratios CAGR/MaxDD de la stratégie et du
  SPY net, alertes rouges, verdict des critères § 2.3 pour l'IS). Le tableau des rejets S4 est indicatif
  et n'affecte aucun verdict. L'alerte rouge « épisodes < 30 » est uniquement visuelle et n'écrase jamais
  le seuil de verdict OOS de 25.
- `evaluate_abandon_criteria(metrics_is, spy_is) -> Verdict` et `evaluate_oos_criteria(...)` (seuils lus
  depuis des constantes qui recopient le README, avec le texte du critère dans la sortie).

### A.6 `main.py`

- `python main.py integrity` : rapport d'intégrité uniquement.
- `python main.py is` : run principal + S1–S4 sur l'IS ; écrit `data/reports/is_<date>.md`. Interdit si le
  rapport d'intégrité n'a pas été produit. Après succès, écrit le manifeste SHA-256 immuable du cache
  utilisé ; aucun rafraîchissement du cache n'est ensuite autorisé avant l'OOS.
- `python main.py --oos` : vérifie l'absence de `data/oos_lock.json`, l'arbre Git propre, écrit le verrou
  (date du verrou OOS, hash du commit, SHA-256 des `.py`, du README et de chaque fichier du cache, hash du
  manifeste IS, dernière séance complète et config), vérifie que le cache est identique à celui du dernier
  run IS, puis lance le run principal + S1–S4 sur 2004 → fin et produit le rapport IS/OOS/Total. Pas
  d'option de contournement.
- Les calculs OOS restent en mémoire sans métrique sur stdout. Juste avant toute sérialisation ou tout
  affichage d'un chiffre OOS, `main.py` positionne irréversiblement `oos_results_exposed=true` dans le
  verrou. Une commande dédiée `python main.py --resume-oos` n'est acceptée qu'une fois, seulement si ce
  marqueur est faux, après documentation § 7 et commit de la correction ; elle incrémente `retry_count`.
  Le verrou et le rapport sont commités immédiatement après le run.

### A.7 Tests (`tests/`)

| Fichier | Ce qui est testé |
|---|---|
| `test_sizing.py` | budget cible/effectif ; run principal fractionné sans arrondi ; S4 entier avec plancher et rejet si quantité 0 ; alerte indicative S4 au-dessus de 30 % de rejets sans effet sur le verdict principal |
| `test_costs.py` | commissions : minimum, plafond 1 %, cas « plafond < minimum » ; SEC et TAF par date (chaque borne de la table, y compris 0 USD) |
| `test_dividends.py` | crédit Q × div × (1 − retenue) à la date de détachement, Q = détention à la veille, aucune détention → 0 |
| `test_no_lookahead.py` | signaux calculés sur les données tronquées à T identiques aux signaux à T sur l'historique complet ; exécution datée strictement à T+1 ; test de perturbation (modifier le Close de T+1 ne change pas le signal de T) |
| `test_episodes.py` | fusion avec écart ≤ 4 séances, non-fusion à 5, plusieurs tickers le même jour = un seul épisode |
| `test_warmup.py` | début de backtest = premier jour ouvré de 2004 ; SMA200 valide ; aucune position avant |
| `test_oos_lock.py` | second lancement refusé ; contenu du verrou (hashs code et cache, dates) ; refus si arbre Git sale ou si le cache diffère du manifeste IS ; refresh interdit entre IS et OOS ; reprise unique permise uniquement avec `oos_results_exposed=false`, correction documentée et commitée ; toute reprise refusée dès qu'un chiffre OOS peut avoir été affiché ou écrit |
| `test_integrity.py` | comparaison limitée à 2003–2015 ; rapport sans niveaux de CAGR ; `DataIntegrityWarning` déclenché au-dessus du seuil sur des données synthétiques, pas en dessous ; distributions > 5 % exclues, listées et annotées après run avec détention et effet sur la VL |

### A.8 Ordre de travail (rappel, respecté strictement)

1. README (ce document) → validation → **commit Git du protocole par le propriétaire**.
2. `data_loader.py` + test de cohérence SPY → rapport d'intégrité présenté → décision § 5.6.
3. `strategy.py` + `portfolio_backtester.py` + `reporting.py` + tests.
4. Run principal (+ S1–S4) sur l'**IS uniquement** → rapport présenté. **Pas d'OOS.**
5. Application des critères d'abandon. Si passage : attente du feu vert explicite, puis OOS **une seule fois**.
