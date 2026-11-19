# Règles métier OnConnect — Télérelève eau (snapshot)

> Snapshot offline de la page wiki Azure DevOps `On-Connect.wiki` — `/Teams/Metering/Règles métier - Télérelève eau` (page id 10914), capturé le 2026-08-15. **La page wiki est la source de vérité** ; ce fichier peut être en retard. En cas de doute, relire la page live via `azure-devops/wiki` (`get_page_content`).

## 1. Horodatage des mesures — "Value Date"

`Value_Date = Jour local (horodatage UTC de la mesure − 12h)`

- 2G / 1G en mode 1G : mesure reçue entre 0h et 5h (heure locale) → rattachée à la veille.
- 1G : mesure reçue le matin → veille ; après-midi → jour même.
- G3BRIDZ : pas de correction, l'horodatage du débit est déjà porté par la trame.
- S'applique au volume permanent de fuite et au volume de retour d'eau journaliers.

**Convention en l'absence d'horodatage dans la trame** (WI 194619, 198157) : la mesure/alarme est horodatée par convention à l'HR (heure de référence) de fin de période de mesure, en sortie du MDC. Une table croise, par type d'équipement/trame/destinataire (BPS ou MDM) : composant de mesure, source dans la trame, horodatage trame, horodatage mesure dans la trame (si existe), horodatage assigné par convention (si absent). C'est l'origine du décalage d'un jour corrigé par la règle Value Date (incident source : WI 187983).

Propagation identique dans : BPS/EMTD (WI 210736), Streamset (WI 210603), BAPS (WI 210986), Supervision (WI 210738, 218097), Supervision SaaS V2 (WI 218919), impact OCR (WI 218140). Feature portante : WI 197337.

## 2. Classification saisonnière / compteur bloqué (règle historique)

Source : WI 192671

`Ratio = Conso moyenne été (Juin-Août) / Conso moyenne hiver (Décembre-Février)`, sur 18 mois glissants.

- Ratio ≥ 2 → logement saisonnier d'été
- Ratio ≤ 0,5 → logement saisonnier d'hiver
- 0,5 < ratio < 2 → résidence principale
- Conso = 0 toutes saisons → compteur bloqué

Cas limites : pas de données sur 3 saisons → "pas de données saisonnières" ; conso nulle sur une saison et non nulle 2× sur l'autre → classement saisonnier de la saison à conso nulle.

⚠️ Détection "compteur bloqué" remplacée en production par le nouvel algo ML (§7).

## 3. Fuites

### 3.1 Table de production (WI 178492, atelier source WI 180055)

| Alarme | Règle |
|---|---|
| Fuite importante | ≥10 500 L/semaine (≈1 500 L/j) ET >30% conso hebdo |
| Fuite modérée | <10 500 L/semaine OU ≤30% conso hebdo |
| Fixation franche | Alarme dans toutes les trames ET conso nulle le jour J |
| Fixation non franche | Alarme non présente dans toutes les trames OU conso non nulle le jour J |
| Micro retour d'eau | Volume journalier <4 L |
| Retour d'eau modéré | Volume journalier 4-150 L |
| Retour d'eau important | Volume journalier >150 L |
| Compteur à l'envers | Tous volumes 7j négatifs et >300 L/semaine |
| Vérifier émetteur | Alarme fixation + retour d'eau/conso négative le jour J (logique différente ITRON 1G vs 2G en mode 1G) |

Alarme affichée le jour J si Qmin non nul le jour J (sans rémanence).

### 3.2 Étude d'harmonisation LEM (WI 218044 lot1, 218045 lot2, 219615 MVP, 196649/211994 Enabler Feature)

