# Analyse du code source — algo ML "compteur bloqué" (BlockedMeter)

Source : packages fournis par l'utilisateur le 27/08/2026 —
`synapse_blockedmtr-0.3.1-py3-none-any.whl`, `codai_calibration-0.1.0-py3-none-any.whl`,
`model_case_a.pkl`, `model_case_c.pkl` (modèles entraînés, non explorés en détail —
trop volumineux et peu exploitables sans le pipeline Spark complet).

Ce document capitalise ce qu'on a pu en extraire pour réduire l'écart entre notre
méthode heuristique KQL et le vrai score `BlockedMeterLevel1-4`.

## 1. Chaîne de traitement (résumé)

1. **Filtre "suspect"** (`measures_daily_prep.py`, `create_suspect_transmitters_view`) :
   sur une fenêtre glissante de **seulement 4 jours** (reporting_date-3 à reporting_date),
   un transmitter est "suspect" s'il n'a **pas** eu 2 jours consécutifs ≥20L. C'est le
   filtre d'entrée du pipeline — très sensible, explique pourquoi la Liste B officielle
   est large (49 570 sur LYONNAISE).
2. **Calcul des features** (`features_computations.py`) — voir section 2.
3. **Conditions d'éligibilité** avant scoring — voir section 3.
4. **Score ML** (modèle entraîné, `model_case_a.pkl`/`model_case_c.pkl` — "case A"/"case C"
   probablement deux segments de parc différents, pas creusé) → probabilité.
5. **Catégorisation en niveau de risque** (`api/model_prediction.py`, fonction
   `risk_category`) :
   ```python
   thresholds = (0.4, 0.25, 0.15)
   # response > 0.4        -> Level 1 (risque FORT)
   # 0.25 < response <= 0.4  -> Level 2
   # 0.15 < response <= 0.25 -> Level 3
   # response <= 0.15      -> Level 4 (risque très faible)
   ```
   ⚠️ **Level 1 = risque le plus fort**, pas le plus faible — contre-intuitif,
   confirmé aussi dans `regles-metier-snapshot.md` (Level1="Risque fort").
6. **Matrice de priorité finale** : croise le niveau de risque avec un "type de
   consommateur" (Small/Medium/Large, dérivé du diamètre × index cumulé) pour
   obtenir une priorité 0-5. Non répliqué en KQL (pas assez d'info sur les seuils
   exacts de la matrice dans le code exploré).

## 2. Définitions clés des features

- **Jour significatif** : consommation (`volume_l`) ≥ 20L **et** au moins un jour
  adjacent (J-1 ou J+1) aussi ≥ 20L. Ce n'est PAS "conso > 0" — un jour isolé à 25L
  entouré de deux jours à 0L n'est pas significatif.
- **T0** : date du dernier jour significatif. Sert d'ancre pour toutes les fenêtres
  suivantes (pas "aujourd'hui" en dur).
- **V7** : dernier index valide (peu importe la granularité — cf `MSR_CODES` ci-dessous).
- **V1** : nombre de jours avec `volume_l` non vide entre T0 et la date du dernier
  index valide (`DayLastValidIndex`).
- `MSR_CODES = ("index_15m_l", "index_1h_l", "index_24h_l", "index_5m_l", "index_6h_l")`
  — la dernière lecture valide est prise sur ces 5 codes combinés, jamais un seul.
- `ALR_CODE (fraude) = ("Fraud_PermanentUnclippingAlert", "Fraud_FuzzyUnclippingAlert")`
  — exactement ces deux codes structurés, pas le code legacy `fraud`, pas les codes
  `OtherDefect_*Misalignment`.

## 3. Conditions d'éligibilité (avant le score ML)

```
V7 >= 1000
ET :
  SI meterDiameter < 25mm : V1 > 60 jours ET pas d'alarme fraude dans les 60 jours
                            précédant DayLastValidIndex
  SI meterDiameter >= 25mm : V1 > 7 jours ET pas d'alarme fraude dans les 7 jours
                             précédant DayLastValidIndex
```

