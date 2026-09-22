# SSH-conf — Hardening di sshd via template Jinja2 e drop-in

Configurazione automatizzata (Vagrant + Ansible) di `sshd` su tre macchine di
appoggio — **proxy-dev**, **proxy-test**, **proxy-prod** — che applica policy di
sicurezza differenziate per ambiente **senza mai toccare `/etc/ssh/sshd_config`**.

Tutta la configurazione custom viene scritta in un file drop-in dedicato
(`/etc/ssh/sshd_config.d/01-users.conf`), generato da un template Jinja2 che
adatta le direttive in base all'host di destinazione. I valori specifici di ogni
host non sono cablati nel template ma centralizzati in `vars/main.yml`.

---

## Perché la cartella drop-in

Su OpenSSH recente `sshd_config` contiene, tipicamente in cima al file, la riga:

```
Include /etc/ssh/sshd_config.d/*.conf
```

I file `.conf` presenti in `/etc/ssh/sshd_config.d/` vengono letti in ordine
alfabetico e, per la maggior parte delle direttive di `sshd`, **vale il primo
valore incontrato**. Poiché l'`Include` è in testa a `sshd_config`, un drop-in
come `01-users.conf` viene valutato *prima* del resto del file principale: le
sue direttive prevalgono senza dover editare — e quindi senza rischiare di
rompere — il `sshd_config` di sistema. Aggiornamenti del pacchetto OpenSSH
possono così sovrascrivere `sshd_config` liberamente, lasciando intatta la
configurazione applicata da questo progetto.

---

## Struttura del progetto

```
SSH-conf/
├── Vagrantfile              # Definizione delle 3 VM (proxy-dev/test/prod)
├── inventory                # Inventario Ansible (host + IP)
├── playbook.yaml            # Playbook che applica il ruolo sshd-conf
└── sshd-conf/               # Ruolo Ansible
    ├── tasks/
    │   └── main.yml         # Assert var → utenti/gruppi/chiavi → rsyslog (prod) → template sshd
    ├── templates/
    │   ├── ssh-conf.j2      # Drop-in sshd_config.d generato dalle variabili per-host
    │   └── rsyslog-conf.j2  # Regola rsyslog per il log SSH di prod
    ├── handlers/
    │   └── main.yml         # Riavvio di sshd e rsyslog
    ├── vars/
    │   └── main.yml         # host_users + sshd_host_configs (utenti, chiavi e config sshd per host)
    ├── defaults/
    │   └── main.yml         # (vuoto)
    ├── tests/               # test.yml + inventory di esempio
    └── meta/                # Metadati del ruolo
```

---

## Le tre macchine

| VM           | Hostname     | IP (private network) | Box              |
|--------------|--------------|----------------------|------------------|
| proxy-dev    | proxy-dev    | 192.168.3.60         | ubuntu/jammy64   |
| proxy-test   | proxy-test   | 192.168.3.61         | ubuntu/jammy64   |
| proxy-prod   | proxy-prod   | 192.168.3.62         | ubuntu/jammy64   |

---

## Policy applicate

### Su tutte le macchine

- **No password auth** — `PasswordAuthentication no`
- **No empty passwords** — `PermitEmptyPasswords no`
- **Solo chiavi SSH moderne** — `PubkeyAuthentication yes` con
  `PubkeyAcceptedAlgorithms` limitato a `ssh-ed25519`,
  `ssh-ed25519-cert-v01@openssh.com`, `rsa-sha2-512`, `rsa-sha2-256`
- **Max tentativi di autenticazione = 2** — `MaxAuthTries 2`

### Accessi per ambiente

| Ambiente   | Chi può accedere                                   | Direttiva sshd                    |
|------------|----------------------------------------------------|-----------------------------------|
| proxy-dev  | utente `sviluppatore` e `root`                     | `AllowUsers sviluppatore root`    |
| proxy-test | utente `tester` e `root`                           | `AllowUsers tester root`          |
| proxy-prod | solo il gruppo `administrators`, utente `admin`, **root escluso** | `AllowGroups administrators`, `AllowUsers admin`, `PermitRootLogin no` |

### Solo su proxy-prod

- **Porta non standard** — `Port 2222` (dev e test restano sulla `22`)
- **No root login** — `PermitRootLogin no` (dev e test hanno `PermitRootLogin yes`)
- **Log verboso** — `LogLevel VERBOSE` (dev e test usano il default `INFO`)
- **Log SSH su file dedicato** — `SyslogFacility local5`, con rsyslog che
  instrada tutto ciò che arriva su `local5.*` verso `/var/log/prod_ssh.log`
  (dev e test usano la facility `AUTH` standard)

---

## Come funziona il template

`templates/ssh-conf.j2` produce il drop-in `01-users.conf`. Le direttive comuni
sono uguali su tutti gli host; i valori che cambiano per ambiente (porta, root
login, utenti/gruppi ammessi, facility e livello di log) vengono letti dal
dizionario `sshd_host_configs[inventory_hostname]`, così lo stesso template
genera tre file diversi a seconda della macchina:

