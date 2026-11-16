# Exercice Compose 2 : Quiz Docker avec Docker Compose

Le fichier `docker-compose.yml` construit l'image `quiz-docker:secure` à partir de `Dockerfile.secure`.
Il lance ensuite le service `quiz-docker` avec les protections du TP sécurité.

## Configuration

| Exigence | Mise en œuvre |
|---|---|
| Service `quiz-docker` | `services: quiz-docker:` |
| Construction automatique | `build.context: .` et `build.dockerfile: Dockerfile.secure` |
| Image `quiz-docker:secure` | `image: quiz-docker:secure` (nom donné à l'image construite) |
| Port 8082 sur l'hôte | `8082:8080` (nginx non‑root écoute sur 8080) |
| Redémarrage automatique | `restart: unless-stopped` : relancé après un crash ou un reboot, sauf s'il a été arrêté volontairement |
| Système de fichiers en lecture seule | `read_only: true` |
| Espace temporaire | `tmpfs` sur `/tmp` (16 Mo, `noexec`, `nosuid`) |
| Aucune capacité Linux | `cap_drop: [ALL]` |
| Pas d'élévation de privilèges | `security_opt: [no-new-privileges:true]` |
| Limites de ressources | 128 Mo de RAM, 0,5 CPU, 50 processus (`deploy.resources.limits`) |
| Utilisateur non‑root | défini dans l'image (`USER 101`) |
| Vérification de santé | définie dans l'image (`HEALTHCHECK`) |

## Commandes du cycle de vie

Toutes les commandes se lancent depuis `dockerfile-exercice2/`.

| Commande | Rôle |
|---|---|
| `docker compose config` | Valide le YAML et affiche la configuration finale interprétée |
| `docker compose up -d --build` | Construit l'image puis crée et démarre le conteneur en arrière‑plan |
| `docker compose ps` | Affiche l'état du service (`healthy` une fois le healthcheck passé) |
| `docker compose logs -f` | Suit les logs nginx en continu (Ctrl+C pour quitter) |
| `docker compose stop` | Arrête le conteneur sans le supprimer |
| `docker compose start` | Redémarre le conteneur existant |
| `docker compose restart` | Enchaîne `stop` et `start` |
| `docker compose down` | Arrête et supprime le conteneur et le réseau du projet (l'image est conservée) |

## Vérifications

```bash
curl -I http://localhost:8082
# HTTP/1.1 200 OK

docker inspect quiz-docker --format 'ReadOnly={{.HostConfig.ReadonlyRootfs}} CapDrop={{.HostConfig.CapDrop}} SecOpt={{.HostConfig.SecurityOpt}} Mem={{.HostConfig.Memory}} Pids={{.HostConfig.PidsLimit}} Restart={{.HostConfig.RestartPolicy.Name}} User={{.Config.User}}'
# ReadOnly=true CapDrop=[ALL] SecOpt=[no-new-privileges:true] Mem=134217728 Pids=50 Restart=unless-stopped User=101
```

Application accessible dans le navigateur : http://localhost:8082
