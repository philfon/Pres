---
name: onconnect-business-rules
description: This skill should be used when the user asks about "règles métier OnConnect", "règles métier télérelève", "business rules OnConnect", water smart metering business rules for Suez, or needs to explore Azure DevOps work items in the "OnConnect" project (organization suezsmartsolutions) to find or verify metering business logic (leak thresholds, Value Date/timestamp convention, seasonal classification, blocked meter detection, frame decoding rules, alarm rules). Also applies when asked to update the OnConnect wiki page documenting these rules.
version: 1.0.0
---

# OnConnect — Water Metering Business Rules

This skill packages the business-rule knowledge and exploration method built for the Azure DevOps project **OnConnect** (organization `suezsmartsolutions`), domain: water remote-metering (télérelève d'eau) for Suez.

## Source of Truth

The canonical, maintained synthesis lives in the Azure DevOps wiki, not in this skill:

- Wiki: `On-Connect.wiki`
- Page path: `/Teams/Metering/Règles métier - Télérelève eau`
- Page id: `10914`
- URL: https://suezsmartsolutions.visualstudio.com/OnConnect/_wiki/wikis/On-Connect.wiki?pagePath=%2FTeams%2FMetering%2FR%C3%A8gles%20m%C3%A9tier%20-%20T%C3%A9l%C3%A9rel%C3%A8ve%20eau

Fetch this page (`mcp__azure-devops__wiki` action `get_page_content`) before answering a precise question about a rule, since it may have been updated since this skill was written. Use `references/regles-metier-snapshot.md` in this skill only as an offline fallback or quick-scan index — treat it as a snapshot, not the live truth.

## Known Business Rules (condensed index)

1. **Value Date / horodatage** — `Value_Date = jour local (horodatage UTC − 12h)`, avec règles spécifiques 1G/2G/G3BRIDZ et convention MDC (HR de fin de période) quand la trame ne porte pas d'horodatage propre.
2. **Classification saisonnière / compteur bloqué (ancienne règle)** — ratio conso été/hiver sur 18 mois glissants (ratio ≥2 → saisonnier été, ≤0,5 → saisonnier hiver, sinon résidence principale ; conso nulle toutes saisons → bloqué).
3. **Fuites** — table de seuils de production (fuite importante ≥10 500 L/semaine ET >30% conso hebdo) **vs** étude d'harmonisation LEM (Leak Event Manager) en cours avec catégorisation Micro/Modérée/Importante et cycle de vie New/Active/Closed.
4. **Alarme GEL** — règle unifiée : alarme dès bit non nul dans le vecteur d'état OU octet non nul en trame PSDV OU alarme gel, avec seuil R1/R2/R3 le plus haut retenu.
5. **Filtrage d'index aberrants WRTP** — rejet de trame si valeurs compressées ≤-125 ET ≥+126 dans la même série, ou si index absolu/intermédiaire = 0 (zone 0H UTC).
6. **Nettoyage Rendement Réseau** — catégorisation des compteurs (validés / à extrapoler / à exclure / non télérelevés) pour `Rendement = Volume consommé / Volume Mis en Distribution`.
7. **Nouvel algo ML "compteur bloqué" (Synapse)** — remplace la règle n°2 en production : score de risque (BlockedMeterLevel1-4) × type de consommateur (Small/Medium/Large selon DN et index) → priorité d'intervention 0-5.
8. **Décodage de trames radio** — règles spécifiques par constructeur/protocole : Maddalena ArrowWAN, ITRON Cyble5 (Docker transcoder), IZAR RC vs G3Birdz (type + longueur de trame), WRT-AL trame 56 (dé-redondance), PSDV 2G Wize (alarme gel bit R1/R2/R3), MBUS.
9. **Alarme Clock shift** — détecte un blocage d'horloge quartz émetteur (index figé, conso à tort nulle).
10. **Clôture d'alarme** — l'absence de données ne doit jamais clôturer une alarme en cours.

See `references/regles-metier-snapshot.md` for the full detail (formulas, thresholds tables, source work item IDs) as captured when this skill was created.

## Known Open Arbitration Points

Flag these explicitly if relevant to the user's question — they are not resolved:

- **Seuils de fuite** : table de production (WI 178492) vs étude LEM (WI 218044 / 218045) ne portent pas les mêmes seuils.
- **Détection compteur bloqué** : coexistence de l'ancienne règle par ratio (WI 192671) et du nouvel algo ML Synapse (WI 215020) — vérifier laquelle fait référence selon le flux (SITR / OCR / ON'Connect).

## Re-exploring Azure DevOps (when a rule needs verification or deepening)

Use the `azure-devops` MCP server, project `OnConnect`, organization `suezsmartsolutions`.

**Scope work item types** to `User Story`, `Enabler Story`, `Feature`, `Enabler Feature` — these carry explicit "Règles de gestion" sections. Exclude `Risque`, `Objectives`, `Incident`, `Epic`, `Bug`: they are far less likely to state business rules directly and inflate result volume without adding rule content.

**Tools**:
- `mcp__azure-devops__search_workitem` — free-text search with ranked relevance and highlighted snippets. Best first step for a new theme. **Do not set `includeFacets: true`** — it can multiply the response size by 5-10x and blow past the token limit even for a handful of results. Keep `top` around 10-20.
- `mcp__azure-devops__wit_query` (`action: "wiql"`) — structured queries, e.g. to enumerate candidates by type + title keywords. Prefer `responseType: "ids"` to keep payloads small, and filter by *narrow* theme keywords (e.g. `fuite`, `leak`, `alarme` together) rather than a broad keyword union — a broad union across all metering vocabulary returned 700+ matches project-wide, which is not triageable by hand. Date filters do not help here: the project's work items are almost entirely from 2024 onward already.
- `mcp__azure-devops__wit_work_item` (`action: "get_batch"`) — fetch `System.Id`, `System.Title`, `System.WorkItemType` first (cheap) to triage candidates by title before pulling `System.Description` for only the promising ones (a handful at a time — batches of 10 full descriptions can already exceed the token limit and get redirected to a saved file).

**When a tool result is too large**: it gets saved to a local file and the response tells you the path. Prefer `Grep` with `output_mode: "count"` or targeted patterns over reading the whole file sequentially — raw HTML description blobs are often single very long lines that `Read`'s offset/limit chunking cannot usefully page through.

## Updating the Wiki Page

To publish new or corrected rules, use `mcp__azure-devops__wiki_upsert_page` with `wikiIdentifier: "On-Connect.wiki"` and `path: "/Teams/Metering/Règles métier - Télérelève eau"`. Read the current content first (`wiki` action `get_page_content`) to merge rather than overwrite, and update `references/regles-metier-snapshot.md` in this skill to match afterward so the offline snapshot does not drift too far from the live page.
