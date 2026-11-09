# Hermes Home Vision

Installation d'un agent IA (Hermes Agent, piloté en CLI) organisée en profils isolés,
dont le second cerveau est un wiki en markdown que l'agent compile et tient à jour
lui-même — pour ceux qui veulent une base de connaissances locale qui s'améliore au
lieu d'être re-cherchée à chaque question.

Projet personnel de Thomas Leroyer (Searching Murphy). Non affilié aux éditeurs de
Hermes Agent.

## Le problème

Un agent IA classique redécouvre la connaissance à chaque requête. Il relit les mêmes
documents, reconstruit les mêmes liens, et paie le même coût de contexte à chaque fois.

Le RAG classique ne capitalise pas non plus : il retrouve des morceaux bruts au moment
de la question, mais rien de ce qu'il a compris n'est conservé. Les synthèses, les
contradictions repérées, les renvois entre sujets — tout est jeté après la réponse.

Reste une tentation coûteuse : envoyer chaque question au plus gros modèle disponible.
Un petit modèle de décision coûte environ 1,5 × 10⁻⁵ $ par appel ; une génération
complète coûte environ 50 fois plus. Router avant de générer n'est pas une optimisation
de confort, c'est ce qui rend l'usage quotidien tenable.

## La solution

Un **wiki** : un ensemble de pages markdown que l'agent écrit, relie et corrige, et qui
reste sur le disque entre deux sessions. L'information est compilée une fois à l'entrée,
puis seulement mise à jour — au lieu d'être re-dérivée à chaque question.

Le partage du travail est explicite : l'humain choisit les sources et oriente l'analyse ;
l'agent résume, cross-référence, classe et signale les contradictions.

État réel de cette installation (novembre 2026) : 20 pages compilées à partir de
5 sources brutes, dans 5 profils isolés.

## Architecture en trois couches

| Couche | Support | Rôle |
|---|---|---|
| 1. Sources | `raw/notes/*.md` | Matériau brut, immuable. L'agent lit, ne modifie jamais. |
| 2. Wiki | `wiki/concepts/`, `wiki/entities/`, `index.md`, `log.md` | Pages compilées, cross-référencées par l'agent. |
| 3. Schéma | `wiki/SCHEMA.md` | Conventions contraignantes : nommage, frontmatter, taxonomie de tags, politique de mise à jour. |

`index.md` est le catalogue (une ligne de résumé par page) ; `log.md` est le journal
append-only des opérations, pour savoir ce qui a été touché et quand.

```
COUCHE 1 : SOURCES         COUCHE 2 : WIKI               COUCHE 3 : SCHÉMA
raw/notes/*.md   ──▶       concepts/   entities/         SCHEMA.md
immuable         ingest    index.md    log.md            ◀───── impose les règles
                           ├── lint ──▶ contradictions.md
                           │
                           │ query  (JEV choisit le niveau)
                           ▼
                           JEV (choice) ── L1 : wiki ──▶ réponse citée
                                        └────────── L2 : RAG  ──▶ réponse citée
```

Les profils — cinq au total, chacun avec sa configuration et ses secrets dédiés (un bot
de messagerie ne sert qu'un profil). `default` est le seul à vivre à la racine du
dépôt ; les quatre autres sont des dossiers sous `profiles/` :

| Profil     | Configuration                   | Rôle                                                           |
|------------|---------------------------------|----------------------------------------------------------------|
| `default`  | `config.yaml` (racine)          | opérateur : code, vidéo, WordPress, second cerveau             |
| `security` | `profiles/security/config.yaml` | analyste sécurité + couche de jugement avant action à risque   |
| `veille`   | `profiles/veille/config.yaml`   | veille technologique : arXiv, HuggingFace, flux RSS → synthèse |
| `watch`    | `profiles/watch/config.yaml`    | surveillances et automatisations planifiées                    |
| `video`    | `profiles/video/config.yaml`    | production vidéo : montage, QA d'analyse                       |

## Les trois opérations

**Ingest** — faire entrer une source dans la base.
La source est déposée telle quelle dans `raw/notes/` (fichier daté, jamais réécrit),
puis `wiki/scripts/compile_wiki.py` lit `SCHEMA.md`, `index.md`, `log.md` et les sources
non encore citées, et écrit ou met à jour les pages concernées. La compilation est un
script unique en un seul appel HTTP, pas un tour d'agent : un agent recoupé par un
timeout de 3 minutes ne peut pas compiler un wiki de façon fiable.
JEV : aucun rôle. Kimi : aucun rôle.

**Query** — répondre à une question.
`index.md` donne les pages candidates, l'agent les lit et répond en citant les pages
(`[[concept]]`). Une réponse substantielle est reversée dans le wiki : c'est là que la
connaissance s'accumule.
JEV : tranche entre deux niveaux — lire le wiki déjà synthétisé (L1), ou interroger le
RAG sur les documents bruts (L2). Kimi : aucun rôle automatique.

**Lint** — contrôler la santé du wiki.
Pages orphelines (aucun lien entrant), liens `[[...]]` morts, page absente de
`index.md`, frontmatter incomplet, tags hors taxonomie, pages de plus de 200 lignes,
et surtout contradictions : deux pages qui affirment des choses incompatibles. Les
contradictions sont inscrites dans `contradictions.md`, et chaque page porte
`contested: true` dans son frontmatter.
JEV : aucun rôle. Kimi : aucun rôle.

## Le routage des modèles

Trois rôles distincts, à ne pas confondre :

**JEV** (TypeSafe System One) est une primitive de décision : `noul` pour juger,
`choice` pour trancher entre N options, `score` pour noter. Il ne rédige pas de texte.
Mesuré ici à ~0,32 s et ~1,46 × 10⁻⁵ $ par appel. Deux usages : choisir le niveau de
recherche (L1 wiki ou L2 RAG), et nommer au plus un skill utile avant l'appel modèle.

**Le modèle de génération** rédige la réponse. Il est pris gratuit d'abord (`eco` via
un routeur local, port 20128), puis la chaîne de repli passe par des modèles gratuits
distants, et le payant n'intervient qu'en dernier.

