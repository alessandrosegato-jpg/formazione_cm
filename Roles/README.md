# Esercitazione Ansible — Registry, Build, Push e Run di container

Raccolta di ruoli Ansible che, partendo dagli step svolti nelle esercitazioni precedenti,
automatizza l'intero ciclo di vita di immagini container su un host di test:

1. installazione del motore container e creazione di una **registry privata**;
2. **build di due immagini** basate su distribuzioni diverse (Ubuntu e Rocky Linux), entrambe con un server SSH configurato per l'accesso a chiave;
3. **run dei container** su porte host differenti, in modo che non vadano in conflitto tra loro né con la registry;
4. **push delle immagini** sulla registry creata al punto 1.

Il ruolo `registry` è inoltre scritto in modo da funzionare **sia con Docker sia con Podman**,
selezionando il motore tramite una singola variabile.

---

## Struttura del repository

```
formazione_cm/
├── Playbooks-roles/
│   ├── container-registry.yaml     # solo registry
│   ├── build-containers.yaml       # solo build + run dei container
│   └── build-and-push.yaml         # flusso completo: registry → build/run → push
└── Roles/
    ├── registry/
    │   ├── defaults/main.yml
    │   ├── tasks/
    │   │   ├── main.yml                 # dispatcher docker/podman
    │   │   ├── docker-install.yaml
    │   │   ├── docker-registry.yaml
    │   │   ├── podman-install.yaml
    │   │   └── podman-registry.yaml
    │   ├── meta/main.yml
    │   ├── vars/main.yml
    │   └── tests/{inventory,test.yml}
    ├── build-container/
    │   ├── defaults/main.yml
    │   ├── files/
    │   │   ├── Dockerfile.ubuntu
    │   │   └── Dockerfile.rocky
    │   ├── tasks/main.yml
    │   ├── meta/main.yml
    │   ├── vars/main.yml
    │   └── tests/{inventory,test.yml}
    └── push-images/
        ├── defaults/main.yml
        ├── handlers/main.yml
        ├── tasks/main.yml
        ├── meta/main.yml
        ├── vars/main.yml
        └── tests/{inventory,test.yml}
```
---

## I ruoli

### Ruolo `registry`

Installa il motore container e mette in piedi una registry privata `registry:2`
con volume persistente.

**Punto chiave dell'esercitazione:** è il ruolo che funziona **sia con Docker sia con Podman**.
`tasks/main.yml` non contiene logica specifica del motore, ma:

1. valida la variabile `container_engine` con un `assert` (accetta solo `docker` o `podman`);
2. include dinamicamente `{{ container_engine }}-install.yaml`;
3. include dinamicamente `{{ container_engine }}-registry.yaml`.

**Variabili (`defaults/main.yml`)**

| Variabile | Default | Descrizione |
|---|---|---|
| `container_engine` | `docker` | Motore da usare: `docker` o `podman` |
| `volume_name` | `registry_data` | Volume persistente per i dati della registry |
| `container_name` | `registry` | Nome del container della registry |
| `container_image` | `registry:2` | Immagine della registry |
| `container_published_ports` | `5000:5000` | Mappatura porta host:container |
| `container_volumes` | `registry_data:/var/lib/registry` | Mount del volume |
| `docker_key_dir` | `/etc/apt/keyrings` | Cartella per la chiave GPG di Docker |
| `docker_key_url` | `https://download.docker.com/linux/ubuntu/gpg` | URL della chiave GPG |

**Cosa fa nel dettaglio**

- `docker-install.yaml` — installa `ca-certificates` e `python3-docker`, crea `/etc/apt/keyrings`,
  scarica la chiave GPG di Docker, scrive il repository in formato `deb822`,
  installa `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`,
  `docker-compose-plugin` e abilita il servizio al boot.
- `podman-install.yaml` — installa il pacchetto `podman` dai repository della distribuzione.
- `docker-registry.yaml` / `podman-registry.yaml` — creano il volume e avviano il container
  della registry con `restart_policy: always`, pull dell'immagine e pubblicazione della porta.
  I due file sono volutamente speculari: cambiano solo i moduli (`docker_*` vs `podman_*`) e
  la sintassi del pull (`pull: true` vs `pull: always`), mentre le variabili usate sono le stesse.

---

### Ruolo `build-container`

Copre i punti "build di almeno due container" e "run senza conflitto di porte".

