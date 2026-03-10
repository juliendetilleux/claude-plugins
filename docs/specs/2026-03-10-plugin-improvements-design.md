# Design — Amelioration du plugin powerbuilder-dev

Date: 2026-03-10
Version cible: v0.3.0 → v0.5.0

## Contexte

Le plugin powerbuilder-dev v0.2.2 fonctionne mais presente des lacunes :
- Chevauchement confus entre agents et skills
- Skills PMIX sans triggers declares
- `pb-analyze` trop court, `pb-debug` sans format de sortie
- Nommage MCP incoherent
- README incomplet
- Pas de skill DataWindow ni testing
- Hook trop simpliste
- Pas de commandes utilitaires

## Principes de design

### Agents vs Skills

**Agents** = sous-agents autonomes en contexte isole pour des analyses lourdes multi-fichiers.
**Skills** = workflows integres dans la conversation pour guider l'action courante.

| Besoin | Composant | Type | Raison |
|--------|-----------|------|--------|
| Architecture complete d'un objet | pb-analyst | Agent | Rapport long, contexte isole |
| Comprendre un objet avant modif | pb-analyze | Skill | Etape rapide dans le flux |
| Revue de code formelle | pb-code-reviewer | Agent | Rapport structure, multi-fichiers |
| Modifier du code PB | pb-modify | Skill | Workflow obligatoire 5 etapes |
| Impact analysis majeure | pb-impact-checker | Agent | Trace l'arbre complet de dependances |
| Impact rapide petite modif | pmix-impact | Skill | Estimation rapide dans le flux |
| Recherche PMIX approfondie | pmix-researcher | Agent | Exploration multi-sources, rapport |
| Question PMIX simple | pmix-navigate | Skill | Reponse directe via RAG |

---

## Phase 1 — Solidifier l'existant (v0.3.0)

### 1.1 Clarification agents vs skills

Chaque agent et skill doit declarer explicitement son role :
- Agents : ajouter en haut `> Cet agent est un sous-agent autonome. Pour une action rapide, utiliser le skill <nom>.`
- Skills : ajouter en haut `> Ce skill guide l'action dans la conversation. Pour une analyse approfondie, utiliser l'agent <nom>.`

### 1.2 Corrections des skills

**Tous les skills PMIX** : ajouter description de trigger dans le frontmatter.

**pb-analyze** (39 → ~80 lignes) :
- Ajouter section analyse heritage (`pb_get_inheritance`)
- Ajouter section dependances (`pb_get_dependencies`)
- Ajouter section patterns architecturaux (DataWindows, menus, NVO)
- Ajouter format de sortie structure

**pb-debug** :
- Ajouter section format de sortie :
  - Symptome observe
  - Cause identifiee
  - Correction appliquee
  - Verification

**Nommage MCP** :
- Utiliser la forme courte partout dans les workflows (`pb_read_object`)
- Ajouter une note en haut de chaque agent : `Les outils MCP sont prefixes par mcp__powerbuilder__ dans Claude Code.`

### 1.3 README du plugin

Reecrire avec :
- Quick-start en 3 etapes : install marketplace → install plugin → `/pb-setup`
- Arbre de decision agent vs skill (tableau ci-dessus)
- Table recapitulative de tous les composants (agents, skills, commandes, hooks)
- Section "Pour les collegues" : comment installer

---

## Phase 2 — Nouvelles fonctionnalites (v0.4.0)

### 2.1 Skill pb-datawindow

**Fichier :** `skills/pb-datawindow/SKILL.md`

**Trigger :** Toute demande impliquant un DataWindow (creer, modifier colonnes, optimiser SQL, changer presentation).

**Workflow :**
1. Identifier le DW cible (nom, type : grid/freeform/composite/crosstab)
2. Lire le SQL source via `pb_get_datawindow_sql` + colonnes via `pb_dw_get_columns`
3. Analyser le contexte (fenetre parente, SetTransObject, arguments de Retrieve)
4. Appliquer la modification (SQL, colonnes, proprietes visuelles)
5. Valider syntaxe + compiler

**Format de sortie :**
- DataWindow : nom, type, library
- SQL avant/apres (si modifie)
- Colonnes ajoutees/supprimees/modifiees
- Impact sur les fenetres consommatrices

### 2.2 Skill pb-test

**Fichier :** `skills/pb-test/SKILL.md`

**Trigger :** Apres une modification, ou demande explicite de test/validation.