```jinja2
{% set cfg = sshd_host_configs[inventory_hostname] %}
PasswordAuthentication no
PermitEmptyPasswords no
PubkeyAuthentication yes
PubkeyAcceptedAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256
MaxAuthTries 2
Port {{ cfg.port }}
PermitRootLogin {{ cfg.permit_root_login }}
AllowUsers {{ cfg.allow_users }}
{% if cfg.allow_groups %}
AllowGroups {{ cfg.allow_groups }}
{% endif %}
SyslogFacility {{ cfg.syslog_facility }}
LogLevel {{ cfg.log_level | default('INFO') }}
```

`AllowGroups` viene emesso solo se per l'host è definito un gruppo (vale per
prod); `LogLevel` ripiega su `INFO` se l'host non ne specifica uno. I valori
per ogni ambiente sono in `vars/main.yml`:

```yaml
sshd_host_configs:
  proxy-dev:
    port: 22
    permit_root_login: "yes"
    allow_users: "sviluppatore root"
    allow_groups: ""
    syslog_facility: "AUTH"
  proxy-test:
    port: 22
    permit_root_login: "yes"
    allow_users: "tester root"
    allow_groups: ""
    syslog_facility: "AUTH"
  proxy-prod:
    port: 2222
    permit_root_login: "no"
    allow_users: "admin"
    allow_groups: "administrators"
    syslog_facility: "local5"
    log_level: "VERBOSE"
```

Il file viene scritto in `/etc/ssh/sshd_config.d/01-users.conf` con permessi
`0644` e, a ogni modifica, notifica l'handler che riavvia `sshd`.

---

## Bonus: log SSH dedicato su prod

Solo su **proxy-prod** l'attività SSH viene scritta in un file dedicato,
`/var/log/prod_ssh.log`, tramite due pezzi che lavorano insieme:

1. Nel drop-in di sshd (`SyslogFacility local5`) si dice a `sshd` di emettere i
   propri messaggi sulla facility syslog `local5` invece che sulla `AUTH` di
   default.
2. La regola rsyslog `templates/rsyslog-conf.j2`

   ```
   local5.* /var/log/prod_ssh.log
   ```

   viene installata in `/etc/rsyslog.d/51-prod-ssh.conf` (task *"Log SSH
   dedicato (solo prod)"* in `tasks/main.yml`, condizionato con
   `when: inventory_hostname == 'proxy-prod'`) e instrada tutto ciò che arriva
   su `local5` verso il file dedicato. Una modifica alla regola notifica
   l'handler che riavvia `rsyslog`.

Questo file rsyslog viene creato **solo** dal task condizionato su prod, quindi
dev e test continuano a loggare l'SSH nel percorso di sistema standard.

---

## Flusso del playbook

`playbook.yaml` gira sul gruppo `proxy` con privilegi elevati (`become: true`)
e applica il ruolo `sshd-conf`. Il ruolo, in `tasks/main.yml`, esegue in ordine:

1. **Verifica delle variabili** — un `assert` controlla che `sshd_host_configs`
   e `host_users` siano definiti e che l'host corrente sia presente in entrambi
   i dizionari; se manca qualcosa il ruolo fallisce subito con un messaggio
   esplicito.
2. **Gruppo `administrators`** — creato solo su `proxy-prod`.
3. **Creazione utenti** — cicla su `host_users[inventory_hostname]` creando ogni
   utente con la sua shell, gli eventuali gruppi supplementari e la home.
4. **Chiavi pubbliche** — installa in `authorized_keys` la chiave indicata da
   ciascun utente (`lookup('file', item.pubkey)`).
5. **Regola rsyslog** — installata solo su `proxy-prod` (vedi sopra).
6. **Drop-in sshd** — genera `sshd_config.d/01-users.conf` dal template e, se
   cambia, notifica l'handler che riavvia `sshd`.

Nomi utente, shell, chiavi e configurazione sshd non sono cablati nei task ma
centralizzati in `vars/main.yml`, nei due dizionari `host_users` (chi creare e
con quali chiavi) e `sshd_host_configs` (i parametri sshd per host).

---

### Verifica

```bash
# Su una qualsiasi VM: controlla il drop-in generato
cat /etc/ssh/sshd_config.d/01-users.conf

# Su proxy-prod: SSH ascolta sulla 2222
ssh -p 2222 admin@192.168.3.62

# Su proxy-prod: il log dedicato si popola
sudo tail -f /var/log/prod_ssh.log
```

> Le chiavi pubbliche installate negli `authorized_keys` degli utenti (e di
> root su dev/test) vengono prese dal campo `pubkey` di ogni utente in
> `host_users` — di default `~/.ssh/id_ed25519.pub` della macchina di controllo
> — tramite `lookup('file', ...)`.

---
