# Docker Registry con Ansible

Esercitazione: automatizzare con Ansible l'installazione di Docker e il deploy di un
**registry privato di immagini container** (`registry:2`).

---

## Cosa fa il playbook

| # | Task | Modulo | Scopo |
|---|---|---|---|
| 1 | Prerequisiti | `apt` | `ca-certificates` per la validazione TLS del repo, `python3-docker` per i moduli |
| 2 | Cartella keyring | `file` | crea `/etc/apt/keyrings` |
| 3 | Chiave GPG | `get_url` | scarica la chiave di firma di Docker |
| 4 | Repository | `copy` | scrive `/etc/apt/sources.list.d/docker.sources` in formato deb822 |
| 5 | Pacchetti Docker | `apt` | `docker-ce`, CLI, `containerd.io`, plugin buildx e compose |
| 6 | Servizio | `systemd` | avvia Docker e lo abilita al boot |
| 7 | Volume | `docker_volume` | crea `registry_data` per i dati persistenti |
| 8 | Container | `docker_container` | avvia `registry:2` con porta pubblicata e volume montato |

### La persistenza

Il volume `registry_data` è montato su `/var/lib/registry`, ovvero il path **interno**
al container dove il registry scrive. Senza volume, ogni ricreazione
del container cancellerebbe tutte le immagini pubblicate.

---

## Verifica

### 1. Stato del servizio

```bash
docker ps --filter name=registry     # "Up", porta 5000 mappata
curl -i http://localhost:5000/v2/    # HTTP/1.1 200 OK
```

L'endpoint `/v2/` restituisce un corpo vuoto (`{}`) con l'header
`Docker-Distribution-Api-Version: registry/2.0`: è il *version check* dell'API.

### 2. Push e pull

```bash
docker pull alpine:latest
docker tag alpine:latest localhost:5000/alpine:test
docker push localhost:5000/alpine:test

curl http://localhost:5000/v2/_catalog
# → {"repositories":["alpine"]}

docker rmi localhost:5000/alpine:test    # elimina la copia locale
docker pull localhost:5000/alpine:test   # può arrivare solo dal registry
```

Il prefisso `localhost:5000/` serve a Docker, sa
a quale registry rivolgersi. Senza prefisso, il push andrebbe a Docker Hub.

---

## Accesso da un'altra macchina

Il registry è in **HTTP puro**. Docker rifiuta per default i registry non cifrati,
tranne quando l'indirizzo è `localhost`. Da un client remoto serve quindi dichiararlo
esplicitamente insicuro, in `/etc/docker/daemon.json`:

```json
{
  "insecure-registries": ["192.168.3.2:5000"]
}
```

```bash
sudo systemctl restart docker
```

---
