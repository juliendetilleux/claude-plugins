# Design : PMIX RAG — Recherche hybride FTS5 + Vecteurs

**Date** : 2026-03-02
**Auteur** : Claude + JUDE
**Statut** : Approuve

---

## 1. Objectif

Ajouter une recherche unifiee (docs + code) au plugin PowerBuilder Toolkit pour que Claude puisse trouver rapidement et precisement toute information PMIX en une seule requete, sans deviner ni generaliser.

**Probleme actuel** : Claude doit faire 3-4 etapes manuelles (routing table → knowledge file → code search → read) pour repondre a une question PMIX. Les knowledge files peuvent generaliser des patterns qui ne sont pas toujours vrais dans le code (chaque client a des developpeurs differents avec des approches differentes).

**Solution** : Recherche hybride FTS5 (mots-cles exacts) + vecteurs (semantique) dans SQLite, avec indexation automatique et incrementale des docs standard ET du code specifique client.

---

## 2. Architecture

### Approche : Integration dans le monorepo existant (Approche A)

Un seul serveur MCP expose tous les outils : 21 outils PB existants + 3 outils RAG.
Un seul dossier a installer pour les collegues.

### Structure des fichiers (ajouts)

```
powerbuilder-toolkit/
├── packages/mcp-server/
│   └── src/
│       ├── tools/
│       │   ├── rag.ts              ← NOUVEAU : 5 outils MCP (search, lookup, correct, learn, reindex)
│       │   └── ... (existants inchanges)
│       ├── rag-indexer.ts          ← NOUVEAU : chunking markdown + indexation FTS5 + embeddings
│       ├── rag-db.ts              ← NOUVEAU : wrapper SQLite (FTS5 + sqlite-vec)
│       ├── server.ts              ← modifie : +registerRagTools()
│       └── cache.ts               ← existant, inchange
├── docs/                           ← Documentation standard PMIX (incluse dans le plugin)
│   ├── knowledge/                  (21 fichiers : routing table + 20 domaines)
│   ├── database/tables/            (471 fichiers)
│   ├── database/views/             (43 fichiers)
│   ├── database/procedures/        (88 fichiers)
│   ├── objects/windows/            (1145 fichiers)
│   ├── objects/datawindows/        (3701 fichiers)
│   ├── objects/userobjects/        (692 fichiers)
│   ├── objects/functions/          (390 fichiers)
│   ├── objects/structures/         (141 fichiers)
│   ├── objects/menus/              (89 fichiers)
│   ├── flux/                       (13 fichiers)
│   ├── modules/                    (50 fichiers)
│   └── index.md
├── data/
│   └── pmix-standard.db           ← auto-genere (index standard, ~50 MB)
└── package.json                   ← +better-sqlite3, +@xenova/transformers
```

### Architecture bi-couche (multi-projet)

```
Couche STANDARD (dans le plugin)
├── docs/                    ← Doc PMIX commune a tous les clients
└── data/pmix-standard.db   ← Index FTS5 + vecteurs (construit 1 fois)

Couche PROJET (dans chaque projet client)
├── Cust_*/                  ← Librairies specifiques client
├── _sysxtra/                ← Surcharges client
└── .pmix-rag.db             ← Index des specifiques (auto-genere par projet)
```

A la recherche, les deux couches sont interrogees simultanement.
Les resultats [custom] sont prioritaires sur [standard] (logique de surcharge).

---

## 3. Schema SQLite

### Table principale

```sql
CREATE TABLE chunks (
  id        INTEGER PRIMARY KEY,
  source    TEXT NOT NULL,      -- 'standard' ou 'custom'
  path      TEXT NOT NULL,      -- chemin relatif du fichier
  domain    TEXT,               -- 'purchasing', 'masters-items', 'stock', etc.
  obj_type  TEXT,               -- 'table', 'window', 'datawindow', 'function', 'flux', 'knowledge', 'code'
  obj_name  TEXT,               -- 'w_item_update', 'linkitad', 'd_purline_update'
  section   TEXT,               -- titre H2/H3 du chunk
  content   TEXT NOT NULL,      -- le texte du chunk
  mtime     REAL NOT NULL       -- timestamp modification fichier (pour incremental)
);
```

### Index FTS5

```sql
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  obj_name, section, content,
  content=chunks, content_rowid=id,
  tokenize='unicode61 remove_diacritics 2'
);
```

- `unicode61 remove_diacritics 2` : gere les accents francais
- Chercher "facture" trouve aussi "facture", "factures", "facturation"
- Chercher "reception" trouve "reception" et "réception"

### Index vectoriel (sqlite-vec)

```sql
-- Vecteurs d'embeddings (384 dimensions pour all-MiniLM-L6-v2)
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  chunk_id INTEGER PRIMARY KEY,
  embedding FLOAT[384]
);
```

---

## 4. Indexation

### Chunking des fichiers markdown

Chaque fichier `.md` est decoupe par sections `##` / `###` :
- Un fichier de 100 lignes avec 5 sections H2 = 5 chunks
- Chaque chunk contient : le titre de la section + le contenu jusqu'a la prochaine section
- Le contexte parent est preserve (nom du fichier, domaine, type d'objet, nom d'objet)

