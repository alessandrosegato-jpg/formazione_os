# Gestione CSR e rilascio certificati con Root CA locale

Procedura per creare una Certification Authority locale self-signed, generare una
richiesta di certificato (CSR) e rilasciare il certificato firmato.

**Obiettivo**

1. Creare una ROOT CA self-signed e locale
2. Creare un CSR
3. Rilasciare i certificati richiesti dal CSR tramite la ROOT CA del punto 1

---

## Concetti di base

**Certification Authority (CA)** — un'entità che firma certificati. È costituita da
una coppia di chiavi (privata + pubblica) e da un certificato che ne dichiara
l'identità.

**Self-signed** — il certificato della CA è firmato con la sua stessa chiave
privata. Non c'è nessuna autorità superiore a garantirla: è il punto di partenza
della catena di fiducia.

**Locale** — la CA resta sulla propria macchina, non è pubblica come Let's Encrypt.
È valida solo per i client che ne importano esplicitamente il certificato.

**CSR (Certificate Signing Request)** — il "modulo di domanda" in formato PKCS#10.
Contiene la chiave pubblica del richiedente, il subject e le eventuali estensioni,
il tutto firmato con la chiave privata del richiedente. **La chiave privata non
entra mai nel CSR** e non lascia mai la macchina di chi la genera.

---

## 1. Creazione della Root CA

### 1.1 Chiave privata della CA

```bash
openssl genrsa -aes256 -out rootCA.key 4096
```

| Opzione | Significato |
|---|---|
| `genrsa` | sottocomando che genera una chiave privata RSA |
| `-aes256` | cifra la chiave su disco con AES-256, protetta da passphrase |
| `-out rootCA.key` | file di output della chiave privata |
| `4096` | dimensione della chiave in bit |

La passphrase verrà richiesta a ogni utilizzo della chiave, quindi a ogni firma di
un CSR. È il comportamento voluto per una root CA: chi ottiene questa chiave può
firmare certificati validi per qualsiasi dominio.

### 1.2 Certificato self-signed della CA

```bash
openssl req -x509 -new -key rootCA.key -sha256 -days 3650 -out rootCA.crt -subj "/C=IT/O=Lab/CN=Lab Root CA"
```

| Opzione | Significato |
|---|---|
| `-x509` | produce un certificato auto-firmato anziché un CSR |
| `-new` | genera una nuova richiesta/certificato da zero |
| `-key rootCA.key` | chiave privata con cui firmare |
| `-sha256` | algoritmo di hash |
| `-days 3650` | validità di 10 anni |
| `-out rootCA.crt` | file di output del certificato pubblico |
| `-subj` | compila i campi del subject senza prompt interattivo |

---

## 2. Creazione del CSR

### 2.1 Chiave privata del server

```bash
openssl genrsa -out server.key 2048
```

Qui **non** si usa `-aes256`. Una chiave cifrata bloccherebbe l'avvio automatico
del servizio (Nginx, Apache) in attesa che qualcuno digiti la passphrase sulla
console — inaccettabile dopo un reboot o dentro un container.

### 2.2 Generazione della richiesta

```bash
openssl req -new -key server.key -out server.csr \
  -subj "/C=IT/O=LabPKI/CN=app.example.local"
```

Il **Common Name deve contenere l'hostname** del servizio. Gli altri campi
(Country, State, Locality, Organization, OU, Email) sono opzionali e ignorati dalla
validazione TLS.

---

## 3. Rilascio del certificato


### 3.1 File delle estensioni

```bash
cat > server_ext.cnf <<'EOF'
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:localhost,IP:127.0.0.1
authorityKeyIdentifier = keyid,issuer
EOF
```

**I SAN sono obbligatori.** Browser e librerie TLS moderne ignorano completamente
il Common Name e validano l'hostname solo contro `subjectAltName`. Un certificato
privo di SAN produce `ERR_CERT_COMMON_NAME_INVALID`.

I valori in `subjectAltName` vanno adattati agli hostname e IP reali del servizio.


### 3.2 Firma

```bash
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out server.crt -days 365 -sha256 -extfile server_ext.cnf
```

| Opzione | Significato |
|---|---|
| `-req` | l'input è un CSR, non un certificato |
| `-CA` / `-CAkey` | certificato e chiave privata della CA firmataria |
| `-CAcreateserial` | crea/aggiorna `rootCA.srl`, il contatore dei numeri seriali |
| `-days 365` | validità del certificato emesso |
| `-extfile` | estensioni da inserire nel certificato |

> **Perché `-extfile` è necessario** — `openssl x509 -req` scarta per default tutte
> le estensioni presenti nel CSR. Non è un limite ma una protezione: altrimenti un
> richiedente potrebbe inserire `basicConstraints=CA:TRUE` nella propria richiesta
> e ottenere una CA firmata. È la CA a decidere cosa concedere, non il richiedente.


---

## Verifica

```bash
# la catena di fiducia è valida
openssl verify -CAfile rootCA.crt server.crt
# atteso: server.crt: OK

# i SAN sono presenti
openssl x509 -in server.crt -noout -text | grep -A1 "Subject Alternative Name"
```

Corrispondenza tra chiave, CSR e certificato — i tre comandi devono restituire lo
stesso hash, altrimenti Nginx fallisce con `key values mismatch`:

```bash
openssl rsa  -in server.key -noout -modulus | openssl sha256
openssl req  -in server.csr -noout -modulus | openssl sha256
openssl x509 -in server.crt -noout -modulus | openssl sha256
```

---

## File prodotti

| File | Contenuto | Riservato |
|---|---|---|
| `rootCA.key` | chiave privata della CA, cifrata con passphrase | **sì — la più critica** |
| `rootCA.crt` | certificato self-signed della CA | no, va distribuito ai client |
| `rootCA.srl` | contatore dei numeri seriali emessi | no |
| `server.key` | chiave privata del server, in chiaro | **sì** |
| `server.csr` | richiesta di firma | no, scartabile dopo l'emissione |
| `server.crt` | certificato firmato del server | no |

Il formato PEM (`-----BEGIN ...-----`) è codifica Base64, **non** cifratura:
chiunque può decodificarlo. L'intestazione distingue i due casi:

- `BEGIN PRIVATE KEY` → chiave in chiaro
- `BEGIN ENCRYPTED PRIVATE KEY` → chiave protetta da passphrase

---

