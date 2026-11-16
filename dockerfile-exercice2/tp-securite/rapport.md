# TP : Sécurisation d'une image et d'un conteneur Docker (Quiz Docker)

Application : Quiz Docker (`dockerfile-exercice2`), application web statique construite avec Vite.

---

## Mission 1 : Audit de l'image initiale

### Dockerfile initial (`Dockerfile.initial`)

```dockerfile
FROM node:22

WORKDIR /app

COPY . .

RUN npm install \
  && npm install -g serve \
  && npm run build

EXPOSE 3000

CMD ["serve", "-s", "dist"]
```

### Commandes utilisées

```bash
docker build -f Dockerfile.initial -t quiz-docker:initial .
docker run -d -p 8095:3000 --name quiz-initial quiz-docker:initial   # application OK sur http://localhost:8095
grep FROM Dockerfile.initial
docker exec quiz-initial cat /etc/os-release | head -2
docker exec quiz-initial ps -o user,pid,cmd
docker image inspect quiz-docker:initial --format 'User: "{{.Config.User}}"'
docker images quiz-docker:initial
docker history quiz-docker:initial
docker scout quickview quiz-docker:initial
docker scout cves quiz-docker:initial
```

Résultats bruts : `audit-initial.txt`, `scan-initial-quickview.txt`, `scan-initial-cves.txt`.

### État initial de l'image

| Critère | Constat |
|---|---|
| Image de base | `node:22` : Debian 12 (bookworm) complète, basée sur `buildpack-deps` (image prévue pour compiler) |
| Tag | imprécis : `22` suit automatiquement les nouvelles versions 22.x |
| Utilisateur du processus | **root** (PID 1 `node /usr/local/bin/serve -s dist`, aucun `USER` défini) |
| Taille | **1,69 Go** sur disque (424 Mo compressée) |
| Couches principales | 619 Mo + 194 Mo + 51 Mo d'outils Debian, 133 Mo de Debian de base, 213 Mo de Node, 48,7 Mo pour le `RUN` de l'appli |
| Paquets Debian installés | 413 |
| Outils de développement présents | `gcc`, `make`, `git`, `curl`, `wget`, `python3`, `apt`, `ssh`, `perl` |
| `node_modules` dans l'image finale | oui (16 Mo, dont le binaire `esbuild`) |
| `.dockerignore` | absent |
| Multi‑stage build | non |
| Limitation des ressources / privilèges | aucune |
| CVE (Docker Scout) | **586** : 9 critiques, 150 élevées, 137 moyennes, 257 faibles, 33 non classées |
| Politiques Docker Scout | 2/7 respectées, note de santé **E (28 %)** |

### Origine des vulnérabilités critiques et élevées

| Paquet | Critiques + élevées | Origine | Nécessaire à l'exécution ? |
|---|---|---|---|
| `golang/stdlib 1.20.7` | 45 (dont 6 critiques) | binaire `esbuild` dans `node_modules` (outil de build de Vite, écrit en Go) | non |
| `postgresql-15` | 34 | librairies client fournies par l'image `node:22` | non |
| `openexr`, `libheif`, `tiff` | 28 | librairies de traitement d'images | non |
| `mariadb` | 9 (dont 2 critiques) | librairies client MariaDB | non |
| `vite`, `rollup`, `nanoid`, `brace-expansion` | ~13 | dépendances npm de build | non |

La quasi‑totalité des vulnérabilités vient de composants qui ne servent pas à faire tourner l'application.

### Risques identifiés

| Risque | Gravité | Explication |
|---|---|---|
| Exécution en root | élevée | une faille dans `serve` donne les droits root dans le conteneur et facilite une évasion vers l'hôte |
| Surface d'attaque | élevée | 586 CVE dont 9 critiques, 413 paquets : chaque paquet est une porte d'entrée possible |
| Outils exploitables par un attaquant | élevée | `curl`/`wget` (télécharger), `gcc`/`make` (compiler), `apt` (installer), `ssh` (rebondir) |
| Outils de build en production | moyenne | Node, npm, Vite, esbuild et `node_modules` n'ont aucune utilité une fois le site compilé |
| Tag imprécis | moyenne | le contenu de l'image change d'un build à l'autre, sans traçabilité |
| Pas de `.dockerignore` | moyenne | `COPY . .` embarque tout le dossier (un `.env` ou une clé partiraient aussi) |
| Aucune limite d'exécution | moyenne | mémoire, CPU, nombre de processus et capacités Linux illimités par défaut |

