# Gabarit Excel — export "compteurs à risque"

Basé sur le fichier `Fuites_LYONNAISE_top10.xlsx` généré et validé en session.

## Structure de la feuille

- Ligne 1 : titre fusionné (fond bleu foncé `1F4E78`, texte blanc, gras, taille 14)
- Ligne 2 : sous-titre — période comparée + source des données (table/code utilisé)
- Ligne 3 : date de génération + critère de tri utilisé
- Ligne 5 : en-têtes de colonnes (fond bleu `2E75B6`, texte blanc, gras, `wrap_text`)
- Lignes 6+ : données, une ligne par compteur
- Ligne de total juste après la dernière ligne de données (fond `D9E1F2`)
- Bloc légende sous le total (couleurs utilisées et leur signification)

## Colonnes standard (dans l'ordre)

1. Rang
2. Client
3. servicePoint
4. Transmitter
5. Sensor (compteur actif)
6. Diamètre (mm)
7. Adresse
8. Secteur hydraulique
9. Conso période N-1 (L)
10. Conso période N (L)
11. Ratio (x) — **formule** `=J{r}/I{r}`, format `0.00"x"`
12. Volume perdu (L) — **formule** `=J{r}-I{r}`, format `#,##0`
13. Alarmes fuite (nb)
14. 1ère alarme fuite
15. Dernière alarme fuite
16. Alarme compteur bloqué ? (texte descriptif + couleur)

## Règles de formules (rappel skill xlsx)

- Toujours écrire les ratios/deltas en formule Excel, jamais en valeur Python codée en dur —
  le fichier doit se recalculer si les volumes bruts sont corrigés.
- Lancer `python scripts/recalc.py fichier.xlsx` après génération et vérifier
  `"status": "success"` avant de livrer.
- Police Arial partout, tailles 10 (corps) / 14 (titre).

## Code couleur

- **Rouge** (`FFC7CE` fond / `9C0006` texte) : plus fort volume perdu (typiquement top 2 de la liste)
- **Orange** (`FFEB9C` / `9C6500`) : antécédent d'alarme "compteur bloqué" (historique complet)
- **Vert** (`C6EFCE` / `006100`) : pas d'antécédent de blocage

## Bordures et largeurs

- Bordure fine grise (`BFBFBF`) sur toutes les cellules de données et la ligne de total
- Largeurs de colonnes ajustées au contenu (adresse ~40, secteur ~26, dates ~14, reste 9-18)
- `freeze_panes = "A6"` pour garder les en-têtes visibles au scroll
