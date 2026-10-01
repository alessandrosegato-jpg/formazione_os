# vagrant-prov

Crea le macchine virtuali del laboratorio con Vagrant (provider VirtualBox) e genera l'inventory Ansible che i ruoli successivi (`docker`, `deploy-containers`) usano per collegarsi alle VM.

Il ruolo gira su **localhost**, cioè sulla macchina da cui lanci Ansible, non sulle VM.

## Cosa fa

1. **Crea la cartella di lavoro delle VM** (`vagrant_workdir`, di default `vms/` accanto al playbook). Lì il modulo scrive il `Vagrantfile` e Vagrant salva lo stato delle macchine (`.vagrant/`).
2. **Crea la cartella dell'inventory** (la cartella che contiene `vagrant_inventory_file`, di default `inventory/`).
3. **Crea e avvia le VM** con `community.vagrant.vagrant`, a partire dalla lista `vms`: nome, gruppo, RAM, CPU e IP sulla rete privata. Se le VM esistono già e sono accese, il task non cambia niente.
4. **Scrive l'inventory** `inventory/vagrant.ini` dal template `inventory.j2`. Per ogni VM crea:
   - la sezione del suo gruppo (`[elasticsearch_server]`, `[grafana_server]`, `[prometheus_server]`);
   - `ansible_host` con l'IP della rete privata;
   - `ansible_ssh_private_key_file` con la chiave che Vagrant genera per ogni VM (`vms/.vagrant/machines/<nome>/virtualbox/private_key`, perché `ssh.insert_key` è attivo);
   - in `[all:vars]`: l'utente `vagrant` e le opzioni SSH che non salvano le host key in `known_hosts`, perché le VM vengono ricreate spesso con gli stessi IP.
5. **Ricarica l'inventory** con `meta: refresh_inventory`. Ansible legge l'inventory una sola volta all'avvio del playbook: senza questo passaggio i play successivi (`hosts: all`) non vedrebbero le VM appena create. Il refresh viene eseguito a ogni run: costa pochissimo e non modifica niente sulle VM.

## Variabili (`defaults/main.yml`)

| Variabile | Valore | Descrizione |
|---|---|---|
| `vagrant_workdir` | `{{ playbook_dir }}/vms` | Cartella con `Vagrantfile` e stato delle VM |
| `vagrant_provider` | `virtualbox` | Provider di Vagrant |
| `vagrant_box` | `generic/rocky9` | Box usata per tutte le VM |
| `vagrant_inventory_file` | `{{ playbook_dir }}/inventory/vagrant.ini` | Inventory generato |
| `vms` | lista | Una voce per VM, con i campi descritti sotto |

Campi di ogni voce di `vms`:

| Campo | Esempio | Descrizione |
|---|---|---|
| `name` | `elasticsearch-host` | Nome della VM e hostname nell'inventory |
| `group` | `elasticsearch_server` | Gruppo Ansible. Solo lettere, numeri e `_`: il trattino genera un warning |
| `memory` | `1024` | RAM in MB |
| `cpus` | `1` | Numero di CPU |
| `interfaces[0].ip` | `192.168.3.40` | IP sulla rete privata, usato come `ansible_host` |

VM attualmente definite:

| VM | Gruppo | IP | RAM |
|---|---|---|---|
| `elasticsearch-host` | `elasticsearch_server` | 192.168.3.40 | 1024 MB |
| `monitoring-grafana` | `grafana_server` | 192.168.3.41 | 512 MB |
| `monitoring-prometheus` | `prometheus_server` | 192.168.3.42 | 512 MB |

I nomi dei gruppi sono usati dal ruolo `deploy-containers` per decidere cosa installare su ogni VM. I nomi delle VM sono usati come chiavi di `node_exporter_ports`.