**Kimi** (kimi-k3, Moonshot) est un alias premium déclaré dans la configuration, à
appeler manuellement en dernier recours : il est plus cher en tokens et n'est pas dans
la chaîne de repli automatique.

Conséquence directe, et c'est le point contre-intuitif : l'escalade n'est pas
« wiki → modèle plus cher ». Si le wiki ne couvre pas la question, JEV route vers le
RAG (documents bruts), et la génération reste sur le même palier de coût. On monte en
précision de source, pas en prix.

## Le jugement (couche 5)

**Le Conseil** est tout sauf un gate de latence : c'est une délibération lourde. Un
`council_gate` fait débattre trois personas — sceptique, oracle, arbitre — puis rend un
verdict : autoriser, autoriser sous conditions, ou refuser. Mesuré sur un cas trivial
(« `git push --force` sur main ») : **109 s** pour un verdict `deny` à **92 %** de
confiance. Deux conséquences : il ne se place pas sur le chemin d'une réponse ordinaire,
et sa confiance n'est pas calibrée — 92 % sur un cas de test en dit plus sur le modèle que
sur le risque réel.

## Le réacteur et le bridge MARI (couche 7)

Le réacteur ANIMA surveille l'état des composants et réagit aux transitions. Il ne répare
rien : il émet un signal vers un bridge MARI, un nœud Orleans hébergé en dehors du dépôt
d'exploitation, sur trois ports dédiés.

| Élément | Mesure du 09/11 |
|---|---|
| Silo Orleans | `127.0.0.1:55551`, en écoute |
| Gateway Orleans | `127.0.0.1:55552`, en écoute |
| REST | `127.0.0.1:55553` |
| `GET /status` | lecture seule — `{"status":"ok","signalsEmitted":1}` |
| `POST /fire` | opérationnel — 200 en 19 ms, `{"status":"signal emitted","total":1}` |

Le signal est compté. La remédiation, elle, n'est pas revendiquée : sur un tir de test, le
compteur s'incrémente, mais aucune demande n'est créée dans le répertoire `remediation/`
du bridge et rien n'apparaît au journal du watcher. La chaîne n'est donc démontrée que
jusqu'au compteur — signal émis, décision non observée.

Ce que le dépôt d'exploitation versionne de cette couche : les sources amont adaptées
(`mari-adapted/`), le lanceur à la demande (`scripts/launch_mari.py`) et le watcher
(`scripts/anima_watcher.py`). Le bridge lui-même — binaire, journal, kill-switch — vit
hors dépôt : un clone ne suffit pas à le faire tourner.

## Installation

Prérequis : Windows 11, PowerShell 5.1+, connexion réseau. Python, git et ffmpeg sont
posés par l'installeur. Aucun droit administrateur n'est nécessaire.

Le mode CLI est le mode nominal : **aucune interface graphique n'est requise**. Un
terminal suffit ; Telegram, Discord ou l'email sont des canaux d'entrée optionnels.

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

Le dépôt d'exploitation complet (configurations, scripts, données, secrets) reste
privé ; ce dépôt public en décrit l'architecture. Les secrets sont créés localement,
par profil, à partir de modèles `.env.example` — aucun n'est versionné.

