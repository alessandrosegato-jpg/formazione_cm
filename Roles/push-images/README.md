# Ruolo `push-images`

Copre il punto "push delle build sul registry precedentemente creato".

**Variabili (`defaults/main.yml`)**

| Variabile | Default | Descrizione |
|---|---|---|
| `registry_host` | `localhost` | Host della registry |
| `registry_port` | `5000` | Porta della registry |
| `image_list` | `ssh-ubuntu-image`, `ssh-rocky-image` | Immagini locali da pubblicare |
| `image_tag` | `latest` | Tag usato per il retag e il push |

**Flusso dei task (`tasks/main.yml`)**

1. verifica dell'esistenza di `/etc/docker/daemon.json` con `stat` e, se presente,
   lettura del contenuto con `slurp` (eseguita solo se il file esiste);
2. costruzione della configurazione con `set_fact`: il contenuto esistente viene
   decodificato da JSON e **fuso** (`combine`) con la voce
   `insecure-registries: ["{{ registry_host }}:{{ registry_port }}"]`, così da
   preservare l'eventuale configurazione già presente invece di sovrascriverla
   (la registry gira in HTTP, senza TLS, quindi va dichiarata come *insecure*);
3. scrittura di `/etc/docker/daemon.json` con `copy` (`to_nice_json`); il task notifica
   l'handler **Riavvia Docker** (`handlers/main.yml`), che riavvia il servizio solo se il file cambia;
4. retag delle immagini locali nel formato `registry_host:registry_port/immagine:tag`
   con `docker_image_tag`, in loop su `image_list`;
5. push con `docker_image_push`, in loop.

---
