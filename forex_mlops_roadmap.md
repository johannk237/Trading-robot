# 🎯 SYSTÈME DE PRÉDICTION USD/EUR - ROADMAP APPRENTISSAGE

> **Niveau d'exigence** : Senior Data Engineer + ML Systems Engineer
> **Durée estimée** : 6-9 mois (3-4h/jour)
> **Prérequis** : Python, SQL intermédiaire, notions finance élémentaires

---

## ⚠️ CRITIQUE DU FRAMEWORK INITIAL

### Pourquoi le roadmap e-commerce ne marche PAS pour le Forex

| Aspect | E-commerce | Forex (Critique) |
|---|---|---|
| **Look-ahead bias** | Moins critique | FATAL - tu entraînes avec l'avenir |
| **Validation temporelle** | K-fold OK | Invalide - rompt l'ordre chronologique |
| **Stationnarité** | Non pertinent | OBLIGATOIRE - USD/EUR n'est pas stationnaire |
| **Coût d'erreur** | Notification inutile | Perte financière réelle |
| **Périodicité** | Batch quotidien OK | Continu 24h/24 - gestion des gaps critique |
| **Drift monitoring** | Optionnel | Obligatoire - le marché change 24/24 |

**Le piège conceptuel** : E-commerce traite les données comme statiques. Forex traite le temps comme dimension critique.

---

# 🚀 ARCHITECTURE SYSTÈME

```
SOURCES TEMPS RÉEL (OHLCV, News, Volatilité)
            ↓
INGESTION & VALIDATION DONNÉES
            ↓
FEATURE ENGINEERING (sans look-ahead)
            ↓
STATIONNARITÉ & TRANSFORMATION
            ↓
TRAIN/VALIDATION/TEST (walk-forward UNIQUEMENT)
            ↓
ENTRAÎNEMENT MODÈLES
            ↓
BACKTEST RÉALISTE (slippage, spread, commissions)
            ↓
SERVING & DÉCISIONS
            ↓
MONITORING FINANCIER (P&L vs backtest, drift)
```

---

# 📚 ROADMAP DÉTAILLÉE (8 PHASES)

## 🔹 PHASE 1 : FONDATIONS DATA ENGINEERING

### 🎯 Objectif
Construire une pipeline d'ingestion **fiable, vérifiable et continue**.

### Pourquoi c'est différent de l'e-commerce ?
- EURUSD fonctionne 24h/24, 5j/5 → gestion des gaps de temps
- Un gap de 4h en données = ton backtest devient mensonger
- Synchronisation temporelle = fondation de tout

### Concepts à maîtriser

**1. Time-Series Databases vs Bases relationnelles**
- Pourquoi PostgreSQL classique n'est pas optimisé pour les séries temporelles
- TimescaleDB/ClickHouse : compromis entre OLTP et OLAP
- Recherche : "Time-series database vs relational" et cas d'usage
- Question clé : Quel est le coût en performance de stocker 5 ans de données horaires ?

**2. Fuseaux horaires et synchronisation**
- Notion de UTC comme référence unique
- Problème réel : EURUSD n'est pas tradé entre 20h et 22h UTC vendredi
- Recherche : "timezone handling in financial data" et "market hours"
- Question clé : Comment représentes-tu les gaps de week-end dans une base temps-série ?

**3. Sources de données pour Forex**
- Gratuit : yfinance, Alpha Vantage (limité)
- Professionnel : OANDA API, Interactive Brokers, FIX protocol
- Différences : latency, fiabilité, granularité (1min vs 5min vs 1h)
- Recherche : "Forex data API comparison" et documentations officielles
- Question clé : Pourquoi les données gratuit et pro donnent-elles parfois des résultats légèrement différents ?

**4. Validation de données financières**
- Cohérence OHLCV : High >= max(Open, Close), Low <= min(Open, Close)
- Détection de gaps anormaux
- Détection d'outliers (un prix qui saute de 20% en 1 heure = anomalie)
- Recherche : "OHLCV data validation rules" et "data quality for trading"
- Question clé : Qu'est-ce qu'une augmentation de volatilité "normale" vs une donnée corrompue ?

**5. Idempotence et reproductibilité**
- Capacité à réingérer des données sans créer de doublons
- Versioning des données brutes
- Tracking de la source d'ingestion (quel broker, quelle API version)
- Recherche : "idempotent data pipelines" et "data lineage"
- Question clé : Si tu réingères le même jour 10 fois, que devrais-tu avoir en base ?

