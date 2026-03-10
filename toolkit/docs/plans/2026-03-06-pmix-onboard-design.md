# PMIX Onboard + Migration Skills dans le Plugin

## Date: 2026-03-06

## Objectif

Creer un systeme d'onboarding automatique pour les nouveaux projets PMIX client, distribue via le plugin `powerbuilder-dev`. Permet aux collegues d'installer le plugin et d'avoir tout le systeme PMIX operationnel immediatement.

## Decisions prises

1. **Option A** : Migrer TOUS les skills PMIX dans le plugin (pas seulement onboard)
2. **Option A** : Auto-detection via instruction CLAUDE.md (pas de hook)
3. **Skills reecrites pour RAG** : Plus de lecture directe de `docs/knowledge/`, tout passe par `pmix_search` / `pmix_lookup`
4. **Marqueur** : `.pmix-client.json` a la racine du projet

## Architecture cible

```
powerbuilder-toolkit/packages/claude-plugin/
  skills/
    pb-modify/          (existant, inchange)
    pb-debug/           (existant, inchange)
    pb-analyze/         (existant, inchange)
    pb-create/          (existant, inchange)
    pmix-onboard/       NOUVEAU
    pmix-navigate/      MIGRE + reecrit pour RAG
    pmix-flux/          MIGRE + reecrit pour RAG
    pmix-impact/        MIGRE + reecrit pour RAG
  commands/
    pb-setup.md         (existant, ajout regle auto-detection)
  hooks/
    hooks.json          (existant, inchange)
```

## Skill pmix-onboard

### Trigger

Invoque automatiquement quand `.pmix-client.json` n'existe pas (regle dans CLAUDE.md genere par `/pb-setup`).

### Etapes

1. Scanner `Cust_*`, `_sysxtra`, `_cust2` via `pb_list_objects` + `pb_get_project_structure`
2. Lire `uo_cust_prg_id` via `pb_read_object` — extraire `cust_id`, `cust_builtno`, `ExpCustDbLvl`
3. Inventorier les personnalisations (nombre d'objets par type/module)
4. Indexer le code custom via `pmix_reindex`
5. Generer `.pmix-client.json` a la racine du projet
6. Afficher resume au developpeur

### Fichier .pmix-client.json

```json
{
  "cust_id": "DUPONT",
  "cust_builtno": "0360",
  "exp_cust_db_lvl": 572,
  "onboarded": "2026-03-06",
  "db_dsn": "Pmix",
  "custom_libs": {
    "_sysxtra": 23,
    "_cust2": 83,
    "cust_peppol": 9
  },
  "custom_summary": "Menu ERP surcharge, barcode reader custom, web services, Peppol"
}
```

## Migration des 3 skills PMIX

### Avant (lecture fichiers)

```
Etape 2 : Read docs/knowledge/00-routing-table.md
Etape 3 : Read docs/knowledge/05-sales-orders.md
```

### Apres (RAG)

```
Etape 2 : pmix_search("commande vente flux")
Etape 3 : pmix_lookup("salhead") si besoin de detail
```

### pmix-navigate

- Trigger: toute question sur PMIX
- Workflow: analyser question -> pmix_search -> pmix_lookup si detail necessaire -> repondre avec references

### pmix-flux

- Trigger: question sur un processus/flux metier PMIX
- Workflow: pmix_search(flux) -> pmix_lookup(tables concernees) -> decrire le flux avec etapes

### pmix-impact

- Trigger: "si je modifie X", "quel impact de changer Y"
- Workflow: pmix_search(objet) -> pb_get_dependencies -> pb_get_inheritance -> analyser impact

## Modification de pb-setup

Ajouter dans le CLAUDE.md genere :

```markdown
## Auto-detection nouveau projet
Si `.pmix-client.json` n'existe pas a la racine, invoquer le skill pmix-onboard
avant toute autre action.
```

## Ce qui ne change PAS

- Les 4 skills PB (pb-modify, pb-debug, pb-analyze, pb-create)
- Le hook PostToolUse
- Le serveur MCP et ses 28 outils
- La DB RAG standard (pmix-standard.db)
