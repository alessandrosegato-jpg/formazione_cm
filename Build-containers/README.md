# Build-containers

Utilizzando Ansible, creare dei playbooks che facciano la build di almeno due container con OS differenti.
Queste build devono generare dei container che abbiano queste caratteristiche:
- Essere sempre in ascolto sulla porta 22 del container
- Avere attivo il servizio ssh
-  Avere un utente abilitato a collegarsi tramite ssh key e poter fare sudo


## Contenuto della cartella

| File | Descrizione |
|---|---|
| `Dockerfile.ubuntu` | Immagine basata su `ubuntu:24.04` con server SSH |
| `Dockerfile.rocky` | Immagine basata su `rockylinux/rockylinux:9` con server SSH |
| `build-containers.yaml` | Playbook Ansible che genera la coppia di chiavi, copia Dockerfile e chiave pubblica nel build context, builda le immagini e avvia i container |

## Cosa fanno le immagini

Entrambi i Dockerfile producono lo stesso risultato, cambia solo il package manager (`apt` vs `dnf`):

1. Installano `sudo` e `openssh-server` (e ripuliscono la cache dei pacchetti).
2. Preparano sshd (`mkdir /var/run/sshd` su Ubuntu, `ssh-keygen -A` per generare le host key su Rocky).
3. Irrigidiscono la configurazione SSH:
   - `PermitRootLogin no`
   - `PasswordAuthentication no`
   - `PubkeyAuthentication yes`
   - `AllowUsers gino` — solo l'utente `gino` può collegarsi
4. Creano l'utente **`gino`** con shell bash e `sudo` senza password (`/etc/sudoers.d/90-gino`).
5. Copiano `id_ed25519.pub` dal build context in `/home/gino/.ssh/authorized_keys` (permessi `600`, owner `gino`).
6. Espongono la porta `22` e avviano `sshd` in foreground (`sshd -D -e`), così il processo resta PID 1 e il container rimane vivo.

## Cosa fa il playbook

`build-containers.yaml` gira su `hosts: all` con `become: true` e `gather_facts: false`.

Definisce una lista `containers` con i parametri dei due ambienti:

| name | image | dockerfile | porta host → container |
|---|---|---|---|
| `ssh-ubuntu` | `ssh-ubuntu-image:latest` | `Dockerfile.ubuntu` | `1025` → `22` |
| `ssh-rocky` | `ssh-rocky-image:latest` | `Dockerfile.rocky` | `1026` → `22` |

1. **Copia i Dockerfile nella directory** — `ansible.builtin.copy` porta i due Dockerfile dal `playbook_dir` locale a `/root/` sull'host target.
2. **Genera la coppia di chiavi SSH** — `community.crypto.openssh_keypair` crea `/home/vagrant/.ssh/id_ed25519` e `.pub` di tipo ed25519, **sull'host target**.
3. **Copia la chiave nel build context** — `ansible.builtin.copy` con `remote_src: true`, perché sorgente e destinazione sono entrambe sull'host target: da `/home/vagrant/.ssh/id_ed25519.pub` a `/root/id_ed25519.pub`, owner `root`, mode `0644`.
4. **Build immagini Docker** — `community.docker.docker_image_build` con build context `/root` e tag `latest`.
5. **Avvio container** — `community.docker.docker_container` con `state: started`, `restart_policy: unless-stopped` e la porta pubblicata.

### Connessione ai container

La chiave privata resta sull'host target, quindi ci si collega da lì:

```bash
ssh -i /home/vagrant/.ssh/id_ed25519 -p 1025 gino@localhost   # Ubuntu
ssh -i /home/vagrant/.ssh/id_ed25519 -p 1026 gino@localhost   # Rocky
```

### Verifica

```bash
docker ps                      # i due container devono essere Up
docker logs ssh-ubuntu         
```