Implication pratique : les gros compteurs (DN≥25) sont évalués beaucoup plus vite
(7 jours) et avec une fenêtre d'exclusion fraude bien plus courte que les petits
compteurs. Une méthode KQL qui applique un seuil uniforme (ex: 60 jours partout)
sous-détecte fortement sur les gros compteurs.

## 4. Résultats de validation (LYONNAISE, référence 24/08/2026, fenêtre 90j)

| Étape | Résultat |
|---|---|
| Liste B (alarme officielle, transmitters distincts, fenêtre 90j) | 49 570 |
| v1 — `index_24h_l` seul, figé >60j, exclusions basiques, conso antérieure >500L/an | 2 856 candidats, 79% confirmés (2 260) |
| v2 — 5 codes d'index combinés | 7 839 candidats, 82% confirmés (6 416) |
| v3 — + seuils/exclusions diamètre-dépendants + V7≥1000 | 9 664 candidats, 79% confirmés globalement, **84% sur DN≥25mm** |
| Décomposition de l'écart v2 (25 479 "index varie" dans Liste B) | via critère T0 : 13 228 candidats supplémentaires (53,5% de cette sous-population) |
| Faux positifs du critère T0 (testé sur 2000 compteurs hors Liste B) | 176 (8,8%) |

### Requête de calcul du T0 (validée, ne PAS utiliser `prev()`/`next()` — trop lourd
sur le connecteur ADX utilisé, préférer une jointure décalée d'un jour) :

```kql
let dailyVol = msr | where client == "X" and code == "volume_l"
  and valueDate >= fenetreGel and valueDate <= ref and transmitter in (candList)
| project transmitter, valueDate, value;
let dailyVolShift = dailyVol | project transmitter, valueDate=valueDate-1d, nextVal=value;
let pairs = dailyVol
| join kind=inner dailyVolShift on transmitter, valueDate
| where value >= 20 and nextVal >= 20
| project transmitter, T0candidate = valueDate + 1d;
let t0 = pairs | summarize T0=max(T0candidate) by transmitter;
// Candidats = transmitters SANS aucun T0 dans la fenêtre (jamais 2 jours consécutifs >=20L)
let noT0 = population | join kind=leftouter (t0 | extend hasT0=true) on transmitter | where isempty(hasT0);
```

⚠️ Pour valider un critère dérivé du code source, **toujours tester sur une
population indépendante de celle utilisée pour le construire** (ex: échantillon de
compteurs hors Liste B) — sinon le taux de confirmation est circulaire et proche
de 100% par construction, ce qui ne prouve rien. Piège rencontré une fois en
session avant correction.

## 5. Pistes non explorées (si besoin d'aller plus loin)

- Réimplémenter les features V2 à V12 complètes pour rapprocher le score exact
  (actuellement seul un sous-ensemble simplifié — T0/V1/V7 — est répliqué).
- Charger `model_case_a.pkl`/`model_case_c.pkl` pour en extraire l'importance des
  features ou les seuils exacts, si le modèle est un type simple (arbre de décision,
  régression logistique) — pas fait faute de temps, et risque de complexité inutile
  si c'est un ensemble (XGBoost/LightGBM) nécessitant l'environnement Spark d'origine.
- Explorer `codai_calibration` plus en détail (technique générique de calibration de
  probabilité — `_calibrators.py`, `_oof_calibration.py`, `_pior_shift.py`) — a priori
  peu de valeur métier directe, plutôt de l'infrastructure ML.
- Vérifier l'alarme "Clock shift" (dérive d'horloge, WI 201584 dans
  `onconnect-business-rules`) — mentionnée dans les règles métier comme cause de
  faux index figé, mais aucun code d'alarme correspondant trouvé dans `alr` pour
  LYONNAISE au moment du test (`code has "Clock"` → vide). À vérifier auprès de
  `devops-rules`/wiki si le nom exact diffère.