**6. Agrégations temporelles**
- 1h → 4h → 1D : comment construire les OHLC à partir de timeframes plus petits
- Pré-calcul vs calcul à la demande
- Recherche : "OHLC aggregation" et "candle consolidation"
- Question clé : Si tu as des données tick-level, comment construis-tu un chandelier 4h sans look-ahead ?

### Ressources à consulter

- **Concepts** : Documentation TimescaleDB sur hypertables
- **Validation** : Great Expectations framework (concepts, pas code)
- **API** : Documentation officielle OANDA/IB sur data feeds
- **Best practices** : Articles sur "data contracts" en finance

### Livrables attendus

- Documentation de ta stratégie d'ingestion (sources choisies, fréquence, validation)
- Schéma de base de données détaillé avec justifications
- Analyse des gaps de données que tu trouves
- Plan de backfill historique (comment récupérer 5 ans)

### Compétences à valider

✅ Comprendre les différences entre PostgreSQL standard et time-series DB
✅ Savoir pourquoi UTC + market hours = critique
✅ Pouvoir énumérer 10+ règles de validation OHLCV
✅ Concevoir idempotence dans un contexte temps-réel

---

## 🔹 PHASE 2 : PIPELINES ROBUSTES & SCHEDULING

### 🎯 Objectif
Zéro intervention manuelle. Les données s'ingèrent continuellement et fiablement.

### Pourquoi différent de l'e-commerce ?

| E-commerce | Forex |
|---|---|
| Batch quotidien acceptable | Besoin continu (gap max 4h) |
| Dépendances métier | Dépendances **temporelles strictes** |
| Retry simple en cas d'erreur | Retry dépend de l'heure du marché |
| Données statiques après batch | Données "vivantes" durant les heures de marché |

### Concepts à maîtriser

**1. DAG et dépendances temporelles**
- Concept de DAG (Directed Acyclic Graph)
- Différence entre dépendance de tâche vs dépendance temporelle
- Cron vs event-driven pour scheduling
- Recherche : "Airflow/Prefect DAG design" et "temporal dependencies"
- Question clé : Si une tâche s'exécute à 14h mais le marché ferme à 17h, comment tu gères les données de 14h-17h ?

**2. Idempotence absolue en production**
- Qu'est-ce qui empêche les doublons lors d'une réexécution
- Upsert vs insert
- State management dans une pipeline
- Recherche : "idempotent operations in data pipelines"
- Question clé : Si une tâche fail, tu la rejoues. Comment s'assurer qu'elle ne crée pas de dupliquatas ?

**3. Backfill massif**
- Ingestion rétrospective de 5+ ans de données
- Gestion des rate limits (API ne permet que X requêtes/sec)
- Parallélisation vs séquentialité
- Recherche : "bulk data import strategies" et "rate-limited API calls"
- Question clé : Combien de temps devrait prendre l'ingestion de 60 ans de 1h candles EURUSD ?

**4. Gestion des erreurs et retry**
- Quand retry automatique a du sens
- Quand alerte humaine est nécessaire
- Exponential backoff
- Recherche : "retry strategies in data pipelines" et "circuit breaker pattern"
- Question clé : Si OANDA est down, tu retry jusqu'à quand ? Combien de fois ?

**5. Monitoring et alertes**
- Qu'est-ce qu'une "tâche qui a l'air OK mais donne des données mauvaises"
- Monitoring du gap de données (dernière donnée de 4h il y a combien)
- Métriques critiques à exposer
- Recherche : "data pipeline monitoring" et "SLA for trading systems"
- Question clé : Tu dois décider : "les données sont trop vieilles, j'arrête le trading". À partir de quel seuil ?

### Ressources à consulter

