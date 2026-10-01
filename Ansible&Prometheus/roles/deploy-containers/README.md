# deploy-containers

Installa lo stack di monitoring sulle VM create da `vagrant-prov`:

- **node_exporter** su **tutte** le VM, come servizio systemd **utente non root** (non in Docker), ognuno su una porta diversa;
- **Elasticsearch** (container) sulla VM del gruppo `elasticsearch_server`, con heap JVM a 1G;
- **Prometheus** (container) sulla VM del gruppo `prometheus_server`, che raccoglie le metriche dei 3 node_exporter;
- **Grafana** (container) sulla VM del gruppo `grafana_server`, con Prometheus come datasource e la dashboard **Node Exporter Full** (ID 1860) importata automaticamente.

## Struttura

```
tasks/
├── main.yml                    # sceglie cosa installare su ogni VM
├── node_exporter.yaml          # tutte le VM
├── elasticsearch-host.yaml     # solo gruppo elasticsearch_server
├── monitoring-prometheus.yaml  # solo gruppo prometheus_server
└── monitoring-grafana.yaml     # solo gruppo grafana_server
templates/
├── node_exporter.service.j2    # unit systemd utente
├── heap.options.j2             # heap JVM di Elasticsearch
├── prometheus.yaml.j2          # configurazione di Prometheus
└── grafana-datasource.yaml.j2  # datasource Prometheus per Grafana
handlers/main.yml               # riavvio dei servizi quando cambia la configurazione
defaults/main.yml               # versioni, porte, percorsi (sovrascrivibili)
```

`tasks/main.yml` include `node_exporter.yaml` su tutti gli host. Gli altri file li include con `when: "'<gruppo>' in group_names"`, cioè solo sulle VM che appartengono a quel gruppo dell'inventory.

## Cosa fa

### node_exporter (tutte le VM)

1. Crea l'utente `node_exporter`, con home in `/home/node_exporter` e shell `/sbin/nologin`: un utente non privilegiato, senza login.
2. Scarica ed estrae il tarball ufficiale (`node_exporter-1.12.1.linux-amd64.tar.gz` da GitHub) direttamente sulla VM, in `/tmp`. Grazie a `creates`, se è già estratto non lo riscarica.
3. Copia il binario in `/usr/local/bin/node_exporter` (proprietario root, `0755`).
4. Abilita il **linger** per l'utente (`loginctl enable-linger`): il systemd dell'utente parte al boot e resta attivo anche senza login.
5. Crea `~/.config/systemd/user/` e ci scrive `node_exporter.service`. È un servizio **utente**: niente `User=`/`Group=`, e `WantedBy=default.target`.
6. Apre la porta dell'host in firewalld (`node_exporter_port/tcp`, permanente e immediata). Serve perché node_exporter non è un container: Prometheus da un'altra VM altrimenti non lo raggiungerebbe.
7. Avvia e abilita il servizio con `scope: user`, eseguito come l'utente `node_exporter` (`become_user`).

La porta di ogni VM viene da `node_exporter_ports` ed è passata al servizio con `--web.listen-address=:<porta>`. Le metriche sono esposte su `http://<ip>:<porta>/metrics`.

### Elasticsearch (gruppo `elasticsearch_server`)

1. Crea `/etc/elasticsearch/jvm.options.d/` sulla VM.
2. Scrive `heap.options` con `-Xms1g` e `-Xmx1g` (valore da `elasticsearch_heap`).
3. Crea il volume Docker `elasticsearch_data`.
4. Avvia il container `docker.elastic.co/elasticsearch/elasticsearch:8.18.0`:
   - porta `9200`;
   - `discovery.type=single-node` (un solo nodo, senza cluster);
   - `xpack.security.enabled=false` (niente HTTPS e password, per il laboratorio);
   - bind mount **in sola lettura** di `heap.options` in `/usr/share/elasticsearch/config/jvm.options.d/heap.options`;
   - volume `elasticsearch_data` su `/usr/share/elasticsearch/data`.

Se `heap.options` cambia, l'handler riavvia il container, così la JVM riparte con il nuovo heap.

### Prometheus (gruppo `prometheus_server`)

1. Crea `/etc/prometheus/` sulla VM.
2. Genera `/etc/prometheus/prometheus.yml` dal template: job `nodi`, `metrics_path: /metrics`, `scrape_interval: 15s` e **tre static target** (uno per VM), costruiti con un ciclo su `node_exporter_ports`, prendendo l'IP di ogni host da `hostvars[host].ansible_host`.
3. Crea il volume persistente `prometheus_data`.
4. Avvia il container `prom/prometheus:v3.4.0`:
   - porta `9090`;
   - bind mount **della cartella** `/etc/prometheus` in sola lettura. Si monta la cartella e non il singolo file, così il container vede sempre il file aggiornato da `template`;
   - volume `prometheus_data` su `/prometheus` (database TSDB).

Il file deve chiamarsi `prometheus.yml`, perché è il percorso con cui l'immagine avvia Prometheus (`--config.file=/etc/prometheus/prometheus.yml`). Se la configurazione cambia, l'handler riavvia il container.

### Grafana (gruppo `grafana_server`)

