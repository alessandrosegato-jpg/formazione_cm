# Esercitazione — Docker-in-Docker + pipeline Jenkins di build & push

Estensione dell'esercitazione precedente (Step 2): al container SSH viene aggiunto un
**daemon Docker funzionante al suo interno**, e viene configurata una **pipeline Jenkins** che builda
un'immagine, la tagga in modo progressivo e ne fa il push su una registry privata.

---

## 1. Obiettivi

| # | Requisito | Dove è implementato |
|---|-----------|---------------------|
| 1 | Container con i requisiti dello Step 2 (SSH, utente non privilegiato, accesso solo a chiave) | `Roles/docker-in-docker/files/Dockerfile.ubuntu.entry` |
| 2 | Servizio Docker attivo dentro il container | `entrypoint.sh` + `privileged: true` |
| 3 | Pipeline Jenkins: build immagine + tag progressivo | `Jenkins&Ansible/Jenkinsfile` (stage *Build* e *Tag*) |
| 4 | Pipeline Jenkins: push sulla registry | `Jenkins&Ansible/Jenkinsfile` (stage *Push*) |

---

## 2. Struttura del progetto

```
formazione_cm/
├── Roles/
│   └── docker-in-docker/
│       ├── defaults/main.yml          # nome container/immagine, porta, path host
│       ├── vars/main.yml              # nomi dei file in files/
│       ├── files/
│       │   ├── Dockerfile.ubuntu          # immagine "Step 2": solo SSH
│       │   ├── Dockerfile.ubuntu.entry    # immagine DinD: SSH + Docker + entrypoint
│       │   └── entrypoint.sh              # avvia dockerd, poi sshd in foreground
│       ├── tasks/main.yml             # logica del ruolo
│       ├── meta/main.yml
│       └── tests/
├── Playbooks-roles/
│   ├── docker-in-docker.yaml          # playbook di questa esercitazione
│ 
└── Jenkins&Ansible/
    └── Jenkinsfile                    # pipeline dichiarativa
```

---

## 3. Flusso complessivo

```
                 ansible-playbook docker-in-docker.yaml
                                  │
              ┌───────────────────┴────────────────────┐
              ▼                                        ▼
    host target (build context /root)         agent Jenkins (/srv/condivisa)
    ├─ genera coppia di chiavi SSH            ├─ Dockerfile.ubuntu
    ├─ copia Dockerfile.ubuntu.entry          └─ id_ed25519.pub
    ├─ copia id_ed25519.pub
    ├─ copia entrypoint.sh
    ├─ docker build  → ssh-ubuntu-image:latest
    └─ docker run    → ssh-ubuntu  (privileged, 1027→22)
                                  │
                                  ▼
                    container con dockerd attivo
                                  │
                 il job Jenkins gira su questo nodo
                                  │
        docker build → tag 1.0.$BUILD_NUMBER → push su 192.168.3.2:5000
```

---

## 4. Il ruolo `docker-in-docker`

### 4.1 Variabili

`defaults/main.yml`:

| Variabile | Default | Significato |
|-----------|---------|-------------|
| `container_name` | `ssh-ubuntu` | nome del container in esecuzione |
| `container_image` | `ssh-ubuntu-image` | nome dell'immagine buildata |
| `host_port` | `1027` | porta host mappata sulla 22 del container |
| `ssh_build_context` | `/root` | directory di build sul target |
| `ssh_key_path` | `/home/vagrant/.ssh/id_ed25519` | chiave privata generata |
| `key_type` | `ed25519` | tipo di chiave |
| `jenkins_workspace` | `/srv/condivisa` | directory condivisa con l'agent Jenkins |

`vars/main.yml` — costanti del ruolo (nomi dei file dentro `files/`), non pensate per essere
sovrascritte: `dockerfile_agent`, `dockerfile_entrypoint`, `entrypoint_script` e
`ssh_key_public_path`, derivata da `ssh_key_path` per restare sempre coerente.

