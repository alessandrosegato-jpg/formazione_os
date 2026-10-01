# ResourceQuota — Lab

## 1. Cos'è

Una ResourceQuota è **un contatore per namespace con una soglia**. Due parti:

| Campo | Chi lo scrive | Cosa contiene |
|---|---|---|
| `spec.hard` | tu | le soglie massime |
| `status.used` | il controller | il consumo attuale, somma degli oggetti del namespace |

L'enforcement avviene nell'**admission controller**: alla creazione di un oggetto Kubernetes valuta `used + richiesta > hard`. Se vero, rifiuta con `403 Forbidden`. Non c'è throttling né coda — è un sì/no sincrono al momento della richiesta all'API server.

> La quota agisce sulle **dichiarazioni** (`requests`/`limits`), non sul consumo reale. Un pod che dichiara 1 CPU e resta inattivo occupa comunque 1 CPU di quota. Un pod che sfora la RAM reale viene ucciso dal kubelet (OOMKill), non dalla quota. Sono due meccanismi indipendenti.

---

## 2. I comportamenti

**Non è retroattiva.** Applicata a un namespace già popolato non rimuove nulla: `used` viene calcolato sull'esistente e può risultare maggiore di `hard`. Da quel momento ogni nuova creazione è bloccata finché non si scende sotto soglia. La quota non fa mai cleanup.

**Se limiti una risorsa compute, quella risorsa diventa obbligatoria.** Con `requests.cpu` in `hard`, ogni container di ogni pod deve dichiarare `requests.cpu`. Senza dichiarazione il controller non saprebbe di quanto scalare il contatore.

**Con i controller l'errore sparisce dalla vista.** Un Deployment viene creato senza problemi (l'oggetto Deployment non consuma quota compute); è il ReplicaSet che poi fallisce sui pod. Vedi `5/8 READY` e nessun errore ovvio.

**Pod terminati: asimmetria.** `Succeeded`/`Failed` escono dal conteggio **compute**, ma restano nel conteggio **oggetti** (`pods`) finché non li cancelli.

---

## 3. Anatomia del manifest

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq
  namespace: quota-lab        # la quota è SEMPRE namespaced
spec:
  hard:                       # i tetti massimi
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "6"
```

### Chiavi utilizzabili in `hard`

**Compute** — somma di request/limit dei pod attivi
`requests.cpu` · `requests.memory` · `limits.cpu` · `limits.memory`

**Storage** —`requests.storage` · `persistentvolumeclaims`

**Conteggio oggetti** —`pods` · `services` · `secrets` · `configmaps` · `services.loadbalancers` · `services.nodeports`

---

## 4. Lab — setup

```bash
kubectl create ns quota-lab
```

**quota.yaml**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq
  namespace: quota-lab
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "6"
```

```bash
kubectl apply -f quota.yaml
kubectl describe quota rq -n quota-lab
```

```
Resource         Used  Hard
--------         ----  ----
limits.cpu       0     2
limits.memory    0     2Gi
pods             0     6
requests.cpu     0     1
requests.memory  0     1Gi
```

Con pod da `requests.cpu: 200m` si satura al quinto (5 × 200m = 1). La chiave che morde è `requests.cpu`, non `pods`.

---

## 5. Lab — pod singoli

**pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: p1
  namespace: quota-lab
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: 200m
          memory: 128Mi
        limits:
          cpu: 400m
          memory: 256Mi
```


### Al quinto pod — saturo

```
Resource         Used    Hard
requests.cpu     1       1
requests.memory  640Mi   1Gi
limits.cpu       2       2
limits.memory    1280Mi  2Gi
pods             5       6
```

`requests.cpu` e `limits.cpu` si saturano insieme (5×200m=1, 5×400m=2).

### Al sesto — rifiuto

```
Error from server (Forbidden): pods "p6" is forbidden: exceeded quota: rq,
requested: limits.cpu=400m,requests.cpu=200m,
used: limits.cpu=2,requests.cpu=1,
limited: limits.cpu=2,requests.cpu=1
```

Il messaggio elenca **solo le chiavi che hanno sforato**. `requests.memory` non compare perché lì c'era ancora spazio. È così che si capisce quale vincolo ci ha fermato.

---

## 6. Lab — Deployment

Più veloce per riempire, ma nasconde l'errore.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-min
  namespace: quota-lab
  labels:
    app: nginx-min
spec:
  replicas: 7
  selector:
    matchLabels:
      app: nginx-min
  template:
    metadata:
      labels:
        app: nginx-min
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
```

### Il fallimento silenzioso

```bash
kubectl get deploy -n quota-lab
```
```
NAME        READY   UP-TO-DATE   AVAILABLE
nginx-min    6/7        6            6
```

Nessun errore. Va pescato altrove:

```bash
kubectl describe rs -n quota-lab
```
```
Events:
  Warning  FailedCreate  replicaset-controller
  Error creating: pods "nginx-min" is forbidden: exceeded quota: rq, ...
```

Questo è il caso reale: in produzione si lavora con Deployment, la quota blocca, e si vedono solo repliche ferme con un controller che ritenta in backoff.

---

## 7. Decodifica degli errori

| Messaggio | Significato | Origine |
|---|---|---|
| `exceeded quota: rq, requested/used/limited` | soglia superata | ResourceQuota |
| `failed quota: rq: must specify requests.cpu` | dichiarazione mancante nel container | ResourceQuota |
| `maximum cpu usage per Container is 500m` | container troppo grande | LimitRange |
| `minimum cpu usage per Container is 50m` | container troppo piccolo | LimitRange |

Le prime due parole del messaggio ti dicono già quale meccanismo ha bloccato.


---

## 8. Verifiche che valgono l'esercizio

**La liberazione è immediata**
```bash
kubectl delete pod p5 -n quota-lab
kubectl describe quota rq -n quota-lab   # requests.cpu torna a 800m
kubectl apply -f p6.yaml                 # ora passa
```

**Nessuna tolleranza** — a quota satura, anche `requests.cpu: 1m` viene rifiutato.

---

## 9. Comandi di verifica

```bash
kubectl get resourcequota -n <ns>
kubectl describe quota -n <ns>                      # tabella Used vs Hard
kubectl get resourcequota rq -n <ns> -o yaml        # status.hard / status.used
kubectl describe rs -n <ns>                         # errori nascosti dei Deployment
```

---