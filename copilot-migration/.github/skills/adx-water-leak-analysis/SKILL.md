---
name: adx-water-leak-analysis
description: "Analyse des compteurs d'eau (télérelève) dans la base Azure Data Explorer 'onconnect' — recherche de compteurs bloqués, fuites, dérives de consommation, et export de rapports terrain. Utiliser cette skill dès qu'une question porte sur un client de télérelève d'eau (ex. SWS, LYONNAISE, LEMANSMETRO, AGUR...), sur des compteurs, sur une consommation anormale/doublée, sur une alarme (fuite, compteur bloqué, batterie faible...), ou qu'un export Excel/rapport de compteurs à risque est demandé. Toujours consulter cette skill avant d'écrire une requête KQL sur les tables meter/msr/alr/module/eqp, pour éviter de redécouvrir le schéma et les pièges à chaque fois."
---

# Analyse fuites & anomalies compteurs — base ADX "onconnect"

Cette skill capitalise sur le schéma et la méthodologie validés lors de sessions précédentes
d'analyse de la base ADX du système de télérelève d'eau. Elle évite de repartir de zéro
(découverte du schéma, des codes d'alarme, des pièges connus) à chaque nouvelle question.

## Vue d'ensemble du modèle de données

Il n'existe **pas de clé unique commune** à toutes les tables — il faut jongler entre 3 identifiants :

| Identifiant | Utilisé dans | Rôle |
|---|---|---|
| `servicePoint` | `meter`, `eqp` | Point de service / adresse physique |
| `sensor` | `meter` | Numéro de série du compteur physique (peut changer si remplacement) |
| `transmitter` | `msr`, `alr`, `module` | Identifiant du module radio qui remonte les données |

**`module` est la table pivot** entre le monde "patrimoine" (`meter`/`eqp`, indexé par `servicePoint`)
et le monde "mesures/alarmes" (`msr`/`alr`, indexé par `transmitter`). Toute analyse qui croise
consommation et infos compteur (adresse, diamètre...) doit passer par une jointure via `module`.

```kql
// Pattern de jointure standard : consommation -> compteur -> adresse
msr
| where client == "<CLIENT>" and code == "volume_l"
| summarize vol = sum(value) by transmitter
| join kind=inner (module | where client == "<CLIENT>") on transmitter
| join kind=inner (meter | where client == "<CLIENT>") on servicePoint
```

```kql
// Pattern de jointure : filtrer par usage (meter_park) puis croiser conso/alarmes
// Ex: compteurs à usage "Incendie" ayant consommé sur une période donnée
let filteredSPs = meter_park
| where servicePointUsageLabel == "Incendie"   // ou Normal/Vert/Chantier/Agricole/Test
| join kind=inner (org | where client == "<CLIENT>" | project servicePoint) on servicePoint
| distinct servicePoint;
let filteredTransmitters = filteredSPs
| join kind=inner (module | where client == "<CLIENT>" | project servicePoint, transmitter) on servicePoint
| distinct servicePoint, transmitter;
msr
| where client == "<CLIENT>" and code == "volume_l" and valueDate >= datetime(...) and valueDate < datetime(...)
| join kind=inner filteredTransmitters on transmitter
| summarize conso = sum(value) by servicePoint, transmitter
```
⚠️ Ce pattern produit souvent des doublons par `servicePoint` — voir piège #10.

### Tables clés

- **`meter`** : patrimoine compteur. Colonnes : `client`, `servicePoint`, `sensor`, `sensorType`,
  `installationDate`, `removalDate`, `meterDiameter`, `meterPulseWeight`, `manufacturer`, `model`.
  ⚠️ Un `servicePoint` peut avoir **plusieurs lignes** (historique de remplacement de compteur).
  Il n'y a pas de flag `isActive` fiable ni de `removalDate` toujours renseignée — pour trouver
  le compteur *actuellement en service*, prendre celui avec le `installationDate` le plus récent
  (heuristique, pas une garantie absolue).
- **`eqp`** : infos complémentaires du point de service — `address`, `hydroSector`, `lat`/`lon`
  (⚠️ souvent vides selon le client), `accessibility`, `type`.
- **`msr`** : mesures, format long (une ligne = un `transmitter` + un `code` + une `valueDate`).
  Voir la liste des codes utiles ci-dessous.
- **`alr`** : alarmes, même format long, indexé par `transmitter`. Voir la liste des codes ci-dessous.
- **`module`** : table de correspondance `transmitter` ↔ `servicePoint`.
- **`meter_park`** : données complémentaires du parc (issues des fichiers SEF/OnConnect), indexée par `servicePoint`. Contient notamment `servicePointUsageLabel` — utile pour filtrer par usage du point de service. Valeurs rencontrées : `Normal`, `Incendie`, `Vert`, `Chantier`, `Agricole`, `Test` (+ vide `""`). Contient aussi `sensorDiameter`, `sensorSerialNumber`, adresse (`clientSiteAddress`...). ⚠️ Voir piège #10 ci-dessous : plusieurs lignes possibles par `servicePoint`.

### Codes `msr` utiles (colonne `code`)

| Code | Signification |
|---|---|
| `volume_l` | Consommation journalière en litres — **à utiliser en priorité pour toute analyse de dérive/doublement de conso** |
| `index_1h_l`, `index_6h_l`, `index_24h_l` | Index cumulé (compteur), pas une consommation directe |
| `night_flow_3h_m3ph`, `night_flow_15m_m3ph` | Débit nocturne — **indicateur le plus fiable de fuite continue** (ne devrait jamais être significativement > 0 la nuit). Pas encore exploité systématiquement dans nos analyses — à privilégier pour fiabiliser une détection de fuite plutôt que la seule conso journalière. |
| `leak_volume_l` | Volume de fuite déjà calculé par le système, si disponible |
| `back_flow_volume_l`, `back_flow_number` | Retour d'eau (backflow) |
| `flow_max_24h_m3ph`, `flow_min_*` | Débits min/max sur fenêtres glissantes |
| `estimated_flag` / `estimatedFlag` | Indique une valeur estimée plutôt que mesurée — vérifier avant de conclure à une vraie anomalie |

### Codes `alr` utiles (colonne `code`)

| Famille | Codes | Signification |
|---|---|---|
| Fuite | `Leak_ModerateLeakAlert`, `Leak_SignificantLeakAlert` | Alarme de fuite, 2 niveaux de sévérité |
| Compteur bloqué | `BlockedMeterLevel1` à `BlockedMeterLevel4` | Sévérité croissante d'un compteur bloqué mécaniquement |
| Fraude | `fraud`, `Fraud_PermanentUnclippingAlert`, `Fraud_FuzzyUnclippingAlert`, `Fraud_MeterAccess`, `Fraud_PossibleTransmitterDefect` | Suspicion de fraude/manipulation |
| Retour d'eau | `backflow`, `highbackflow`, `BackFlow_ModerateBackflowAlert`, `BackFlow_SignificantBackflowAlert`, `BackFlow_MinorBackflowAlert`, `BackFlow_InvertedMeterMountingAlert` | Retour d'eau anormal |
| Technique | `lowBattery`/`Battery_LowBatteryAlert`, `freeze`/`Freeze_FreezeAlert`, `damage`, `detectionError`, `DetectionAnomaly`, `OtherDefect_TransmitterMisalignment`, `OtherDefect_MeterMisalignment`, `Other_OverFlowAlert` | Défauts matériels divers |

Ne pas supposer que ces codes existent pour tous les clients : toujours vérifier avec
`alr | where client == "<CLIENT>" | distinct code` avant de filtrer, car la liste peut varier.

## Pièges connus (déjà rencontrés — ne pas re-découvrir)

1. **Un client présent dans `meter` peut n'avoir aucune donnée dans `msr`** (ex: client "MALTA" :
   469k compteurs au patrimoine, 0 mesure). Toujours vérifier `msr | where client == "X" | count`
   avant de lancer une analyse de consommation — si 0, le dire clairement plutôt que de conclure
   à tort à "pas de fuite".