1. Crea `/etc/grafana/provisioning/datasources/` sulla VM.
2. Scrive il datasource **Prometheus** (provisioning): `uid: prometheus`, `access: proxy`, predefinito (`isDefault: true`), con URL `http://<ip VM Prometheus>:9090` ricavato dall'inventory (`groups['prometheus_server'][0]`).
3. Crea il volume persistente `grafana_data`.
4. Avvia il container `grafana/grafana:12.0.0`:
   - porta `3000`;
   - password dell'utente `admin` da `GF_SECURITY_ADMIN_PASSWORD` (dal vault);
   - bind mount in sola lettura di `/etc/grafana/provisioning`;
   - volume `grafana_data` su `/var/lib/grafana` (database di Grafana: utenti, dashboard, impostazioni).
5. Attende che Grafana risponda su `/api/health` (fino a 30 tentativi, uno ogni 2 secondi).
6. Importa la dashboard **Node Exporter Full** (ID `1860`, revisione `41`) con `community.grafana.grafana_dashboard`, con `overwrite: true` per poter rilanciare il playbook.

## Handler

| Handler | Quando parte | Cosa fa |
|---|---|---|
| `Riavvia node_exporter` | cambia il binario o la unit | `systemctl --user restart` (con `daemon_reload`) come utente `node_exporter` |
| `Riavvia elasticsearch` | cambia `heap.options` | riavvia il container |
| `Riavvia prometheus` | cambia `prometheus.yml` | riavvia il container |
| `Riavvia grafana` | cambia il datasource | riavvia il container |

Gli handler servono perché i programmi leggono la configurazione solo all'avvio. Partono solo se il file è cambiato davvero, e una sola volta a fine play.

## Variabili (`defaults/main.yml`)

### node_exporter

| Variabile | Valore | Descrizione |
|---|---|---|
| `node_exporter_version` | `1.12.1` | Versione da scaricare |
| `node_exporter_arch` | `linux-amd64` | Architettura |
| `node_exporter_release` | `node_exporter-<versione>.<arch>` | Nome del tarball e della cartella estratta |
| `node_exporter_url` | URL della release su GitHub | Da dove scaricare il tarball |
| `node_exporter_user` | `node_exporter` | Utente non root che esegue il servizio |
| `node_exporter_home` | `/home/node_exporter` | Home dell'utente (contiene `.config/systemd/user/`) |
| `node_exporter_ports` | `elasticsearch-host: 9100`, `monitoring-grafana: 9101`, `monitoring-prometheus: 9102` | Porta per ogni VM. Le chiavi devono essere i nomi degli host nell'inventory |
| `node_exporter_port` | `{{ node_exporter_ports[inventory_hostname] }}` | Porta dell'host corrente, ricavata dal dizionario |

### Elasticsearch

| Variabile | Valore |
|---|---|
| `elasticsearch_container_name` | `elasticsearch` |
| `elasticsearch_config_dir` | `/etc/elasticsearch` |
| `elasticsearch_data_volume` | `elasticsearch_data` |
| `elasticsearch_image` | `docker.elastic.co/elasticsearch/elasticsearch` |
| `elasticsearch_version` | `8.18.0` |
| `elasticsearch_port` | `9200` |
| `elasticsearch_heap` | `1g` |

### Prometheus

| Variabile | Valore |
|---|---|
| `prometheus_container_name` | `prometheus` |
| `prometheus_config_dir` | `/etc/prometheus` |
| `prometheus_data_volume` | `prometheus_data` |
| `prometheus_image` | `prom/prometheus` |
| `prometheus_version` | `v3.4.0` |
| `prometheus_port` | `9090` |

### Grafana

| Variabile | Valore |
|---|---|
| `grafana_container_name` | `grafana` |
| `grafana_data_volume` | `grafana_data` |
| `grafana_image` | `grafana/grafana` |
| `grafana_version` | `12.0.0` |
| `grafana_port` | `3000` |
| `grafana_admin_password` | `{{ vault_grafana_admin_password }}` (il valore vero è nel vault, `group_vars/all/vault.yaml`) |
| `grafana_provisioning_dir` | `/etc/grafana/provisioning` |
| `grafana_dashboard_id` | `1860` (Node Exporter Full) |
| `grafana_dashboard_revision` | `41` |

## Porte e firewall

| Servizio | Porta | VM | Aperta in firewalld? |
|---|---|---|---|
| node_exporter | 9100 / 9101 / 9102 | tutte | Sì, dal ruolo (è un processo sulla VM) |
| Elasticsearch | 9200 | 192.168.3.40 | No, la pubblica Docker con le sue regole |
| Grafana | 3000 | 192.168.3.41 | No, la pubblica Docker |
| Prometheus | 9090 | 192.168.3.42 | No, la pubblica Docker |

## Verifica

```bash
# node_exporter
curl -s http://192.168.3.40:9100/metrics | head
curl -s http://192.168.3.41:9101/metrics | head
curl -s http://192.168.3.42:9102/metrics | head
```

- Prometheus: `http://192.168.3.42:9090/targets`, job `nodi` con 3 target **UP**.
- Grafana: `http://192.168.3.41:3000`, dashboard **Node Exporter Full** con le 3 VM nel menu *Host*.