### Extraction des metadonnees

Le chemin du fichier determine les metadonnees :
- `docs/database/tables/linkitad.md` → domain=purchasing, obj_type=table, obj_name=linkitad
- `docs/objects/windows/w_item_update.md` → domain=masters-items, obj_type=window, obj_name=w_item_update
- `docs/knowledge/04-masters-suppliers.md` → domain=masters-suppliers, obj_type=knowledge
- `docs/flux/purchasing.md` → domain=purchasing, obj_type=flux

Le mapping domain est derive de la routing table (`00-routing-table.md`).

### Indexation des fichiers PB source (couche projet)

Les fichiers `.srw`, `.srd`, `.sru`, `.srf` des librairies `Cust_*` et `_sysxtra` :
- obj_type=code
- obj_name extrait du nom de fichier
- Chunking par event/function (le parser PB existant fournit les bornes)

### Indexation incrementale

Au demarrage du serveur MCP :
1. Verifier si la DB existe
2. Si non → indexation complete (~15-20 secondes pour ~13 000 fichiers standard)
3. Si oui → comparer les `mtime` des fichiers avec ceux en base → ne reindexer que les changes
4. Scanner `PB_SOLUTION_PATH/Cust_*` et `PB_SOLUTION_PATH/_sysxtra` pour les specifiques client

### Embeddings locaux

- Modele : `all-MiniLM-L6-v2` via `@xenova/transformers`
- 384 dimensions, ~22 MB de modele
- Telecharge automatiquement au premier lancement
- Fonctionne 100% offline apres le premier telechargement

---

## 5. Outils MCP

### Outil 1 : `pmix_search(query, limit?, source?)`

Recherche hybride FTS5 + vecteurs. L'outil principal.

**Parametres** :
- `query` (string, requis) — question en langage naturel ou terme technique
- `limit` (number, defaut 8) — nombre max de resultats
- `source` (enum, optionnel) — filtrer par "standard", "custom", ou les deux

**Algorithme de scoring** :
1. Executer la recherche FTS5 (score BM25)
2. Executer la recherche vectorielle (similarite cosinus)
3. Fusionner : `score = 0.4 * fts_score_norm + 0.6 * vector_score`
4. Trier par score descendant
5. Retourner les top N resultats

**Format de retour** :
```json
{
  "total_results": 42,
  "results": [
    {
      "score": 0.92,
      "source": "standard",
      "path": "knowledge/04-masters-suppliers.md",
      "domain": "masters-suppliers",
      "obj_type": "knowledge",
      "obj_name": null,
      "section": "Flux : Liaison article-fournisseur",
      "content": "Depuis la fiche article (w_item_update)... onglet Fournisseurs...",
      "highlight": "...liaison article-**fournisseur**..."
    },
    ...
  ],
  "index_stats": { "standard_chunks": 75000, "custom_chunks": 1200, "last_indexed": "..." }
}
```

### Outil 2 : `pmix_lookup(name)`

Recherche directe par nom d'objet (exact + prefix match via FTS5).

**Parametres** :
- `name` (string, requis) — nom de table, fenetre, DW, fonction

**Retourne** : Tous les chunks qui mentionnent ce nom, tries par pertinence.
Pas de recherche vectorielle — uniquement FTS5 pour la precision maximale.

### Outil 3 : `pmix_correct(path, old_text, new_text, reason)`

Corrige une erreur decouverte dans la documentation standard.
Quand Claude detecte une incoherence entre la doc et le code source reel.

**Parametres** :
- `path` (string, requis) — chemin relatif du fichier dans docs/ (ex: "knowledge/04-masters-suppliers.md")
- `old_text` (string, requis) — texte exact a remplacer (doit exister une seule fois)
- `new_text` (string, requis) — texte de remplacement
- `reason` (string, requis) — raison de la correction (pour tracabilite)

**Comportement** :
1. Verifie que `old_text` existe exactement une fois dans le fichier
2. Remplace `old_text` par `new_text`
3. Reindexe les chunks du fichier modifie
4. Log la correction dans `data/corrections.log` avec timestamp + raison
5. Retourne confirmation

**Garde-fous** :
- Ne modifie que les fichiers dans `docs/` (jamais le code source PB)
- `old_text` doit matcher exactement une fois (sinon erreur)

### Outil 4 : `pmix_learn(topic, content, tags?)`

Capture une nouvelle connaissance decouverte pendant le travail.
Permet a Claude d'enrichir l'index avec des informations non presentes dans la doc initiale.

**Parametres** :
- `topic` (string, requis) — sujet court (ex: "Navigation onglets PBTabControl")
- `content` (string, requis) — contenu de la connaissance (texte libre)
- `tags` (string[], optionnel) — tags pour faciliter la recherche

**Comportement** :
1. Cree un chunk de type `learned` dans la DB avec domain="learned"
2. Genere l'embedding pour la recherche vectorielle
3. Les chunks "learned" sont retournes par `pmix_search` avec tag [learned]

### Outil 5 : `pmix_reindex()`

Force une reindexation complete des deux couches.

