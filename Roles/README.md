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
    │   ├── templates/docker.sources.j2   # sorgente repo Docker (deb822)
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

## To Do:

1 - sostituire sul docker-install dal modulo copy utilizzando il modulo template (verificare che non ce ne siano altri, in caso sostituisci tutto con template)
2 - check prima di eseguire install docker o podman se non siano gia installati
3 - check sulla differenza tra le variabili nei ruoli: defaults o vars e in caso spostarle nel punto giusto
4 - crea un /etc/docker/daemon.json di esempio, poi al task dove metti gli insecure registy, utilizza blockinfile o lineinfile
5 - studiarti per bene l'opzione di ansible NO_LOG e in caso implementarla dove pensi che serva