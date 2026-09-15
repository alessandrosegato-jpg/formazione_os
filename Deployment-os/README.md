# Deployment OpenShift — Nginx a footprint minimale


## Obiettivi

1. **Comprendere in profondità** la composizione del manifest che descrive un **Deployment** (oggetto Kubernetes) e un **DeploymentConfig** (oggetto storico di OpenShift), capendo cosa fa ogni campo e in cosa i due si differenziano.
2. **Creare un Deployment** che installi Nginx (o un'applicazione dal footprint minimo) su un cluster OpenShift/Kubernetes.

---

## Contenuto della cartella

| File | Descrizione |
|------|-------------|
| `deployment-nginx.yaml` | Manifest di un Deployment che avvia un singolo pod Nginx con richieste/limiti di risorse ridotti al minimo. |

---

## 1. Anatomia del manifest `deployment-nginx.yaml`

Di seguito il manifest commentato riga per riga.

```yaml
apiVersion: apps/v1        # gruppo/versione dell'API a cui appartiene l'oggetto Deployment
kind: Deployment           # tipo di risorsa che stiamo creando
metadata:
  name: nginx-min          # nome univoco della risorsa nel namespace
  labels:
    app: nginx-min         # etichetta che identifica/raggruppa la risorsa
spec:                      # stato DESIDERATO dell'oggetto
  replicas: 1              # numero di pod identici da mantenere sempre attivi
  selector:
    matchLabels:
      app: nginx-min       # il Deployment "possiede" i pod con questa label
  template:                # modello con cui vengono creati i pod
    metadata:
      labels:
        app: nginx-min     # DEVE combaciare con selector.matchLabels
    spec:
      containers:
        - name: nginx                  # nome del container dentro al pod
          image: nginx:1.27-alpine     # immagine (variante alpine = leggera)
          ports:
            - containerPort: 80        # porta esposta dal processo nel container
          resources:
            requests:                  # risorse GARANTITE al container (scheduling)
              cpu: "10m"               # 10 millicore = 0,01 CPU
              memory: "16Mi"           # 16 mebibyte di RAM
            limits:                    # tetto MASSIMO che il container può usare
              cpu: "100m"              # oltre → throttling della CPU
              memory: "64Mi"           # oltre → il container viene terminato (OOMKilled)
```

- **Footprint minimale**: `nginx:1.27-alpine` + `10m/16Mi` di request rendono questo il carico più leggero possibile per un web server funzionante.

---

## 2. Deployment vs DeploymentConfig

Entrambi gestiscono il rilascio e la scalabilità di un'applicazione, ma appartengono a due mondi diversi.

| Aspetto | **Deployment** (`apps/v1`) | **DeploymentConfig** (`apps.openshift.io/v1`) |
|---------|----------------------------|-----------------------------------------------|
| Origine | Kubernetes standard | Specifico di OpenShift (storico/legacy) |
| Portabilità | Funziona su qualsiasi cluster K8s | Solo su OpenShift |
| Controller di rilascio | Gestito dai controller K8s (ReplicaSet) | Gestito dal deployer di OpenShift (ReplicationController) |
| Trigger automatici | No nativi (serve pipeline) | Sì: `ImageChange` (redeploy al cambio immagine), `ConfigChange` |
| Strategie di rollout | `RollingUpdate`, `Recreate` | `Rolling`, `Recreate`, **`Custom`** |
| Hook lifecycle | No | Sì (`pre`, `mid`, `post` deployment hooks) |
| Stato attuale | **Consigliato** anche su OpenShift | **Deprecato**: da usare solo per compatibilità |

> **In sintesi**: su OpenShift moderno si usa il **Deployment** (come in questa esercitazione). Il `DeploymentConfig` va conosciuto perché è ancora presente in molti progetti esistenti, ma non è più la scelta consigliata.

---

## 3. Verifica

```bash

# Verifica che il pod sia in esecuzione
kubectl get deployment nginx-min
kubectl get pods -l app=nginx-min


# Verifica funzionale (Nginx risponde)
# Esponi la porta del pod in locale
kubectl port-forward deploy/nginx-min 8080:80

# In un altro terminale
curl http://localhost:8080
# → deve restituire la pagina di benvenuto di Nginx
```
---