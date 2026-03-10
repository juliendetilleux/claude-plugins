# CLAUDE.md — pmix-plugins

Marketplace de plugins Claude Code pour le développement PowerBuilder / PMIX ERP.

## Architecture

```
.claude-plugin/marketplace.json    # Registre des plugins (version, source, catégorie)
plugins/
  powerbuilder-dev/                # Plugin principal (v0.2.0)
    .claude-plugin/plugin.json     # Manifeste du plugin
    agents/                        # 4 agents autonomes (.md)
    skills/                        # 8 skills réutilisables (SKILL.md dans chaque dossier)
    commands/                      # 1 commande CLI (.md)
    hooks/                         # hooks.json (PostToolUse)
```

## Conventions

### Langue
- **Métadonnées** (plugin.json, marketplace.json) : anglais
- **Contenu** (agents, skills, commandes) : français
- Les agents et skills communiquent en **français**

### Format des fichiers
- Line endings : **LF uniquement** (`.gitattributes` appliqué)
- Agents : `agents/<nom>.md` avec front matter YAML (name, description, model, tools, etc.)
- Skills : `skills/<nom>/SKILL.md` avec front matter YAML (name, description, trigger)
- Commandes : `commands/<nom>.md` avec front matter YAML
- Hooks : `hooks/hooks.json` (format Claude Code hooks)

### Front matter obligatoire

**Agent :**
```yaml
---
name: nom-agent
description: Description courte
model: sonnet  # ou opus, haiku
subagent_type: nom-agent
allowed_tools: [Read, Grep, Glob, mcp__xxx]
---
```

**Skill :**
```yaml
---
name: nom-skill
description: Description courte
---
```

**Commande :**
```yaml
---
name: nom-commande
description: Description courte
allowed_tools: [Read, Write, Bash, etc.]
---
```

## Règles de développement

1. **Ne pas modifier** la structure des dossiers existants sans mettre à jour `plugin.json`
2. **Toujours synchroniser** `marketplace.json` quand on change la version dans `plugin.json`
3. **Tester les front matter** YAML — un front matter invalide casse le chargement du plugin
4. **Pas de dépendances Node/npm** dans ce repo — c'est un repo de contenu Markdown/JSON uniquement
5. **Hooks** : le format `hooks.json` suit la spec Claude Code (type, matcher, command)

## plugin.json — champs clés

```json
{
  "name": "powerbuilder-dev",
  "version": "0.2.0",
  "description": "...",
  "keywords": ["powerbuilder", "pb", "pmix"],
  "agents": ["agents/*.md"],
  "skills": ["skills/*/SKILL.md"],
  "commands": ["commands/*.md"],
  "hooks": ["hooks/hooks.json"]
}
```

Quand tu ajoutes un agent/skill/commande, vérifie que le glob dans `plugin.json` le capture.

## Workflow git

- Branche principale : `main`
- Commits en anglais, format conventionnel : `feat:`, `fix:`, `docs:`, `chore:`
- Pas de CI/CD — validation manuelle

## MCP — référence rapide

Le plugin utilise `@pb-toolkit/mcp-server` (21 outils). Les agents déclarent leurs outils MCP autorisés via `allowed_tools` dans le front matter. Préfixe MCP : `mcp__pb-toolkit__`.

**Catégories d'outils :**
- Analyse : `pb_read_object`, `pb_search_code`, `pb_get_inheritance`, `pb_get_dependencies`, `pb_get_call_graph`
- Modification : `pb_modify_script`, `pb_create_object`, `pb_validate_syntax`, `pb_compile`
- PMIX : `pmix_search`, `pmix_lookup`, `pmix_sql`, `pmix_tables`, `pmix_describe`
- Visuel : `pb_launch_app`, `pb_screenshot_window`, `pb_visual_compare`

## Checklist nouveau plugin

1. Créer `plugins/<nom>/.claude-plugin/plugin.json`
2. Ajouter agents/skills/commandes/hooks selon besoin
3. Enregistrer dans `.claude-plugin/marketplace.json`
4. Ajouter `.gitattributes` (LF)
5. Documenter dans un `README.md`