### Réponses aux questions

**L'image utilisée est‑elle adaptée à une utilisation en production ?**
Non. `node:22` est une image de développement et de compilation : 1,69 Go, 413 paquets, des compilateurs et des clients de bases de données. Docker Scout lui attribue la note E.

**Quels composants sont réellement nécessaires pour exécuter l'application ?**
Un serveur HTTP capable de servir des fichiers statiques, et le dossier `dist/` produit par `npm run build` (environ 300 Ko de HTML, CSS, JS et images). Node, npm, `node_modules`, Vite, esbuild et les outils Debian ne servent qu'au build.

**Quels risques de sécurité identifiez‑vous ?**
Voir le tableau ci‑dessus : exécution en root, surface d'attaque très large (586 CVE), outils exploitables par un attaquant, tag imprécis, absence de `.dockerignore` et de limites d'exécution.

**Quelles améliorations recommanderiez‑vous en priorité ?**

1. Build multi‑stage avec une image finale minimale : c'est ce qui supprime le plus de vulnérabilités d'un coup.
2. Exécution avec un utilisateur non‑root.
3. Tags d'images précis, idéalement épinglés par digest (`@sha256:...`).
4. Fichier `.dockerignore`.
5. Limites à l'exécution : `--memory`, `--cpus`, `--pids-limit`, `--read-only`, `--cap-drop ALL`, `--security-opt no-new-privileges`.
6. Scan de vulnérabilités régulier (Docker Scout).

---

## Mission 2 : Sécuriser l'image et le conteneur

### Dockerfile sécurisé (`Dockerfile.secure`)

```dockerfile
# ---------- Étape 1 : build (jetée à la fin) ----------
FROM node:22.23.5-alpine3.24@sha256:1314fb30ec37592d5845e5a160d522c0e680972aa534549940b8d73bde88481a AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---------- Étape 2 : image finale minimale ----------
FROM nginxinc/nginx-unprivileged:1.31-alpine3.24@sha256:b5047b9b76845eac124358373c6852377c52e4b05834a47647e955dd91cb8f1e
USER root
RUN apk upgrade --no-cache \
  && apk del --no-cache curl \
  && apk --purge del apk-tools
COPY --from=build /app/dist /usr/share/nginx/html
USER 101
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -q --spider http://127.0.0.1:8080/ || exit 1
```

### Fichier `.dockerignore`

```
.git
.gitignore
node_modules
dist
Dockerfile*
.dockerignore
*.md
.env
.env.*
*.pem
*.key
**/*.old
**/*.bak
**/*~
```

Exemple concret : le dossier `public/` contenait `vite.svg.old`. Vite copie `public/` tel quel dans `dist/`, ce fichier de sauvegarde aurait donc été servi publiquement. Le `.dockerignore` l'exclut (vérifié : absent de l'image).

### Commandes utilisées

```bash
docker build -f Dockerfile.secure -t quiz-docker:secure .

docker run -d --name quiz-secure -p 8096:8080 \
  --read-only --tmpfs /tmp:rw,noexec,nosuid,size=16m \
  --cap-drop ALL --security-opt no-new-privileges:true \
  --memory 128m --cpus 0.5 --pids-limit 50 \
  --restart unless-stopped \
  quiz-docker:secure

docker scout quickview quiz-docker:secure
docker scout cves quiz-docker:secure
```

Résultats bruts : `scan-secure-quickview.txt`, `scan-secure-cves.txt`.

### Application des recommandations

