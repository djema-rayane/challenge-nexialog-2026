# Challenge NEXIALOG 2026 — Modélisation des Valeurs Résiduelles Automobiles
### NEXIALOG Consulting × Master MOSEF Data Science | Mobilize Financial Services (Groupe Renault)

---

## Table des matières

1. [Contexte du challenge](#1-contexte-du-challenge)
2. [Compréhension du problème](#2-compréhension-du-problème)
3. [Les données](#3-les-données)
4. [Découvertes clés lors de l'exploration](#4-découvertes-clés-lors-de-lexploration)
5. [Indices méthodologiques extraits des PDF](#5-indices-méthodologiques-extraits-des-pdf)
6. [Plan de modélisation](#6-plan-de-modélisation)
7. [Livrables](#7-livrables)
8. [Prochaines étapes](#8-prochaines-étapes)

---

## 1. Contexte du challenge

**Commanditaire :** Mobilize Financial Services (filiale financière du Groupe Renault)  
**Organisateur :** NEXIALOG Consulting  
**Partenaire académique :** Master MOSEF (Modélisation Économique et Statistique des Marchés Financiers)

**Enjeu métier :** Mobilize Financial Services propose des contrats de crédit-bail (LLD / LOA) sur des véhicules. À la fin du contrat, le véhicule est restitué et revendu sur le marché de l'occasion. La **valeur résiduelle (VR)** est le prix auquel ce véhicule sera vendu à cette date. Une mauvaise estimation de la VR expose la société à des **risques financiers majeurs** (pertes sur cession, provisions inadéquates).

**Objectif :** Prédire le **prix de vente en euros** de chaque véhicule du portefeuille à la fin de son contrat de financement.

**Calendrier :**
- Soumission des prédictions : **17 avril 2026 avant minuit**
- Soutenance : **21 avril 2026**

**Format de soumission :** Fichier CSV nommé `groupe_NOM1_NOM2` avec deux colonnes : `id` et `prediction`

---

## 2. Compréhension du problème

### 2.1 Qu'est-ce que la Valeur Résiduelle ?

La valeur résiduelle représente l'estimation de la valeur de revente future d'un véhicule au terme d'un contrat de crédit-bail. Elle dépend principalement de :

- **L'âge du véhicule** (ancienneté depuis la production)
- **Le kilométrage** (usage cumulé)
- **Le segment / motorisation** (thermique, hybride, électrique)
- **La conjoncture du marché** (offre/demande sur l'occasion)
- **Les risques climatiques** (réglementations Crit'Air, dépréciation des batteries EV)

### 2.2 La Décote

La décote mesure la dépréciation relative d'un véhicule :

```
Décote(t) = 1 - V(t) / V₀
```

Où :
- `V(t)` = valeur au temps `t`
- `V₀` = prix catalogue d'origine (neuf)

### 2.3 Modèle de dépréciation exponentielle

La littérature actuarielle et le guide méthodologique fourni s'appuient sur un modèle **log-linéaire** :

```
V(t) = V₀ × e^(-k×t)
```

En prenant le logarithme :
```
ln(V(t) / V₀) = -k × t
```

Le **taux de dépréciation k** se calcule empiriquement :
```
k = -(12 / âge_en_mois) × ln(prix_vente / prix_catalogue)
```

### 2.4 Modèle à deux paramètres (âge + kilométrage)

Extension du modèle pour incorporer simultanément l'âge et le kilométrage :

```
ln(V_{g,t} / V_{g,0}) = α_g + k_g1 × Age_t + k_g2 × Kilométrage_t + ε
```

Où `g` désigne un groupe de véhicules homogènes (segment, motorisation…).

Ce modèle implique :
- `α_g` ≈ 0 théoriquement (si V(0) = V₀)
- `k_g1` : taux de dépréciation lié à l'âge
- `k_g2` : taux de dépréciation lié au kilométrage

### 2.5 Problème de prédiction

Le challenge consiste à :
1. Estimer les paramètres `(α_g, k_g1, k_g2)` par groupe à partir des données `used_market`
2. Appliquer ces paramètres aux véhicules du `portfolio` en tenant compte de leur **âge et kilométrage projetés à la fin du contrat**

---

## 3. Les données

### 3.1 `used_market.xlsb` — Données d'entraînement

**Source :** Transactions du marché de l'occasion automobile en Allemagne (Groupe Renault)  
**Volume :** 736 958 lignes × 22 colonnes  
**Couverture temporelle :** 2018–2025

| Colonne | Description |
|---------|-------------|
| `country_code` | Code pays (DE = Allemagne) |
| `brand` | Marque (Renault, Dacia, Alpine…) |
| `MODEL` | Modèle du véhicule |
| `MARK` | Génération du modèle |
| `Phase` | Phase (Facelift) |
| `RANGE TYPE` | Gamme (Entry, Standard, Premium…) |
| `FUEL TYPE` | Motorisation (Essence, Diesel, Hybride, Électrique) |
| `Engine Power(HP)` | Puissance moteur en chevaux |
| `gearbox` | Boîte de vitesses |
| `BODY TYPE` | Type de carrosserie |
| `MODEL SEGMENT` | Segment de marché |
| `grouping` | Regroupement pour la modélisation |
| `modelyear` | Année modèle |
| `productionYear` | Année de production |
| `id_group` | Identifiant du groupe |
| `id` | Identifiant de l'observation |
| `date de vente` | Date de la transaction (serial Excel) |
| `mileage` | Kilométrage |
| `age` | Âge en mois |
| `prix de vente` | Prix de vente sur le marché de l'occasion (€) — **variable cible** |
| `prix catalogue d'origine` | Prix neuf d'origine (€) |
| `sample_size` | Nombre de transactions agrégées |

### 3.2 `portfolio.xlsx` — Données à prédire

**Volume :** 1 951 lignes × 19 colonnes  
**Description :** Contrats de crédit-bail actifs ou proches de leur terme

| Colonne | Description |
|---------|-------------|
| `id` | Identifiant unique du contrat |
| `country_name` | Pays |
| `produit financier` | Type de contrat (LLD, LOA…) |
| `version_name` | Nom de version du véhicule |
| `brand` | Marque |
| `model` | Modèle |
| `production_year` | Année de production |
| `fuel_type` | Motorisation |
| `range_type` | Gamme |
| `contract_start_date` | Date de début du contrat |
| `current_contract_planned_end_date` | Date de fin prévue |
| `contract_start_year` | Année de début |
| `contract_end_year` | Année de fin projetée |
| `contract_duration` | Durée totale (mois) |
| `remaining_contract_duration` | Durée restante (mois) — peut être négatif |
| `contract_mileage` | Kilométrage prévu au contrat |
| `initial_car_age` | Âge initial du véhicule au démarrage du contrat (mois) |
| `initial_mileage` | Kilométrage initial |
| `prix catalogue d'origine` | Prix neuf d'origine (€) |

---

## 4. Découvertes clés lors de l'exploration

### 4.1 Structure des données `used_market`

**Données agrégées, non individuelles**

Chaque ligne ne représente **pas** une transaction individuelle mais une **moyenne de `sample_size` transactions réelles**. Cela signifie :
- Les modèles doivent être entraînés avec **pondération par `sample_size`**
- La variance des prix est sous-estimée si on ignore ce poids
- `sample_size` = 1 sur certaines lignes → potentiellement des observations rares à traiter avec précaution

**Colonnes dupliquées identifiées**

| Paire | Observation |
|-------|-------------|
| `id` = `id_group` | Parfaitement identiques → supprimer l'une |
| `modelyear` = `productionYear` | Corrélation parfaite → supprimer l'une |

**Paliers discrets d'âge et de kilométrage**

Le marché de l'occasion structure les cotations par tranches :
- **Âge** : 14 paliers discrets (non continus) en mois
- **Kilométrage** : 12 paliers discrets

**Conséquence** : Les véhicules du portfolio n'auront pas exactement ces valeurs → nécessité d'**interpolation** ou de **projection sur le palier le plus proche**.

### 4.2 Analyse temporelle — Effet COVID

Le ratio `prix_vente / prix_catalogue` varie significativement dans le temps :

| Année | Ratio moyen |
|-------|-------------|
| 2018  | 0.449       |
| 2020  | 0.437       |
| 2022  | 0.562       |
| 2023  | 0.577       |
| 2025  | 0.507       |

**Interprétation :** La crise COVID (2020-2021) a entraîné une forte perturbation de l'offre (pénurie de semi-conducteurs → moins de voitures neuves → hausse des prix de l'occasion). Le pic de 2023 reflète cet effet retardé. Le retour vers 0.50 en 2025 suggère une normalisation progressive.

**Implication pour la modélisation :** Les prédictions pour 2026-2030 doivent tenir compte de cette **dynamique temporelle** — ne pas utiliser un modèle statique entraîné uniquement sur les données historiques sans variable temporelle.

### 4.3 Qualité des données

**Données `used_market` :** Aucune valeur manquante (NaN) détectée. Les colonnes numériques n'ont pas de zéros aberrants.

**Outliers identifiés :**
- 11 lignes avec `ratio > 1` (prix de vente > prix catalogue neuf) → possiblement des véhicules rares ou des erreurs de saisie
- 22 lignes avec `ratio < 0.05` (quasi nuls) → erreurs probables

**Portfolio :**
- **106 contrats avec `remaining_contract_duration < 0`** → contrats déjà expirés ; les véhicules ont peut-être déjà été restitués (à traiter avec attention)
- Nombreux zéros en `initial_car_age` et `initial_mileage` → véhicules neufs au démarrage du contrat (normal pour LLD)

### 4.4 Déséquilibre VE / Thermique

| Motorisation | Part portfolio | Part used_market |
|---|---|---|
| Électrique (EV) | ~20% | ~5% |
| Thermique + Hybride | ~80% | ~95% |

**Conséquence :** Les modèles EV sont **sous-représentés** dans les données d'entraînement. En particulier, la **Renault R5** (nouveau modèle électrique) n'a que ~45 observations dans `used_market`.  
**Risque :** Sur-apprentissage sur les VE → résidus élevés sur 20% du portfolio.

### 4.5 Ancienneté du modèle (variable à créer)

```python
ancienneté_modèle = année_vente - modelyear
```

Cette variable capture le **risque générationnel** : un modèle en fin de cycle de vie se déprécie plus vite car les acheteurs anticipent l'arrivée d'une nouvelle génération. Cette variable est mentionnée explicitement dans le guide méthodologique comme facteur important.

### 4.6 Variables clés à calculer pour la modélisation

Pour `used_market` :
```python
log_ratio = log(prix_vente / prix_catalogue)       # variable cible log-linéaire
k = -(12 / age) * log(prix_vente / prix_catalogue) # taux de dépréciation empirique
ancienneté_modèle = année_vente - modelyear
```

Pour `portfolio` :
```python
age_fin = (contract_end_year - production_year) * 12   # âge prévu à la fin du contrat
km_fin = initial_mileage + contract_mileage              # km prévus à la fin
log_ratio_prédit = α_g + k_g1 * age_fin + k_g2 * km_fin
prix_prédit = prix_catalogue * exp(log_ratio_prédit)
```

---

## 5. Indices méthodologiques extraits des PDF

### 5.1 Du guide "Conférence Crédit Bail ML et Climat"

**Modèle classique recommandé :**
- Régression log-linéaire par groupe de véhicules homogènes
- Pondération obligatoire par `sample_size`
- Contrainte de monotonie décroissante via **régression isotonique (algorithme PAVA)**

**Familles de dépréciation (clustering) :**
- Regrouper les groupes selon leurs vecteurs `(α_g, k_g1, k_g2)` estimés
- Utiliser **K-means** pour créer des familles de dépréciation
- Avantage : robustesse pour les groupes avec peu d'observations (emprunt de force aux voisins)

**Modèle ML recommandé :**
- **CatBoost avec régression quantile** pour prédire `k` (taux de dépréciation)
- Avantages cités : robuste aux outliers, gère les variables catégorielles nativement, quantile pour l'intervalle de confiance
- **SHAP values** pour l'interprétabilité

**Données externes recommandées :**
- Indice de prix des voitures d'occasion (INSEE / Banque de France)
- URL Banque de France : `webstat.banque-france.fr/fr/catalogue/icp/ICP.M.FR.N.071100.4.ANR`

**Variables importantes identifiées dans le guide :**
1. Âge (mois)
2. Kilométrage
3. Motorisation (EV vs thermique)
4. Segment de marché
5. Ancienneté du modèle (génération)
6. Indice de prix conjoncturel (externe)
7. Variables temporelles (effet COVID, saisonnalité)

### 5.2 Risques climatiques (4 approches)

Le guide distingue **4 approches de stress climatique** :

| Approche | Description |
|----------|-------------|
| **Transition réglementaire** | Restrictions Crit'Air → dépréciation accélérée des véhicules thermiques |
| **Risque physique** | Dégradation des batteries EV liée aux chaleurs extrêmes |
| **Choc de demande** | Shift accéléré vers l'électrique → effondrement des prix thermiques |
| **Stress test intégré** | `V(t)^Stress = V₀ × e^(-kt) × Δ^(stress_climatique)` |

### 5.3 Approche hybride recommandée

Le guide suggère une approche en **deux étages** :
1. **Modèle classique** : Log-linéaire par groupe → baseline solide et interprétable
2. **Modèle ML** : CatBoost pour capturer les non-linéarités et effets temporels → amélioration du baseline

**Combinaison** : Blend pondéré ou stacking des deux modèles.

---

## 6. Plan de modélisation

### Phase 1 — Feature Engineering

```python
# used_market
df['log_ratio'] = np.log(df['prix de vente'] / df["prix catalogue d'origine"])
df['k'] = -(12 / df['age']) * df['log_ratio']
df['date'] = pd.to_datetime(df['date de vente'])  # conversion serial Excel
df['year'] = df['date'].dt.year
df['ancienneté_modèle'] = df['year'] - df['modelyear']

# portfolio
df_p['age_fin'] = (df_p['contract_end_year'] - df_p['production_year']) * 12
df_p['km_fin'] = df_p['initial_mileage'] + df_p['contract_mileage']
```

### Phase 2 — Modèle 1 : Régression Log-Linéaire Classique

Pour chaque groupe `g` défini par `(brand, MODEL, FUEL TYPE, MODEL SEGMENT)` :

```
ln(V_{g,t} / V_{g,0}) = α_g + k_g1 × Age_t + k_g2 × Kilométrage_t + ε
```

- Entraînement **WLS** (Weighted Least Squares) avec poids = `sample_size`
- Contrainte de monotonie par régression isotonique (PAVA)
- Fallback : K-means pour les groupes avec < N observations → emprunt d'un groupe proche

### Phase 3 — Modèle 2 : CatBoost Quantile

Features : `[age, mileage, FUEL TYPE, MODEL SEGMENT, brand, MODEL, year, ancienneté_modèle, log(prix_catalogue), indice_bdf]`  
Target : `k` (taux de dépréciation)  
Loss : `QuantileLoss(quantile=0.5)` (médiane robuste)

### Phase 4 — Prédictions Portfolio

```python
# Assemblage final
log_ratio_pred = alpha_g + k_g1 * age_fin + k_g2 * km_fin
prix_pred = prix_catalogue * np.exp(log_ratio_pred)
```

Ajustements :
- Correction temporelle par l'indice Banque de France
- Plancher à 500€ (valeur ferraille)
- Clip sur [0.05, 1.5] × prix_catalogue pour éviter les aberrations

### Phase 5 — Stress Test Climatique (si requis)

```python
V_stress = V0 * np.exp(-k * t) * delta_stress_climatique
```

Où `delta_stress_climatique` est estimé selon le scénario réglementaire (Crit'Air, SNBC…)

---

## 7. Livrables

Selon le cahier des charges :

| Livrable | Description |
|----------|-------------|
| Fichier de prédictions | CSV `groupe_NOM1_NOM2` avec colonnes `id`, `prediction` |
| Rapport synthétique | Démarche méthodologique, résultats, limites |
| Support de présentation | Slides pour la soutenance du 21 avril 2026 |
| Code source | Notebooks Python documentés |

---

## 8. Prochaines étapes

- [ ] Feature engineering complet (log_ratio, k, age_fin, km_fin, ancienneté_modèle)
- [ ] Téléchargement de l'indice Banque de France (URL : `webstat.banque-france.fr`)
- [ ] Entraînement Modèle 1 : WLS log-linéaire par groupe avec PAVA
- [ ] Clustering K-means des familles de dépréciation
- [ ] Entraînement Modèle 2 : CatBoost quantile
- [ ] Génération des prédictions et validation des bornes
- [ ] Analyse SHAP pour l'interprétabilité
- [ ] Rédaction du rapport et des slides
- [ ] Traitement spécifique des VE (R5, Mégane E-Tech, Zoé)
- [ ] Stress test climatique

---

## Structure du projet

```
CHALLENGE-NEXIALOG-2026/
├── README.md                          # Ce fichier
├── data/
│   ├── used_market.xlsb               # Données marché occasion (736k lignes)
│   ├── portfolio.xlsx                 # Portefeuille à prédire (1951 contrats)
│   └── external/                      # Indices externes (Banque de France…)
├── notebooks/
│   ├── 01_exploration.ipynb           # Exploration des données
│   ├── 02_feature_engineering.ipynb   # Construction des features
│   ├── 03_model_classique.ipynb       # Régression log-linéaire WLS
│   ├── 04_model_catboost.ipynb        # CatBoost quantile
│   └── 05_predictions.ipynb           # Génération des prédictions finales
├── src/
│   ├── preprocessing.py               # Nettoyage et feature engineering
│   ├── models.py                      # Classes de modèles
│   └── utils.py                       # Fonctions utilitaires
└── output/
    └── groupe_NOM1_NOM2.csv           # Fichier de soumission final
```

---

*Challenge NEXIALOG 2026 — Master MOSEF — Mobilize Financial Services*