Costruisce due immagini SSH da distribuzioni diverse e le avvia mappando ciascuna
sulla propria porta host.

**Variabili (`defaults/main.yml`)**

| Variabile | Default | Descrizione |
|---|---|---|
| `ssh_containers` | lista di 2 elementi (vedi sotto) | Definizione delle immagini/container da creare |
| `ssh_build_context` | `/root` | Directory usata come build context sull'host |
| `ssh_key_path` | `/home/vagrant/.ssh/id_ed25519` | Percorso della chiave privata generata |
| `ssh_key_public_path` | `/home/vagrant/.ssh/id_ed25519.pub` | Percorso della chiave pubblica |
| `key_type` | `ed25519` | Tipo di chiave |

| `name` | `image` | `dockerfile` | `port` |
|---|---|---|---|
| `ssh-ubuntu` | `ssh-ubuntu-image` | `Dockerfile.ubuntu` | `1025` |
| `ssh-rocky` | `ssh-rocky-image` | `Dockerfile.rocky` | `1026` |

**Flusso dei task (`tasks/main.yml`)**

1. copia dei Dockerfile nel build context, in loop su `ssh_containers`;
2. generazione della coppia di chiavi SSH con `community.crypto.openssh_keypair`
   (idempotente: se la chiave esiste già non viene rigenerata);
3. copia della chiave **pubblica** nel build context come `id_ed25519.pub`
   (`remote_src: true`, perché la chiave è stata appena creata sull'host);
4. build delle immagini con `community.docker.docker_image_build`, in loop;
5. avvio dei container con `community.docker.docker_container`, `restart_policy: unless-stopped`,
   pubblicando `{{ item.port }}:22`.

---

### Ruolo `push-images`

Copre il punto "push delle build sul registry precedentemente creato".

**Variabili (`defaults/main.yml`)**

| Variabile | Default | Descrizione |
|---|---|---|
| `registry_host` | `localhost` | Host della registry |
| `registry_port` | `5000` | Porta della registry |
| `image_list` | `ssh-ubuntu-image`, `ssh-rocky-image` | Immagini locali da pubblicare |
| `image_tag` | `latest` | Tag usato per il retag e il push |

**Flusso dei task (`tasks/main.yml`)**

1. scrittura di `/etc/docker/daemon.json` che dichiara la registry come *insecure*
   (necessario perché la registry è in HTTP, senza TLS); il task notifica l'handler
   **Riavvia Docker** (`handlers/main.yml`), che riavvia il servizio solo se il file cambia;
2. retag delle immagini locali nel formato `registry_host:registry_port/immagine:tag`
   con `docker_image_tag`, in loop su `image_list`;
3. push con `docker_image_push`, in loop.

---

## I playbook

Tutti i playbook sono in `Playbooks-roles/`, girano su `hosts: all` con `become: true`.

| Playbook | Ruoli | Scopo |
|---|---|---|
| `container-registry.yaml` | `registry` | Installa il motore container e la sola registry privata |
| `build-containers.yaml` | `build-container` | Build e run delle due immagini SSH  |
| `build-and-push.yaml` | `registry` → `build-container` → `push-images` | Flusso completo end-to-end |

L'ordine in `build-and-push.yaml` è vincolante: la registry deve esistere prima che le
immagini vengano costruite e pubblicate.

---

## Mappa delle porte

Le porte host sono state scelte in modo da non collidere tra loro né con la porta 22
dell'host, soddisfacendo il requisito "run dei container in modo che non vadano in
conflitto di porte tra loro".

| Servizio | Container | Porta host | Porta container |
|---|---|---|---|
| Registry privata | `registry` | `5000` | `5000` |
| SSH Ubuntu 24.04 | `ssh-ubuntu` | `1025` | `22` |
| SSH Rocky Linux 9 | `ssh-rocky` | `1026` | `22` |


---

## Verifiche

Sull'host di destinazione:

```bash
# Registry attiva e container in esecuzione
docker ps            # oppure: podman ps

# Immagini presenti nella registry
curl http://localhost:5000/v2/_catalog

# Accesso SSH ai due container, con la chiave generata dal ruolo
ssh -i /home/vagrant/.ssh/id_ed25519 -p 1025 gino@localhost   # Ubuntu
ssh -i /home/vagrant/.ssh/id_ed25519 -p 1026 gino@localhost   # Rocky
```

---