- Sévérité Micro/Modérée/Importante — seuils en cours d'harmonisation avec §3.1.
- Type de fuite : constante (chasse d'eau, ballon ECS, robinet/mitigeur) / variable (joint chasse d'eau) / en augmentation (canalisation).
- Cycle de vie `LeakStatus` : New (1er jour) → Active → Closed (dès qu'à une journée donnée tous les Qmin = 0).
- Gestion des trous de données : fuite toujours présente après un trou → même fuite.
- Petites fuites (micro/modérées) : sortie de l'état dès Qmin nul.
- Fuite importante réparée + fuite faible résiduelle : comptabilisées séparément.

### 3.3 Formules

- `Leak % = (leakVolume_l / volume_l) × 100` (WI 190593) — pas de calcul si l'un des deux termes est absent ou nul.
- Volume total de fuite depuis le début affiché dans le rapport Alarmes Calculées (WI 207085, 209036).
- Interpolation Qmin (WI 217708) : `Volume hors fuite = Volume total consommé − Volume de fuite`. Résolution journalière : Qmin5 et Qmin15 présents → `min(Qmin5+8, Qmin15)` ; un seul → cette valeur ; aucun → NULL. Interpolation si NULL : moyenne temporelle pondérée entre dernier Qmin valide (7j avant) et premier Qmin valide (7j après), seulement si les deux ancres existent et sont >0. Priorité finale : mesuré > interpolé > NULL.

## 4. Alarme GEL (WI 189724)

- Problème : DIEHL ne déclenche que sur A1/R3 (pas R1 T<3°, ni R2 T<-2°) ; ITRON déclenche dès R1 (bruit).
- Nouvelle règle : alarme "risque de gel" si bit non nul vecteur d'état OU octet non nul trame PSDV OU alarme gel — seuil le plus élevé retenu (R1/R2/R3). Une seule alerte affichée, avec seuil dépassé indiqué pour priorisation (R3 > R2 > R1).
- R3 = température ≤-2°C ET absence de consommation pendant X jours.

## 5. Filtrage de données aberrantes

### 5.1 WRTP (WI 194618, 195649, 200835, 201567)

Trame = 2 séries de 12 valeurs compressées sur 1 octet.
- Règle 1 : valeur compressée ≤-125 (0x81/82/83) ET une autre ≥+126 (0x7E/7F) dans la même série → rejet.
- Règle 2 (zone 0H UTC — UK, Portugal) : index absolu (CI)=0 OU (CI+DCI12h)=0 OU (CI+DCI12h+DCI24h)=0 → rejet.
- Trames rejetées archivées dans un log (id trame, horodatage réception, motif).

### 5.2 Rendement Réseau (WI 193443)

`Rendement brut = Volume consommé / Volume Mis en Distribution`

| Catégorie | Traitement |
|---|---|
| Télérelevés validés | Exact ou interpolé |
| Non valides à inclure (données manquantes, conso négative, conso sous alarme fixation, données aberrantes, compteurs bloqués) | Extrapolation |
| Non valides à exclure (compteurs divisionnaires, changement patrimoine) | Exclusion par filtres |
| Non télérelevés (relève manuelle) | Approche CMJ ou Data Science |

Voir aussi WI 190297/198141 (flux ONC↔AAWN), 185698 (VLAR), 193369 (REBM).

## 6. Nouvel algo ML "compteur bloqué" (WI 215020, design review wiki page 10402)

Calcul dans Synapse par algo ML (équipe Data Suez, validé PMM avril 2026), publié vers SITR/ON'Connect/ON'Connect Revenue.

| Code | Catégorie |
|---|---|
| BlockedMeterLevel1 | Risque fort |
| BlockedMeterLevel2 | Risque moyen |
| BlockedMeterLevel3 | Risque faible |
| BlockedMeterLevel4 | Risque très faible |

Type de consommateur (DN × index cumulé) :

| DN\Index | <2M L | 2M-10M L | ≥10M L |
|---|---|---|---|
| DN≤15 | Small | Small | Small |
| 15<DN≤20 | Small | Medium | Medium |
| DN>20 | Small | Medium | Large |

Priorité d'intervention (type × risque, 0-5) :

| Type\Risque | Lvl4 | Lvl3 | Lvl2 | Lvl1 |
|---|---|---|---|---|
| Small | 0 | 1 | 2 | 3 |
| Medium | 0 | 2 | 3 | 4 |
| Large | 0 | 3 | 4 | 5 |

## 7. Décodage de trames radio

- Maddalena ArrowWAN (WI 218021) : trame brute → données métier (id compteur, volumes selon unité VIF).
- ITRON Cyble5 (WI 218005, 218008) : POST Docker constructeur avec trame + clés ; retour index/alarmes/fuite/débit/retour d'eau/horodatage ; mapping Metering sans perte de précision.
- IZAR RC vs G3Birdz (WI 219213) : reconnaissance par type ET longueur de trame (`G3B_DS51_A`=10, `LRZ102_DS40_OQ`=9).
- DS40_OQ/IzarRC (WI 213458) : trame identifiée si bits 15-10 des 2 premiers octets = 9 ; Little Endian ; 1ère trame le 2e jour civil après connexion.
- WRT-AL trame 56 (WI 183304) : identiques si 48 premiers caractères égaux ; 54e caractère 0=originale/2=redondante ; originale toujours transmise, redondante seulement si originale non reçue.
- PSDV 2G Wize gel (WI 191914) : alarme Freeze selon bits R1/R2/R3 du champ FreezeValue.
- MBUS (WI 191221) : header d'identification, contrat `Frame`, index via `FormatFrames`.

## 8. Alarme Clock shift (WI 201584)

Détecte blocage d'horloge quartz émetteur (index figé, conso calculée à tort nulle pendant l'arrêt).

## 9. Clôture d'alarme (WI 210632)

L'absence de données ne doit pas clôturer une alarme en cours (événement générateur peut-être toujours actif).

## Points à arbitrer

- Seuils de fuite : production (WI 178492) vs étude LEM (WI 218044/218045).
- Compteur bloqué : ancienne règle ratio (WI 192671) vs nouvel algo ML (WI 215020).

## Périmètre de l'exploration source

Recherches texte + WIQL sur OnConnect, restreintes aux types User Story/Enabler Story/Feature/Enabler Feature (Risque/Objectives/Incident/Epic/Bug exclus). Thèmes couverts : fuites/alarmes/retour d'eau, horodatage/Value Date, filtrage données aberrantes, décodage trames radio, Rendement Réseau. Non exhaustif (711 items au total correspondent à un filtre mots-clés large sur ces 4 types).
