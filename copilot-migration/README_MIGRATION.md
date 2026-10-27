# Migration Claude → GitHub Copilot (VS Code)

## Installation
1. Copier le contenu de ce dossier à la racine du dépôt de travail (`.github/` et `.vscode/`).
2. Construire/avoir l'image Docker `adx-mcp-server` ; `az login` / `npx` disponibles pour Azure DevOps.
3. Dans VS Code : ouvrir `.vscode/mcp.json`, lancer les serveurs (« Start »), saisir tenant/client/secret à l'invite (rien n'est stocké dans les fichiers).
4. Dans le chat Copilot (mode Agent), choisir l'agent `ocr-analyste` ou un des 3 sous-agents.

## Contenu
- `.github/copilot-instructions.md` : ex-CLAUDE.md (pipeline + règles essentielles)
- `.github/agents/` : devops-rules, adx-query, adx-validator + orchestrateur ocr-analyste
- `.github/skills/adx-water-leak-analysis/` : schéma ADX, pièges, méthodes silencieux/bloqués (+ analyse du code Synapse)
- `.github/skills/onconnect-business-rules/` : index des règles métier
- `.vscode/mcp.json` : serveurs adx et azure-devops, sans secret

## Non migré / à vérifier
- L'historique de conversation n'est pas transférable (les connaissances utiles sont dans les skills).
- Les noms d'outils MCP dans les `.agent.md` (`adx/execute_query`, `azure-devops/*`) sont à vérifier dans le sélecteur d'outils de VS Code selon votre version.
- Le champ `agents:` de l'orchestrateur (sous-agents) peut dépendre de la version de VS Code ; sinon, invoquer les agents à la main.
- `onconnect-business-rules` : le fichier `references/regles-metier-snapshot.md` cité dans la skill n'était pas dans la copie synchronisée ; la source de vérité reste la page wiki `/Teams/Metering/Règles métier - Télérelève eau`.
- Les fichiers parquet / modèles .pkl / wheels Synapse et les exports Excel ne sont pas inclus.
- Licence : vérifier avec GitHub / Digital Solutions la couverture Copilot pour un usage par les utilisateurs finaux d'OCR.