**Retourne** : Statistiques d'indexation (fichiers traites, chunks crees, duree).

---

## 6. Auto-amelioration

### Tables d'apprentissage

```sql
-- Synonymes appris (enrichissement des recherches)
CREATE TABLE learned_synonyms (
  id        INTEGER PRIMARY KEY,
  term      TEXT NOT NULL,       -- "fournisseur article"
  target    TEXT NOT NULL,       -- "linkitad lktyp P"
  hits      INTEGER DEFAULT 1,   -- nombre de fois utilise
  created   TEXT NOT NULL
);

-- Feedback de recherche (quels chunks sont utiles)
CREATE TABLE search_feedback (
  id        INTEGER PRIMARY KEY,
  query     TEXT NOT NULL,       -- la question originale
  chunk_id  INTEGER NOT NULL,    -- le chunk qui a donne la bonne reponse
  useful    BOOLEAN DEFAULT 1,   -- utile ou pas
  created   TEXT NOT NULL
);

-- Log des corrections (tracabilite)
CREATE TABLE corrections_log (
  id        INTEGER PRIMARY KEY,
  path      TEXT NOT NULL,
  old_text  TEXT NOT NULL,
  new_text  TEXT NOT NULL,
  reason    TEXT NOT NULL,
  created   TEXT NOT NULL
);
```

### Scoring avec apprentissage

Le scoring hybride integre le feedback :
```
score = 0.3 * fts_score_norm
      + 0.5 * vector_score
      + 0.2 * feedback_boost
```

Ou `feedback_boost` = nombre de fois que ce chunk a ete marque utile / max feedback.

### Synonymes

Quand Claude cherche "fournisseur article" et finit par trouver la reponse dans un chunk contenant "linkitad" :
- Il appelle `pmix_learn` ou le systeme enregistre automatiquement l'association
- La prochaine recherche pour "fournisseur article" booste directement les chunks "linkitad"

### Cercle vertueux

```
Claude travaille sur PMIX
  → pmix_search("question")
  → trouve la bonne reponse dans chunk X
  → feedback: chunk X est utile pour cette question
  → decouvre une erreur dans la doc
  → pmix_correct() corrige la doc
  → decouvre un pattern non documente
  → pmix_learn() capture la connaissance
  → prochaine session : recherches plus precises, doc plus fiable
```

---

## 7. Dependencies nouvelles

| Package | Version | Taille | Role |
|---------|---------|--------|------|
| `better-sqlite3` | ^11 | ~3 MB (natif) | SQLite avec FTS5 |
| `sqlite-vec` | ^0.1 | ~1 MB | Extension vectorielle SQLite |
| `@xenova/transformers` | ^3 | ~2 MB (+ 22 MB modele au premier run) | Embeddings locaux |

**Note** : `better-sqlite3` necessite une compilation native. Sur Windows avec Node.js, `npm install` le compile automatiquement via `node-gyp`.

---

## 7. Installation pour les collegues

```bash
# 1. Cloner le repo
git clone <url> powerbuilder-toolkit

# 2. Installer (compile better-sqlite3 automatiquement)
cd powerbuilder-toolkit && npm install

# 3. Build
npm run build

# 4. Configurer .mcp.json dans le projet PMIX client
{
  "mcpServers": {
    "powerbuilder": {
      "command": "node",
      "args": ["<chemin>/powerbuilder-toolkit/packages/mcp-server/dist/server.js"],
      "env": {
        "PB_SOLUTION_PATH": "<chemin du projet PMIX client>",
        "PB_EXE_PATH": "<chemin>/pmix.exe",
        "PYTHON_EXE": "<chemin>/python.exe"
      }
    }
  }
}
```

Le modele d'embeddings se telecharge automatiquement au premier `pmix_search`.
L'index se construit automatiquement au premier demarrage (~15-20 sec).

---

## 8. Performance attendue

| Metrique | Valeur |
|----------|--------|
| Indexation complete (standard) | ~15-20 secondes |
| Indexation incrementale | <1 seconde (si peu de changes) |
| Recherche FTS5 seule | <5 ms |
| Recherche vectorielle seule | ~50 ms |
| Recherche hybride | ~60 ms |
| Taille DB standard | ~50 MB |
| Taille DB par projet | ~1-5 MB (selon taille Cust_*) |
| RAM supplementaire | ~50 MB (modele embeddings en memoire) |

---

## 9. Risques et mitigations

| Risque | Mitigation |
|--------|------------|
| `better-sqlite3` ne compile pas sur certaines machines | Fallback sur `sql.js` (pur JS, plus lent mais zero compilation) |
| Modele embeddings trop gros a telecharger | Mode "FTS5 only" sans embeddings (perd la recherche semantique) |
| Index desynchronise avec le code | Verification mtime a chaque demarrage + outil `pmix_reindex()` |
| Conflit entre standard et custom | Resultats [custom] toujours prioritaires avec tag visible |

---

## 10. Hors scope (v1)

- Interface web pour naviguer dans l'index
- Synchronisation multi-utilisateurs de l'index
- Embeddings via API externe (Anthropic, OpenAI)
- Indexation du schema SQL Anywhere en direct (via ODBC)
