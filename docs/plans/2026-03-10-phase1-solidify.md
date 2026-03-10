# Phase 1 — Solidifier l'existant (v0.3.0)

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix all quality issues in existing agents, skills, and README for v0.3.0

**Architecture:** Modify 11 existing files — no new files created. Add agent/skill cross-references, fix triggers, expand thin skills, harmonize MCP naming.

**Tech Stack:** Markdown (.md), YAML frontmatter, JSON

---

## Chunk 1: Agents — add cross-references and MCP note

### Task 1: pb-analyst.md — add skill cross-reference

**Files:**
- Modify: `plugins/powerbuilder-dev/agents/pb-analyst.md`

- [ ] **Step 1: Add cross-reference note after frontmatter**

Insert after the `---` closing the frontmatter, before `# PowerBuilder Code Analyst`:

```markdown
> **Agent autonome** — Lance une analyse approfondie en contexte isole.
> Pour une vue rapide dans la conversation, utiliser le skill `pb-analyze`.
```

- [ ] **Step 2: Add MCP prefix note**

Insert at the end of the `## Your Capabilities` section:

```markdown
> **Note MCP** : Dans Claude Code, les outils sont prefixes par `mcp__powerbuilder__` (ex: `mcp__powerbuilder__pb_read_object`). Ce document utilise la forme courte pour la lisibilite.
```

- [ ] **Step 3: Commit**

```bash
git add plugins/powerbuilder-dev/agents/pb-analyst.md
git commit -m "docs(pb-analyst): add skill cross-ref and MCP note"
```

### Task 2: pb-code-reviewer.md — add cross-reference

**Files:**
- Modify: `plugins/powerbuilder-dev/agents/pb-code-reviewer.md`

- [ ] **Step 1: Add cross-reference note after frontmatter**

Insert after the `---` closing the frontmatter, before `# PowerBuilder Code Reviewer`:

```markdown
> **Agent autonome** — Lance une revue de code formelle avec rapport structure.
> Pour valider la syntaxe dans le flux de modification, utiliser le skill `pb-modify` (etape 5).
```

- [ ] **Step 2: Add MCP prefix note**

Insert at the end of the `## Review Checklist` intro (before `### 1. SQL & Database`):

```markdown
> **Note MCP** : Dans Claude Code, les outils sont prefixes par `mcp__powerbuilder__`. Ce document utilise la forme courte.
```

- [ ] **Step 3: Commit**

```bash
git add plugins/powerbuilder-dev/agents/pb-code-reviewer.md
git commit -m "docs(pb-code-reviewer): add skill cross-ref and MCP note"
```

### Task 3: pb-impact-checker.md — add cross-reference

**Files:**
- Modify: `plugins/powerbuilder-dev/agents/pb-impact-checker.md`

- [ ] **Step 1: Add cross-reference note after frontmatter**

Insert after the `---` closing the frontmatter, before `# PowerBuilder Impact Checker`:

```markdown
> **Agent autonome** — Trace l'arbre complet de dependances en contexte isole.
> Pour une estimation rapide d'impact dans la conversation, utiliser le skill `pmix-impact`.
```

- [ ] **Step 2: Add MCP prefix note**

Insert after `## Analysis Process` heading, before `### For Object Modifications`:

```markdown
> **Note MCP** : Dans Claude Code, les outils sont prefixes par `mcp__powerbuilder__`. Ce document utilise la forme courte.
```

- [ ] **Step 3: Commit**

```bash
git add plugins/powerbuilder-dev/agents/pb-impact-checker.md
git commit -m "docs(pb-impact-checker): add skill cross-ref and MCP note"
```

### Task 4: pmix-researcher.md — add cross-reference

**Files:**
- Modify: `plugins/powerbuilder-dev/agents/pmix-researcher.md`

- [ ] **Step 1: Add cross-reference note after frontmatter**