| Recommandation | Mise en œuvre |
|---|---|
| Images de base minimales | `node:*-alpine` pour le build, `nginx-unprivileged` Alpine pour l'exécution. Ni Node, ni npm, ni compilateur dans l'image finale. |
| Pas d'exécution en root | Image `nginx-unprivileged` (nginx officiel prévu pour l'UID 101, port 8080 car un port < 1024 demande root). `USER 101` explicite. Vérifié : `uid=101(nginx)`. |
| `.dockerignore` | Exclut Git, dépendances locales, Dockerfiles, documentation, secrets et fichiers de sauvegarde. |
| Tags précis | Version complète **et** digest `@sha256:` : l'image de base ne peut pas changer sans modification du Dockerfile. |
| `COPY` plutôt que `ADD` | Uniquement des `COPY`. `ADD` télécharge des URL et décompresse des archives implicitement. |
| Limiter ressources et privilèges | `--memory 128m`, `--cpus 0.5`, `--pids-limit 50`, `--read-only` (+ `tmpfs` `noexec` pour `/tmp`), `--cap-drop ALL`, `--security-opt no-new-privileges:true`. |
| Build multi‑étapes | Étape `build` (Node) jetée, seul `dist/` (53 Ko) passe dans l'image finale. |
| Pas d'accès au socket Docker | Aucun montage de `/var/run/docker.sock` (vérifié : `Mounts` vide). Ce socket donnerait le contrôle total de Docker, donc de l'hôte. |
| Vérification de la sécurité | Scan Docker Scout avant et après. `HEALTHCHECK` pour vérifier le bon fonctionnement en continu. |
| Mesure complémentaire | `apk upgrade` (correctifs Alpine publiés après l'image de base), suppression de `curl` et du gestionnaire de paquets `apk`. |

### Étape intermédiaire : l'effet de `apk upgrade`

Un premier build sans `apk upgrade` donnait encore **9 critiques et 27 élevées**, toutes issues de l'image de base `nginx-unprivileged` (Alpine 3.24.2) : `curl` (7C 17H), `zlib` (1C 1H), `brotli` (1C), `expat`, `libxslt`, `musl`. Des correctifs existaient côté Alpine (3.24.4), mais l'image de l'éditeur n'avait pas encore été reconstruite. `apk upgrade` les applique, et la suppression de `curl` (inutile : le `HEALTHCHECK` utilise `wget` de busybox) retire 24 CVE à elle seule.

Contrepartie : la taille passe de 91 à 104 Mo. Les nouvelles versions sont écrites dans une nouvelle couche, les anciennes restent dans la couche de l'image de base (une couche ne peut pas libérer l'espace des couches précédentes).

---

## Mission 3 : Comparaison avant / après

### Tableau comparatif

| Critère | `quiz-docker:initial` | `quiz-docker:secure` |
|---|---|---|
| Image de base | `node:22` (Debian 12 complète) | `nginxinc/nginx-unprivileged:1.31-alpine3.24` (Alpine 3.24.4 après mise à jour) |
| Tag | imprécis (`22`) | version complète + digest `@sha256:` |
| Taille | **1,69 Go** | **104 Mo** (÷ 16) |
| Serveur | `serve` (Node.js) | nginx |
| Utilisateur | **root** | **nginx (UID 101)** |
| Outils de développement présents | `gcc`, `make`, `git`, `curl`, `wget`, `python3`, `apt`, `ssh`, `perl`, `node`, `npm` | `wget` (busybox, utilisé par le `HEALTHCHECK`) |
| Gestionnaire de paquets | `apt` présent | supprimé (`apk` retiré) |
| `node_modules` / outils de build | présents | absents |
| `.dockerignore` | non | oui |
| Multi‑stage build | non | oui |
| `HEALTHCHECK` | non | oui |
| Système de fichiers | lecture/écriture | **lecture seule** (`--read-only`, `/tmp` en `tmpfs noexec`) |
| Limitation des ressources | aucune | 128 Mo RAM, 0,5 CPU, 50 processus max |
| Restriction des privilèges | aucune | `--cap-drop ALL`, `no-new-privileges` |
| Accès au socket Docker | non monté | non monté |
| CVE critiques | **9** | **0** |
| CVE élevées | **150** | **2** |
| CVE moyennes | 137 | 5 |
| CVE faibles | 257 | 5 |
| CVE non classées | 33 | 5 |
| **Total CVE** | **586** (69 paquets) | **17** (3 paquets) |
| Politiques Docker Scout | 2/7 | 4/7 |
| Note de santé Docker Scout | **E (28 %)** | **C (56 %)** |