2. **Un client peut avoir très peu de compteurs réellement télérelevés** malgré un grand parc
   patrimoine (ex: LEMANSMETRO : seulement 2 transmitters actifs dans `msr` sur toute la base).
   Vérifier `msr | where client == "X" | distinct transmitter | count` avant de promettre "10
   compteurs" — si le nombre de transmitters actifs est < 10, le dire et donner ce qu'il y a.
3. **`volume_l` peut n'avoir qu'1 enregistrement/jour pour certains clients** (usage de test/pilote)
   — ne pas confondre avec un vrai flux production.
4. **Le dernier jour disponible est souvent incomplet** (journée en cours) — pour une comparaison
   jour/jour ou mois/mois, utiliser le dernier jour *complet*, pas `now()`.
5. **Des ratios extrêmes (x50, x100...) viennent souvent d'une base de comparaison quasi nulle**
   (ex: 6L -> 1500L). Toujours filtrer sur un seuil minimum de consommation de référence
   (`vA > seuil`) avant de calculer un ratio, sinon le "top 10 par ratio" est dominé par du bruit
   statistique plutôt que par de vraies fuites.
6. **Un ratio x2-x3 sur un très large ensemble de compteurs (dizaines de milliers) est normal** —
   la variance naturelle de consommation fait qu'énormément de compteurs passent dans cette
   fourchette par hasard. Pour prioriser, trier par **volume perdu en absolu** (`vB - vA`), pas
   par ratio seul, sauf si le nombre de résultats est déjà petit.
