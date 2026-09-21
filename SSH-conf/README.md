# SSH-conf — Hardening di sshd via template Jinja2 e drop-in

Configurazione automatizzata (Vagrant + Ansible) di `sshd` su tre macchine di
appoggio — **proxy-dev**, **proxy-test**, **proxy-prod** — che applica policy di
sicurezza differenziate per ambiente **senza mai toccare `/etc/ssh/sshd_config`**.

Tutta la configurazione custom viene scritta in un file drop-in dedicato
(`/etc/ssh/sshd_config.d/01-users.conf`), generato da un template Jinja2 che
adatta le direttive in base all'host di destinazione.

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
    │   ├── main.yml         # Include per-host + template sshd
    │   ├── proxy-dev.yaml   # Utente sviluppatore + chiavi
    │   ├── proxy-test.yaml  # Utente tester + chiavi
    │   └── proxy-prod.yaml  # Gruppo administrators, admin, log dedicato
    ├── templates/
    │   ├── ssh-conf.j2      # Drop-in sshd_config.d generato via Jinja2
    │   └── rsyslog-conf.j2  # Regola rsyslog per il log SSH di prod
    ├── handlers/
    │   └── main.yml         # Riavvio di sshd e rsyslog
    ├── vars/
    │   └── main.yml         # Nomi utente, gruppo e shell per ambiente
    ├── defaults/            # (vuoto)
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
| proxy-prod | solo il gruppo `administrators`, utente `admin`, **root escluso** | `AllowGroups administrators`, `AllowUsers admin` `PermitRootLogin no` |

### Solo su proxy-prod

- **Porta non standard** — `Port 2222`
- **No root login** — `PermitRootLogin no`
- **Log verboso** — `LogLevel VERBOSE`
- **Log SSH su file dedicato** — `SyslogFacility LOCAL5`, con rsyslog che
  instrada tutto ciò che arriva su `local5.*` verso `/var/log/prod_ssh.log`

---

## Come funziona il template

`templates/ssh-conf.j2` produce il drop-in `01-users.conf`. La parte comune è
identica su tutti gli host; le sezioni specifiche sono attivate con `{% if
inventory_hostname == '...' %}`, così lo stesso template genera tre file
diversi a seconda della macchina:

```jinja2
PasswordAuthentication no
PermitEmptyPasswords no
PubkeyAuthentication yes
PubkeyAcceptedAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256
MaxAuthTries 2
{% if inventory_hostname == 'proxy-dev' %}
AllowUsers sviluppatore root
{%- endif %}
{% if inventory_hostname == 'proxy-test' %}
AllowUsers tester root
{%- endif %}
{% if inventory_hostname == 'proxy-prod' %}
LogLevel VERBOSE
PermitRootLogin no
Port 2222
AllowUsers admin
AllowGroups administrators
SyslogFacility LOCAL5
{%- endif %}
```

Il file viene scritto in `/etc/ssh/sshd_config.d/01-users.conf` con permessi
`0644` e, a ogni modifica, notifica l'handler che riavvia `sshd`.

---

## Bonus: log SSH dedicato su prod

Solo su **proxy-prod** l'attività SSH viene scritta in un file dedicato,
`/var/log/prod_ssh.log`, tramite due pezzi che lavorano insieme:

1. Nel drop-in di sshd (`SyslogFacility LOCAL5`) si dice a `sshd` di emettere i
   propri messaggi sulla facility syslog `local5` invece che sulla `AUTH` di
   default.
2. La regola rsyslog `templates/rsyslog-conf.j2`

   ```
   local5.* /var/log/prod_ssh.log
   ```

   viene installata in `/etc/rsyslog.d/51-prod-ssh.conf` (task *"Log SSH
   dedicato"* in `proxy-prod.yaml`) e instrada tutto ciò che arriva su
   `local5` verso il file dedicato. Una modifica alla regola notifica
   l'handler che riavvia `rsyslog`.

Questo file rsyslog viene creato **solo** dal task di prod, quindi dev e test
continuano a loggare l'SSH nel percorso di sistema standard.

---

## Flusso del playbook

`playbook.yaml` gira sul gruppo `proxy` con privilegi elevati (`become: true`)
e applica il ruolo `sshd-conf`. Il ruolo, in `tasks/main.yml`:

1. Include il file di task specifico dell'host —
   `include_tasks: "{{ inventory_hostname }}.yaml"` — che crea utenti/gruppi,
   installa le chiavi pubbliche e (solo su prod) la regola rsyslog.
2. Genera il drop-in `sshd_config.d/01-users.conf` dal template e, se cambia,
   riavvia `sshd`.

Nomi utente, gruppo e shell non sono cablati nei task ma centralizzati in
`vars/main.yml` (`dev_name`, `test_name`, `prod_name`, `prod_group`, ecc.).

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

>Le chiavi pubbliche installate negli `authorized_keys` degli utenti (e
> di root su dev/test) vengono prese da `~/.ssh/id_ed25519.pub` della macchina
> di controllo tramite `lookup('file', ...)`.

---