### Réponses aux questions

**Quelles modifications ont apporté les améliorations les plus significatives ?**

1. Le **build multi‑stage avec une image finale minimale** : de 586 à 52 CVE et de 1,69 Go à 91 Mo. Il élimine d'un coup Debian, les compilateurs, Node, npm et `node_modules` (dont le binaire `esbuild`, 45 CVE critiques et élevées à lui seul).
2. La **mise à jour des paquets Alpine et la suppression de `curl`** : de 9 critiques à 0.
3. L'**utilisateur non‑root** combiné aux **options d'exécution** (`--read-only`, `--cap-drop ALL`, `no-new-privileges`) : elles ne changent pas le nombre de CVE, mais limitent fortement ce qu'un attaquant peut faire s'il exploite une faille.

**La taille de l'image a‑t‑elle diminué ? Pourquoi ?**

Oui, de 1,69 Go à 104 Mo (÷ 16). L'image finale ne contient plus que nginx sur Alpine et les 53 Ko du site compilé. Tout ce qui sert à construire l'application reste dans l'étape `build`, qui n'est pas conservée. Alpine (musl, busybox) est aussi bien plus léger que Debian.

**Quelles recommandations concernent le Dockerfile et lesquelles concernent l'exécution du conteneur ?**

| Dockerfile (construction de l'image) | Exécution (`docker run` ou orchestrateur) |
|---|---|
| Image de base minimale | Limitation mémoire / CPU / processus |
| Tags précis et digest | Système de fichiers en lecture seule |
| `.dockerignore` | Suppression des capacités Linux (`--cap-drop`) |
| `COPY` plutôt que `ADD` | `no-new-privileges` |
| Multi‑stage build | Ne pas monter le socket Docker |
| `USER` non‑root | |
| `HEALTHCHECK` | |
| Scan des vulnérabilités (à chaque build, et régulièrement) | |

Le non‑root est à cheval : défini dans l'image (`USER`), mais forçable aussi à l'exécution (`--user`). Les protections d'exécution ne sont pas portées par l'image : il faut les reporter dans chaque environnement (fichier Compose, manifeste Kubernetes…).

**Votre image est‑elle désormais totalement sécurisée ? Justifiez votre réponse.**

Non. Elle est bien plus sûre, mais :

- il reste **17 CVE**, dont 2 élevées (`libxml2`, `musl`) sans correctif disponible à la date du scan ;
- de nouvelles CVE sont publiées chaque jour : une image sûre aujourd'hui ne le sera plus dans quelques semaines sans reconstruction ;
- nginx lance un processus de travail par cœur du serveur hôte (20 ici) sans tenir compte de `--cpus` : en production, il faudrait fixer `worker_processes` ;
- sans *user namespaces*, l'UID 101 du conteneur est le même UID sur l'hôte (`docker top` l'affiche sous le nom `uuidd`, le compte UID 101 de l'hôte) ;
- `apk upgrade` rend le build légèrement non reproductible (les correctifs appliqués dépendent du jour du build) ;
- l'image n'est ni signée ni accompagnée d'attestations de provenance (politique Scout « supply chain attestations » non respectée) ;
- la sécurité de l'hôte, du réseau, du registre et du code de l'application reste hors du périmètre de l'image.

**Quelles mesures complémentaires proposeriez‑vous pour un déploiement en production ?**

- Scan automatique dans la CI à chaque build, avec blocage si une CVE critique est détectée, et scan planifié des images déjà déployées.
- Mise à jour automatique des images de base (Renovate ou Dependabot) plutôt qu'un `apk upgrade`.
- Signature des images (Docker Content Trust, cosign) et génération de SBOM et d'attestations de provenance (`docker build --sbom=true --provenance=true`).
- Utilisation d'une **Docker Hardened Image** (voir mission 4).
- Configuration nginx durcie : `worker_processes` fixe, en‑têtes de sécurité (CSP, `X-Content-Type-Options`…), `server_tokens off`.
- HTTPS via un reverse proxy ou un load balancer.
- Activation des *user namespaces* (`userns-remap`) ou du mode Docker *rootless* sur l'hôte.
- Profils seccomp ou AppArmor adaptés.
- Supervision et centralisation des logs.

### Publication sur Docker Hub

```bash
docker tag quiz-docker:secure clementraoult/quiz-docker:secure
docker tag quiz-docker:initial clementraoult/quiz-docker:initial
docker push clementraoult/quiz-docker:secure
docker push clementraoult/quiz-docker:initial
```

Images publiées : https://hub.docker.com/r/clementraoult/quiz-docker/tags

- `clementraoult/quiz-docker:initial`
- `clementraoult/quiz-docker:secure`

---

## Mission 4 : Docker Hardened Image

### Choix de la méthode

L'atelier officiel (https://docs.docker.com/guides/lab-dhi-node/) se lance comme un « labspace » (`docker compose -f oci://dockersamples/labspace-dhi-node up -d`). Avant de le lancer, son fichier Compose a été inspecté (`docker compose ... config`). Il :

- monte le socket Docker de l'hôte (`/var/run/docker.sock`, via un proxy filtrant) ;
- transmet aux conteneurs les identifiants de registres de l'hôte (`use_api_socket: true`, volume `docker-creds`).

Sur un poste professionnel qui détient des accès à des registres d'entreprise, ce n'est pas acceptable : c'est précisément ce que la mission 2 recommande d'éviter. Les étapes de l'atelier (analyse, migration multi‑stage vers DHI, comparaison, inspection des attestations) ont donc été reproduites à la main, directement sur l'application Quiz, ce qui permet en plus une comparaison directe avec notre image sécurisée.

### Accès aux images DHI

Les DHI sont publiées sur le registre `dhi.io`, qui n'accepte pas les accès anonymes. Connexion avec le compte Docker Hub :

```bash
docker login dhi.io -u clementraoult    # mot de passe = Personal Access Token Docker Hub
```

Deux variantes par image :

- **`-dev`** : contient les outils de build (npm, shell, gestionnaire de paquets), s'exécute en root. Utilisée uniquement dans l'étape de build.
- **runtime** (sans suffixe) : minimale, non‑root (UID 65532), sans gestionnaire de paquets. Utilisée pour l'image finale.

### Dockerfile DHI (`Dockerfile.dhi`)

```dockerfile
FROM dhi.io/node:22-alpine3.24-dev@sha256:55a3bbae91f21a08ae5769898c4601c5bb075c7dbc4836fd6fea8510c5cb8d63 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM dhi.io/nginx:1.31-alpine3.24@sha256:34c5f1f63c57c0fb1b52505aecbb213520e15918dc822eb16371edff5183cbd1
COPY --from=build /app/dist /usr/share/nginx/html
USER 65532
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -q --spider http://127.0.0.1:8080/ || exit 1
```

Par rapport à `Dockerfile.secure`, plus besoin de `apk upgrade` ni de supprimer `curl` ou `apk` : l'image DHI est déjà à jour, ne contient pas `curl` et n'a pas de gestionnaire de paquets.

### Commandes utilisées

```bash
docker build -f Dockerfile.dhi -t quiz-docker:dhi .

docker run -d --name quiz-dhi -p 8097:8080 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=16m \
  --tmpfs /var/lib/nginx/tmp:rw,noexec,nosuid,size=16m,uid=65532,gid=65532 \
  --tmpfs /run/nginx:rw,noexec,nosuid,size=1m,uid=65532,gid=65532 \
  --cap-drop ALL --security-opt no-new-privileges:true \
  --memory 128m --cpus 0.5 --pids-limit 50 \
  --restart unless-stopped \
  quiz-docker:dhi

docker scout quickview quiz-docker:dhi
docker scout cves quiz-docker:dhi
docker scout attest list dhi.io/nginx:1.31-alpine3.24
```

Différence d'exécution : la configuration nginx de l'image DHI écrit ses fichiers temporaires dans `/var/lib/nginx/tmp` et son PID dans `/run/nginx`. Avec `--read-only`, ces deux dossiers doivent être fournis en `tmpfs`, appartenant à l'UID 65532 (sinon nginx s'arrête au démarrage avec `Read-only file system`).