7. **Le diamètre du compteur (`meterDiameter`) change l'interprétation** : un doublement sur un
   DN15 résidentiel n'a pas le même enjeu qu'un doublement sur un DN100+ (collectif/industriel).
   Toujours croiser avec `meterDiameter` avant de qualifier la sévérité.
8. **Un compteur récemment installé** (`installationDate` < 30 jours) qui "double" sa conso est
   probablement juste une montée en charge normale, pas une fuite — à exclure ou signaler à part.
9. **`data_only`/formats de date** : les timestamps sont en UTC (`...T00:00:00Z` pour `valueDate`,
   souvent `...T22:00:00Z` ou `23:00:00Z` pour `eventDate` des alarmes — décalage horaire probable,
   ne pas s'étonner de ces heures atypiques).
10. **Un "compteur silencieux" peut en réalité être un remplacement de compteur/transmitter, pas une
    panne.** La table `module` (comme `meter`) peut avoir **plusieurs lignes par `servicePoint`** :
    un ancien `transmitter` avec `removalDate` renseignée, et un nouveau `transmitter` avec une
    `installationDate` récente. Si on ne prend que l'ancien transmitter (ex: via `any()` ou en
    ignorant `removalDate`), on qualifie à tort de "silencieux" un point de service qui continue en
    réalité de remonter des données sous un nouvel identifiant radio. **Avant de conclure qu'un
    servicePoint est silencieux**, vérifier dans `module` (et `meter`) s'il existe un second
    enregistrement pour le même `servicePoint` avec une `installationDate` postérieure à la date
    du dernier signal de l'ancien transmitter — si oui, chercher les données `msr` de ce nouveau
    transmitter avant d'inclure le point dans une liste de compteurs muets. Filtrer idéalement sur
    `removalDate` étant vide (`isempty(removalDate)`) pour ne garder que les transmitters
    actuellement en service dans l'analyse de silence.
10. **`meter_park` peut avoir plusieurs lignes par `servicePoint`** (comme `meter` — historique de
    remplacement de compteur physique, plusieurs `sensorSerialNumber` pour un même point de service).
    Une jointure sur `servicePoint` produit donc des doublons de conso/alarmes. Toujours dédupliquer
    après jointure, par exemple avec `summarize ... by servicePoint, transmitter` ou `distinct
    servicePoint` en aval, avant de compter ou sommer au niveau service point.
