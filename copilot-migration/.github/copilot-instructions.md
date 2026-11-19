# Instructions du dépôt — OnConnect / OCR (télérelève d'eau, Suez Smart Solutions)

Base de données : Azure Data Explorer, cluster `adxoptirevenue.northeurope.kusto.windows.net`, base `onconnect`.
Projet Azure DevOps : organisation `suezsmartsolutions`, projet `OnConnect`.

## Pipeline d'analyse ADX

Pour toute question d'analyse sur les compteurs d'eau nécessitant une règle métier (seuil, statut, définition), suivre systématiquement cette séquence :

1. `devops-rules` — confirme la règle métier applicable
2. `adx-query` — exécute l'analyse avec cette règle confirmée
3. `adx-validator` — valide la cohérence du résultat avant livraison

Ne jamais sauter l'étape 1 si la question implique un seuil ou une définition non triviale (fuite, compteur bloqué, statut, etc.).

## Skills à consulter
- `adx-water-leak-analysis` : schéma ADX, pièges connus, méthodes (silencieux, bloqués, fuites). À lire avant toute requête KQL sur meter/msr/alr/module/eqp.
- `onconnect-business-rules` : index des règles métier et points d'arbitrage ouverts.

## Règles essentielles (rappel)
- Le dernier jour de données est incomplet : l'exclure.
- Compteur silencieux = ni consommation ni index sur TOUS les codes `msr`, et pas de remplacement de transmitter (`removalDate`/émetteur plus récent).
- KQL : `has` n'est pas une recherche de sous-chaîne, utiliser `contains`.
- Index : utiliser les 5 codes (`index_5m_l`, `index_15m_l`, `index_1h_l`, `index_6h_l`, `index_24h_l`), pas seulement `index_24h_l`.
- Deux nomenclatures d'alarmes fraude/fixation (`Fraud_PermanentUnclippingAlert`, `Fraud_FuzzyUnclippingAlert`, ancien code `fraud`).
- Fenêtre d'exclusion fraude : 7 jours si DN ≥ 25 mm, 60 jours si DN < 25 mm.
- Index figé ≠ bloqué (un logement vacant donne le même signal).
- La qualité des données varie par client (`module` riche sur SWS, vide sur LYONNAISE ; `meter_park` rempli pour LYONNAISE uniquement).
- Ne jamais présenter un motif statistique comme une règle : le formuler comme « HYPOTHÈSE À VALIDER PAR UN HUMAIN ».
- Accès ADX en lecture seule ; ne jamais écrire dans le wiki sans demande explicite.
