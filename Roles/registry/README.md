# Ruolo `registry`

Installa il motore container e mette in piedi una registry privata `registry:2`
con volume persistente.

**Punto chiave dell'esercitazione:** è il ruolo che funziona **sia con Docker sia con Podman**.
`tasks/main.yml` non contiene logica specifica del motore, ma:

1. valida la variabile `container_engine` con un `assert` (accetta solo `docker` o `podman`);
2. verifica se un motore container è **già installato** sull'host
   (`command -v docker || command -v podman`), registrando l'esito;
3. include dinamicamente `{{ container_engine }}-install.yaml` **solo se nessun motore
   è già presente** (`when: container_engine_installed.rc != 0`), per non reinstallare inutilmente;
4. include dinamicamente `{{ container_engine }}-registry.yaml`.

**Variabili (`defaults/main.yml`)**

| Variabile | Default | Descrizione |
|---|---|---|
| `container_engine` | `docker` | Motore da usare: `docker` o `podman` |
| `volume_name` | `registry_data` | Volume persistente per i dati della registry |
| `container_name` | `registry` | Nome del container della registry |
| `container_image` | `registry:2` | Immagine della registry |
| `container_published_ports` | `5000:5000` | Mappatura porta host:container |
| `container_volumes` | `registry_data:/var/lib/registry` | Mount del volume |

**Variabili (`vars/main.yml`)**

| Variabile | Default | Descrizione |
|---|---|---|
| `docker_key_dir` | `/etc/apt/keyrings` | Cartella per la chiave GPG di Docker |
| `docker_key_url` | `https://download.docker.com/linux/ubuntu/gpg` | URL della chiave GPG |

**Cosa fa nel dettaglio**

- `docker-install.yaml` — installa `ca-certificates` e `python3-docker`, crea `/etc/apt/keyrings`,
  scarica la chiave GPG di Docker, genera il file del repository in formato `deb822` a partire
  dal template `templates/docker.sources.j2` (modulo `template`, più leggibile e riutilizzabile
  del vecchio `copy`), installa `docker-ce`, `docker-ce-cli`, `containerd.io`,
  `docker-buildx-plugin`, `docker-compose-plugin` e abilita il servizio al boot.
- `podman-install.yaml` — installa il pacchetto `podman` dai repository della distribuzione.
- `docker-registry.yaml` / `podman-registry.yaml` — creano il volume e avviano il container
  della registry con `restart_policy: always`, pull dell'immagine e pubblicazione della porta.
  I due file sono volutamente speculari: cambiano solo i moduli (`docker_*` vs `podman_*`) e
  la sintassi del pull (`pull: true` vs `pull: always`), mentre le variabili usate sono le stesse.

---