### 4.2 Task, in ordine

1. **Copia `Dockerfile.ubuntu.entry`** nel build context (`/root`).
2. **Genera la coppia di chiavi SSH** con `community.crypto.openssh_keypair` (`no_log: true`,
   così la chiave non finisce nell'output di Ansible). Il modulo è idempotente: se la chiave
   esiste già non viene rigenerata.
3. **Copia la chiave pubblica nel build context** (`remote_src: true`: la sorgente è già sul target).
4. **Copia `entrypoint.sh`** nel build context con permessi `0755`.
5. **Copia `Dockerfile.ubuntu` e la chiave pubblica in `/srv/condivisa`**, la directory da cui
   l'agent Jenkins farà la propria build.
6. **`community.docker.docker_image_build`** → costruisce `ssh-ubuntu-image:latest` dal
   `Dockerfile.ubuntu.entry`.
7. **`community.docker.docker_container`** → avvia `ssh-ubuntu` con `privileged: true`,
   `restart_policy: unless-stopped` e la porta `1027:22`.

### 4.3 Come è ottenuto il Docker dentro il container

Tre pezzi lavorano insieme:

**a. Il pacchetto** — `Dockerfile.ubuntu.entry` installa `docker.io` accanto a `openssh-server` e
aggiunge `gino` al gruppo `docker`, così l'utente può usare il socket senza `sudo`.

**b. La registry insecure** — la registry di laboratorio è in HTTP puro, quindi il daemon interno
va istruito a fidarsene:

```dockerfile
RUN mkdir -p /etc/docker && \
    echo '{"insecure-registries":["192.168.3.2:5000"]}' > /etc/docker/daemon.json
```

Senza questa riga il push fallisce con `http: server gave HTTP response to HTTPS client`.

**c. L'avvio di due servizi** — un container ha un solo PID 1, quindi l'entrypoint avvia `dockerd`
in background e lascia `sshd` in foreground:

```bash
#!/bin/bash
set -e
dockerd &> /var/log/dockerd.log &
sleep 10
exec /usr/sbin/sshd -D -e
```

`exec` è la parte importante: `sshd` diventa il PID 1 e riceve correttamente i segnali di
`docker stop`.

**d. `privileged: true`** — `dockerd` ha bisogno di creare cgroup, montare filesystem e gestire
`iptables`: senza il flag privileged non parte. È il prezzo del Docker-in-Docker reale.

---

## 5. La pipeline Jenkins

```groovy
pipeline {
  agent { label 'docker' }

  environment {
    REGISTRY = '192.168.3.2:5000'
    IMAGE = 'ssh-docker'
    TAG = "1.0.${BUILD_NUMBER}"
  }

  stages {
    stage('Build immagine') { ... }
    stage('Tag immagine')   { ... }
    stage('Push immagine')  { ... }
  }
}
```

- **Tag progressivo** — `1.0.${BUILD_NUMBER}`: `BUILD_NUMBER` è la variabile che Jenkins
  incrementa a ogni esecuzione, quindi ogni build produce un tag nuovo e mai riusato
  (`1.0.1`, `1.0.2`, …). Nessuna immagine viene sovrascritta nella registry.
- **Tag verso la registry** — `docker tag` aggiunge il prefisso `192.168.3.2:5000/`, che è ciò
  che dice a Docker *dove* fare il push (per Docker il registry fa parte del nome dell'immagine).
- **Push** — `docker push 192.168.3.2:5000/ssh-docker:1.0.N`.

### Verifica

```bash
# elenco dei repository nella registry
curl http://192.168.3.2:5000/v2/_catalog

# tag disponibili per l'immagine
curl http://192.168.3.2:5000/v2/ssh-docker/tags/list
```

---

## 6. Esecuzione

Accesso al container:

```bash
ssh -i /home/vagrant/.ssh/id_ed25519 -p 1027 gino@<host>
docker info          # deve rispondere: il daemon interno è attivo
```

---