11. **Un compteur à usage non-résidentiel (`servicePointUsageLabel` = `Incendie`, `Vert`, `Chantier`...)
    n'est normalement pas censé avoir de consommation régulière**, contrairement à un usage `Normal`.
    Pour ces usages, la présence même d'une consommation sur une période donnée est déjà un signal à
    investiguer (fuite, fraude, usage légitime ponctuel — pompiers, arrosage, chantier) : pas besoin de
    comparer deux périodes comme pour la méthodologie fuite classique (§ Méthodologie standard), un
    simple filtre `conso > 0` sur la période suffit à isoler les cas à vérifier.

## Méthodologie standard : détecter des fuites / dérives de consommation

Suivre cette checklist dans l'ordre pour toute demande du type "identifie les compteurs avec une
consommation anormale" :

1. **Vérifier la disponibilité des données** pour le client demandé (`msr | where client == "X" |
   count`, `distinct transmitter | count`). Si les données sont absentes ou très limitées, le dire
   immédiatement plutôt que de forcer une analyse peu fiable.
2. **Définir la fenêtre de comparaison** selon la demande utilisateur :
   - "2 derniers jours" → comparer le dernier jour complet vs l'avant-dernier
   - "2 derniers mois" → comparer 2 fenêtres glissantes de 30 jours consécutives
   - Toujours préciser les dates exactes utilisées dans la réponse (transparence sur l'hypothèse)
3. **Calculer** `vA` (période précédente) et `vB` (période récente) par `transmitter`, avec
   `code == "volume_l"`.
4. **Filtrer le bruit** : `vA > seuil_minimum` (ex: 500L pour du résidentiel, plus pour du
   collectif) pour éviter les ratios artificiels sur des bases quasi nulles.
5. **Prioriser par volume perdu absolu** (`vB - vA`) plutôt que par ratio seul si le nombre de
   résultats dans la fourchette demandée est grand (> quelques dizaines).
6. **Enrichir avec les alarmes** : croiser avec `alr` (codes `Leak_*` et `BlockedMeter*` a minima)
   sur la même période ou en historique complet, pour confirmer/expliquer la dérive.
7. **Enrichir avec le débit nocturne** (`night_flow_*`) quand disponible pour fiabiliser un
   diagnostic de fuite continue (à proposer si pas explicitement demandé, car plus fiable qu'une
   simple comparaison de volumes).
8. **Croiser avec `meterDiameter`** pour qualifier la sévérité (résidentiel vs collectif/industriel).
9. **Exclure ou signaler séparément** les compteurs récemment installés (< 30 jours).
10. Présenter les résultats avec : `servicePoint`, `transmitter`, adresse (si disponible),
    diamètre, volumes des 2 périodes, ratio, volume perdu, statut alarme fuite, statut alarme
    compteur bloqué (historique complet, pas seulement sur la fenêtre récente — un antécédent
    ancien reste une information utile).

## Méthodologie standard : détecter des compteurs silencieux

Suivre cette checklist pour toute demande du type "quels compteurs sont devenus silencieux" :

1. **Identifier le dernier jour complet** de données (`msr | where client == "X" | summarize nb=dcount(transmitter) by valueDate` — le dernier jour a typiquement un volume nettement inférieur, signe qu'il est partiel).
2. **Repérer les transmitters dont la dernière donnée `volume_l`** tombe dans la fenêtre de silence voulue (ex: entre 7 et 30 jours avant le dernier jour complet), avec un historique fiable préalable (`nbDays > 30` par ex., cf piège #1/#2).
3. **Avant de livrer la liste, exclure les remplacements de compteur/transmitter** (piège #10) :
   pour chaque `servicePoint` candidat, vérifier dans `module` s'il existe un autre `transmitter`
   avec une `installationDate` postérieure à la dernière donnée de l'ancien — si oui, chercher ses
   données `msr` : s'il en a de récentes, ce n'est **pas** un compteur silencieux, c'est un
   remplacement réussi, à retirer de la liste.
4. **Sur les gros clients** (> quelques centaines de milliers de transmitters, ex: LYONNAISE),
   `join`/`summarize` sur la table `msr` complète peut dépasser le budget mémoire ADX
   (`E_RUNAWAY_QUERY`). Filtrer d'abord `msr` avec `transmitter in (candList)` où `candList` est
   une liste précalculée et réduite de candidats (via `toscalar(... | summarize make_list(...))`),
   plutôt que de joindre directement sur la table complète.
5. **Enrichir avec le patrimoine** (adresse, diamètre) et les alarmes batterie faible en historique
   complet — sur les jeux de données observés, la quasi-totalité des silences ne sont **pas**
   précédés d'une alarme batterie, ce qui pointe vers un problème radio/transmetteur plutôt qu'une
   pile en fin de vie.
6. **Prioriser par enjeu** : un gros diamètre (collectif/industriel) qui devient silencieux est plus
   critique qu'un DN15 résidentiel, même si le nombre de jours de silence est identique.

## Méthodologie : détecter des compteurs bloqués

⚠️ La vraie règle officielle est un algo ML Synapse (package `synapse_blockedmtr`,
alarmes `BlockedMeterLevel1-4` dans `alr`). **Si ce code est accessible
(wheels/pkl fournis par l'utilisateur), le lire en premier** — il a permis de
diviser par 3 le taux de faux négatifs de la méthode heuristique ci-dessous
(voir `references/blockedmeter-source-analysis.md` pour l'analyse complète
du code source et tous les chiffres de validation). Cette section documente
la **méthode de substitution** (signal brut, sans le vrai modèle ML) à
utiliser quand le score officiel n'est pas exposé/accessible.

**Correction importante sur le sens des niveaux** : `BlockedMeterLevel1` =
risque **le plus fort** (score>0.4), `Level4` = risque le plus faible
(score≤0.15) — c'est l'inverse de ce qu'on pourrait intuiter. Confirmé à la
fois dans le code (`model_prediction.py`, fonction `risk_category`) et dans
`onconnect-business-rules`.

**Piège n°11 — index figé ≠ compteur bloqué.** Un index strictement identique
pendant plusieurs semaines ne prouve qu'une chose : zéro consommation sur la
période. Ce signal est **rigoureusement identique** pour un vrai compteur en
panne ET pour un logement réellement inoccupé (vacance, déménagement,
résidence secondaire). Ne jamais présenter un index figé seul comme un
"compteur bloqué" — c'est un candidat, pas une conclusion.

**Piège n°12 — `index_24h_l` seul rate ~75% des cas.** L'algo officiel calcule
la dernière lecture valide sur **5 granularités d'index en parallèle** :
`index_15m_l`, `index_1h_l`, `index_24h_l`, `index_5m_l`, `index_6h_l`. Sur
LYONNAISE, 35 140 des 49 570 compteurs en alarme officielle (75%) n'avaient
**aucune** donnée `index_24h_l` mais remontaient bien une autre granularité.
Toujours interroger `code in (indexCodes)` avec les 5 codes, jamais un seul.

**Piège n°13 — deux nomenclatures d'alarme coexistent pour "fraude/fixation".**
Le format structuré (`Fraud_PermanentUnclippingAlert`, `Fraud_FuzzyUnclippingAlert`
— ce sont exactement les codes utilisés par l'algo officiel, cf `ALR_CODE` dans
le code source) coexiste avec un ancien format générique (`fraud`, minuscule,
sans détail dans `value`). Exclure les deux à défaut de savoir laquelle
s'applique à un transmitter donné — l'ancien format n'est pas utilisé par
l'algo officiel mais reste un bon signal de fiabilité des données.
Piège voisin déjà connu : `OtherDefect_TransmitterMisalignment` /
`OtherDefect_MeterMisalignment` (désalignement, concept différent du
décrochage/fixation) — ces codes ont cessé d'être générés après le
01/04/2025 sur LYONNAISE ; vérifier si c'est encore le cas avant de s'y fier.

**Piège n°14 — la fenêtre d'exclusion "fraude récente" dépend du diamètre.**
L'algo officiel n'exclut pas un transmitter ayant *un jour* eu une alarme
fraude, mais seulement s'il y en a eu une **récemment** : 60 jours de
lookback pour DN<25mm, **7 jours seulement** pour DN≥25mm. Une exclusion
uniforme "à tout moment de l'historique" est trop agressive, surtout sur les
gros compteurs (DN≥25) où une fraude ancienne ne doit plus disqualifier un
vrai blocage récent.

### Méthode v3 (recommandée) — index multi-codes + seuils diamètre-dépendants

```kql
let ref = datetime(...);                    // dernier jour complet
let indexCodes = dynamic(["index_15m_l","index_1h_l","index_24h_l","index_5m_l","index_6h_l"]);
let diametres = meter | where client == "X" and isempty(removalDate) | summarize meterDiameter=max(meterDiameter) by servicePoint;
let modLink = module | where client == "X" and isempty(removalDate) | project transmitter, servicePoint;
let transDiam = modLink | join kind=inner diametres on servicePoint | project transmitter, meterDiameter;
let fenetreMax = ref - 90d;
// 1. Index figé, tous codes confondus
let bloques = msr
| where client == "X" and code in (indexCodes) and valueDate >= fenetreMax and valueDate <= ref
| summarize indexMin=min(value), indexMax=max(value), nbJours=dcount(valueDate) by transmitter
| where (indexMax - indexMin) == 0;
// 2. Seuil d'historique dépendant du diamètre (7j gros compteurs, 60j petits — valeurs officielles)
let avecDiam = bloques | join kind=inner transDiam on transmitter
| extend seuilJours = iif(meterDiameter >= 25, 7, 60) | where nbJours > seuilJours;
// 3. Exclusions : fraude (fenêtre dépendante du diamètre, codes officiels) + gel (garde-fou data-quality)
let fraudeGrandDN = alr | where client == "X" and code in ("Fraud_PermanentUnclippingAlert","Fraud_FuzzyUnclippingAlert") and eventDate >= ref - 7d | distinct transmitter;
let fraudePetitDN = alr | where client == "X" and code in ("Fraud_PermanentUnclippingAlert","Fraud_FuzzyUnclippingAlert") and eventDate >= ref - 60d | distinct transmitter;
let gel = alr | where client == "X" and code has "Freeze" and eventDate >= fenetreMax | distinct transmitter;
let candidats = avecDiam
| join kind=leftouter (fraudeGrandDN | extend exGrand=true) on transmitter
| join kind=leftouter (fraudePetitDN | extend exPetit=true) on transmitter
| join kind=leftouter (gel | extend exGel=true) on transmitter
| where not((meterDiameter >= 25 and isnotempty(exGrand)) or (meterDiameter < 25 and isnotempty(exPetit)) or isnotempty(exGel))
| project transmitter, meterDiameter;
// 4. Consommation antérieure significative (filtre anti-"déjà vacant de longue date")
let candList = toscalar(candidats | project transmitter | summarize make_list(transmitter));
let avantGel = msr | where client == "X" and code == "volume_l" and transmitter in (candList) and valueDate >= fenetreMax - 365d and valueDate < fenetreMax | summarize volAvant = sum(value) by transmitter;
candidats | join kind=inner avantGel on transmitter | where volAvant > 500
```

**Validation croisée sur LYONNAISE** (référence pour juger un futur résultat) :

| Version | Candidats | Confirmés par `BlockedMeterLevel` officiel | Couverture Liste B (49 570) |
|---|---|---|---|
| `index_24h_l` seul | 2 856 | 79% | 4,6% |
| + 5 codes d'index | 7 839 | 82% | 13% |
| **v3 (ci-dessus, complète)** | **9 664** | 79% global, **84% sur DN≥25mm** | **15,3%** |

### Limite structurelle qui ne se résout pas avec un seuil absolu

L'écart restant (~85% de la Liste B non couvert) vient très majoritairement de
compteurs dont l'index **varie** mais reste anormalement bas — pas un vrai
signal "figé". Le code officiel définit un jour "significatif" comme
**≥20L ET au moins un jour adjacent aussi ≥20L** (2 jours consécutifs),
et calcule `T0` = date de la dernière période significative. Un critère
expérimental basé sur ce `T0` (transmitter sans aucun `T0` sur 90j = candidat)
a permis de retrouver **13 228 candidats supplémentaires** sur la population
"index non figé" (53,5% de cette sous-population), avec un **taux de faux
positifs mesuré de 8,8%** sur un échantillon de compteurs hors Liste B —
signal réel mais imparfait. Voir `references/blockedmeter-source-analysis.md`
pour le détail du calcul T0 (jointure décalée d'un jour, pas de `prev()`/
`next()` — plus stable sur ce connecteur) et sa méthode de validation.

**Ce qui reste hors de portée sans réimplémenter le modèle ML complet** :
les features V2 à V12 (ratios, durées de périodes non significatives),
la matrice de priorité finale (type de consommateur DN×index cumulé ×
niveau de risque → priorité 0-5), et la calibration du score
(`codai_calibration`, technique générique de calibration de probabilité,
peu de valeur métier directe).

**Limite persistante même après tout affinage** : un vrai déménagement récent
produit le même signal qu'une panne. Seul un statut d'occupation officiel
externe (ex: tag `conso log vacant en cours` vu côté LYONNAISE) permet de
trancher avec certitude. Présenter systématiquement le résultat comme une
liste de candidats à vérifier, jamais comme des pannes confirmées.

## Export Excel — format standard

Quand un export terrain est demandé, suivre le gabarit déjà validé (voir
`references/excel_template_notes.md`) :
- Titre + sous-titre expliquant la méthodologie (période comparée, source des données)
- Colonnes : rang, client, servicePoint, transmitter, sensor actif, diamètre, adresse, secteur
  hydraulique, conso période N-1, conso période N, ratio (**formule Excel**, pas une valeur
  codée en dur), volume perdu (**formule**), nb alarmes fuite, 1ère/dernière alarme fuite,
  statut compteur bloqué
- Ligne de total en bas (sommes en formules)
- Code couleur : rouge = plus fort volume perdu, orange = antécédent de blocage, vert = pas
  d'antécédent
- Toujours lancer `recalc.py` après génération (voir skill `xlsx` pour le détail des règles
  de formules et de formatage)

## Liste des clients connus avec données de mesure (`msr`)

AGUR, BRECLAV, BRNO, CAEN, CAEPERNAY, CESKY STERBERK, CHOCEN, DEMONSTRATION, DRAHNOVICE, EBER,
EVREUX_PDN, FOUILLOUSE, GDPARISSUD, JABLONNE, LANŠKROUN, LEMANSMETRO (données très limitées, cf.
piège #2), LESMUREAUX, LETOHRAD, LE_HAVRE, LIANCOURTOIS, LIBEŽ, LIBOURNE, LYONNAISE, MAROLLESCRESNES,
MARTIGUES, MONTDEMARSAN, MULHOUSE2A, OCEA, OMFLEURY, OSTRAVA, PILOTESV2, POLYNESIE, PYŠELY,
RAMBOUILLET, REBM, RESO, SAT_32, SENEAU, SEOE, SIAEP_BRAY, SIDERM, SIEVA, STETIENNE, STGALMIER,
SWS, TEHOV, UNITYWATER, USTI, VBG, VHS BENEŠOV, VYVS.

⚠️ Cette liste peut évoluer — si un client demandé n'y figure pas (ex: MALTA, présent dans `meter`
mais pas dans `msr`), le vérifier avec une requête plutôt que de supposer.

Si un nouveau client ou une nouvelle particularité est découvert(e) en session, envisager de
mettre à jour cette skill pour capitaliser sur la découverte future.