Insert after the `---` closing the frontmatter, before `# PMIX ERP Knowledge Researcher`:

```markdown
> **Agent autonome** — Recherche approfondie multi-sources avec rapport structure.
> Pour une reponse rapide a une question PMIX, utiliser le skill `pmix-navigate`.
```

- [ ] **Step 2: Add MCP prefix note**

Insert after `## Your Capabilities` heading, before the tool list:

```markdown
> **Note MCP** : Dans Claude Code, les outils sont prefixes par `mcp__powerbuilder__`. Ce document utilise la forme courte.
```

- [ ] **Step 3: Commit**

```bash
git add plugins/powerbuilder-dev/agents/pmix-researcher.md
git commit -m "docs(pmix-researcher): add skill cross-ref and MCP note"
```

---

## Chunk 2: Skills — fix triggers and expand content

### Task 5: pb-analyze — expand from 39 to ~80 lines

**Files:**
- Modify: `plugins/powerbuilder-dev/skills/pb-analyze/SKILL.md`

- [ ] **Step 1: Add cross-reference and trigger to frontmatter**

Replace the entire frontmatter with:

```yaml
---
name: pb-analyze
description: Use when you need to understand existing PowerBuilder code — explore architecture, inheritance, dependencies, and data flow. For deep analysis with a full report, use the pb-analyst agent instead.
---
```

- [ ] **Step 2: Rewrite the skill body**

Replace the entire body (everything after frontmatter) with:

```markdown
# PowerBuilder Code Analysis

> **Skill integre** — Guide l'analyse dans la conversation courante.
> Pour une analyse approfondie avec rapport complet, lancer l'agent `pb-analyst`.

## Trigger

Demande de comprendre un objet PB existant : "c'est quoi w_xxx ?", "montre-moi l'architecture de uo_xxx", "qui utilise d_xxx ?".

## Workflow

### Etape 1 : Vue d'ensemble

Utiliser `pb_get_object_summary` pour obtenir rapidement :
- Type, ancetre, library
- Nombre de fonctions et events
- Variables d'instance

### Etape 2 : Heritage

Utiliser `pb_get_inheritance` :
- Ancetres : de quoi herite cet objet ? Quels comportements sont herites ?
- Descendants : qui herite de cet objet ? (impact en cas de modification)

### Etape 3 : Dependances

Utiliser `pb_get_dependencies` :
- Qui appelle cet objet ? (dependances entrantes)
- Quels objets cet objet utilise-t-il ? (dependances sortantes)

### Etape 4 : Lecture du code source

Utiliser `pb_read_object` pour lire le code complet.
Identifier :
- Fonctions publiques (`of_*`) et privees (`wf_*`)
- Events personnalises (`ue_*`)
- Variables d'instance (`is_`, `il_`, `ib_`, `id_`)
- DataWindows utilises et leur SQL (`pb_get_datawindow_sql`)

### Etape 5 : Patterns architecturaux

Determiner le role de l'objet :
- **Fenetre sheet** : fenetre principale avec toolbar et menu
- **Fenetre response** : dialogue modal (saisie/confirmation)
- **NVO metier** : logique business sans UI (nvo_*, nv_*)
- **NVO utilitaire** : services transversaux (log, mail, config)
- **DataWindow** : presentation de donnees (grid, freeform, composite)
- **Menu** : barre de menu et items (m_*)

### Etape 6 : Structure du projet

Si necessaire, utiliser `pb_get_project_structure` pour comprendre :
- Organisation des 69+ libraries
- Repartition par module metier
- Conventions de nommage des libraries

## Format de sortie

```
## Analyse : [nom_objet]

### Identite
- Type : [type] | Ancetre : [ancetre] | Library : [library]
- Fonctions : [nb] | Events : [nb] | Variables : [nb]

### Heritage
- Chaine : [objet] → [parent] → [grand-parent] → ...
- Descendants : [nb] objets heritent de [objet]

