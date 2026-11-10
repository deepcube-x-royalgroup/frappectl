# Guida completa a frappectl

`frappectl` è un client da riga di comando per l'API REST v2 di Frappe (v16+),
progettato per essere usato indifferentemente da persone e da agenti AI.

Questa guida documenta l'intera superficie del comando così com'è implementata
nel codice: ogni sottocomando, ogni flag, le regole di parsing dei valori, il
contratto di input/output, i meccanismi di autenticazione e le protezioni di
sicurezza applicate *prima* che una richiesta lasci la macchina.

Versione di riferimento: **0.23.0** · Python richiesto: **>= 3.11**

---

## Indice

1. [Filosofia e modello mentale](#1-filosofia-e-modello-mentale)
2. [Installazione e aggiornamento](#2-installazione-e-aggiornamento)
3. [Autenticazione](#3-autenticazione)
4. [Profili read-only: la protezione principale](#4-profili-read-only-la-protezione-principale)
5. [Opzioni globali](#5-opzioni-globali)
6. [Contratto di input/output](#6-contratto-di-inputoutput)
7. [`auth` — profili e credenziali](#7-auth--profili-e-credenziali)
8. [`doctype` — introspezione dello schema](#8-doctype--introspezione-dello-schema)
9. [`doc` — CRUD e ciclo di vita dei documenti](#9-doc--crud-e-ciclo-di-vita-dei-documenti)
10. [Sintassi dei filtri](#10-sintassi-dei-filtri)
11. [Coercizione dei valori](#11-coercizione-dei-valori)
12. [`report` — esecuzione di report](#12-report--esecuzione-di-report)
13. [`query` — SQL in sola lettura](#13-query--sql-in-sola-lettura)
14. [`method` — discovery e invocazione di metodi](#14-method--discovery-e-invocazione-di-metodi)
15. [`api` — accesso grezzo all'API v2](#15-api--accesso-grezzo-allapi-v2)
16. [`file` — upload e download](#16-file--upload-e-download)
17. [`guide` — primer per agenti](#17-guide--primer-per-agenti)
18. [`assistant` — lanciare un agente come assistente Frappe](#18-assistant--lanciare-un-agente-come-assistente-frappe)
19. [`update` — aggiornamento in place](#19-update--aggiornamento-in-place)
20. [Errori, exit code e suggerimenti](#20-errori-exit-code-e-suggerimenti)
21. [Sicurezza: cosa viene rifiutato e perché](#21-sicurezza-cosa-viene-rifiutato-e-perché)
22. [File, percorsi e variabili d'ambiente](#22-file-percorsi-e-variabili-dambiente)
23. [Endpoint HTTP usati](#23-endpoint-http-usati)
24. [Ricettario](#24-ricettario)
25. [Risoluzione dei problemi](#25-risoluzione-dei-problemi)
26. [Riferimento rapido](#26-riferimento-rapido)

---

## 1. Filosofia e modello mentale

Sei principi guidano ogni scelta di design del CLI (vedi `PHILOSOPHY.md`):

- **Per umani e agenti, allo stesso modo.** Un terminale interattivo ottiene
  tabelle colorate e prompt; una pipe (o `--json`) ottiene JSON pulito su stdout
  e nient'altro. Lo stesso comando serve entrambi i pubblici.
- **Un wrapper sottile e onesto.** I verbi mappano l'API REST v2 di Frappe con
  poca magia, e `api` è sempre la via di fuga: il CLI non nasconde mai la
  piattaforma sottostante.
- **Contratto I/O prevedibile.** stdout è dato, stderr è chiacchiera. Exit code
  `0`/`1`/`2`. Gli errori sono testo semplice con un suggerimento che indica la
  correzione.
- **La conferma spetta al chiamante.** Il CLI non chiede *mai* conferma prima di
  una mutazione, quindi non c'è nulla da aggirare negli script. Umani e harness
  di agenti decidono quando chiedere. La vera protezione sta altrove: un profilo
  read-only rifiuta i verbi non sicuri.
- **Le credenziali restano fuori vista.** Chiave e segreto vivono nell'ambiente
  o nel keyring del sistema operativo — mai in chiaro su disco, mai in flag o
  pipe. OAuth è il login preferito: non memorizza alcun segreto.
- **Auto-orientante.** `guide` e `doctype show` permettono di imparare la
  superficie prima di toccare un sito vivo. Orientati, non indovinare.

### Il flusso canonico

```
frappectl guide                    # 1. impara il CLI (nessun sito, nessuna rete)
frappectl auth list                # 2. scopri quali siti sono raggiungibili
frappectl doctype show "<DocType>" # 3. scopri lo schema prima di indovinare i campi
frappectl doc list ...             # 4. leggi
frappectl doc create|update ...    # 5. scrivi (serve un profilo writable)
```

### Architettura interna (utile per capire i messaggi d'errore)

```
cli.py            app Typer, opzioni globali, hoisting dei flag
  commands/*.py   un modulo per gruppo di comandi
    session.py    ponte da contesto CLI a client configurato
      config.py   profili + credenziali (file JSON + keyring)
      oauth.py    flusso OAuth 2.0 Authorization Code + PKCE
      client.py   facciata pubblica
        resources.py  DocumentsAPI, MethodsAPI, DiscoveryAPI, FilesAPI
          transport.py  l'unico livello che parla HTTP; applica le policy di rete
```

`transport.py` è l'unico punto in cui passa HTTP: lì vivono il rifiuto del
plain HTTP, il blocco dei verbi non sicuri sui profili read-only, il refresh
automatico del token su 401 e la redazione delle credenziali nei log di debug.

---

## 2. Installazione e aggiornamento

```sh
uv tool install frappectl
# oppure
pip install frappectl
# versione di sviluppo
uv tool install git+https://github.com/frappe/frappectl
```

L'installazione fornisce **due** comandi equivalenti:

| Comando | Note |
|---|---|
| `frappectl` | nome completo |
| `fr` | alias breve, identico in tutto |

Dipendenze runtime: `typer>=0.12`, `httpx2>=2.16`, `rich>=13.0`,
`keyring>=24.0`, `authlib>=1.8`.

Aggiornamento: `frappectl update` (vedi [§19](#19-update--aggiornamento-in-place)).

---

## 3. Autenticazione

Esistono **tre** percorsi di autenticazione, con una precedenza precisa.

### 3.1 Ordine di risoluzione

1. **Variabili d'ambiente** — `FRAPPE_SITE` vince sempre su tutto e non tocca
   mai il keyring. Si applica solo se `-s/--site` **non** è stato passato.
2. **Profilo memorizzato** — selezionato con `-s/--site`, oppure il profilo di
   default.
3. Nient'altro. Non esiste un fallback con segreto in chiaro su disco: se il
   keyring è rotto, devi usare le variabili d'ambiente.

### 3.2 Variabili d'ambiente (percorso headless / agente)

```sh
export FRAPPE_SITE=https://erp.example.com
export FRAPPE_API_KEY=xxxxxxxx
export FRAPPE_API_SECRET=yyyyyyyy
export FRAPPE_READ_ONLY=1          # opzionale: rende la sessione read-only
```

Se `FRAPPE_SITE` è impostata ma mancano `FRAPPE_API_KEY` o
`FRAPPE_API_SECRET`, il comando fallisce con exit code `2` e il messaggio
«FRAPPE_SITE is set but FRAPPE_API_KEY / FRAPPE_API_SECRET are missing».

`FRAPPE_READ_ONLY` è considerata vera per i valori `1`, `true`, `yes`, `on`
(case-insensitive, spazi ignorati).

### 3.3 Profili memorizzati con API key/secret

Genera chiave e segreto da **User → Settings → API Access**, poi:

```sh
frappectl auth login https://erp.example.com
```

Il comando è **interattivo per design**: rifiuta di partire se stdin non è un
vero terminale. Chiave e segreto non sono accettati come flag né via pipe,
perché finirebbero nella history della shell, nella lista dei processi e nei
log di CI.

Dove finiscono i dati:

| Dato | Posizione |
|---|---|
| URL del sito, nome profilo, descrizione, flag read-only | `~/.config/frappe/config.json` (permessi `0600`) |
| `api_key:api_secret` | **keyring del sistema operativo**, servizio `frappectl` |

Le installazioni precedenti al rename usavano il servizio keyring
`frappe-cli`: le letture ricadono su quel nome e **migrano in avanti**
automaticamente, così i login esistenti sopravvivono.

### 3.4 OAuth (metodo preferito)

Il login interattivo propone OAuth come default. OAuth **non memorizza nessun
segreto**: conserva un access token a vita breve e un refresh token, e rinnova
l'access token automaticamente.

```sh
frappectl auth login https://erp.example.com --oauth
frappectl auth login https://erp.example.com --oauth --client-id <public-client-id>
frappectl auth login https://erp.example.com --oauth --oauth-port 9876
```

Meccanica implementata in `oauth.py`:

- **Flusso**: Authorization Code + PKCE (`S256`), client *pubblico* (nessun
  secret emesso né memorizzato). La parte protocollare gira attraverso Authlib.
- **Discovery**: `GET <site>/.well-known/oauth-authorization-server`. Se
  risponde 404 si ricade sulla tabella di endpoint noti di Frappe
  (`/api/method/frappe.integrations.oauth2.*`).
- **client_id**: se i metadata annunciano un endpoint di registrazione, il
  client viene registrato dinamicamente (RFC 7591) con un solo redirect URI.
  Altrimenti **devi** passare `--client-id` di un client pubblico già
  registrato sul sito con quel redirect URI esatto. Frappe v15+ supporta la
  registrazione dinamica di default.
- **Redirect URI**: `http://127.0.0.1:9876/callback`. La porta è **fissa**, non
  effimera, perché Frappe confronta il redirect URI byte per byte (non rispetta
  la regola RFC 8252 "qualsiasi porta di loopback"). `--oauth-port` la cambia,
  ma la porta deve corrispondere a un redirect URI registrato.
- **Scope**: `openid all` — `openid` per confermare l'identità via userinfo,
  `all` concede la stessa superficie API di una API key.
- **Timeout del login**: 300 secondi di attesa del redirect dal browser.
- **Refresh**: l'access token viene rinnovato quando mancano **meno di 60
  secondi** alla scadenza, oppure in risposta a un 401 (la richiesta viene
  rigiocata una volta). Se la risposta di refresh non include un nuovo refresh
  token, quello precedente viene conservato (Frappe riusa il vecchio).
- **Persistenza del refresh**: se il keyring non è scrivibile, il token fresco
  resta valido per il processo corrente — un keyring rotto non trasforma un
  refresh riuscito in una richiesta fallita.
- **Logout**: `auth logout` revoca prima il token sul server (best-effort) e poi
  rimuove il profilo; se il server è irraggiungibile il logout locale riesce
  comunque.

OAuth richiede un terminale **e** un browser locale, quindi non è il percorso
headless. Per CI e agenti senza terminale, usa le variabili d'ambiente.

### 3.5 Risoluzione non interattiva: la regola dell'ambiguità

Questa è una delle regole meno ovvie e vale la pena memorizzarla.

Quando l'output **non** è un terminale (pipe, redirect, agente, CI) il profilo
di *default* configurato viene onorato **solo se non è ambiguo**:

| Profili configurati | Interattivo (TTY) | Non interattivo |
|---|---|---|
| 0 | errore: nessun sito configurato | errore: nessun sito configurato |
| 1 | usa quello | usa quello |
| 2+ | usa il default | **errore**: devi passare `-s/--site` o le `FRAPPE_*` |

Il razionale: uno script o un agente non deve mai operare per caso sul sito
sbagliato perché qualcuno ha cambiato il default. Il messaggio d'errore è
esplicito e suggerisce entrambe le soluzioni.

---

## 4. Profili read-only: la protezione principale

Un profilo nuovo è **read-only per default** e rifiuta ogni metodo HTTP non
sicuro **prima che la richiesta lasci la tua macchina**. Questo elimina il
footgun di produzione più ovvio: un comando esplorativo non può mutare il sito
per sbaglio.

I soli verbi permessi su un profilo read-only sono `GET`, `HEAD`, `OPTIONS`.

```sh
frappectl auth login https://prod.example.com --name prod            # read-only
frappectl auth login https://dev.example.com  --name dev --writable  # scrivibile
frappectl auth configure prod --read-only                            # blocca
frappectl auth configure prod --writable                             # sblocca
```

In login interattivo senza flag, la domanda è «Allow writes through this
profile?» con default **no**.

### Cosa viene bloccato e cosa continua a funzionare

| Comando | Verbo HTTP | Su profilo read-only |
|---|---|---|
| `doc list`, `doc get` | GET | ✅ funziona |
| `doctype list`, `doctype show` | GET | ✅ funziona |
| `report run` | GET | ✅ funziona |
| `query` | POST (eccezione, vedi sotto) | ✅ funziona |
| `method search/list/show` | GET | ✅ funziona |
| `file download` | GET | ✅ funziona |
| `auth whoami` | GET | ✅ funziona |
| `doc create` | POST | ❌ bloccato |
| `doc update` | PATCH | ❌ bloccato |
| `doc delete` | DELETE | ❌ bloccato |
| `doc submit/cancel/amend` | POST | ❌ bloccato |
| `file upload` | POST | ❌ bloccato |
| `method call` (default) | POST | ❌ bloccato — vedi nota |
| `api <path>` senza `-X` | GET | ✅ funziona |
| `api <path> -X POST` | POST | ❌ bloccato |

Il messaggio di rifiuto è:

> Refusing to send a POST request: this profile is read-only. Only GET, HEAD,
> and OPTIONS requests are permitted. Use a writable profile, or pass `-X GET`
> for a whitelisted read method.

**Nota su `method call`**: il verbo di default è `POST`, ma su una sessione
read-only diventa automaticamente `GET`, perché è il solo verbo che potrebbe
riuscire. Per invocare esplicitamente un metodo di lettura whitelisted, rendi
il verbo sicuro esplicito: `frappectl api method/… -X GET`.

**Nota su `query`**: `frappectl query` è un POST verso l'endpoint System
Console, ma usa un percorso interno (`request_read_only_post`) che
deliberatamente aggira il controllo sul verbo. Il razionale: il contratto
server-side di quell'endpoint garantisce l'assenza di scritture (la query gira
con `commit: 0`). Non esiste un modo generico per aggirare il blocco dal CLI.

---

## 5. Opzioni globali

Valgono per ogni sottocomando.

| Flag | Effetto |
|---|---|
| `-s`, `--site <profilo>` | Profilo da usare; sovrascrive il profilo di default. Accetta anche `--site=<profilo>` e `-s=<profilo>`. |
| `--json` | Forza l'output JSON (già implicito quando l'output è in pipe). |
| `--debug` | Stampa su stderr ogni richiesta HTTP, e dove disponibile l'SQL eseguito dal server. |
| `--version` | Stampa la versione ed esce. |
| `-h`, `--help` | Aiuto. Disponibile su ogni livello di comando. |

### Posizione dei flag: hoisting automatico

I flag globali funzionano anche in posizione finale, stile `gh`. Prima di
avviare Typer, `cli.py` riordina `argv` spostando `--json`, `--debug` e
`-s/--site <valore>` davanti al sottocomando. Quindi questi sono equivalenti:

```sh
frappectl doc list "Sales Invoice" --json
frappectl --json doc list "Sales Invoice"
frappectl -s raven doc list "Raven Channel"
frappectl doc list "Raven Channel" -s raven
```

L'hoisting si **ferma al primo `--`**: tutto ciò che segue `--` viene lasciato
intatto. Questo è ciò che permette a `frappectl assistant claude -- --json …`
di passare `--json` al tool figlio invece di consumarlo.

### Comportamento dell'help

L'help usa il rendering `click` semplice, senza box, colori o padding
(`rich_markup_mode=None`): su un terminale stretto quelle decorazioni costano
una riga di larghezza e un token per riga senza dire nulla in più. Qualsiasi
comando o gruppo invocato senza argomenti stampa il proprio help
(`no_args_is_help=True`).

### Notifica di aggiornamento

A ogni invocazione, il callback radice chiama `notify_if_outdated`, che può
stampare su stderr una riga gialla «A new version of frappectl is available».
È deliberatamente non invadente:

- solo su output TTY interattivo — **mai** in modalità `--json` o in pipe, così
  non può corrompere stdout machine-readable;
- salta le installazioni editable/di sviluppo e le esecuzioni non da package;
- al massimo **un** controllo di rete ogni 24 ore, con cache su disco, timeout
  2 secondi, e silenzio totale su qualsiasi errore.

---

## 6. Contratto di input/output

### stdout contro stderr

| Canale | Contenuto |
|---|---|
| **stdout** | solo dati: tabelle (TTY) o JSON (pipe / `--json`) |
| **stderr** | errori, suggerimenti, prompt, messaggi di progresso, trace `--debug`, conferme tipo `created ToDo abc123` |

Conseguenza pratica: `frappectl … --json | jq` è sempre sicuro. Nemmeno
`--debug` toccherà mai stdout.

### Rilevamento della modalità

`Output.json` è vero se è stato passato `--json` **oppure** se `sys.stdout` non
è un terminale. Non serve passare `--json` quando si fa pipe.

### Formato JSON

Il JSON su stdout usa un layout compatto scritto a mano: un valore sta su una
riga sola finché la riga rientra in **160 colonne**, e si espande solo quando
non ci sta. Il razionale è il costo in token: padding e indentazione profonda
costano un token per riga e non dicono nulla al lettore. Una riga figlia o una
lista corta restano quindi su una riga.

Il risultato viene **riparsato prima di essere emesso**: un chiamante che fa
pipe non deve mai ricevere JSON rotto. Se il layout compatto fallisse (sarebbe
un bug), un avviso va su stderr e il valore viene riscritto con l'encoder
standard.

### Rendering TTY

- Liste → tabella `rich` con header ciano in grassetto, celle che vanno a capo
  (`overflow="fold"`). Se non ci sono righe, su stderr appare `No records.`
- Record singolo → tabella chiave/valore a due colonne senza bordi.
- Booleani → `✓` per vero, cella vuota per falso.
- `None` → cella vuota.
- Dict e liste annidate → JSON su una riga nella cella.
- Sorgente Python (da `method show`) → syntax highlighting con sfondo
  trasparente, così si fonde col tema del terminale.

### Nessuna conferma, mai

Il CLI non chiede conferma prima di una mutazione. Non esiste `--yes` perché
non esiste nulla da confermare. La conferma è responsabilità del chiamante —
che sia tu o un harness di agente. La protezione che conta è il profilo
read-only.

### `--debug` in dettaglio

```sh
frappectl doc list ToDo --debug
```

Su stderr compaiono:

```
→ GET https://erp.example.com/api/v2/document/ToDo?start=0&limit=20&debug=true
  host: erp.example.com
  authorization: token ***
  accept: application/json
  user-agent: frappectl
← 200 OK
  [server] SELECT `name` FROM `tabToDo` ...
```

Dettagli importanti:

- L'header `Authorization` è **sempre redatto**: resta visibile solo lo schema
  (`token` o `Bearer`), il valore diventa `***`. La credenziale non viene
  mostrata nemmeno al terminale locale.
- Il body della richiesta viene stampato, troncato a **2000 caratteri**; un
  body binario viene riassunto come `<N bytes of binary body>`.
- Per gli endpoint che lo espongono (come `doc list`, che aggiunge
  `debug=true` alla query) viene stampato l'SQL eseguito server-side.
- Su una richiesta fallita, `--debug` stampa il **traceback completo del
  server**. Senza `--debug`, se il body conteneva un traceback, l'errore
  include il suggerimento «re-run with --debug to see the full server
  traceback».
- Caratteri di controllo e sequenze ANSI vengono rimossi dall'output del
  server, così un server ostile non può iniettare escape sequence nel tuo
  terminale. Tab e newline sono preservati.

---

## 7. `auth` — profili e credenziali

```
frappectl auth login <url> [opzioni]
frappectl auth list
frappectl auth whoami
frappectl auth default <profilo>
frappectl auth configure <profilo> [opzioni]
frappectl auth logout <profilo>
```

### `auth login`

```sh
frappectl auth login https://erp.example.com
frappectl auth login https://raven.example.com --name raven --default
frappectl auth login https://erp.example.com --oauth --description "ERP di produzione"
```

| Flag | Default | Descrizione |
|---|---|---|
| `<url>` | obbligatorio | URL del sito. Se manca lo schema viene normalizzato a `https://`; lo slash finale viene rimosso. |
| `--name <nome>` | prompt, default = host del sito | Nome breve del profilo. |
| `--description <testo>` | prompt, default vuoto | Nota descrittiva; la modalità assistant la usa per scegliere il sito. |
| `--default` / `--no-default` | `--no-default` | Rende questo il profilo di default. **Il primo profilo diventa sempre il default**, anche senza il flag. |
| `--read-only` / `--writable` | prompt, default read-only | Se il profilo rifiuta le scritture. |
| `--oauth` | off | Usa OAuth senza mostrare il prompt sul metodo di autenticazione. |
| `--client-id <id>` | riusa quello memorizzato | client_id OAuth pubblico, per siti senza registrazione dinamica. |
| `--oauth-port <porta>` | `9876` | Porta di loopback fissa per il redirect OAuth. |

Comportamenti da conoscere:

- **Richiede un terminale vero.** Se stdin non è un TTY, il comando rifiuta con
  exit code `2` e indirizza verso le variabili d'ambiente. Questo impedisce che
  le credenziali vengano iniettate via pipe, heredoc o redirect — tutte strade
  che finiscono nella history o nei log.
- **Verifica prima di memorizzare.** Le credenziali vengono provate con una
  chiamata a `frappe.auth.get_logged_user`; solo se l'identità viene confermata
  (utente diverso da `Guest`) il profilo viene salvato. Non vengono mai
  persistite credenziali morte.
- **Riuso del client OAuth.** Un re-login su un profilo OAuth esistente riusa il
  `client_id` già registrato, così il sito non si riempie di righe `OAuth
  Client` duplicate. Un `--client-id` esplicito vince.
- **Scrittura transazionale.** Se il salvataggio del file di config fallisce, il
  segreto precedente nel keyring viene ripristinato.

Alla fine viene stampata su stderr una riga di conferma:

```
logged in as Administrator — profile 'erp' [oauth] (default) [read-only]
```

### `auth list`

```sh
frappectl auth list
frappectl auth list --json
```

Colonne: `profile`, `site`, `auth`, `description`, `writable`, `default`.

Nota la polarità: la colonna è **`writable`**, non `read_only`. Le scritture
sono il rischio, quindi sono i profili che le permettono a essere marcati.

Se non ci sono profili, su stderr compare un suggerimento con il comando di
login.

### `auth whoami`

```sh
frappectl auth whoami
frappectl auth whoami -s prod --json
```

Risolve il profilo attivo, apre una connessione e interroga il server.
Campi restituiti:

| Campo | Significato |
|---|---|
| `site` | URL risolto e normalizzato |
| `user` | utente autenticato secondo il server |
| `source` | `env` se da variabili d'ambiente, altrimenti il nome del profilo |
| `auth` | `oauth` o `api_key` |
| `description` | descrizione del profilo |
| `read_only` | se le scritture sono rifiutate |
| `token_expires_at` | solo OAuth: scadenza ISO-8601 UTC del token **in uso dopo un eventuale refresh**, non di quello stale memorizzato |

È il comando diagnostico da usare per primo ogni volta che un errore parla di
permessi o autenticazione.

### `auth default`

```sh
frappectl auth default raven
```

Cambia il profilo di default. Ricorda che in modalità non interattiva il
default viene ignorato se esistono più profili ([§3.5](#35-risoluzione-non-interattiva-la-regola-dellambiguità)).

### `auth configure`

```sh
frappectl auth configure prod --name production
frappectl auth configure prod --description "Nuovo testo"
frappectl auth configure prod --description ""        # svuota la descrizione
frappectl auth configure prod --read-only
frappectl auth configure prod --writable
frappectl auth configure prod                          # prompt interattivo
```

| Flag | Descrizione |
|---|---|
| `--name <nuovo>` | Rinomina il profilo; sposta il segreto nel keyring e il puntatore di default. |
| `--description <testo>` | Nuova descrizione; stringa vuota per cancellarla. |
| `--read-only` / `--writable` | Commuta la policy di scrittura. |

Comportamenti:

- Senza nessun flag su un terminale, nome e descrizione vengono chiesti con i
  valori correnti come default; la policy read-only resta invariata se il flag
  non è passato.
- Senza nessun flag e **senza** terminale, il comando rifiuta con exit code `2`:
  non c'è nulla da cambiare e nessun posto dove chiedere.
- **Le credenziali non vengono mai toccate** qui. Per reinserire una chiave
  API usa `auth login`.
- Il rename è progettato per ripristinare lo stato precedente se il salvataggio
  del config o la cancellazione del vecchio segreto fallisce a metà.

### `auth logout`

```sh
frappectl auth logout prod
```

Rimuove il profilo e le sue credenziali. Per un profilo OAuth, l'access token
viene prima revocato sul server (best-effort), così il logout chiude davvero la
sessione invece di limitarsi a dimenticarla localmente. Se la revoca non va a
buon fine (server irraggiungibile), la rimozione locale procede comunque.

Se il profilo rimosso era il default, il default passa al primo profilo
rimanente (o a nessuno).

---

## 8. `doctype` — introspezione dello schema

**Orientati prima di indovinare.** Questo è il gruppo di comandi che evita il
90% degli errori «unknown column» e «mandatory field».

### `doctype list`

```sh
frappectl doctype list
frappectl doctype list --module HR
frappectl doctype list --custom
frappectl doctype list --module HR --custom --limit 50
```

| Flag | Default | Descrizione |
|---|---|---|
| `--module <nome>` | — | Filtra per modulo. |
| `--custom` | off | Solo DocType custom. |
| `--limit <n>` | `10000` | Numero massimo di righe. |

Dettaglio di implementazione utile: la lista passa da
`frappe.desk.search.search_link`, non dalla lettura dei documenti `DocType`.
Leggere i documenti `DocType` richiederebbe il ruolo System Manager, ma la
ricerca per link su `DocType` ignora i permessi, quindi **qualunque utente desk
può elencare i DocType**. I nomi vengono restituiti ordinati
alfabeticamente. In modalità JSON l'output è un semplice array di stringhe.

### `doctype show`

```sh
frappectl doctype show "Sales Invoice"
frappectl doctype show "Sales Invoice" --json
frappectl doctype show "Sales Invoice" --raw
```

| Flag | Descrizione |
|---|---|
| `--raw` | Stampa i meta completi non elaborati. Usalo **solo** per scripting: l'output è molto grande. |

Senza `--raw`, l'output è ridotto a ciò che serve per costruire una chiamata
corretta:

- **Intestazione**: `name`, `module`, `is_submittable`, `issingle`,
  `title_field`, `autoname`, e `is_virtual` solo se vero. In JSON si aggiunge
  `istable`.
- **Campi**: per ciascuno `fieldname`, `label`, `fieldtype`, più `options` solo
  se presente e `reqd: true` solo se obbligatorio. I fieldtype di puro layout
  (`Section Break`, `Column Break`, `Tab Break`, `HTML`, `Heading`) sono
  esclusi.
- **Permessi**: in JSON, per ogni regola vengono mostrate solo le chiavi con
  valore diverso da zero (più `permlevel`, sempre), e la boilerplate dei
  documenti figlio viene rimossa.
- La colonna `in_list_view` compare solo nel rendering TTY, non in JSON.

Le informazioni chiave da leggere prima di scrivere:

| Campo | Perché conta |
|---|---|
| `reqd` | i campi obbligatori: senza di essi `doc create` falla con un errore «mandatory» |
| `fieldtype` | distingue scalari, `Link`, `Table` (tabelle figlio), `Select` |
| `options` | per un `Link` è il DocType puntato; per un `Select` sono i valori ammessi |
| `is_submittable` | se `submit`/`cancel`/`amend` hanno senso |
| `issingle` | un singleton non si elenca come una collezione |
| `autoname` | se il `name` è generato dal server o lo devi fornire |

---

## 9. `doc` — CRUD e ciclo di vita dei documenti

```
frappectl doc list   <DocType> [opzioni]
frappectl doc get    <DocType> <name>
frappectl doc create <DocType> [--set k=v]… [--input file.json]
frappectl doc update <DocType> <name> [--set k=v]… [--input file.json] [--force]
frappectl doc delete <DocType> <name>
frappectl doc submit <DocType> <name>
frappectl doc cancel <DocType> <name>
frappectl doc amend  <DocType> <name>
```

### `doc list`

```sh
frappectl doc list ToDo
frappectl doc list "Sales Invoice" -f status=Overdue -f 'grand_total>1000'
frappectl doc list "Sales Invoice" --fields name,customer,grand_total --limit 50
frappectl doc list "Sales Invoice" --fields '*' --limit 1
frappectl doc list "Sales Invoice" --order-by 'modified desc'
frappectl doc list "Sales Invoice" --all --json
frappectl doc list "Sales Invoice" --filters-json '[["status","in",["Paid","Overdue"]]]'
```

| Flag | Default | Descrizione |
|---|---|---|
| `-f <filtro>` | — | Filtro semplice, ripetibile. Vedi [§10](#10-sintassi-dei-filtri). |
| `--filters-json <json>` | — | Sintassi filtri Frappe completa come JSON. **Vince su `-f`.** |
| `--fields <lista>` | guidato dai meta | Campi separati da virgola, oppure `*` per tutti. |
| `--order-by <espr>` | `creation desc` | Ordinamento. |
| `--limit <n>` | `20` | Righe massime. |
| `--all` | off | Auto-paginazione di tutte le righe corrispondenti. |

**Campi di default.** Se non passi `--fields`, il CLI legge i meta del DocType
e costruisce colonne simili a quelle del Desk: `name`, più il `title_field`,
più tutti i campi con `in_list_view`, escludendo i fieldtype di layout e i
`Button`/`Fold`. Se la lettura dei meta falla, ricade su `["name"]`. Con
`--fields '*'` il parametro `fields` non viene inviato affatto e il server
decide.

**Paginazione con `--all`.** Richiede pagine da 200 righe finché il server non
segnala `has_next_page: false` o restituisce un batch vuoto. Il progresso
(`fetched 400…`) va su **stderr** con ritorno a capo carrello, quindi non
sporca mai lo JSON su stdout. Attenzione: su una collezione grande questo
significa molte richieste e un payload grande in memoria — restringi i filtri
e i campi prima di usarlo.

**Con `--debug`** la richiesta include `debug=true` e il server restituisce
l'SQL eseguito, che viene stampato su stderr.

### `doc get`

```sh
frappectl doc get "Sales Invoice" SINV-0001
frappectl doc get "Sales Invoice" SINV-0001 --json
```

Restituisce il documento completo, ma **snellito** (`_slim_document`) sia in
TTY sia in JSON:

- i campi con valore `null` vengono rimossi — un campo nullo dice solo che il
  campo esiste, cosa che lo schema già afferma;
- il campo `idx` viene rimosso;
- nelle righe delle tabelle figlio viene rimossa la boilerplate che ripete il
  genitore: `name`, `doctype`, `owner`, `creation`, `modified`, `modified_by`,
  `docstatus`, `parent`, `parentfield`, `parenttype`, `idx`; e anche lì i valori
  `null` sono scartati.

Il razionale è il costo: su un documento con tabelle grandi quella boilerplate
domina l'output. Se ti serve il documento *esattamente* come lo manda il
server, usa la via di fuga: `frappectl api document/<DocType>/<name>`.

### `doc create`

```sh
frappectl doc create ToDo --set description="Follow up" --set priority=High
cat invoice.json | frappectl doc create "Sales Invoice"
frappectl doc create "Sales Invoice" --input invoice.json
frappectl doc create "Sales Invoice" --input base.json --set status=Draft
```

| Flag | Descrizione |
|---|---|
| `--set field=value` | Scalare, ripetibile. Il valore viene coercito (vedi [§11](#11-coercizione-dei-valori)). |
| `--input <file>` | File JSON, per tabelle figlio e strutture annidate. |

Regole di composizione dell'input:

1. Se è presente `--input`, il file viene letto; **altrimenti** se stdin non è
   un terminale, viene letto stdin. (`--input` ha la precedenza su stdin.)
2. L'input deve essere un **oggetto** JSON; un array o uno scalare danno exit
   code `2`.
3. I valori `--set` vengono applicati **sopra** l'input, quindi vincono in caso
   di conflitto.
4. Una chiave `doctype` nell'input viene rimossa: il DocType è già
   l'argomento posizionale.
5. JSON non valido su stdin o nel file dà exit code `2` con il dettaglio del
   parse error.

Su TTY stampa su stderr `created ToDo abc123`; su stdout emette il documento
creato (con il `name` assegnato dal server).

### `doc update`

```sh
frappectl doc update ToDo abc123 --set status=Closed
frappectl doc update ToDo abc123 --input patch.json
frappectl doc update ToDo abc123 --set status=Closed --force
```

| Flag | Descrizione |
|---|---|
| `--set field=value` | Scalare, ripetibile. |
| `--input <file>` | File JSON. |
| `--force` | Salta il controllo di concorrenza ottimistica (sovrascrive). |

**Concorrenza ottimistica — il comportamento di default.** Prima di inviare la
PATCH, se il payload non contiene già `modified`, il CLI legge il documento
corrente e allega il suo timestamp `modified`. Se il documento è stato
modificato nel frattempo, il server rifiuta e il CLI traduce l'errore in:

> `<DocType> <name>` was modified since you read it. Re-read and retry, or pass
> `--force` to overwrite.

Il conflitto è riconosciuto da uno status `409`, oppure da un messaggio che
contiene `timestampmismatch`, `modified after you have opened` o `please
refresh`.

Con `--force`, un eventuale `modified` viene **rimosso** dal payload e la
scrittura procede sovrascrivendo.

Altre regole:

- Le chiavi `doctype` e `name` vengono rimosse dal payload.
- Se dopo tutto questo non resta nulla da aggiornare, exit code `2` con
  «Nothing to update. Use --set, --input, or pipe JSON.»
- L'input segue le stesse regole di `doc create` (`--input` > stdin, `--set`
  sopra tutto).

### `doc delete`

```sh
frappectl doc delete ToDo abc123
```

Nessuna conferma, nessun flag. In JSON restituisce
`{"deleted": "abc123", "doctype": "ToDo"}`; su TTY stampa `deleted ToDo abc123`
su stderr.

### `doc submit` / `doc cancel`

```sh
frappectl doc submit "Sales Invoice" SINV-0001
frappectl doc cancel "Sales Invoice" SINV-0001
```

Invocano il metodo di controller omonimo sul documento e poi **rileggono** il
documento, così l'output riflette lo stato post-transizione (`docstatus` 1 per
submit, 2 per cancel).

### `doc amend`

```sh
frappectl doc amend "Sales Invoice" SINV-0001
```

L'amend è implementato lato client in tre passi:

1. Legge il documento originale e **verifica che sia cancellato**
   (`docstatus == 2`). Se non lo è, falla con «is not cancelled; only cancelled
   documents can be amended».
2. Copia tutti i campi tranne `name`, `owner`, `creation`, `modified`,
   `modified_by`, `idx`.
3. Imposta `docstatus: 0` e `amended_from: <name originale>`, poi crea il nuovo
   documento.

Su TTY stampa `amended Sales Invoice SINV-0001 → SINV-0001-1`.

---

## 10. Sintassi dei filtri

### Filtri semplici con `-f`

Ripeti il flag per combinare più condizioni (sono in AND):

```sh
frappectl doc list ToDo -f status=Open -f priority=High
frappectl doc list "Sales Invoice" -f 'grand_total>1000'
frappectl doc list "Sales Invoice" -f 'grand_total>=1000' -f 'grand_total<=5000'
frappectl doc list ToDo -f 'subject like %report%'
frappectl doc list ToDo -f 'subject not like %draft%'
frappectl doc list ToDo -f 'status!=Closed'
```

Operatori supportati:

| Operatore | Forma | Nota |
|---|---|---|
| `=` | `field=value` | |
| `!=` | `field!=value` | |
| `>` | `field>value` | |
| `<` | `field<value` | |
| `>=` | `field>=value` | |
| `<=` | `field<=value` | |
| `like` | `field like %x%` | operatore a parole, spazi obbligatori attorno |
| `not like` | `field not like %x%` | operatore a parole |

Dettagli di parsing che evitano sorprese:

- Gli operatori a parole (` like `, ` not like `) vengono cercati **per primi**,
  case-insensitive, e `not like` prima di `like`.
- Gli operatori simbolici vengono provati **dal più lungo al più corto**
  (`>=`, `<=`, `!=`, `=`, `>`, `<`), così `grand_total>=1000` non viene letto
  come `grand_total>` + `=1000`.
- L'operatore deve trovarsi a indice **maggiore di zero**: un token che comincia
  con l'operatore non ha un nome di campo e viene rifiutato.
- Un token non parsabile dà exit code `2` con un messaggio che indica le forme
  valide e suggerisce `--filters-json`.
- **Quoting della shell**: `>` e `<` sono redirezioni per la shell. Metti sempre
  il filtro tra apici singoli: `-f 'grand_total>1000'`.

### Filtri complessi con `--filters-json`

Per `in`, `not in`, `between`, filtri su tabelle figlio, o qualsiasi cosa che
non entra pulitamente in un argomento di shell, passa il JSON dei filtri Frappe:

```sh
frappectl doc list "Sales Invoice" \
  --filters-json '[["status","in",["Paid","Overdue"]]]'

frappectl doc list "Sales Invoice" \
  --filters-json '[["posting_date","between",["2026-01-01","2026-03-31"]]]'

frappectl doc list "Sales Invoice" \
  --filters-json '[["Sales Invoice Item","item_code","=","ITEM-001"]]'
```

`--filters-json` **ha la precedenza** su `-f`: se passi entrambi, i `-f` vengono
ignorati. JSON non valido dà exit code `2`.

### Filtri nei report

`report run` accetta gli stessi flag ma i report prendono i filtri come un
dizionario piatto `{campo: valore}`. Quindi:

- con `-f` sono ammessi **solo** `field=value`; qualunque altro operatore dà
  exit code `2` con il suggerimento di usare `--filters-json`;
- `--filters-json` per un report deve essere un **oggetto** JSON, non un array.

---

## 11. Coercizione dei valori

Vale per i valori di `--set` e per i valori dei filtri `-f`, e (nella forma
`key=value`) per `-F` di `method call` e `api`.

| Input | Risultato | Tipo |
|---|---|---|
| `""` (vuoto) | `""` | stringa vuota |
| `"testo"` o `'testo'` | `testo` | stringa (apici rimossi) |
| `true` / `True` / `TRUE` | `True` | booleano |
| `false` | `False` | booleano |
| `null`, `none` (qualsiasi case) | `None` | null |
| `42` | `42` | intero |
| `3.14` | `3.14` | float |
| `007` | `"007"` | **stringa** |
| `0123` | `"0123"` | **stringa** |
| `0` | `0` | intero |
| `0.5` | `0.5` | float |
| `-7` | `-7` | intero |
| `ITEM-001` | `"ITEM-001"` | stringa |

**La regola degli zeri iniziali** merita attenzione. I valori che *sembrano*
interi ma hanno uno zero iniziale significativo (CAP, numeri di telefono,
codici articolo come `007`) restano **stringhe**: coercirli a intero
cancellerebbe silenziosamente gli zeri e corromperebbe il valore scritto. `0` e
i decimali come `0.5` sono esenti dalla regola e vengono coerciti normalmente.

**Forzare una stringa.** Metti il valore tra apici *dentro* l'argomento:

```sh
--set 'code="42"'       # stringa "42", non intero 42
--set "code='42'"       # idem
```

**Per valori strutturati** (oggetti, array) usa `--input` con un file JSON, o la
forma `-F key:=<json>` dove disponibile.

---

## 12. `report` — esecuzione di report

```sh
frappectl report run "Accounts Receivable" -f company="Frappe"
frappectl report run "Accounts Receivable" -f company="Frappe" --json
frappectl report run "General Ledger" --filters-json '{"company":"Frappe","from_date":"2026-01-01"}'
```

| Flag | Descrizione |
|---|---|
| `-f field=value` | Filtro, ripetibile. **Solo** `=`. |
| `--filters-json <json>` | Filtri come oggetto JSON. |

Comportamento:

- Esegue `frappe.desk.query_report.run` in **GET**, quindi funziona anche su un
  profilo read-only.
- Le righe restituite da Frappe possono essere liste posizionali o dizionari. Il
  CLI normalizza: le liste vengono zippate con le chiavi delle colonne
  (`fieldname`, o `label` come fallback, o la parte prima dei `:` per le colonne
  in formato stringa). Le righe vuote vengono scartate.
- In modalità JSON l'output è `{"columns": [...], "result": [...]}` — le colonne
  **grezze** del server e le righe grezze, non quelle normalizzate. Se stai
  scriptando, aspettati le righe nella forma originale.
- Su TTY vedi una tabella con il nome del report come titolo.

---

## 13. `query` — SQL in sola lettura

```sh
frappectl query 'select count(*) as users from tabUser'
frappectl query 'select name, creation from tabToDo order by creation desc limit 10' --json
frappectl query - < query.sql
echo 'select 1 as x' | frappectl query -
frappectl query --input query.sql
frappectl query -i query.sql
```

| Argomento / flag | Descrizione |
|---|---|
| `QUERY` | La query SQL. `-` legge da stdin. |
| `--input`, `-i <file>` | Legge l'SQL da un file; `-` per stdin. |

Regole:

- Passare **sia** `QUERY` **sia** `--input` dà exit code `2`.
- Non passare nessuno dei due dà exit code `2`.
- Una query vuota o di soli spazi dà exit code `2`.

### Requisiti e meccanica

La query passa attraverso **System Console**
(`frappe.desk.doctype.system_console.system_console.execute_code` con
`type: "SQL"` e `commit: 0`). Conseguenze:

- Serve il ruolo **System Manager** o l'utente **Administrator**. Senza, la
  chiamata torna un errore di permesso.
- È una POST, ma usa il percorso interno che aggira il controllo read-only,
  perché il contratto server-side garantisce l'assenza di scritture. Quindi
  `query` funziona anche su un profilo read-only.
- System Console **cattura** gli errori SQL e di permesso e li mette nel campo
  `output` **mantenendo una risposta HTTP di successo**. Il CLI rileva questo
  caso: se `output` non è JSON parsabile, lo tratta come errore; se comincia con
  `Traceback`, estrae l'ultima riga come messaggio. Quindi un errore SQL ti
  arriva comunque come errore con exit code `1`, non come successo silenzioso.
- Risposte con una forma inattesa (nessun `output`, `output` non lista, righe
  non dizionari) producono un errore esplicito.

Le colonne della tabella sono dedotte dall'unione ordinata delle chiavi di tutte
le righe, così una query che restituisce righe con chiavi diverse non perde
colonne.

---

## 14. `method` — discovery e invocazione di metodi

Il gruppo `method` si appoggia agli endpoint `/api/v2/discovery` di Frappe.

### I due tipi di metodo

| `kind` | Che cos'è | Come si invoca |
|---|---|---|
| `rpc` | funzione whitelisted, percorso con punti | `/api/v2/method/<path>` |
| `doctype` | metodo di controller whitelisted | `/api/v2/document/<DocType>/<name>/method/<method>` — richiede un documento esistente |

Nelle liste, i due tipi vengono ripiegati in un'unica colonna `ref`:

- `rpc` → il percorso puntato, es. `frappe.tests.test_api.test`
- `doctype` → `DocType.metodo`, es. `User.populate_role_profile_roles`

Un `kind` ignoto (compatibilità in avanti) non fa cadere la riga: viene usato
qualunque identificatore l'entry porti.

### Visibilità e requisiti

- **Tutti gli endpoint di discovery richiedono il ruolo System Manager.**
- I risultati sono filtrati per la sessione corrente: un guest vede i metodi
  accessibili ai guest, una sessione autenticata vede i propri metodi
  whitelisted, e gli Server Script di tipo API appaiono solo con permesso di
  lettura su `Server Script`.
- La discovery è servita da cache. A cache fredda il server risponde `503`
  mentre genera: il CLI **ritenta fino a 4 volte**, rispettando l'header
  `Retry-After` (fallback 2 secondi, tetto 30 secondi), stampando su stderr
  «Discovery cache is being generated; retrying in N seconds.» Così un comando
  si riprende da una cache fredda senza mai restare appeso.
- Se la feature non esiste sul sito, il messaggio è «Method discovery is not
  available on this site.» Il CLI distingue questo caso dal «metodo non
  trovato» sondando la radice della discovery una volta.

### `method search`

```sh
frappectl method search --query unread
frappectl method search -q "invoice" --json
```

| Flag | Descrizione |
|---|---|
| `--query`, `-q <termini>` | Obbligatorio. Cerca per kind, path, doctype, nome e docstring. |

### `method list`

```sh
frappectl method list
frappectl method list --doctype User
frappectl method list --json
```

| Flag | Descrizione |
|---|---|
| `--doctype <nome>` | Elenca i metodi di un solo DocType. È una lettura **live** (non da cache) e include i metodi standard ereditati. |

Con `--doctype`, un 404 significa che il DocType è ignoto o non ha metodi
scopribili (è il caso delle tabelle figlio e dei nomi sbagliati), non che la
feature manca: il CLI verifica e dà il messaggio giusto.

### `method show`

```sh
frappectl method show frappe.ping
frappectl method show frappe.tests.test_api.test
frappectl method show --doctype User add_comment
```

| Flag | Descrizione |
|---|---|
| `--doctype <nome>` | Risolve l'argomento come metodo di quel DocType. |

Per un metodo `rpc` mostra: `path`, `name`, `allow_guest`, `http_methods`,
`endpoint`, `docstring`. Per un metodo `doctype`: `doctype`, `method`,
`defined_in`, `http_methods`, `permission`, `endpoint`, `docstring`.

Poi, in entrambi i casi, la tabella dei **parametri** (`name`, `type`,
`required`, `default`) o «No parameters.», e infine il **sorgente** Python con
syntax highlighting — ma solo se l'app del metodo lo espone tramite l'hook
`expose_discovery_source`; gli altri siti omettono la chiave del tutto.

### `method call`

```sh
frappectl method call frappe.ping
frappectl method call gameplan.api.get_unread_count -F project=1
frappectl method call add_comment --doctype User --name Administrator \
  -F comment_type=Comment -F content="Ciao"
frappectl method call frappe.client.get_count -F doctype=User -X GET
```

| Flag | Descrizione |
|---|---|
| `--doctype <nome>` | Invoca il metodo su un documento di questo DocType. |
| `--name <nome>` | Nome del documento esistente. **Obbligatorio** con `--doctype`. |
| `-F`, `--field key=value` | Parametro con valore tipizzato; `key:=value` invia JSON grezzo. Ripetibile. |
| `-f`, `--raw-field key=value` | Parametro sempre stringa. Ripetibile. |
| `--method`, `-X <verbo>` | Verbo HTTP. Default `POST`, oppure `GET` su sessione read-only. |

Validazioni: `--doctype` senza `--name` dà exit code `2` («--name is required
when --doctype is given»), e `--name` senza `--doctype` pure («--name is only
valid together with --doctype»). Un `-f` senza `=` dà exit code `2`.

L'output di `method call` è **sempre JSON**, anche su TTY: il valore di ritorno
di un metodo arbitrario non ha una forma tabellare prevedibile.

### `-F` contro `-f`: la differenza che conta

| Forma | Esempio | Cosa viene inviato |
|---|---|---|
| `-F key=value` | `-F limit=100` | intero `100` (coercizione, vedi [§11](#11-coercizione-dei-valori)) |
| `-F key=value` | `-F unread=true` | booleano `True` |
| `-F key:=value` | `-F 'filters:={"enabled":1}'` | oggetto JSON strutturato |
| `-F key:=value` | `-F 'emails:=["a@x.it","b@x.it"]'` | array JSON |
| `-f key=value` | `-f limit=100` | stringa `"100"` |

I valori strutturati vengono inviati come JSON nativo nel body di una richiesta
con body, e codificati in JSON nella query string per le GET — quindi lo stesso
`-F` si comporta in modo identico nei due casi. Un `-F key:=` con JSON non
valido dà exit code `2` con il dettaglio del parse error.

---

## 15. `api` — accesso grezzo all'API v2

È la via di fuga senza zucchero. Tutti i verbi `doc` sono zucchero sopra questo
stesso client; quando i verbi non bastano, qui c'è la piattaforma nuda.

```sh
frappectl api method/frappe.client.get_count -F doctype=User
frappectl api method/gameplan.api.get_unread_count
frappectl api method/frappe.client.get_list -F doctype=User -F 'filters:={"enabled":1}'
frappectl api document/ToDo --method GET
frappectl api document/ToDo/abc123
frappectl api document/ToDo -X POST --input nuovo.json
cat patch.json | frappectl api document/ToDo/abc123 -X PATCH --input -
```

| Flag | Default | Descrizione |
|---|---|---|
| `<path>` | obbligatorio | Percorso API, es. `method/frappe.ping` o `document/User`. |
| `-F`, `--field key=value` | — | Valore tipizzato; `key:=value` per JSON grezzo. Ripetibile. |
| `-f`, `--raw-field key=value` | — | Valore sempre stringa. Ripetibile. |
| `--method`, `-X <verbo>` | `GET`, o `POST` se c'è `--input` | Verbo HTTP. |
| `--input <file>` | — | File JSON per il body; `-` legge da stdin. |

Normalizzazione del percorso: lo slash iniziale viene rimosso, e un prefisso
`api/v2/` eventualmente presente viene tolto per non duplicarlo. Il percorso
finale è sempre `/api/v2/<path>`.

Regole su verbo e body:

- Senza `--input` e senza `-X`, il verbo è `GET` e i parametri `-F`/`-f` vanno
  nella query string.
- Con `--input` e senza `-X`, il verbo diventa `POST` e il contenuto del file
  diventa il body JSON; i `-F`/`-f` vanno nella query string **accanto** al body.
- Con `-X GET` **e** `--input`, un body non avrebbe senso: il JSON deve essere
  un **oggetto** e le sue chiavi di primo livello vengono fuse nei parametri di
  query. Un array o uno scalare dà exit code `2`.
- Senza `--input`, per un verbo con body i parametri `-F`/`-f` diventano **il
  body**.

L'output è sempre JSON su stdout.

Il livello di trasporto rimuove i parametri con valore `None` e codifica in JSON
i valori dict/list, perché `httpx2` serializza gli scalari per la query string
ma non i container annidati, e Frappe li aspetta comunque come stringhe JSON.

---

## 16. `file` — upload e download

### `file upload`

```sh
frappectl file upload ./contract.pdf
frappectl file upload ./contract.pdf --private
frappectl file upload ./contract.pdf --doctype "Sales Invoice" --name SINV-0001
frappectl file upload ./logo.png --doctype "Company" --name "Frappe" --fieldname company_logo
```

| Flag | Descrizione |
|---|---|
| `--doctype <nome>` | Allega al DocType indicato. |
| `--name <nome>` | Allega al documento indicato. **Richiede `--doctype`.** |
| `--fieldname <campo>` | Allega al campo indicato. |
| `--private` | Marca il file come privato. |

Validazioni locali: se il percorso non è un file esistente, exit code `2`; se
passi `--name` senza `--doctype`, exit code `2`.

Il file viene inviato come multipart a `/api/v2/method/upload_file` con
`is_private` (0/1) e `folder: "Home"`. Su TTY stampa
`uploaded contract.pdf → /private/files/contract.pdf` su stderr, e il record
`File` su stdout.

### `file download`

```sh
frappectl file download abc123def -o contratto.pdf
frappectl file download /files/contract.pdf
frappectl file download /private/files/contract.pdf -o -   # su stdout
frappectl file download abc123def --json
```

| Flag | Descrizione |
|---|---|
| `<ref>` | Nome del documento `File`, oppure un URL `/files/…`. |
| `--output`, `-o <percorso>` | Percorso di destinazione. Default: nome derivato. `-` scrive su stdout. |

Risoluzione del riferimento: se `ref` non comincia con `/` e non contiene
`://`, viene trattato come **nome di documento `File`**: il CLI lo legge, ne
prende `file_url` e `file_name`. Se il documento non ha `file_url`, errore.
Altrimenti `ref` è usato direttamente come URL.

Scelta del nome di output, in ordine: `--output`, poi il `file_name` del
documento, poi il basename dell'URL senza query string, poi `download`.

Protezioni di sicurezza nel download — vale la pena conoscerle:

- Il `file_name` arriva dal server. Viene passato per `basename`, così un nome
  costruito ad arte come `../../x` non può uscire dalla directory corrente.
- Il file non viene scritto direttamente sulla destinazione: viene scritto in un
  **file temporaneo creato in modo esclusivo** nella stessa directory (prefisso
  `.<nome>.`, suffisso `.part`) e poi rinominato atomicamente. Un percorso
  prevedibile come `<output>.part` potrebbe essere un symlink verso un altro
  file, e il download finirebbe per sovrascrivere *quello*.
- Su qualsiasi errore il temporaneo viene rimosso.
- Il corpo viene **streammato** a blocchi, non bufferizzato in memoria: funziona
  anche su file grandi. Con `-o -` lo stream va direttamente su
  `sys.stdout.buffer`.

In JSON l'output è `{"saved": "...", "bytes": N, "file_url": "..."}`; su TTY
`downloaded contratto.pdf (12345 bytes)` su stderr.

---

## 17. `guide` — primer per agenti

```sh
frappectl guide
```

Stampa un primer completo del CLI che **non richiede né sito né rete**. Un
agente può eseguirlo come primissimo passo per imparare il CLI prima di toccare
un sito.

L'unica parte dinamica è l'elenco dei profili configurati, letto dal config
locale. Questo è il punto: la guida da sola permette a un agente di mappare «l'ERP
di staging» su un concreto `-s <profilo>`. Ogni profilo è elencato con il suo
URL, la descrizione, e i tag `default` e `writable`.

La guida copre, in una schermata: accesso ai siti, orientamento
(`doctype list/show`), i verbi sui documenti, filtri, report e SQL, file,
discovery dei metodi e la via di fuga `api`. Dice esplicitamente agli agenti di
usare sempre `--oauth` per il login e di non mettere mai credenziali in
argomenti, file o log.

---

## 18. `assistant` — lanciare un agente come assistente Frappe

```sh
frappectl assistant                       # auto-sceglie il primo tool installato
frappectl assistant claude
frappectl assistant pi -- "elenca le fatture scadute su staging"
frappectl -s staging assistant claude     # sessione limitata a un profilo
frappectl assistant codex --dry-run
```

| Argomento / flag | Descrizione |
|---|---|
| `[tool]` | `pi`, `claude`, `codex`, `flow` o `opencode`. Default: il primo installato, in quest'ordine. |
| `--dry-run` | Stampa cosa verrebbe eseguito (env, file, comando) ed esce, senza lanciare nulla né scrivere nulla. |
| `-- <args>` | Tutto ciò che segue `--` viene passato intatto al tool figlio. |

### Come funziona

È un launcher sottile: avvia un tool agente supportato con un system prompt
orientato a Frappe, costruito dall'output di `frappectl guide` — così l'agente
sa che deve pilotare `frappectl` e quali siti autenticati può raggiungere.

**Non tocca mai la directory di lavoro e non vi scrive nulla.** Ogni tool
riceve invece una directory di configurazione privata sotto la cartella di
config di frappectl (`~/.config/frappe/assistant/<tool>`).

| Tool | Meccanismo |
|---|---|
| `pi` | `--append-system-prompt`; usa la sua vera config dir (auth incluso) intatta. Avviato con `--approve --offline --no-skills --no-context-files`. |
| `claude` | `--append-system-prompt`; permessi lasciati al default interattivo. |
| `codex` | Non ha `--append-system-prompt`, ma legge un `AGENTS.md` globale da `$CODEX_HOME`: il CLI punta `CODEX_HOME` alla cartella privata, vi scrive `AGENTS.md` e **symlinka** `auth.json` e `config.toml` alla vera home di codex, così il login e la configurazione restano validi. |
| `flow` | `flow web --append-system-prompt …`: avvia il daemon, serve la UI web e apre il browser. |
| `opencode` | Risolve `--agent` per nome scandendo `{agent,agents}/**/*.md`: il CLI scrive `~/.config/opencode/agents/frappe.md` e lancia `opencode --agent frappe`. |

### Scoping a un singolo profilo

Con il flag globale `-s <profilo>`, la sessione viene limitata a quel sito: il
prompt elenca **solo** quel profilo (così l'agente non viene a sapere
dell'esistenza degli altri) e include un blocco `SESSION SCOPE` che gli dice di
passare `-s <profilo>` a ogni comando e di non usare altri profili né fare
login su altri siti. Un profilo inesistente dà exit code `2`.

### Controlli pre-lancio

- Il binario del tool deve essere su `PATH`, altrimenti exit code `127`.
- **`frappectl` stesso** deve essere su `PATH`: l'agente ci shella sopra per
  raggiungere i siti. Se manca, exit code `127`.
- Se nessun tool supportato è installato, exit code `127` con la lista di quelli
  supportati.

Dopo i controlli, il processo viene sostituito con `execvpe`, così la TUI del
tool possiede il terminale.

---

## 19. `update` — aggiornamento in place

```sh
frappectl update
frappectl update --json
```

Esegue `uv tool upgrade frappectl`, perché `uv tool install frappectl` è il
metodo di installazione supportato.

Rifiuta (senza cambiare nulla):

- se l'installazione è **editable/di sviluppo** (`pip install -e .`): aggiorna
  quelle con git, exit code `1`;
- se `uv` non è disponibile: exit code `1` con il link a uv.

Se `uv tool upgrade` falla, l'exit code del CLI è quello di uv.

### Controllo passivo delle versioni

Indipendentemente dal comando, il CLI può segnalare la disponibilità di una
versione nuova. Dettagli:

- sorgente: l'API JSON pubblica di PyPI (`info.version`, cioè esattamente ciò
  che installerebbe un upgrade per nome);
- cache: `~/.config/frappe/update-check.json`, TTL **24 ore**; anche un
  controllo *fallito* registra il tentativo, così una macchina offline non
  interroga PyPI a ogni comando per il resto della giornata — continua a servire
  l'ultima risposta buona;
- timeout 2 secondi, qualsiasi errore è silenzioso;
- il confronto di versione estrae `X.Y.Z` tollerando un `v` iniziale e
  ignorando suffissi di pre-release o build. L'ordinamento delle pre-release non
  è modellato di proposito: questo guida un suggerimento morbido, non un gate.

---

## 20. Errori, exit code e suggerimenti

### Exit code

| Codice | Significato |
|---|---|
| `0` | successo |
| `1` | fallimento (errore del server, errore di rete, aggiornamento fallito) |
| `2` | errore d'uso (argomenti non validi, config non valida, nessun sito configurato, ambiguità non interattiva) |
| `127` | solo `assistant`: binario non trovato su PATH |
| *altro* | solo `update`: l'exit code di `uv` propagato |

Il dettaglio vive nel messaggio, non nel codice.

### Forma dei messaggi di errore

```
error: <messaggio in testo semplice>
tip: <suggerimento, se il messaggio corrisponde a una forma nota>
tip: re-run with --debug to see the full server traceback.
```

Tutto su **stderr**. L'HTML viene rimosso dai messaggi del server e le entità
`&amp;`, `&lt;`, `&gt;`, `&#39;`, `&quot;` vengono decodificate.

### Estrazione del messaggio dal server

L'API v2 restituisce gli errori come `{"errors": [{"type", "message",
"exception", …}]}`. Gli endpoint più vecchi usano `_server_messages`, una lista
JSON di dict JSON il cui `message` contiene spesso HTML. Il CLI riduce entrambi
a testo pulito, provando in ordine:

1. `errors[].message` di ciascun errore (deduplicati, uno per riga);
2. se un errore ha solo `exception`, l'**ultima riga significativa** del
   traceback, togliendo il prefisso della classe di eccezione
   (`frappe.exceptions.MandatoryError: [ToDo, …]: description` → la parte
   destra);
3. `_server_messages`;
4. le chiavi `exception`, `message`, `exc_type`, `_error_message`;
5. fallback: `HTTP <status>`.

### Suggerimenti automatici

Se il messaggio d'errore (minuscolizzato) contiene uno dei pattern noti, viene
aggiunta una riga `tip:` che punta al comando che risolve quella *classe* di
errore. L'elenco, in ordine di priorità:

| Pattern nel messaggio | Suggerimento |
|---|---|
| `authentication failed` | imposta le `FRAPPE_*` o esegui `auth login`; controlla il profilo con `auth whoami` |
| `permission` | conferma l'identità con `auth whoami` |
| `not permitted` | conferma l'identità con `auth whoami` |
| `could not reach` | controlla l'URL e la raggiungibilità; `auth whoami` mostra il sito risolto |
| `does not exist` | elenca i record con `doc list`, o verifica il DocType con `doctype list` |
| `not found` | come sopra |
| `mandatory` | guarda i campi obbligatori con `doctype show` |
| `value missing` | come sopra |
| `unknown column` | elenca i fieldname validi con `doctype show` |
| `invalid field` | come sopra |

Il sistema è conservativo per design: solo le forme d'errore ben conosciute
ricevono un suggerimento, così gli errori d'uso e i messaggi non riconosciuti
restano silenziosi. I suggerimenti sono generici: puntano al comando che
risolve la classe di errore, senza indovinare il DocType o il campo esatto.

### Casi particolari

- **401**: «Authentication failed (401). Check the API key/secret for this
  site.» Prima di restituire l'errore, il transport prova **una volta** a
  rinfrescare la credenziale e a rigiocare la richiesta. Se il refresh riesce,
  non vedi alcun errore.
- **403**: il messaggio del server, o «Permission denied (403).» se non c'è.
- **Errore di rete**: «Could not reach `<site>`: `<dettaglio>`».

---

## 21. Sicurezza: cosa viene rifiutato e perché

Un elenco consolidato dei controlli attivi, con il razionale.

| Controllo | Comportamento | Razionale |
|---|---|---|
| **HTTP in chiaro** | Rifiuta di parlare con un host remoto via `http://`. Permesso solo per `localhost`, `*.localhost`, `127.0.0.1`, `::1`, `0.0.0.0`. | La credenziale viaggerebbe in chiaro. |
| **OAuth su HTTP** | Stesso rifiuto, con messaggio dedicato. | Access e refresh token sono credenziali bearer: chi li vede può agire come l'utente. |
| **Credenziali nei flag** | `auth login` non accetta chiave/segreto come flag né via pipe. | Finirebbero in history, lista processi, log CI. |
| **`auth login` non interattivo** | Rifiuta se stdin non è un TTY. | Vedi sopra; e OAuth richiede comunque un browser locale. |
| **Segreti su disco** | Nessun fallback in chiaro. Keyring o variabili d'ambiente. | Un segreto su disco è un segreto perso. |
| **Permessi del config** | `config.json` è `chmod 0600`. | Contiene metadati, non segreti, ma resta privato. |
| **Redazione nel debug** | L'header `Authorization` mostra solo lo schema. | Non si perde la credenziale nemmeno sul terminale locale o in uno screenshot. |
| **Verbi su profilo read-only** | Solo `GET`/`HEAD`/`OPTIONS`, bloccati **prima** della rete. | Un comando esplorativo non può mutare un sito di produzione. |
| **Redirect cross-origin** | `httpx2` rimuove l'header `Authorization` sui redirect verso un'altra origine. | La credenziale non viene mai consegnata a un host che non hai configurato. |
| **Path traversal nel download** | `file_name` dal server passa per `basename`. | Un nome come `../../x` non può uscire dalla cwd. |
| **Symlink nel download** | Scrittura su temporaneo esclusivo + rename atomico, invece di `<output>.part`. | Un `.part` prevedibile potrebbe essere un symlink verso un altro file. |
| **Escape ANSI dal server** | Caratteri di controllo rimossi dall'output del server. | Un server ostile non può iniettare sequenze escape nel tuo terminale. |
| **Encoding dei segmenti di path** | Ogni segmento di URL è URL-encoded (`safe=""`). | Un nome di documento con `/` o caratteri speciali non può alterare il percorso. |
| **Verifica prima della persistenza** | Le credenziali vengono provate contro il sito prima di essere salvate. | Mai persistere credenziali morte. |
| **Revoca al logout** | Il token OAuth viene revocato sul server. | Il logout chiude la sessione, non si limita a dimenticarla. |
| **PKCE S256 + client pubblico** | Nessun client secret emesso né memorizzato. | Un CLI non può custodire un secret. |
| **Timeout** | 60 secondi di default su ogni richiesta HTTP. | Nessun comando resta appeso indefinitamente. |

---

## 22. File, percorsi e variabili d'ambiente

### Percorsi

| Percorso | Contenuto |
|---|---|
| `$XDG_CONFIG_HOME/frappe/config.json` (default `~/.config/frappe/config.json`) | profili: URL, nome, descrizione, flag `read_only`, tag `auth`. Permessi `0600`. |
| `~/.config/frappe/update-check.json` | cache del controllo versione (`checked_at`, `latest`). |
| `~/.config/frappe/assistant/<tool>/` | config privata per tool agente (`AGENTS.md`, symlink). |
| `~/.config/opencode/agents/frappe.md` | definizione dell'agente per opencode (scritta da `assistant opencode`). |
| keyring OS, servizio `frappectl` | `api_key:api_secret`, oppure il blob JSON dei token OAuth. |
| keyring OS, servizio `frappe-cli` | nome legacy; letto come fallback e migrato in avanti. |

### Forma del file di config

```json
{
  "default": "erp",
  "profiles": {
    "erp": {
      "site": "https://erp.example.com",
      "auth": "oauth",
      "description": "ERP di produzione",
      "read_only": true
    },
    "dev": {
      "site": "https://dev.example.localhost"
    }
  }
}
```

Le chiavi `auth`, `description` e `read_only` sono omesse quando valgono il
default (`api_key`, vuota, `false`).

### Forma del segreto nel keyring

| Tipo di profilo | Valore memorizzato |
|---|---|
| API key | `"<api_key>:<api_secret>"` |
| OAuth | `{"access_token", "refresh_token", "expires_at", "token_type": "bearer", "client_id"}` come JSON |

Il `client_id` è persistito proprio perché i login successivi riusino lo stesso
client registrato.

### Variabili d'ambiente

| Variabile | Effetto |
|---|---|
| `FRAPPE_SITE` | URL del sito; attiva il percorso environment (vince su tutto se `-s` non è passato). |
| `FRAPPE_API_KEY` | chiave API; obbligatoria se `FRAPPE_SITE` è impostata. |
| `FRAPPE_API_SECRET` | segreto API; obbligatorio se `FRAPPE_SITE` è impostata. |
| `FRAPPE_READ_ONLY` | se vera (`1`/`true`/`yes`/`on`), la sessione rifiuta i verbi non sicuri. |
| `XDG_CONFIG_HOME` | base della directory di config. |
| `CODEX_HOME` | letta da `assistant codex` per trovare la vera home di codex da symlinkare. |

---

## 23. Endpoint HTTP usati

Utile quando devi replicare un comando con `api`, o leggere un trace `--debug`.

| Comando | Metodo e percorso |
|---|---|
| `doc list` | `GET /api/v2/document/<DocType>` con `start`, `limit`, `fields`, `filters`, `order_by` |
| `doc get` | `GET /api/v2/document/<DocType>/<name>/` |
| `doc create` | `POST /api/v2/document/<DocType>` |
| `doc update` | `PATCH /api/v2/document/<DocType>/<name>/` |
| `doc delete` | `DELETE /api/v2/document/<DocType>/<name>/` |
| `doc submit`/`cancel` | `POST /api/v2/document/<DocType>/<name>/method/<submit\|cancel>/` |
| `doc amend` | `GET` dell'originale, poi `POST /api/v2/document/<DocType>` |
| `doctype show` | `GET /api/v2/doctype/<DocType>/meta` |
| `doctype list` | `GET /api/v2/method/frappe.desk.search.search_link` |
| (conteggio) | `GET /api/v2/doctype/<DocType>/count` |
| `report run` | `GET /api/v2/method/frappe.desk.query_report.run` |
| `query` | `POST /api/v2/method/frappe.desk.doctype.system_console.system_console.execute_code` |
| `method call` (rpc) | `POST`/`GET` `/api/v2/method/<path>` |
| `method call` (doctype) | `POST`/`GET` `/api/v2/document/<DocType>/<name>/method/<method>` |
| `method search` | `GET /api/v2/discovery/search?q=…` |
| `method list` | `GET /api/v2/discovery/method` |
| `method list --doctype` | `GET /api/v2/discovery/doctype/<DocType>` |
| `method show` | `GET /api/v2/discovery/method/<path>` |
| `method show --doctype` | `GET /api/v2/discovery/doctype/<DocType>/method/<method>` |
| `file upload` | `POST /api/v2/method/upload_file` (multipart) |
| `file download` | `GET <file_url>` in streaming |
| `auth whoami`, verifica login | `GET /api/v2/method/frappe.auth.get_logged_user` |
| OAuth discovery | `GET /.well-known/oauth-authorization-server` |
| OAuth fallback | `/api/method/frappe.integrations.oauth2.{authorize,get_token,revoke_token,openid_profile,register_client}` |

Il transport restituisce il payload **scartato dall'involucro**: se il corpo
della risposta è un dict che contiene `data`, viene restituito `body["data"]`.

---

## 24. Ricettario

### Esplorare un sito nuovo senza rischi

```sh
frappectl auth login https://nuovo.example.com --oauth --name nuovo   # read-only per default
frappectl auth whoami -s nuovo
frappectl -s nuovo doctype list --custom
frappectl -s nuovo doctype show "Custom DocType"
```

### Contare i documenti che corrispondono a un filtro

```sh
frappectl api method/frappe.client.get_count -F doctype=ToDo -F 'filters:=[["status","=","Open"]]'
```

### Estrarre tutte le fatture scadute in CSV

```sh
frappectl doc list "Sales Invoice" \
  -f status=Overdue \
  --fields name,customer,grand_total,due_date \
  --all --json \
| jq -r '(.[0] | keys_unsorted) as $k | $k, (.[] | [.[$k[]]]) | @csv' \
> scadute.csv
```

### Aggiornare in massa leggendo una lista

```sh
frappectl doc list ToDo -f status=Open --fields name --all --json \
| jq -r '.[].name' \
| while read -r name; do
    frappectl doc update ToDo "$name" --set status=Closed
  done
```

### Creare un documento con tabelle figlio

```json
{
  "customer": "ACME Srl",
  "posting_date": "2026-11-10",
  "items": [
    { "item_code": "ITEM-001", "qty": 2, "rate": 100 },
    { "item_code": "ITEM-002", "qty": 1, "rate": 250 }
  ]
}
```

```sh
frappectl doc create "Sales Invoice" --input fattura.json
# oppure
cat fattura.json | frappectl doc create "Sales Invoice"
```

### Ciclo completo: crea, invia, annulla, modifica

```sh
NAME=$(frappectl doc create "Sales Invoice" --input fattura.json --json | jq -r .name)
frappectl doc submit "Sales Invoice" "$NAME"
frappectl doc cancel "Sales Invoice" "$NAME"
frappectl doc amend  "Sales Invoice" "$NAME"
```

### Scoprire e invocare un metodo mai visto

```sh
frappectl method search -q unread
frappectl method show gameplan.api.get_unread_count
frappectl method call gameplan.api.get_unread_count -F project=1
```

### Un comando read-only su un profilo bloccato

```sh
frappectl -s prod api method/frappe.client.get_count -F doctype=User -X GET
```

### Due siti nello stesso script

```sh
for sito in prod staging; do
  echo "== $sito"
  frappectl -s "$sito" doc list "Sales Invoice" -f status=Overdue --fields name --json
done
```

### Confrontare lo schema fra due siti

```sh
diff <(frappectl -s prod    doctype show "Sales Invoice" --json) \
     <(frappectl -s staging doctype show "Sales Invoice" --json)
```

### Uso in CI (headless)

```sh
export FRAPPE_SITE=https://erp.example.com
export FRAPPE_API_KEY=$CI_FRAPPE_KEY
export FRAPPE_API_SECRET=$CI_FRAPPE_SECRET
export FRAPPE_READ_ONLY=1          # sicurezza in più per un job di sola lettura

frappectl doc list "Sales Invoice" -f status=Overdue --json > scadute.json
```

### Controllare un exit code in uno script

```sh
if ! out=$(frappectl doc get ToDo nonesiste --json 2>err.txt); then
  echo "fallito con exit code $? :"
  cat err.txt
fi
```

---

## 25. Risoluzione dei problemi

| Sintomo | Causa e rimedio |
|---|---|
| «Multiple profiles are configured. Non-interactive invocations must pick a site explicitly» | Stai facendo pipe o sei in CI con più di un profilo. Passa `-s <profilo>` o usa le `FRAPPE_*`. Vedi [§3.5](#35-risoluzione-non-interattiva-la-regola-dellambiguità). |
| «No site configured» | Nessun profilo e nessuna `FRAPPE_SITE`. Esegui `auth login` o imposta le variabili. |
| «No such profile: X» | Controlla i nomi con `auth list`. |
| «Refusing to send a POST request: this profile is read-only» | Il profilo blocca le scritture. Usa un profilo writable, `auth configure <p> --writable`, o rendi esplicito `-X GET` per un metodo di lettura. |
| «Refusing to talk to … over plain HTTP» | Usa `https://`, oppure un indirizzo localhost per lo sviluppo locale. |
| «Could not read credentials from the OS keyring» | Macchina senza keyring funzionante. Usa `FRAPPE_SITE`/`FRAPPE_API_KEY`/`FRAPPE_API_SECRET`. |
| «Could not store credentials in the OS keyring» | Stesso problema in scrittura. frappectl non scrive segreti su disco: usa le variabili d'ambiente. |
| «No stored credentials for profile 'X'» | Il config conosce il profilo ma il keyring no. Rifai `auth login` per quel sito. |
| «auth login is interactive only and needs a terminal» | Per design. In headless usa le variabili d'ambiente. |
| «Authentication failed (401)» | Chiave/segreto errati o revocati, o sessione OAuth non più rinfrescabile. `auth whoami` per vedere chi sei; rifai il login. |
| «Could not refresh the OAuth session. Log in again.» | Refresh token scaduto o revocato. `auth login … --oauth`. |
| «Permission denied (403)» | L'utente non ha il permesso. `auth whoami` per confermare l'identità; verifica i ruoli e i permessi con `doctype show`. |
| «was modified since you read it» | Conflitto di concorrenza su `doc update`. Rileggi e ritenta, o `--force` per sovrascrivere. |
| «is not cancelled; only cancelled documents can be amended» | `doc amend` richiede `docstatus == 2`. Prima `doc cancel`. |
| «Method discovery is not available on this site» | Il sito non espone `/api/v2/discovery`. Usa `api` direttamente. |
| «Discovery cache is being generated; retrying in N seconds» | Normale a cache fredda; il CLI ritenta fino a 4 volte. |
| Discovery dà sempre 403 | Gli endpoint di discovery richiedono **System Manager**. |
| `query` dà un errore di permesso | System Console richiede **System Manager** o **Administrator**. |
| Una `-f 'campo>valore'` non viene parsata | Quota il filtro con apici singoli: la shell interpreta `>` come redirezione. |
| Il valore `007` è diventato `7` in un campo | Non può accadere con `--set`: gli zeri iniziali sono preservati. Se vedi questo, il valore è stato convertito a monte (per esempio da `jq`). |
| Un intero è diventato stringa | Zeri iniziali, o apici dentro l'argomento. Vedi [§11](#11-coercizione-dei-valori). |
| «Could not bind the loopback port 9876» | Qualcosa occupa la porta. Chiudilo, o `--oauth-port <porta>` — ma la porta deve corrispondere a un redirect URI registrato sul sito. |
| «does not advertise dynamic client registration and no client id was configured» | Passa `--client-id` di un client pubblico già registrato con redirect URI `http://127.0.0.1:9876/callback`. |
| «Timed out waiting for the authorization redirect» | 300 secondi scaduti. Completa il login nel browser e ritenta. |
| «This is an editable/development install» su `update` | Aggiorna con `git pull`. |
| «`frappectl update` needs uv» | Installa uv, o aggiorna con il tuo package manager. |
| JSON rotto in pipe | Non dovrebbe succedere: il JSON è riparsato prima dell'emissione, e `--debug` scrive solo su stderr. Se accade, è un bug. |
| L'help è «spoglio» | Per design: `click` semplice, senza box né colori, per risparmiare larghezza e token. |

---

## 26. Riferimento rapido

### Tutti i comandi

```
frappectl [-s PROFILO] [--json] [--debug] [--version] [-h] <comando>

auth login <url> [--name N] [--description D] [--default|--no-default]
                 [--read-only|--writable] [--oauth] [--client-id ID]
                 [--oauth-port P]
auth list
auth whoami
auth default <profilo>
auth configure <profilo> [--name N] [--description D] [--read-only|--writable]
auth logout <profilo>

doctype list [--module M] [--custom] [--limit N]
doctype show <DocType> [--raw]

doc list   <DocType> [-f FILTRO]… [--filters-json J] [--fields F]
                     [--order-by O] [--limit N] [--all]
doc get    <DocType> <name>
doc create <DocType> [--set k=v]… [--input FILE]
doc update <DocType> <name> [--set k=v]… [--input FILE] [--force]
doc delete <DocType> <name>
doc submit <DocType> <name>
doc cancel <DocType> <name>
doc amend  <DocType> <name>

report run <nome> [-f campo=valore]… [--filters-json J]

query <SQL>|- [--input|-i FILE]

method search --query|-q TERMINI
method list [--doctype D]
method show <metodo> [--doctype D]
method call <metodo> [--doctype D --name N] [-F k=v]… [-f k=v]… [-X VERBO]

api <path> [-F k=v]… [-f k=v]… [-X VERBO] [--input FILE|-]

file upload   <path> [--doctype D] [--name N] [--fieldname F] [--private]
file download <ref>  [--output|-o PATH|-]

guide
assistant [pi|claude|codex|flow|opencode] [--dry-run] [-- <args>]
update
```

### Default che conviene ricordare

| Cosa | Default |
|---|---|
| Nuovo profilo | **read-only** |
| Metodo di login interattivo | **OAuth** |
| Primo profilo creato | diventa il default |
| `doc list --limit` | `20` |
| `doc list --order-by` | `creation desc` |
| `doc list --fields` | guidato dai meta (`name` + title + `in_list_view`) |
| `doc list --all` dimensione pagina | `200` |
| `doctype list --limit` | `10000` |
| `doc update` | concorrenza ottimistica attiva |
| `method call` verbo | `POST` (`GET` se read-only) |
| `api` verbo | `GET` (`POST` se c'è `--input`) |
| Timeout HTTP | 60 s |
| Porta loopback OAuth | `9876` |
| Timeout login OAuth | 300 s |
| Margine di refresh OAuth | 60 s |
| Retry discovery | 4, backoff 2 s, tetto 30 s |
| TTL controllo aggiornamenti | 24 h |
| Larghezza riga JSON | 160 colonne |
| Troncamento body nel debug | 2000 caratteri |

### Le cinque regole che evitano la maggior parte dei problemi

1. **Orientati prima di indovinare**: `doctype show` prima di scrivere.
2. **Quota i filtri** con apici singoli: `-f 'grand_total>1000'`.
3. **In script e CI passa `-s`** o le `FRAPPE_*`: il default non è affidabile in
   modalità non interattiva.
4. **Lascia i profili di produzione read-only.** È la sola protezione che il CLI
   ti dà, e funziona prima della rete.
5. **stdout è dato, stderr è chiacchiera.** Fai pipe senza paura; usa `--debug`
   senza paura.

---

## Licenza

MIT. Vedi `LICENSE`.
