# Hermes Home Vision

Installation d'un agent IA (Hermes Agent, piloté en CLI) organisée en profils isolés et en huit couches, de
l'entrée d'un message jusqu'à la remédiation. Son second cerveau est un wiki en markdown que l'agent
compile et tient à jour lui-même — pour ceux qui veulent un système local dont chaque affirmation se
vérifie, plutôt qu'une base re-cherchée à chaque question.

Projet personnel de Thomas Leroyer (Searching Murphy). Non affilié aux éditeurs de Hermes Agent.

## Le problème

Un agent IA classique redécouvre la connaissance à chaque requête : mêmes documents relus, mêmes liens
reconstruits, même coût de contexte. La recherche augmentée par génération (RAG : retrouver des extraits au
moment de la question) ne capitalise pas non plus — elle rend des morceaux bruts, puis jette ce qu'elle en a
compris : synthèses, contradictions repérées, renvois entre sujets disparaissent avec la réponse.

Reste la tentation coûteuse : envoyer chaque question au plus gros modèle. Une primitive de décision coûte
environ 1,5 × 10⁻⁵ $ par appel, une génération complète environ 50 fois plus. Router avant de générer n'est
pas un confort, c'est ce qui rend l'usage quotidien tenable.

## La solution

Un **wiki** : des pages markdown que l'agent écrit, relie et corrige, et qui restent sur le disque entre deux
sessions. Compilé une fois à l'entrée, il n'est ensuite que mis à jour.

Une **primitive de décision**, JEV (TypeSafe System One) : elle tranche entre quelques options pour ~0,32 s et
~1,46 × 10⁻⁵ $ par appel, sans rédiger de texte, et choisit la source à interroger comme le skill utile.

Et une organisation en **huit couches numérotées de 0 à 7**, décrites ci-dessous dans l'ordre du flux. Note
historique : les documents d'origine numérotaient en quatre couches Aegis (1-4) puis « couche 5 » pour ANIMA ;
la numérotation 0-7 les englobe, ces valeurs ne servant plus qu'à lire les textes antérieurs.

## Les huit couches en un coup d'œil

| # | Couche | Composant principal | Port |
|---|---|---|---|
| 0 | Entrées | CLI, Telegram, email (Discord et A2A non configurés) | — |
| 1 | Gateway et profils | gateway multiplexé, profils isolés | — |
| 2 | Routage LLM | OmniRoute, proxy NIM | 20128, 20200 |
| 3 | Outils et extensions | MCP, skills, plugins | — |
| 4 | Cerveau | RAG hybride, SiYuan, wiki, Laya | 8200, 6806 |
| 5 | Jugement | Conseil (trois personas) | — |
| 6 | Sécurité | Aegis — secrets, ACL, loopback | — |
| 7 | Réacteur | watcher ANIMA, bridge MARI | 55551-55553 |

```
  0 Entrées     CLI · Telegram · email (Discord, A2A non configurés)
  1 Gateway     gateway unique, profils isolés
  2 Routage     OmniRoute 20128 · proxy NIM 20200
  3 Outils      MCP · skills · plugins
  4 Cerveau     RAG 8200 · SiYuan 6806 · wiki · Laya
  5 Jugement    Conseil — sceptique · oracle · arbitre
  6 Sécurité    Aegis — isolation, secrets, loopback
  7 Réacteur    watcher ANIMA → bridge MARI 55553
                ↓ le flux descend de 0 à 7
```

## Couche 0 — Entrées