### Dependances
- [nb] objets utilisent [objet]
- [objet] utilise [nb] objets

### Role architectural
[Description du role dans l'application]

### Fonctions cles
| Fonction | Visibilite | Role |
|----------|------------|------|

### Donnees
| DataWindow | SQL resume | Tables |
|------------|-----------|--------|
```
```

- [ ] **Step 3: Commit**

```bash
git add plugins/powerbuilder-dev/skills/pb-analyze/SKILL.md
git commit -m "feat(pb-analyze): expand skill with full workflow and output format"
```

### Task 6: pb-debug — add output format

**Files:**
- Modify: `plugins/powerbuilder-dev/skills/pb-debug/SKILL.md`

- [ ] **Step 1: Update frontmatter description**

Replace the frontmatter with:

```yaml
---
name: pb-debug
description: Use when diagnosing bugs, errors, or unexpected behavior in PowerBuilder code. Provides a systematic debugging approach with structured output.
---
```

- [ ] **Step 2: Add output format section at the end of the file**

Append after `### Step 5: Propose and Apply Fix`:

```markdown
## Format de sortie

```
## Diagnostic : [description du probleme]

### Symptome
- Erreur : [message d'erreur ou description du comportement]
- Contexte : [quand ca se produit]
- Reproductible : oui/non

### Cause identifiee
- Objet : [nom_objet]
- Localisation : [fonction/event, ligne]
- Explication : [pourquoi l'erreur se produit]

### Correction
- Action : [ce qui a ete fait]
- Code avant : `[ancien code]`
- Code apres : `[nouveau code]`

### Verification
- [ ] Syntaxe validee (`pb_validate_syntax`)
- [ ] Compile sans erreur (`pb_compile`)
- [ ] Comportement corrige (test manuel)
```
```

- [ ] **Step 3: Commit**

```bash
git add plugins/powerbuilder-dev/skills/pb-debug/SKILL.md
git commit -m "feat(pb-debug): add structured output format"
```

### Task 7: pmix-flux — add trigger to frontmatter

**Files:**
- Modify: `plugins/powerbuilder-dev/skills/pmix-flux/SKILL.md`

- [ ] **Step 1: Update frontmatter**

Replace the frontmatter with:

```yaml
---
name: pmix-flux
description: Use when documenting or explaining any PMIX business process, workflow, or data flow. Activates for questions like "comment faire X dans PMIX", "quel est le processus de Y", "decris le flux Z". For deep research with cross-referencing, use the pmix-researcher agent instead.
---
```

- [ ] **Step 2: Add cross-reference note after frontmatter**

Insert after `# PMIX Flux`, before `## Trigger`:

```markdown
> **Skill integre** — Documente les processus metier dans la conversation.
> Pour une recherche approfondie multi-sources, lancer l'agent `pmix-researcher`.
```

- [ ] **Step 3: Commit**

```bash
git add plugins/powerbuilder-dev/skills/pmix-flux/SKILL.md
git commit -m "docs(pmix-flux): add agent cross-ref to frontmatter"
```

### Task 8: pmix-impact — add trigger to frontmatter

**Files:**
- Modify: `plugins/powerbuilder-dev/skills/pmix-impact/SKILL.md`

- [ ] **Step 1: Update frontmatter**

Replace the frontmatter with:

```yaml
---
name: pmix-impact
description: Use when analyzing the business impact of modifying a PMIX object, table, or process. Activates for questions like "si je modifie X", "quel impact de changer Y", "quels flux sont touches par Z". For full dependency tree analysis, use the pb-impact-checker agent instead.
---
```

- [ ] **Step 2: Add cross-reference note after frontmatter**

Insert after `# PMIX Impact`, before `## Trigger`:

```markdown
> **Skill integre** — Estimation rapide d'impact dans la conversation.
> Pour une analyse complete de l'arbre de dependances, lancer l'agent `pb-impact-checker`.
```

- [ ] **Step 3: Commit**

```bash
git add plugins/powerbuilder-dev/skills/pmix-impact/SKILL.md
git commit -m "docs(pmix-impact): add agent cross-ref to frontmatter"
```

### Task 9: pmix-navigate — add trigger to frontmatter

**Files:**
- Modify: `plugins/powerbuilder-dev/skills/pmix-navigate/SKILL.md`

- [ ] **Step 1: Update frontmatter**

Replace the frontmatter with:

```yaml
---
name: pmix-navigate
description: Use when answering any question about PMIX ERP — routes to the right knowledge file and provides accurate answers with references. Activates for any question about PmiGest, PMIX business processes, tables, windows, or ERP functionality. For deep research with full report, use the pmix-researcher agent instead.
---
```

- [ ] **Step 2: Add cross-reference note after frontmatter**

Insert after `# PMIX Navigation`, before `## Trigger`:

```markdown
> **Skill integre** — Reponse rapide via RAG dans la conversation.
> Pour une recherche approfondie multi-sources avec rapport, lancer l'agent `pmix-researcher`.
```

- [ ] **Step 3: Commit**

```bash
git add plugins/powerbuilder-dev/skills/pmix-navigate/SKILL.md
git commit -m "docs(pmix-navigate): add agent cross-ref to frontmatter"
```

### Task 10: pmix-onboard — clarify trigger

**Files:**
- Modify: `plugins/powerbuilder-dev/skills/pmix-onboard/SKILL.md`

- [ ] **Step 1: Update frontmatter**

Replace the frontmatter with:

```yaml
---
name: pmix-onboard
description: Use when opening a new PMIX client project for the first time. Automatically triggered when .pmix-client.json does not exist at the project root and the project contains PMIX libraries (_sysxtra, Cust_Empty). Scans custom libraries, identifies the client, indexes custom code, and generates a project summary.
---
```

- [ ] **Step 2: Commit**

```bash
git add plugins/powerbuilder-dev/skills/pmix-onboard/SKILL.md
git commit -m "docs(pmix-onboard): clarify trigger condition in frontmatter"
```

---

## Chunk 3: README — rewrite with quick-start and decision tree

### Task 11: Rewrite README.md

**Files:**
- Modify: `plugins/powerbuilder-dev/README.md`

- [ ] **Step 1: Rewrite the full README**

Replace the entire file with:

```markdown
# powerbuilder-dev — Claude Code Plugin

Plugin Claude Code pour le developpement PowerBuilder 2025, avec support PMIX ERP (PmiGest).

## Quick Start

```bash
# 1. Ajouter le marketplace
/marketplace add juliendetilleux/claude-plugins

# 2. Installer le plugin
/plugin install powerbuilder-dev

# 3. Redemarrer Claude Code, puis dans votre projet PB :
/pb-setup
```

## Agents vs Skills — quand utiliser quoi ?

Les **agents** sont des sous-agents autonomes lances en contexte isole pour des analyses lourdes.
Les **skills** sont des workflows integres dans la conversation pour guider l'action courante.

| Besoin | Utiliser | Type |
|--------|----------|------|
| Analyse approfondie d'un objet | `pb-analyst` | Agent |
| Comprendre un objet rapidement | `pb-analyze` | Skill |
| Revue de code formelle | `pb-code-reviewer` | Agent |
| Modifier du code PB | `pb-modify` | Skill |
| Impact analysis complete | `pb-impact-checker` | Agent |
| Impact rapide d'une petite modif | `pmix-impact` | Skill |
| Recherche PMIX approfondie | `pmix-researcher` | Agent |
| Question PMIX simple | `pmix-navigate` | Skill |

**Regle simple** : pour une action rapide → skill. Pour un rapport detaille → agent.

## Composants

### Agents (4)

| Agent | Description |
|-------|-------------|
| **pb-analyst** | Analyse approfondie d'objets PB — heritage, dependances, call graph |
| **pb-code-reviewer** | Revue de code — bugs, mauvaises pratiques, conventions PMIX |
| **pb-impact-checker** | Analyse d'impact avant modification — trace toutes les dependances |
| **pmix-researcher** | Recherche PMIX — processus metier, tables, flux via RAG |

### Skills (8)

| Skill | Trigger | Description |
|-------|---------|-------------|
| **pb-modify** | Toute modification de code PB | Workflow obligatoire en 5 etapes (read → heritage → deps → modify → validate) |
| **pb-analyze** | Comprendre un objet existant | Vue d'ensemble, heritage, dependances, patterns |
| **pb-debug** | Bug ou comportement inattendu | Diagnostic systematique avec format de sortie structure |
| **pb-create** | Creer un nouvel objet PB | Identification ancetre, library, creation, compilation |
| **pmix-navigate** | Question sur PMIX | Reponse via RAG (recherche hybride FTS5 + semantique) |
| **pmix-flux** | Processus metier PMIX | Documentation des 12 macro-flux (vente, achat, stock...) |
| **pmix-impact** | Impact d'une modification | Analyse rapide croisant RAG + dependances PB |
| **pmix-onboard** | Nouveau projet PMIX | Scan custom, identification client, indexation RAG |

### Commandes (1)

| Commande | Description |
|----------|-------------|
| `/pb-setup [path]` | Setup complet : installe le toolkit, analyse le projet, genere CLAUDE.md et .mcp.json |

### Hooks (1)

| Hook | Trigger | Action |
|------|---------|--------|
| PostToolUse | Edit/Write sur `.sr*` | Rappel de compilation |

## MCP Tools (21)

Tous les outils proviennent de `@pb-toolkit/mcp-server` :

**Analyse** : pb_list_objects, pb_read_object, pb_search_code, pb_get_project_structure, pb_get_inheritance, pb_get_dependencies, pb_get_object_summary, pb_get_call_graph, pb_get_datawindow_sql, pb_dw_get_columns

**Modification** : pb_modify_script, pb_create_object, pb_validate_syntax, pb_compile, pb_refresh_cache

**Test visuel** : pb_launch_app, pb_screenshot_window, pb_list_controls, pb_interact_control, pb_save_reference, pb_visual_compare

**PMIX** : pmix_search, pmix_lookup, pmix_sql, pmix_tables, pmix_describe

## Pre-requis

- Claude Code avec support plugins
- `@pb-toolkit/mcp-server` connecte (fournit les outils pb_* et pmix_*)
- PowerBuilder 2025 installe
- Variables d'environnement : `PB_SOLUTION_PATH`, `PB_EXE_PATH`

## Installation pour un collegue

```bash
/marketplace add juliendetilleux/claude-plugins
/plugin install powerbuilder-dev
# Redemarrer Claude Code
# Dans le projet PB : /pb-setup
```
```

- [ ] **Step 2: Commit**

```bash
git add plugins/powerbuilder-dev/README.md
git commit -m "docs(README): rewrite with quick-start, decision tree, and complete reference"
```

---

## Chunk 4: Version bump and final commit

### Task 12: Bump version to 0.3.0

**Files:**
- Modify: `plugins/powerbuilder-dev/.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Update plugin.json version to 0.3.0**

Change `"version": "0.2.2"` to `"version": "0.3.0"`.

- [ ] **Step 2: Update marketplace.json version to 0.3.0**

Change `"version": "0.2.2"` to `"version": "0.3.0"`.

- [ ] **Step 3: Update CLAUDE.md to reflect new structure**

Update the CLAUDE.md at project root if needed (version reference).

- [ ] **Step 4: Commit and push**

```bash
git add plugins/powerbuilder-dev/.claude-plugin/plugin.json .claude-plugin/marketplace.json CLAUDE.md
git commit -m "chore: bump to v0.3.0 — solidified agents, skills, and README"
git push
```