- **Orchestration** : Documentation Airflow ou Prefect (concepts d'exécution, pas tutorial)
- **Design patterns** : Articles sur patterns de backfill et retry
- **Monitoring** : Articles sur "observability in data pipelines"

### Livrables attendus

- Diagramme DAG de ta pipeline avec dépendances explicites
- Stratégie de retry et alertes documentée
- Plan d'ingestion historique avec estimations de temps
- Métriques à monitorer : liste + seuils

### Compétences à valider

✅ Pouvoir dessiner et expliquer ta DAG
✅ Citer 5 scénarios d'erreur et ta réaction à chaque
✅ Expliquer pourquoi le retry exponentiel > retry linéaire
✅ Définir quand tu arrêtes le trading (data trop vieille)

---

## 🔹 PHASE 3 : DATA QUALITY & CONTRACTS

### 🎯 Objectif
Détecter les anomalies AVANT qu'elles ne cassent ton modèle et ton P&L.

### Pourquoi CRITIQUE en trading

**Scénario réel problématique** :
- Une source retourne tous les prix identiques pendant 2h
- Tu entraînes un modèle avec ces 2h de bruit pur
- Ton modèle apprend un pattern invalide
- Pertes financières en production

### Concepts à maîtriser

**1. Great Expectations framework**
- Notion d'expectation (assertion sur les données)
- Data quality suite
- Validation checkpoint
- Recherche : "Great Expectations tutorial" et "data contract concept"
- Question clé : Pourquoi vérifier "close > open" est plus fort que "pas de NULL" ?

**2. Validations OHLCV spécifiques**
- High >= max(Open, Close) : toujours vrai?
- Low <= min(Open, Close) : toujours vrai?
- Volume > 0 : une bonne règle?
- Close range [X, Y] : comment set les bornes?
- Recherche : "OHLC consistency validation" et "outlier detection in price data"
- Question clé : EUR/USD peut-il monter de plus de 2% en une heure?

**3. Détection d'anomalies temporelles**
- Gaps de temps : pas d'entrée pendant X temps = anomalie?
- Patterns impossibles : détection
- Microstructure anormale (ex: prix montant strictement durant 50 min d'affilée)
- Recherche : "time-series anomaly detection" et "market data anomaly"
- Question clé : Comment distingues-tu une vraie volatilité d'une donnée corrompue?

**4. Stationnarité des données**
- Concept de série stationnaire vs non-stationnaire
- Pourquoi c'est important : si une série dérive, entraîner un modèle dessus = overfitting
- Tests : ADF, KPSS
- Solutions : différenciation, log-returns
- Recherche : "stationary vs non-stationary time series" et "augmented dickey fuller test"
- Question clé : EUR/USD est-il stationnaire? Et si tu prends les returns (changements %) plutôt que le prix?

**5. Data contracts**
- Accord formel : "les données auront cette forme"
- Versioning des contrats (si contrat change = problème)
- Schéma évolutif
- Recherche : "data contracts in data engineering" et "schema evolution"
- Question clé : Si OANDA change le format de l'API demain, tu fais quoi?

**6. Validation custom pour séries temporelles**
- Look-ahead bias : impossibilité que t[n] dépende de t[n+1]
- Autocorrélation : si X[t] dépend de X[t-1], c'est normal ou artifact?
- Distribution : la distribution des prix est-elle stable (ou change-t-elle = drift)?
- Recherche : "look-ahead bias detection" et "autocorrelation in time series"
- Question clé : Comment vérifie-tu qu'une feature n'utilise PAS d'information future?

### Ressources à consulter

- **Validation** : Great Expectations documentation (concepts)
- **Statistiques** : Cours sur stationnarité et tests ADF/KPSS
- **Finance** : Littérature sur "market data quality"
- **Anomaly detection** : Articles sur detection en time-series

### Livrables attendus

- Suite complète de 15+ assertions de validation
- Documentation de chaque assertion (pourquoi elle existe)
- Plan de test de stationnarité pour tes données
- Définition de data contract pour USD/EUR OHLCV

### Compétences à valider

✅ Citer 10 vérifications d'OHLCV et leur logique
✅ Expliquer stationnarité sans math (intuitivement)
✅ Pouvoir identifier look-ahead bias dans une feature
✅ Faire tourner et interpréter un test ADF

---

## 🔹 PHASE 4 : VERSIONING & LINEAGE

### 🎯 Objectif
**Reproductibilité scientifique** : relancer l'expérience 6 mois plus tard = résultats identiques.

### Pourquoi c'est junior-level d'ignorer ça

**Scénario réel** :
- Modèle X fait 55% d'accuracy
- 3 mois plus tard, tu veux le réentraîner
- Tu ne sais plus QUELLE version des données il a utilisé
- Résultats sont différents
- Impossible de debugger : données change, code a changé, paramètres hyper ont changé = tu ne sais pas quoi accuser

### Concepts à maîtriser

**1. Versioning des datasets**
- Notion de dataset immutable
- Hash/checksum de dataset (SHA256)
- Versioning via Git LFS ou DVC
- Métadonnées : qui a créé, quand, avec quelle source
- Recherche : "dataset versioning" et "DVC (Data Version Control)"
- Question clé : Si tu regeneres les features avec 1% des données en plus, tu incrémente la version comment?

**2. Reproducibility vs Repeatability**
- Reproductibilité : exécuter EXACTEMENT la même chose = même résultat
- Repeatabilité : réexécuter = résultats stables
- Seed aléatoire, versions de libs, ordre d'exécution
- Recherche : "reproducibility in ML" et "random seed importance"
- Question clé : Tu relances ton entraînement, tu obtiens 55.01% vs 55.00% avant. C'est grave?

**3. Lineage des données**
- Tracking : d'où viennent les données (source → transformation → feature)
- Criticité : savoir qu'une feature dépend de telle transformation de telle source
- Recherche : "data lineage" et "data provenance"
- Question clé : Si tu découvres un bug dans ta feature de RSI, comment tu identifies tous les modèles impactés?

**4. Feature versioning**
- Feature comme entité immuable
- Si tu changes une formule de feature = nouvelle version
- Tracking des features utilisées par quel modèle
- Recherche : "feature store" et "feature versioning"
- Question clé : Comment gérer quand tu découvres un bug dans le calcul d'une feature utilisée par 5 modèles?

**5. Model checkpoints vs production**
- Sauvegarde à chaque étape du training
- Capacité de rollback si nouveau modèle est pire
- Métadonnées attachées : score, date, données utilisées
- Recherche : "model versioning" et "model registry"
- Question clé : Tu déploies un modèle, il fait -10% de P&L en 1 jour. Tu reverses à quel point?

**6. Configurations immuables**
- Hyperparamètres version-controlled
- Seeds aléatoires fixés
- Versions des libs freezées
- Recherche : "configuration management ML" et "dependency pinning"
- Question clé : Tu upgrades XGBoost de 1.5 à 1.6, résultats changent légèrement. Tu reverses ou acceptes?

### Ressources à consulter

- **Versioning** : DVC documentation (concepts)
- **Lineage** : Apache Atlas, Marquez (concepts)
- **Reproducibility** : Littérature sur reproduciblité en ML/Science
- **MLOps** : MLflow Model Registry documentation

### Livrables attendus

- Plan complet de versioning (datasets, features, modèles, configs)
- Structure Git pour tracer les changements
- Schéma de lineage (données → features → modèles)
- Procédure de rollback documentée

### Compétences à valider

✅ Expliquer différence reproducibilité vs repeatabilité
✅ Faire un schema d'impact si une feature change
✅ Pouvoir reverser un modèle (identifier quelle version)
✅ Tracked changement entre 2 versions d'un modèle

---

## 🔹 PHASE 5 : FEATURE ENGINEERING (Pont critique)

### 🎯 Objectif
Transformer données brutes en signaux pertinents pour prédire EURUSD.

### Pourquoi c'est le cœur de la stratégie

**Réalité** : 90% de la valeur prédictive vient des features, pas du modèle. Un modèle basique sur bonnes features > modèle complexe sur mauvaises features.

### Concepts à maîtriser

**1. Principes de feature engineering en time-series**
- Feature = fonction déterministe des données
- Pas de look-ahead (critère absolu)
- Domaine-specific > data-mining
- Causality : la feature doit avoir une relation causale avec le target
- Recherche : "feature engineering for time series" et "look-ahead bias"
- Question clé : "Close price de demain" est-elle une bonne feature pour prédire "Direction demain"?

**2. Indicateurs techniques financiers**
- RSI (Relative Strength Index) : momentum
- MACD : trend
- Bollinger Bands : volatilité
- ATR (Average True Range) : volatilité
- SMA/EMA : trend
- Recherche : "technical indicators explanation" et "RSI calculation"
- Question clé : RSI > 70 = suracheté. À quoi ça correspond vraiment statistiquement?

**3. Statistical features**
- Volatility (écart-type des returns)
- Autocorrelation
- Skewness, Kurtosis
- Return distribution
- Recherche : "statistical features for price prediction" et "volatility estimation"
- Question clé : Si volatilité = 2%, ça te dit quoi sur les chances que prix monte demain?

**4. Market microstructure**
- Bid-ask spread (indication de liquidité)
- Volume profile
- Time-weighted average price
- Order flow imbalance
- Recherche : "market microstructure" et "FIX protocol basics"
- Question clé : Un volume élevé avec prix stable = haussier ou baissier?

**5. Lagged features et rolling windows**
- Price[t-1], [t-2], [t-3]
- 20-period rolling average
- 5-period rolling volatility
- Attention : chaque lag = perte de donnée
- Recherche : "lag features in time series" et "window function"
- Question clé : Combien de lags tu dois garder avant que l'information soit trop vieille?

**6. Temporal features**
- Hour, day of week, month, quarter
- Is_market_open, Is_before_FOMC
- Days_since_economic_event
- Recherche : "cyclical features" et "temporal factors in trading"
- Question clé : EURUSD se comporte-t-il différemment lundi matin vs jeudi après-midi?

**7. Cross-asset features**
- Corrélation avec S&P500, Or, Pétrole
- Forex indices (DXY = Dollar Index)
- Interest rates spread (US vs Eurozone)
- Recherche : "forex correlations" et "carry trade concept"
- Question clé : Si la FED relève les taux + le S&P monte, EUR/USD baisse. C'est universel?

**8. Absence de leakage**
- Information ne doit pas venir du future
- Dépendances circulaires
- Forward-looking vs backward-looking
- Recherche : "data leakage in ML" et "target leakage detection"
- Question clé : Tu utilises "volatility realized demain". C'est du leakage?

**9. Feature scaling**
- Normalization (0-1) vs Standardization (mean=0, std=1)
- Impact sur différents modèles
- Recherche : "feature scaling for ML" et "when to normalize"
- Question clé : Tree-based model (XGBoost) a besoin de scaling? Et neural networks?

**10. Feature selection**
- Collinéarité : si 2 features sont corrélées à 0.95, garde-tu les 2?
- Importance (via modèle)
- Domain knowledge vs data-driven
- Recherche : "feature selection methods" et "multicollinearity"
- Question clé : Tu as 100 features. Tu les gardes toutes ou tu purges les moins importantes?

### Ressources à consulter

- **Tech indicators** : Littérature sur indicateurs techniques (Wikipedia, Trading books)
- **Statistics** : Cours sur statistiques temps-série
- **Feature design** : Articles sur feature engineering en ML
- **Finance** : Littérature sur "what moves forex"

### Livrables attendus

- Catalogue complet des features envisagées (50+)
- Pour chaque feature : logique, calcul (sans code), justification
- Analyse de corrélations (quelles features sont redondantes?)
- Plan de détection de look-ahead bias

### Compétences à valider

✅ Citer 30 features pertinentes pour EURUSD
✅ Expliquer pourquoi chaque feature a un pouvoir prédictif
✅ Identifier look-ahead bias dans 5 features suspectes
✅ Justifier absence de leakage dans ta feature set

---

## 🔹 PHASE 6 : ML TRAÇABLE

### 🎯 Objectif
Transformer du ML (notebooks) en ingénierie reproductible et tracée.

### Pourquoi différent d'un notebook Kaggle

| Notebook | Production |
|---|---|
| "J'ai 88% d'accuracy" | "Quel modèle, quelle version, quel résultat?" |
| Trial and error | Expériences tracées systématiquement |
| Pas de checkpoints | Checkpoints à chaque étape |
| Pas de historique | Comparaison de 50 runs différents |
| Pas de dépendances | Requirements.txt + version freezé |

### Concepts à maîtriser

**1. Baseline modeling**
- Modèle naïf : "demain = aujourd'hui"
- Modèle statistique : ARIMA
- Importance : savoir si ton modèle complexe > baseline
- Recherche : "baseline models for forecasting" et "persistence model"
- Question clé : Si tu prédis "USD/EUR monte" 100% du temps, t'as quel accuracy?

**2. Training/Validation/Test split en time-series**
- Walk-forward : jamais future dans train
- Pourquoi K-fold invalide pour time-series
- Gap entre train et test (prévient look-ahead)
- Recherche : "time series cross validation" et "walk forward validation"
- Question clé : Comment splits-tu 5 ans de données en train/val/test?

**3. Métrique d'évaluation appropriée**
- Accuracy ≠ utile (peut être trompeur)
- Precision/Recall (pour classification binaire)
- RMSE/MAE (pour régression)
- Recherche : "evaluation metrics for trading" et "why accuracy is misleading"
- Question clé : Prédire la direction (haut/bas) vs magnitude du mouvement = métriques différentes?

**4. Tuning d'hyperparamètres**
- Grid search vs Random search
- Validation set pour tuning
- Risque d'overfitting aux hyperparams
- Recherche : "hyperparameter tuning" et "random search vs grid search"
- Question clé : Tu tune sur train/val, tu testes sur test. Test est-il vraiment "unseen"?

**5. Multiple models vs Single model**
- Ensemble : combiner plusieurs modèles
- Stacking, Bagging, Boosting
- Quand ensemble aide, quand ça fait du mal
- Recherche : "ensemble methods in ML" et "when to use ensemble"
- Question clé : 5 modèles prédisant en moyenne > 1 bon modèle?

**6. Tracking avec MLflow**
- Concepts : experiments, runs, parameters, metrics, artifacts
- Logging automatique
- Comparaison de runs
- Recherche : "MLflow concepts" et "experiment tracking importance"
- Question clé : Comment retrouver "le modèle qui faisait 55% accuracy" 6 mois après?

**7. Model registry et versioning**
- Production-ready vs development
- Metadata : score, date, équipe
- Transition entre versions
- Recherche : "model registry" et "model lifecycle"
- Question clé : Tu déploies version 5, elle fail. Tu reverses à version 4. C'est facile?

**8. Model interpretability**
- Feature importance : quelles features font la décision
- SHAP, LIME pour explainability
- Pourquoi c'est important en trading (tracabilité d'une perte)
- Recherche : "model interpretability" et "SHAP values"
- Question clé : Si modèle prédit "EUR monte" basé sur 1 feature seule, c'est sus?

**9. Failing gracefully**
- Quand PAS faire de prédiction (confiance insuffisante)
- Threshold de confiance
- Fallback mechanisms
- Recherche : "prediction confidence" et "confidence threshold"
- Question clé : Si ton modèle est 51% sûr EUR monte, tu trades?

### Ressources à consulter

- **Modèles** : Littérature sur XGBoost, LightGBM, ARIMA
- **Validation** : Articles sur walk-forward validation
- **Tracking** : MLflow documentation (concepts)
- **Metrics** : Articles sur métriques en trading

### Livrables attendus

- Baseline model documenté (naïf + ARIMA)
- Plan de train/val/test walk-forward
- 3-5 modèles différents essayés (résumé résultats)
- Dashboard MLflow avec 20+ runs comparés
- Feature importance analysis

### Compétences à valider

✅ Expliquer pourquoi K-fold ne marche pas
✅ Implémenter walk-forward validation
✅ Comparer 3 modèles objectivement
✅ Interpréter feature importance d'un modèle

---

## 🔹 PHASE 7 : SERVING & DÉPLOIEMENT

### 🎯 Objectif
Rendre le modèle consommable en temps réel par des décisions de trading.

### Pourquoi c'est critique

**Différence** :
- Notebook : "Génère prédiction pour données fixes, une fois"
- Production : "Reçoit nouveau datapoint toutes les heures, génère signal en < 100ms, gère erreurs"

### Concepts à maîtriser

**1. Prédiction en production vs training**
- Distribution de données peut changer (drift)
- Latency requirements (100ms max)
- Batch vs online predictions
- Recherche : "prediction serving" et "online vs batch prediction"
- Question clé : Tu reçois nuevo price à 14:00:00.000. À 14:00:00.100, tu veux une décision. Possible?

**2. API REST fundamentals**
- Endpoint = URL qui reçoit données, retourne prédiction
- Request/Response format
- Error handling
- Recherche : "REST API basics" et "API design for ML"
- Question clé : Si l'API répond "error 500", ton robot de trading fait quoi?

**3. Framework FastAPI vs Flask**
- Différences en performance
- Async/await pour latency
- Validation de données entrantes
- Recherche : "FastAPI vs Flask" et "async Python for APIs"
- Question clé : FastAPI c'est 10x plus vite que Flask. Ça vaut la peine d'apprendre?

**4. Model serving patterns**
- Single model : une prédiction = un modèle
- Ensemble : combiner prédictions de plusieurs modèles
- Canary deployment : nouveaux modèles sur petit % du traffic
- Recherche : "model serving patterns" et "canary deployment"
- Question clé : Tu as modèle V5 et V6. Comment tu testes V6 sur 10% du traffic?

**5. Caching et performance**
- Requêtes identiques = même réponse (sans recalcul)
- Cache invalidation (quand le cache devient périmé)
- Recherche : "caching in APIs" et "cache invalidation"
- Question clé : Si tu mets en cache "EUR/USD close = 1.095", tu mets le cache jusqu'à quand?

**6. Monitoring de l'API**
- Latency : temps de réponse
- Throughput : requêtes/seconde
- Error rate : % d'erreurs
- Recherche : "API monitoring" et "SLA metrics"
- Question clé : Si API répond lentement (500ms au lieu de 100ms), c'est une alerte?

**7. Rollback et versioning**
- Capacité de reverser rapidement si buggu
- Blue-green deployment : 2 versions en parallèle
- Recherche : "blue-green deployment" et "zero-downtime deployment"
- Question clé : Si nouveau modèle tue le P&L, tu le desactive en combien de temps?

**8. Docker et containerization**
- Isolement : chaque version du modèle dans son container
- Reproductibilité : même image = même résultats
- Recherche : "Docker for ML" et "containerization concepts"
- Question clé : Docker c'est quoi exactement et pourquoi ça aide?

**9. Feedback loop et récollecte de données**
- Chaque prédiction = données d'apprentissage potentielles
- Tracking de "prédiction vs réalité"
- Collecte de ces écarts pour retraining
- Recherche : "feedback loop" et "active learning"
- Question clé : Une prédiction "EUR monte" quand réalité "EUR baisse". Tu l'utilises pour retraining?

### Ressources à consulter

- **API** : Documentation FastAPI (concepts, architecture)
- **Serving** : Articles sur model serving patterns
- **Deployment** : Articles sur blue-green, canary
- **Docker** : Docker documentation (concepts)

### Livrables attendus

- Spécification API (endpoint, payload, response)
- Architecture de serving (schéma)
- Plan de monitoring (quoi monitorer, seuils d'alerte)
- Procédure de rollback documentée
- Docker strategy (si applicable)

### Compétences à valider

✅ Dessiner une API REST pour prédictions
✅ Expliquer problème de latency vs accuracy
✅ Concevoir plan de rollback < 5 minutes
✅ Monitorer 5 métriques critiques

---

## 🔹 PHASE 8 : MONITORING & DRIFT

### 🎯 Objectif
Savoir QUAND ton modèle ment, AVANT qu'il ne coûte cher.

### Pourquoi c'est différent de "normal monitoring"

| Système normal | Trading |
|---|---|
| API lente = bad UX | API lente = trade manqué = perte |
| Données manquantes = "retry" | Données manquantes = décision sur données vieilles = perte |
| Modèle drift = résultats moins bons | Modèle drift = pertes financières = CRITIQUE |

### Concepts à maîtriser

**1. Data drift (Feature drift)**
- Distribution des features change
- Exemple : volatilité moyenne passe de 2% à 5%
- Détection : comparer distribution actuelle vs historique
- Recherche : "data drift detection" et "statistical tests for drift"
- Question clé : Si volatilité monte de 2% à 2.5%, c'est un drift significatif?

**2. Prediction drift**
- Distribution des prédictions change
- Exemple : modèle prédit "EUR monte" 70% du temps vs 40% avant
- Indication possibilité que données ont changé
- Recherche : "prediction drift" et "monitoring predictions"
- Question clé : Quelle % de "EUR monte" vs "EUR baisse" est normal?

**3. Target drift / Performance drift**
- Réalité change
- Exemple : modèle prédisait 55% accuracy, maintenant 48%
- Mettre en evidence le plus grave
- Recherche : "target drift" et "model performance monitoring"
- Question clé : À quelle baisse d'accuracy tu décides "le modèle ne marche plus"?

**4. Méthodologie d'alerting**
- Seuils : "accuracy < 50%" ou "data drift p-value < 0.01"
- Sensibilité vs Spécificité
- False alarms vs missed problems
- Recherche : "alerting strategies" et "false positive rate"
- Question clé : Tu veux une alerte à chaque 0.1% de changement (trop) ou une fois par semaine (trop tard)?

**5. Tools pour monitoring**
- Evidently : monitoring drift
- Great Expectations : validation continue
- Custom dashboards
- Recherche : "Evidently AI" et "production monitoring tools"
- Question clé : Quels sont les outils gratuit vs payant pour monitoring?

**6. Retraining trigger**
- À quel moment tu réentraînes
- Automatiqu vs manuel
- Risque : réentraîner sur données bugguées = pire
- Recherche : "model retraining strategies" et "drift detection"
- Question clé : Tous les jours? Une fois par semaine? Ou à la demande?

**7. A/B testing**
- Tester nouveau modèle sur subset du traffic
- Mesurer impact sur P&L
- Rollout progressif
- Recherche : "A/B testing in production" et "online experiments"
- Question clé : Tu as modèle V5 vs V6. Comment tu compares lequel est meilleur?

**8. Model shadows et canary**
- Lancer nouveau modèle en parallel, comparer sans utiliser ses prédictions
- Risque zéro de déploiement mauvais modèle
- Recherche : "shadow models" et "canary deployment"
- Question clé : Comment tu testes un modèle avant de lui donner 100% du traffic?

**9. Incident response**
- Processus : détecter anomalie → alerter → investiguer → fixer
- Communication : qui doit être notifié
- Rollback : reverser à quoi?
- Recherche : "incident response procedures" et "on-call playbooks"
- Question clé : Une alerte drift à 3am. Qui se lève? Quel est le plan?

**10. P&L tracking et backtesting**
- Backtest du modèle actuel vs modèle ancien
- Tracking : "ce modèle a perdu 500€ ce mois"
- Decision : reverser ou laisser?
- Recherche : "live trading P&L tracking" et "performance attribution"
- Question clé : Comment tu sais si baisse de P&L = bad model ou bad market?

### Ressources à consulter

- **Drift detection** : Littérature sur tests statistiques (KS test, chi-square)
- **Tools** : Evidently, Great Expectations documentation
- **Monitoring** : Articles sur "observability in ML"
- **Finance** : Articles sur "model governance in trading"

### Livrables attendus

- Dashboard de monitoring (7+ métriques)
- Définition de seuils d'alerte (avec justifications)
- Procédure d'incident response
- Plan de retraining automatique
- Stratégie de A/B testing pour nouveaux modèles

### Compétences à valider

✅ Détecter 5 types de drift différents
✅ Faire un dashboard de monitoring (même simple)
✅ Concevoir une procédure d'alerte
✅ Interpréter un résultat "P&L baisse : drift ou marché?"

---

# 🎓 COMPÉTENCES FINALES ACQUISES

| Domaine | Niveau | Distinction |
|---|---|---|
| **Data Engineering** | Avancé | Gestion du temps comme dimension critique |
| **SQL** | Expert | Time-series optimization |
| **Data Quality** | Rare | Détection d'anomalies financières |
| **Pipelines** | Solide | Idempotence, backfill, monitoring |
| **Feature Engineering** | Expert | No look-ahead, causalité |
| **Time-series ML** | Avancé | Walk-forward validation, stationnarité |
| **MLOps** | Professionnel | Versioning, reproducibility, monitoring |
| **Serving** | Opérationnel | Latency, rollback, feedback loops |
| **Risk & Trading** | Professionnel | P&L tracking, incident response |

---

# 🚫 CE QUE TU N'APPRENDRAS PAS

❌ Kubernetes (complexe trop tôt)
❌ LLMs (hors scope trading)
❌ High-frequency trading (microseconde latency)
❌ Algorithmes complexes type deep RL
❌ Options pricing / Derivatives (finance avancée)

---

# 📊 RESSOURCES CRITIQUES PAR PHASE

## Phase 1
- TimescaleDB official docs
- OANDA API documentation  
- "Time Series Databases" article/white-papers
- Great Expectations concepts

## Phase 2
- Airflow/Prefect documentation (DAG concepts)
- "Data Pipeline Patterns"
- Monitoring tools comparison

## Phase 3
- Great Expectations framework
- ADF/KPSS test tutorials
- "Data Quality for ML"
- "Look-ahead bias in practice"

## Phase 4
- DVC conceptual overview
- MLflow documentation
- "Reproducibility in ML" papers
- Data lineage tools

## Phase 5
- "Technical Analysis Explained"
- Time-series feature engineering guides
- "Microstructure of Financial Markets"
- Forex correlation studies

## Phase 6
- XGBoost/LightGBM documentation
- "Time Series Forecasting" textbooks
- Walk-forward validation tutorials
- MLflow tracking concepts

## Phase 7
- FastAPI documentation
- REST API design principles
- "Model Serving at Scale"
- Docker concepts

## Phase 8
- Evidently documentation
- "Monitoring ML Systems"
- Statistical drift tests
- Trading incident response papers

---

# 🎯 PHRASE CLÉ

> **"En trading, tu échoues rarement à cause des modèles.
> Tu échoues à cause des données mal gérées, du timing, et du drift non détecté."**

Maîtriser ces 8 phases = Senior Data Engineer + ML Systems Engineer. Prêt?