### Comparaison avec l'image sécurisée

| Critère | `quiz-docker:secure` | `quiz-docker:dhi` |
|---|---|---|
| Image de base (runtime) | `nginxinc/nginx-unprivileged:1.31-alpine3.24` | `dhi.io/nginx:1.31-alpine3.24` |
| Taille | 104 Mo | **14,9 Mo** (÷ 7) |
| Paquets dans l'image de base | 71 | **15** |
| Utilisateur | UID 101 | UID 65532 |
| Gestionnaire de paquets | supprimé à la main (`apk del apk-tools`) | absent d'origine |
| `curl` | supprimé à la main | absent d'origine |
| Shell | oui (busybox) | oui (busybox, variante Alpine) |
| Corrections manuelles dans le Dockerfile | `apk upgrade`, `apk del` | aucune |
| CVE critiques | 0 | 0 |
| CVE élevées | 2 | 1 (`musl`, correctif publié après l'image) |
| CVE moyennes | 5 | 0 |
| CVE faibles | 5 | 0 |
| CVE non classées | 5 | 5 (`openssl`) |
| **Total CVE** | **17** | **6** |
| Note Docker Scout | C (56 %) | C (56 %) |
| SBOM fourni par l'éditeur | non | oui (CycloneDX et SPDX) |
| Provenance SLSA | non | oui (build traçable jusqu'à sa définition sur GitHub) |
| VEX (CVE déclarées non exploitables) | non | oui (OpenVEX) |
| Autres attestations | aucune | scan de secrets, scan antivirus, tests, journal des modifications |

### Analyse

- **Contenu** : l'image DHI ne contient que ce qui est nécessaire à nginx (15 paquets, 4,2 Mo pour la base). Le travail de réduction de la surface d'attaque fait à la main dans `Dockerfile.secure` est fait en amont par l'éditeur, et de façon plus poussée.
- **Vulnérabilités** : 6 CVE contre 17, sans aucune correction manuelle. L'image reste toutefois dépendante du rythme de reconstruction de l'éditeur : la CVE `musl` restante est corrigée dans une version publiée après la construction de l'image.
- **Chaîne d'approvisionnement** : c'est l'apport principal des DHI. SBOM signés, provenance SLSA et VEX permettent de savoir précisément ce que contient l'image, d'où elle vient, et quelles CVE sont réellement exploitables. Le VEX peut être exporté vers d'autres scanners (Grype, Trivy) pour qu'ils ne signalent pas de faux positifs.
- **Note Scout identique (C)** : les politiques non respectées sont communes aux deux images (licences copyleft, attestations de supply chain absentes sur **notre** image finale, qui n'est ni signée ni accompagnée de SBOM).
- **Limites** : la variante Alpine contient encore un shell busybox. La variante Debian (`dhi.io/nginx:1`, 50 Mo) n'a pas de shell du tout (« distroless »), mais le `HEALTHCHECK` par `wget` n'y est alors plus possible : la vérification de santé doit être faite par l'orchestrateur. L'accès à `dhi.io` demande aussi un compte Docker.

**Conclusion** : pour la production, l'image DHI est préférable. Elle est plus petite, contient moins de vulnérabilités, ne nécessite aucun correctif manuel dans le Dockerfile et apporte la traçabilité (SBOM, provenance, VEX) qui manque à l'image construite à la main.

---

## Synthèse

### Point de départ

L'image initiale (`node:22`, un seul stage) fonctionnait, mais elle embarquait un environnement de développement complet pour servir quelques centaines de Ko de fichiers statiques : 1,69 Go, 413 paquets Debian, des compilateurs et des outils réseau, un processus exécuté en root, et 586 vulnérabilités dont 9 critiques. Presque toutes ces vulnérabilités venaient de composants inutiles à l'exécution (binaire `esbuild`, librairies PostgreSQL, MariaDB et de traitement d'images).

### Améliorations apportées

| Étape | Effet principal |
|---|---|
| Build multi‑stage + image finale nginx Alpine | 1,69 Go → 91 Mo, 586 → 52 CVE |
| Correctifs Alpine (`apk upgrade`) + suppression de `curl` et `apk` | 9 → 0 CVE critiques, plus de gestionnaire de paquets |
| Image `nginx-unprivileged` + `USER` | exécution en UID 101 au lieu de root |
| Tags précis épinglés par digest | image de base figée et traçable |
| `.dockerignore` | contexte de build maîtrisé, aucun fichier sensible ou de sauvegarde dans l'image (un `vite.svg.old` aurait sinon été publié) |
| Options d'exécution | système de fichiers en lecture seule, aucune capacité Linux, pas d'élévation de privilèges, mémoire / CPU / processus limités |
| `HEALTHCHECK` | état réel de l'application visible par Docker |
| Migration vers une Docker Hardened Image | 104 → 14,9 Mo, 17 → 6 CVE, sans correction manuelle, avec SBOM, provenance SLSA et VEX |

| | Initiale | Sécurisée | DHI |
|---|---|---|---|
| Taille | 1,69 Go | 104 Mo | 14,9 Mo |
| Utilisateur | root | UID 101 | UID 65532 |
| CVE critiques / élevées | 9 / 150 | 0 / 2 | 0 / 1 |
| Total CVE | 586 | 17 | 6 |

L'amélioration la plus forte vient de la séparation entre la construction et l'exécution (multi‑stage) : on ne livre que ce qui sert à faire tourner l'application. Les options d'exécution, elles, ne réduisent pas le nombre de vulnérabilités, mais limitent fortement ce qu'un attaquant pourrait faire en exploitant l'une d'elles.

### Risques résiduels

- Quelques CVE restent présentes (2 élevées sur l'image sécurisée, 1 sur l'image DHI), sans correctif disponible ou corrigées après la construction de l'image de base.
- De nouvelles CVE apparaissent chaque jour : sans reconstruction et scan réguliers, l'image se dégrade avec le temps.
- Les options d'exécution ne sont pas portées par l'image : elles doivent être reprises dans chaque environnement (Compose, Kubernetes…).
- Sans *user namespaces* ou mode *rootless*, l'UID du conteneur est le même sur l'hôte.
- nginx ignore la limite `--cpus` pour dimensionner ses processus (`worker_processes` à fixer).
- Notre image finale n'est ni signée ni accompagnée de SBOM ou d'attestation de provenance.
- Les variantes Alpine conservent un shell busybox.
- Le code de l'application, le réseau, le registre et l'hôte restent hors du périmètre de ce TP.

### Recommandations pour la production

1. Partir d'une Docker Hardened Image pour l'image finale.
2. Intégrer le scan Docker Scout dans la CI, avec blocage sur les CVE critiques, et scanner régulièrement les images déployées.
3. Automatiser la mise à jour des images de base (Renovate, Dependabot).
4. Signer les images et publier SBOM et provenance (`docker build --sbom=true --provenance=true`).
5. Reporter les options d'exécution dans le fichier de déploiement, et activer *user namespaces* ou le mode *rootless* sur les hôtes.
6. Durcir la configuration nginx (`worker_processes`, `server_tokens off`, en‑têtes de sécurité) et servir l'application en HTTPS.

### Livrables

| Livrable | Emplacement |
|---|---|
| Dockerfile initial | `dockerfile-exercice2/Dockerfile.initial` |
| Dockerfile sécurisé | `dockerfile-exercice2/Dockerfile.secure` |
| Dockerfile DHI | `dockerfile-exercice2/Dockerfile.dhi` |
| `.dockerignore` | `dockerfile-exercice2/.dockerignore` |
| Commandes | sections « Commandes utilisées » des missions 1, 2 et 4 |
| Résultats des analyses | `tp-securite/scan-*.txt`, `audit-initial.txt`, `attestations-dhi.txt` |
| Tableau comparatif | mission 3 (et mission 4 pour l'image DHI) |
| Dépôt Git | https://github.com/clementraoult/docker-introduction (dossier `dockerfile-exercice2`) |
| Images publiées | https://hub.docker.com/r/clementraoult/quiz-docker/tags |