Canaux réellement configurés au 2026-11-09 : CLI (toujours), Telegram (`default` + `veille`), email
(`default`, en échec d'authentification IMAP). Discord et A2A ne sont pas configurés. Aucun canal n'écoute
sur un port propre : les plateformes sont des connexions sortantes, configurées par profil. Un canal ne sert
qu'un profil — jamais de bot partagé ni de `.env` copié.

## Couche 1 — Gateway et profils

Un gateway unique, multiplexé (`multiplex_profiles: true` dans le `config.yaml` racine), sert tous les
profils. Chacun a sa configuration et ses secrets dédiés.

| Profil | Configuration | Rôle |
|---|---|---|
| `default` | `config.yaml` (racine) | opérateur : code, vidéo, WordPress, second cerveau |
| `security` | `profiles/security/` | analyste sécurité + couche de jugement |
| `veille` | `profiles/veille/` | veille technologique : arXiv, HuggingFace, flux |
| `watch` | `profiles/watch/` | surveillances et automatisations planifiées |
| `video` | `profiles/video/` | production vidéo : montage, QA d'analyse |

S'y ajoutent trois profils d'alias ou non versionnés : `deepseek-omniroute` et `kimi-omniroute` (bascule de
modèle), `docs-writer` (local, gateway arrêté).

## Couche 2 — Routage LLM

- **OmniRoute** (`127.0.0.1:20128`) : routeur de modèles, combos et chaîne ordonnée — gratuits d'abord,
  payant en dernier, chaque bascule tracée.
- **Proxy NIM** (`127.0.0.1:20200`) : normalise les appels NVIDIA NIM (préfixes de modèle et paramètres
  rejetés) pour que le routeur les traite comme les autres.

**Kimi** (kimi-k3, Moonshot) circule dans la configuration sans être un étage de cette couche : c'est un
alias premium, appelé manuellement en dernier recours, plus cher en tokens et absent de la chaîne de repli.

## Couche 3 — Outils et extensions

- **MCP** (Model Context Protocol, le standard par lequel l'agent appelle des serveurs d'outils) :
  `premiere` (`default`, `video`), `council` et `rea` (`security`). Aucun autre MCP configuré au 2026-11-09.
- **Skills** : bibliothèque de procédures markdown versionnées, chacune décrivant quand l'utiliser.
- **Plugins** : deux hooks avant l'appel au modèle — `jev-skill-router` (nomme au plus un skill pertinent)
  et `anima-memoire-router` (injecte le contexte mémoire).

## Couche 4 — Cerveau

- **RAG hybride** (`127.0.0.1:8200`) : chaque résultat porte un rang lexical et un rang vectoriel (embeddings
  `e5-base`), sur 4 284 fragments répartis entre skills, SiYuan, wiki et WordPress.
- **SiYuan** (`127.0.0.1:6806`) : notes structurées, projets et décisions.
- **Le wiki** : pages compilées, décrites ci-dessous.
- **Laya** : décision locale en ONNX, sur CPU, en repli quand JEV est injoignable — hors flux nominal.

### Le wiki compilé (patron Karpathy)

Le wiki suit le patron dit « LLM Wiki » : l'agent maintient une base interreliée au lieu de re-chercher. Il
occupe trois couches internes, à ne pas confondre avec les huit couches du système.

| Couche interne | Support | Rôle |
|---|---|---|
| Sources | `raw/notes/` | Matériau brut, immuable. L'agent lit, ne modifie jamais. |
| Wiki | `wiki/concepts/`, `wiki/entities/`, `index.md`, `log.md` | Pages compilées, cross-référencées. |
| Schéma | `wiki/SCHEMA.md` | Conventions : nommage, frontmatter, taxonomie de tags. |

Trois opérations le font vivre :

- **Ingest** — faire entrer une source : dépôt dans `raw/notes/`, puis compilation par
  `wiki/scripts/compile_wiki.py`, qui lit le schéma et les sources non citées et écrit les pages en un seul
  appel. Pas un tour d'agent : un agent coupé par un timeout ne compile pas de façon fiable.
- **Query** — répondre : `index.md` donne les pages candidates, l'agent les lit et cite. Une réponse
  substantielle est reversée dans le wiki — c'est là que la connaissance s'accumule. JEV choisit le niveau :
  L1 (wiki déjà synthétisé) ou L2 (RAG sur les documents bruts).
- **Lint** — contrôler : liens morts, pages orphelines, index incomplet, frontmatter invalide, et surtout
  contradictions — inscrites dans `contradictions.md`, la page portant `contested: true`.

Dernière vérification fonctionnelle : 2026-11-09.

## Couche 5 — Jugement

Le **Conseil** est une couche de délibération, pas un modèle de plus : il ne produit pas de contenu, il juge
une action avant qu'elle soit prise. Un `council_gate` fait débattre trois personas — sceptique, oracle,
arbitre — puis rend un verdict : autoriser, sous conditions, ou refuser.

Ce n'est pas un gate de latence. Mesuré sur un cas trivial (« `git push --force` sur main ») : 109 s pour un
verdict `deny` à 92 % de confiance, non calibrée — 92 % sur un cas de test en dit plus sur le modèle que sur le
risque réel. Il ne se place donc pas sur le chemin d'une réponse ordinaire.

Dernière vérification fonctionnelle : 2026-11-09.

## Couche 6 — Sécurité Aegis

**Aegis** est le nom de la couche de sécurité. Quatre principes vérifiables : un `.env` par profil (3 ACE
héritées du dossier parent au 2026-11-09, 0 explicite — durcissement non fait) ; aucun `.env` versionné
(`git ls-files | grep .env` = 0) ; moindre privilège ; services
à l'écoute de la boucle locale seulement, plages réservées au 2026-11-09 : 13508 et 49560-50444 ; aucune ne
couvre la plage de 54985 à 55084 ; un SIEM (Wazuh) complète la couche sur le papier, mais il n'était pas en
écoute au 2026-11-09 avant 16:21 ; remis en ligne à 16:21, cluster green.

## Couche 7 — Réacteur et bridge MARI

**ANIMA** est la couche de supervision : un watcher lit l'état de chaque composant toutes les 30 s (à
l'arrêt depuis le 2026-11-03 23:44 — sa relance est décidée séparément ; le compteur MARI reste à 1 tant
qu'il ne tourne pas) et détecte les transitions (READY/DEGRADED vers FAILED/BLOCKED). Il ne répare rien, il
émet un signal.

**MARI** est le système cible : un nœud Orleans (framework d'acteurs) dont le bridge expose trois ports fixes
— 55551 silo, 55552 gateway, 55553 API REST — sur la boucle locale uniquement. `GET /status` est en lecture
seule (`{"status":"ok","signalsEmitted":N}`) ; `POST /fire` écrit, répond en 19 ms avec
`{"status":"signal emitted","total":N}` et incrémente le compteur.

La remédiation n'est pas revendiquée : sur un tir de test, le compteur s'incrémente, mais aucune demande n'est
créée dans `remediation/` du bridge et rien n'apparaît au journal du watcher. La chaîne est démontrée jusqu'au
compteur — signal émis, décision non observée.

Le bridge vit hors dépôt (`C:/mari-poc/bridge`, journal et kill-switch compris) ; le dépôt d'exploitation
versionne les sources adaptées (`mari-adapted/`), le lanceur (`scripts/launch_mari.py`) et le watcher.

Dernière vérification fonctionnelle : 2026-11-09.

## Installation

Prérequis : Windows 11, PowerShell 5.1+, connexion réseau ; Python, git et ffmpeg sont posés par l'installeur,
sans droits administrateur. **Aucune interface graphique n'est requise** : un terminal suffit, Telegram,
Discord et l'email étant optionnels.

```powershell
# 1. Installer l'agent
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
hermes --version

# 2. Vérifier la compilation du wiki sans rien écrire
python wiki/scripts/compile_wiki.py --dry-run

# 3. Compiler les sources non encore citées
python wiki/scripts/compile_wiki.py

# 4. Router une question avant de générer (L1 wiki ou L2 RAG)
python wiki/scripts/jev_router.py "Qu'est-ce que le LLM Wiki ?"
```

Le dépôt d'exploitation (configurations, scripts, données, secrets) reste privé ; ce dépôt public en décrit
l'architecture. Les secrets sont créés localement, par profil, à partir de `.env.example`.

## Limites

Le wiki n'est pas un moteur de recherche. Tant que le corpus reste petit — 20 pages et 5 sources — lire
`index.md` puis grep suffit, et c'est plus fiable qu'un index approximatif.

Le RAG, lui, est déjà hybride : 4 284 fragments, dont 349 de wiki contre 3 353 de skills. Le vrai seuil est là :
au-delà de 100 pages ou 100 000 tokens de wiki compilé, il faut sortir le wiki dans son propre index — sinon une
question sur le wiki ramène des pages de skills. L'implémentation de référence (94 articles, 99 sources
quotidiennes) situe la dégradation entre 50 000 et 100 000 tokens de wiki curé.

Autres limites, mesurées :

- les décisions de JEV et du Conseil ne sont pas calibrées : elles s'ordonnent, elles ne se lisent pas comme
  des probabilités ;
- les modèles gratuits de repli tombent régulièrement (429, 504) : des secours, jamais des cibles ;
- le wiki ne se vérifie pas tout seul : sans lint régulier, `index.md` dérive du disque ;
- le bridge MARI vit hors dépôt : cloner le dépôt ne suffit pas à faire tourner la couche 7.

Dernière vérification fonctionnelle : 2026-11-09.

## Références

- Andrej Karpathy, *LLM Wiki* (gist) : https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- `Astro-Han/karpathy-llm-wiki` (implémentation de référence, MIT) : https://github.com/Astro-Han/karpathy-llm-wiki
- Agent Skills, le standard de packaging suivi ici : https://agentskills.io
- MARI (amont) : https://github.com/kaonomi-net/mari — fork adapté : https://github.com/Antoine-Thomas/mari

## FAQ

**Ça tourne sans interface graphique ?** Oui, c'est le mode nominal. Tout passe par le terminal ; les canaux
de messagerie sont optionnels.

**Pourquoi deux modèles au lieu d'un ?** Parce que décider coûte 50 fois moins cher que rédiger. JEV tranche
pour ~1,5 × 10⁻⁵ $ avant l'appel.

**Comment j'ajoute une source ?** Dépose un markdown dans `raw/notes/`, lance `compile_wiki.py --dry-run`
pour voir les pages touchées, puis sans le flag. Le fichier brut n'est jamais réécrit.

**Quelle différence avec un RAG classique ?** Le RAG retrouve des extraits bruts et jette sa synthèse. Ici la
synthèse est écrite, datée, reliée et conservée. Les deux coexistent : wiki en L1, RAG en L2, JEV choisit.

**Combien ça coûte ?** Le palier gratuit d'abord. Une décision coûte ~1,5 × 10⁻⁵ $. Un modèle premium existe
en dernier recours, appelé à la main, jamais automatiquement.

**Ça envoie mes données dans le cloud ?** `raw/` et `wiki/` restent sur le disque. Les services n'écoutent
qu'en local, et les secrets ne sont jamais versionnés : un `.env` par profil.

**Combien de pages avant que ça casse ?** Rien ne casse brutalement : la lecture et le grep se dégradent.
Repère : 50 000 à 100 000 tokens de wiki curé, soit environ 100 pages ici — c'est le moment de sortir le wiki
dans son propre index.

## Licence

© 2026 Thomas Leroyer (Searching Murphy). Le contenu de ce dépôt est sous **CC BY 4.0** — réutilisation,
modification et redistribution autorisées avec attribution. Texte complet dans [`LICENSE`](LICENSE).

Les skills commerciaux publiés séparément sur Agensi.io font l'objet d'une licence propre ; leur mention ici
ne confère aucun droit d'usage ni de redistribution.
