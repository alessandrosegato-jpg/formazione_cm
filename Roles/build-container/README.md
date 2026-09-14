# Ruolo `build-container`

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
   (idempotente: se la chiave esiste già non viene rigenerata; con `no_log: true`
   per non registrare dati sensibili nel log di Ansible);
3. copia della chiave **pubblica** nel build context come `id_ed25519.pub`
   (`remote_src: true`, perché la chiave è stata appena creata sull'host);
4. build delle immagini con `community.docker.docker_image_build`, in loop;
5. avvio dei container con `community.docker.docker_container`, `restart_policy: unless-stopped`,
   pubblicando `{{ item.port }}:22`.

---
