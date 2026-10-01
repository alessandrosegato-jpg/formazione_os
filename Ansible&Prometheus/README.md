# Ansible & Prometheus

Progetto Ansible che crea da zero un piccolo laboratorio di monitoring:

1. crea **3 VM Rocky Linux 9** con Vagrant e VirtualBox, e genera l'inventory;
2. installa **Docker** su tutte le VM;
3. installa **node_exporter** su tutte le VM (servizio systemd utente, non root) e fa il deploy dei container **Elasticsearch**, **Prometheus** e **Grafana**, con la dashboard *Node Exporter Full* già importata.

## Architettura

| VM | IP | Cosa gira | Porte |
|---|---|---|---|
| `elasticsearch-host` | 192.168.3.40 | Elasticsearch (container), node_exporter | 9200, 9100 |
| `monitoring-grafana` | 192.168.3.41 | Grafana (container), node_exporter | 3000, 9101 |
| `monitoring-prometheus` | 192.168.3.42 | Prometheus (container), node_exporter | 9090, 9102 |

Prometheus raccoglie le metriche dei 3 node_exporter (`http://<ip>:<porta>/metrics`). Grafana usa Prometheus come datasource.

## Struttura del progetto

```
.
├── ansible.cfg              # inventory = inventory/, niente controllo host key
├── playbook.yaml            # playbook principale
├── requirements.yaml        # collection Ansible necessarie
├── group_vars/
│   └── all/vault.yaml       # segreti cifrati con Ansible Vault
└── roles/
    ├── vagrant-prov/        # controlli iniziali, creazione VM, inventory
    ├── docker/              # installazione Docker CE
    └── deploy-containers/   # node_exporter, Elasticsearch, Prometheus, Grafana
```

## Prerequisiti

### Sulla macchina da cui lanci Ansible

| Requisito | Note |
|---|---|
| **Ansible** (`ansible-core`) | Versione recente: controlla con `ansible --version` |
| **Collection Ansible** | Elencate in `requirements.yaml`, vedi la tabella qui sotto |
| **Vagrant** | Crea e gestisce le VM |
| **VirtualBox** | Provider usato da Vagrant |
| **CPU x86_64** | La box `generic/rocky9` per VirtualBox è `amd64` |
| **RAM libera** | Circa 2 GB per le 3 VM (1024 + 512 + 512 MB) |

### Collection (`requirements.yaml`)

| Collection | Usata per |
|---|---|
| `community.vagrant` | creare le VM (`community.vagrant.vagrant`) |
| `community.docker` | container e volumi (`docker_container`, `docker_volume`) |
| `ansible.posix` | aprire le porte in firewalld (`ansible.posix.firewalld`) |
| `community.grafana` | importare la dashboard (`grafana_dashboard`) |
| `community.general` | controllo iniziale delle collection (lookup `collection_version`) |

Sono tutte collection **non incluse in `ansible-core`**, quindi vanno installate esplicitamente, anche `community.general`.

## Setup

1. **Installa le collection**, dalla cartella del progetto:

   ```bash
   ansible-galaxy collection install -r requirements.yaml
   ```

2. **Crea il vault** con la password dell'utente `admin` di Grafana:

   ```bash
   ansible-vault create group_vars/all/vault.yaml
   ```

   con il contenuto:

   ```yaml
   vault_grafana_admin_password: 'UnaPassword!'
   ```

## Esecuzione

Sempre **dalla cartella del progetto**, così viene letto `ansible.cfg`:

```bash
ansible-playbook playbook.yaml --ask-vault-pass
```

Il playbook esegue nell'ordine:

1. **`vagrant-prov`** (localhost): controlli iniziali, creazione delle VM, generazione e ricaricamento dell'inventory;
2. **`docker`** (tutte le VM): installazione di Docker CE;
3. **`deploy-containers`** (tutte le VM): node_exporter su tutte, e su ognuna il container del proprio gruppo.

Il playbook è idempotente: rilanciandolo senza modifiche, i task risultano `ok` e niente viene ricreato.

## Verifica

| Servizio | Indirizzo | Cosa controllare |
|---|---|---|
| node_exporter | `http://192.168.3.40:9100/metrics` (e .41:9101, .42:9102) | risponde con le metriche |
| Elasticsearch | `http://192.168.3.40:9200` | JSON con `"tagline": "You Know, for Search"` |
| Prometheus | `http://192.168.3.42:9090/targets` | job `nodi` con 3 target **UP** |
| Grafana | `http://192.168.3.41:3000` | login `admin`, dashboard *Node Exporter Full* con le 3 VM |