**Workflow :**
1. Identifier les objets modifies (via git diff ou historique de session)
2. Generer un plan de test :
   - Cas nominaux (flux normal)
   - Cas limites (null, vide, max length)
   - Regressions (fonctionnalites adjacentes)
3. Lancer l'app via `pb_launch_app`
4. Naviguer et capturer via `pb_screenshot_window` + `pb_interact_control`
5. Comparer visuellement via `pb_visual_compare` si reference existe
6. Produire un rapport : OK / KO / a verifier manuellement

**Format de sortie :**
- Objets testes
- Resultats par cas de test (OK/KO/Manuel)
- Captures d'ecran (references)
- Actions requises

### 2.3 Commande /pb-status

**Fichier :** `commands/pb-status.md`

**Sortie structuree :**
```
=== Projet PowerBuilder ===
Nom: <nom>
Chemin: <chemin>
Libraries: <nb> (.pbl)
Objets: <nb windows> / <nb DW> / <nb NVO> / <nb menus>

=== Toolkit ===
Chemin: <chemin server.js>
Statut: Connecte / Non connecte

=== MCP ===
Serveur: powerbuilder
Outils disponibles: <nb>/21

=== PMIX (si applicable) ===
Client: <nom>
RAG: <taille> chunks indexes
Derniere synchro: <date>

=== Git ===
Branche: <branche>
Fichiers modifies: <nb>
```

### 2.4 Hook validation syntaxe ameliore

**Fichier :** `hooks/hooks.json` (modifier l'existant)

Remplacer le hook actuel (simple rappel) par :
- Apres Edit/Write sur `.sr*` → appeler `pb_validate_syntax`
- Si erreur syntaxe → afficher l'erreur clairement
- Si OK → message discret "Syntaxe validee"

---

## Phase 3 — Polish (v0.5.0)

### 3.1 Commande /pb-validate

**Fichier :** `commands/pb-validate.md`

**Workflow :**
1. Compiler le projet complet via `pb_compile`
2. Lister erreurs/warnings
3. Verifier conventions de nommage (scan des objets)
4. Detecter patterns dangereux :
   - SQL sans check `SQLCA.SQLCode`
   - `Destroy` manquants
   - `IsNull`/`IsValid` non testes avant usage
   - `AcceptText()` manquant avant `Update()`
5. Rapport structure : erreurs bloquantes / warnings / suggestions

### 3.2 Hook pre-commit

**Ajout dans** `hooks/hooks.json` :

- Type : `PreCommit`
- Scanner les `.sr*` dans le staging git
- Valider la syntaxe de chacun via `pb_validate_syntax`
- Si erreur → bloquer le commit avec message explicite
- Si OK → laisser passer

### 3.3 Amelioration pb-setup

**Modifier** `commands/pb-setup.md` :
- Apres generation de `.mcp.json`, tenter `pb_get_project_structure` pour verifier la connexion MCP
- Si echec → diagnostic clair (serveur non demarre, chemin invalide, node manquant)
- Detection PMIX assouplie : accepter projets avec seulement certaines libraries standard
- Detection version PB via le `.pbproj` (2019R3 / 2022 / 2025)

---

## Recapitulatif des fichiers impactes

### Phase 1 (v0.3.0)
| Action | Fichier |
|--------|---------|
| Modifier | agents/pb-analyst.md |
| Modifier | agents/pb-code-reviewer.md |
| Modifier | agents/pb-impact-checker.md |
| Modifier | agents/pmix-researcher.md |
| Modifier | skills/pb-analyze/SKILL.md |
| Modifier | skills/pb-debug/SKILL.md |
| Modifier | skills/pmix-flux/SKILL.md |
| Modifier | skills/pmix-impact/SKILL.md |
| Modifier | skills/pmix-navigate/SKILL.md |
| Modifier | skills/pmix-onboard/SKILL.md |
| Reecrire | README.md |

### Phase 2 (v0.4.0)
| Action | Fichier |
|--------|---------|
| Creer | skills/pb-datawindow/SKILL.md |
| Creer | skills/pb-test/SKILL.md |
| Creer | commands/pb-status.md |
| Modifier | hooks/hooks.json |

### Phase 3 (v0.5.0)
| Action | Fichier |
|--------|---------|
| Creer | commands/pb-validate.md |
| Modifier | hooks/hooks.json |
| Modifier | commands/pb-setup.md |
