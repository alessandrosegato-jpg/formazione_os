# docker

Installa e avvia Docker CE sulle VM Rocky Linux / RHEL, e le prepara per essere gestite con i moduli `community.docker` di Ansible.

## Cosa fa

1. **Verifica il sistema operativo:** si ferma con un errore chiaro se l'host non è della famiglia RedHat (Rocky, RHEL, Alma…).
2. **Rimuove i pacchetti in conflitto** (`podman`, `buildah`, `runc`), solo se `docker_remove_conflicting` è `true`. Su RHEL/Rocky il `runc` di sistema va in conflitto con quello incluso in `containerd.io`, come indicato nella guida ufficiale di Docker.
3. **Importa la chiave GPG** del repository Docker, per verificare la firma dei pacchetti.
4. **Aggiunge il repository Docker CE** scaricando `docker-ce.repo` in `/etc/yum.repos.d/`.
5. **Installa Docker:** engine, CLI, containerd, plugin buildx e plugin compose.
6. **Installa le librerie Python** necessarie ai moduli `community.docker` sulle VM (`python3-requests`). Senza, i task `docker_container` e `docker_volume` del ruolo `deploy-containers` falliscono con `Failed to import the required Python library (requests)`.
7. **Avvia e abilita il servizio `docker`**, così riparte da solo al boot.
8. **Aggiunge gli utenti al gruppo `docker`** (`docker_users`, di default `vagrant`), così possono usare `docker` senza `sudo`. Usa `append: true` per non togliere gli altri gruppi dell'utente, come `wheel`.

## Variabili

### `defaults/main.yml`: configurabili

| Variabile | Valore | Descrizione |
|---|---|---|
| `docker_packages` | `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin` | Pacchetti di Docker |
| `docker_remove_conflicting` | `false` | Se `true`, disinstalla `docker_conflicting_packages` prima dell'installazione |
| `docker_conflicting_packages` | `podman`, `buildah`, `runc` | Pacchetti in conflitto con Docker CE |
| `docker_users` | `vagrant` | Utenti da aggiungere al gruppo `docker` |

### `vars/main.yml`: interne al ruolo

Hanno una precedenza alta e non sono pensate per essere cambiate dall'esterno: sono dettagli di implementazione, che il resto del ruolo dà per scontati.

| Variabile | Valore | Descrizione |
|---|---|---|
| `docker_repo_url` | `https://download.docker.com/linux/rhel/docker-ce.repo` | File del repository Docker CE |
| `docker_gpg_key` | `https://download.docker.com/linux/rhel/gpg` | Chiave GPG del repository |
| `docker_python_packages` | `python3-requests` | Librerie Python necessarie ai moduli `community.docker` |

## Verifica

Sulla VM:

```bash
systemctl is-active docker        # active
docker version                    # client e server
id vagrant                        # deve comparire il gruppo docker
docker ps                         # funziona senza sudo (in una nuova sessione SSH)
```