## Limites

Ce wiki n'est pas un moteur de recherche. Tant que le corpus reste petit — l'ordre de
grandeur actuel est de 20 pages et 5 sources — lire `index.md` puis grep suffit, et
c'est plus fiable qu'un index approximatif.

L'implémentation de référence (94 articles, 99 sources en production quotidienne)
considère qu'au-delà de 50 000 à 100 000 tokens de wiki curé, la lecture et le grep
cessent d'être fiables et qu'il faut mesurer la perte de rappel avant d'ajouter de
l'outillage de recherche.

Seuil opérationnel retenu ici : **au-delà de 100 pages ou 100 000 tokens de wiki
compilé, il faut un index dédié au wiki**. Pas parce que la recherche hybride manque —
elle existe déjà — mais parce que le wiki est aujourd'hui noyé dans un index mutualisé :
349 fragments de wiki contre 3 353 fragments de skills. Le jour où le wiki pèse autant
que les skills, il faut le sortir dans son propre index, sinon une question sur le wiki
ramène des pages de skills. Ce seuil est une règle d'exploitation de ce dépôt, pas une
loi : il se vérifie en mesurant le rappel.

Autres limites, mesurées :

- Le RAG (L2) est **hybride**, pas seulement vectoriel : `POST /search` renvoie pour
  chaque résultat un rang lexical et un rang vectoriel (embeddings e5-base), sur
  `127.0.0.1:8200`. Mesuré le 09/11 : 4 284 fragments, index du 08/11 à 02:09 —
  skills 3 353, siyuan 493, wiki 349, wordpress 89.
- Les décisions de JEV ne sont pas calibrées : elles s'ordonnent, elles ne se lisent pas
  comme des probabilités.
- Les modèles gratuits de repli tombent régulièrement (429, 504) : ce sont des secours,
  jamais des cibles nominales.
- Le wiki ne se vérifie pas tout seul. Sans lint régulier, `index.md` dérive du disque.

## Références

- Andrej Karpathy, *LLM Wiki* (gist) — l'idée d'origine :
  https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- `Astro-Han/karpathy-llm-wiki` — implémentation de référence, MIT, Agent Skills :
  https://github.com/Astro-Han/karpathy-llm-wiki
- Agent Skills, le standard de packaging suivi ici : https://agentskills.io

## FAQ

**Ça tourne sans interface graphique ?**
Oui, c'est même le mode nominal. Tout passe par le terminal ; Telegram, Discord et
l'email sont des canaux d'entrée facultatifs. Aucun droit administrateur.

**Pourquoi deux modèles au lieu d'un ?**
Parce que décider coûte 50 fois moins cher que rédiger. JEV tranche pour ~1,5 × 10⁻⁵ $
avant que le modèle de génération ne soit appelé ; payer une génération complète pour
savoir « wiki ou RAG ? » serait du gaspillage.

**Comment j'ajoute une source ?**
Dépose le fichier en markdown dans `raw/notes/`, lance `compile_wiki.py --dry-run` pour
voir les pages qui seraient touchées, puis sans le flag. Le fichier brut n'est jamais
réécrit.

**Quelle différence avec un RAG classique ?**
Le RAG retrouve des morceaux bruts à chaque question et jette sa synthèse. Ici la
synthèse est écrite, datée, reliée et conservée. Les deux coexistent : le wiki est le
niveau L1, le RAG le niveau L2, et JEV choisit lequel interroger.

**Combien ça coûte ?**
Le palier gratuit d'abord. Une décision JEV coûte ~1,5 × 10⁻⁵ $. Un modèle premium
existe en dernier recours, appelé à la main, jamais automatiquement.

**Ça envoie mes données dans le cloud ?**
`raw/` et `wiki/` restent sur le disque. Les services annexes n'écoutent que sur
l'interface locale, et les secrets ne sont jamais versionnés : un `.env` par profil.

**Combien de pages avant que ça casse ?**
Rien ne casse brutalement : la lecture et le grep se dégradent. Repère : 50 000 à
100 000 tokens de wiki curé, soit environ 100 pages ici — c'est le moment de sortir le
wiki dans son propre index.

## Licence

© 2026 Thomas Leroyer (Searching Murphy). Le contenu de ce dépôt (documentation,
schémas) est sous **CC BY 4.0** — réutilisation, modification et redistribution
autorisées avec attribution. Texte complet dans [`LICENSE`](LICENSE).

Les skills commerciaux publiés séparément sur Agensi.io font l'objet d'une licence
propre ; leur mention ici ne confère aucun droit d'usage ni de redistribution.
