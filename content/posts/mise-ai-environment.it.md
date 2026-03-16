---
title: "mise en place per l'era dell'AI"
date: 2026-03-16
draft: false
ai: true
tags: ["AI", "mise", "Bash", "Workflow", "Automation", "Tools"]
description: "L'AI è ottima a scrivere script bash. mise è ottimo a eseguirli in modo coerente. Insieme risolvono uno dei problemi meno discussi nello sviluppo assistito da AI: quello che succede tra 'capisco come farlo' e 'funziona sempre allo stesso modo'."
image: "images/posts/mise-ai-environment.png"
---

Prima che una cucina da ristorante apra per la sera, lo chef prepara. Gli ingredienti vengono tagliati, le salse avviate, gli strumenti disposti a portata di mano. I francesi chiamano questo *mise en place*: ogni cosa al suo posto. Il servizio va bene non perché lo chef sia più veloce sul momento, ma perché il pensiero è avvenuto in anticipo.

Esiste uno strumento per sviluppatori chiamato [mise](https://mise.jdx.dev/). È un gestore di versioni e un task runner, e per caso prende il nome dallo stesso concetto. Non penso sia un caso: è uno strumento che esiste per avere tutto pronto prima che il lavoro inizi. Con l'integrazione degli agent AI nel mio workflow quotidiano, la metafora è diventata sempre più letterale.

## La tassa nascosta

L'AI agentiva ha un costo che non emerge nelle demo: il consumo di token. Ogni round-trip in un ciclo agentivo costa token in diverse categorie. I token in input coprono il system prompt, le definizioni degli strumenti, l'intera cronologia della conversazione e il nuovo messaggio. I token in output coprono tutto ciò che il modello genera: ragionamento, prosa e invocazioni degli strumenti.

L'asimmetria di prezzo è importante. Sui modelli Anthropic attuali, [i token in output costano circa tre-cinque volte di più rispetto a quelli in input](https://www.anthropic.com/pricing). Non è un dettaglio: è il motivo per cui il modo in cui strutturi i workflow agentivi ha una diretta conseguenza economica.

Quando un agent deve costruire un comando shell complesso da zero, genera quel comando come output. Una chiamata `curl` realistica con autenticazione bearer, header content-type e un filtro `jq` per estrarre i campi che ti interessano può arrivare a cento o più token di output. Il comando equivalente nominato, `mise run jira:get PROJ-1234`, è otto token. Se quell'operazione avviene dieci volte in una sessione, la differenza è circa 920 token di output. Su una sessione lunga con molte chiamate agli strumenti, questo si accumula in modo misurabile.

Vale la pena notare: la [documentazione sul conteggio dei token di Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/token-counting) mostra che anche una singola semplice definizione di strumento aggiunge circa 390 token a un messaggio che altrimenti ne avrebbe 14. Il system prompt, gli schema degli strumenti e la cronologia della conversazione sono tutti token in input, e si accumulano.

## Il ciclo di retry

C'è un secondo costo, meno ovvio ma probabilmente più rilevante in pratica. I comandi costruiti dinamicamente falliscono. Il flag sbagliato, una virgoletta mancante, un formato di autenticazione errato: il comando viene eseguito, restituisce un errore, l'agent legge l'errore, ragiona sulla correzione e genera un nuovo tentativo. Ogni ciclo è un round-trip completo: più token in output, più token in input (il risultato dell'errore entra nel contesto), più costo, più latenza.

Uno script predefinito e testato non indovina. Esegue lo stesso comando con la stessa sintassi ogni volta. I punti di fallimento si spostano da "l'AI costruisce il comando in modo errato" a "il servizio esterno è effettivamente non disponibile", un elenco molto più breve.

Il team di ingegneria di Anthropic nel loro [articolo su SWE-bench](https://www.anthropic.com/engineering/swe-bench-sonnet) descrive run agentive che consumano oltre 100.000 token per task, in gran parte nei cicli di chiamata agli strumenti. Ridurre l'overhead per chiamata si moltiplica su quei cicli. Il loro post su [come costruire agent efficaci](https://www.anthropic.com/research/building-effective-agents) nota che il design del formato degli strumenti influisce direttamente sull'efficienza del modello, e che "abbiamo passato più tempo a ottimizzare i nostri strumenti che il prompt complessivo" in quel lavoro.

## L'alternativa ovvia, e perché non basta

La risposta ovvia a tutto questo è: script shell. Scrivi il comando curl una volta, mettilo in una cartella `scripts/`, fatto. E funziona, fino a un certo punto. Ma porta con sé una serie di problemi che si accumulano con la crescita del progetto.

Le variabili d'ambiente devono essere impostate da qualche parte. In pratica significa un file `.env` che va sourcato manualmente, o voci in `~/.zshrc` che esistono solo sul tuo laptop, o una sezione del `README` che dice "assicurati di avere questi impostati" e che è sempre leggermente obsoleta. I segreti finiscono nella cronologia di git più spesso di quanto chiunque ammetta. L'AI, da parte sua, non ha idea di quali variabili siano disponibili e quali no, a meno che tu non glielo dica ogni volta.

Le versioni degli strumenti divergono. Il `jq` che viene distribuito con macOS è più vecchio di quello su Linux CI. La versione di Node che funziona sul tuo laptop non è necessariamente quella che il tuo collega ha installato. Gli script che girano in locale falliscono silenziosamente altrove, e il fallimento sembra un bug nello script piuttosto che una mancata corrispondenza di versione.

E la discoverabilità scompare del tutto. Una cartella di script shell non ha un indice, nessuna descrizione, nessuna dichiarazione di dipendenze. L'AI può fare `ls scripts/` e indovinare, o può chiederti, o può leggere ogni script fino a trovare quello di cui ha bisogno. Nessuna di queste è efficiente.

## Cos'è mise

[mise](https://mise.jdx.dev/) (pronunciato "meez", da *mise en place*) è un gestore di versioni poliglotta e un task runner. Rimpiazza l'ecosistema frammentato di `nvm`, `rbenv`, `pyenv`, `asdf` e strumenti simili, più Make, con un'unica interfaccia unificata. L'installazione è un one-liner:

```bash
curl https://mise.run | sh
```

O via Homebrew su macOS:

```bash
brew install mise
```

Una volta installato, un `mise.toml` alla radice di un progetto gestisce tre problemi in un unico file:

```toml
[tools]
node = "22"

[env]
JIRA_BASE_URL = "https://jira.tuaazienda.com"
JIRA_PAT = "{{ env.JIRA_PAT }}"

[tasks.jira-get]
description = "Recupera un issue Jira tramite chiave"
run = """
curl -s \
  -H "Authorization: Bearer $JIRA_PAT" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/2/issue/$1" \
  | jq '{key:.key, summary:.fields.summary, status:.fields.status.name}'
"""
```

`[tools]` fissa ogni runtime e strumento CLI a una versione specifica, installata in modo coerente su ogni macchina tramite `mise install`. `[env]` rende i segreti disponibili come variabili d'ambiente, letti dall'ambiente shell dove vivono in sicurezza: un keychain, un secrets manager o un file locale gitignored. Non appaiono mai in un prompt. `[tasks]` dà a ogni workflow un nome e una descrizione.

I task inline nel TOML funzionano per i casi semplici. Per qualcosa di più complesso, mise scopre i task da una cartella dedicata: `.mise/tasks/`, `mise/tasks/` e altri percorsi convenzionali. Ogni file è uno script eseguibile con un header strutturato:

```bash
#!/usr/bin/env bash
# [MISE]
# description = "Recupera un issue Jira tramite chiave"
# alias = "jg"
# depends = ["check-env"]

set -euo pipefail

ISSUE_KEY="${1:?Uso: mise run jira:get ISSUE-KEY}"

curl -s \
  -H "Authorization: Bearer $JIRA_PAT" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/2/issue/$ISSUE_KEY" \
  | jq '{
      key: .key,
      summary: .fields.summary,
      status: .fields.status.name,
      description: .fields.description
    }'
```

Questo file vive in `.mise/tasks/jira/get` e si invoca come `mise run jira:get PROJ-1234`. La sottocartella diventa un namespace, usando `:` come separatore. `mise tasks` elenca tutto con descrizioni. `depends` dichiara prerequisiti che vengono eseguiti automaticamente prima.

La sintassi `# [MISE]` (con spazio dopo `#`) è sicura per i formatter: la maggior parte dei linter e autoformatter che aggiungono silenziosamente spazi ai marcatori di commento non la corromperà.

I modelli AI scrivono bash fluentemente e senza difficoltà. Lo stesso modello che scrive il codice della tua applicazione scrive i tuoi script di tooling, e il risultato è leggibile e corretto. I Makefile, al confronto, hanno una sintassi piena di insidie: spazi significativi rispetto alle tab, variabili automatiche come `$@` e `$<`, dichiarazioni `.PHONY`. I task mise sono solo script. La barriera è più bassa, e l'AI la supera senza sforzo.

## Un esempio concreto: Jira on-premise

Il [server MCP ufficiale di Atlassian](https://support.atlassian.com/atlassian-rovo-mcp-server/docs/getting-started-with-the-atlassian-remote-mcp-server/) (Rovo) collega gli strumenti AI a Jira, ma supporta solo Jira Cloud. I team che gestiscono Jira Server o Data Center on-premise sono attualmente esclusi: esiste una feature request aperta ([JRASERVER-78874](https://jira.atlassian.com/browse/JRASERVER-78874)) senza impegno pubblico di roadmap.

La REST API di Jira, tuttavia, funziona in modo identico su ogni versione. I Personal Access Token per l'autenticazione sono supportati dal Server 8.14. Il task mostrato sopra è l'intera integrazione: una singola chiamata curl, un PAT nell'ambiente, un filtro jq. Nessun MCP, nessun plugin, nessuna dipendenza aggiuntiva.

Questo pattern si generalizza. Ogni REST API è, nella sostanza, un comando curl. Non esiste integrazione che richieda strettamente uno strumento dedicato: c'è solo la questione se l'API esiste, e quasi sempre esiste.

## Collegare l'AI alla cucina

L'ultimo tassello è dire all'AI come usare ciò che hai preparato. In Claude Code, le skill sono brevi file markdown che danno all'agent istruzioni riutilizzabili. Ecco una skill che collega Claude al task Jira:

```markdown
---
name: jira-context
description: Recupera un issue Jira da usare come contesto prima di iniziare il lavoro
---

Quando lavori su qualcosa che fa riferimento a un issue Jira, recuperalo prima di scrivere codice:

    mise run jira:get {{ISSUE_KEY}}

Usa l'output per capire scope e criteri di accettazione.
Non chiedere all'utente di incollare manualmente il contenuto Jira.
```

L'AI legge queste istruzioni, sa esattamente quale comando eseguire e lo chiama nello stesso modo ogni volta: uno script già testato, nessuna ricostruzione, nessun retry. Il PAT non entra mai nella conversazione. L'URL non viene mai ricostruito dalla memoria.

Confronta questo con quello che la skill dovrebbe contenere senza mise. Per dare all'agent istruzioni equivalenti, dovrebbe codificare la struttura dell'endpoint, l'header di autenticazione, i nomi delle variabili d'ambiente e i campi jq da estrarre:

```markdown
---
name: jira-context
description: Recupera un issue Jira da usare come contesto prima di iniziare il lavoro
---

Recupera gli issue Jira tramite la REST API:

    curl -s \
      -H "Authorization: Bearer $JIRA_PAT" \
      -H "Accept: application/json" \
      "$JIRA_BASE_URL/rest/api/2/issue/ISSUE-KEY" \
      | jq '{key:.key, summary:.fields.summary, status:.fields.status.name, description:.fields.description}'

JIRA_BASE_URL è https://jira.tuaazienda.com. JIRA_PAT è un Personal Access Token dal tuo profilo.
Se ricevi un 401, il token potrebbe essere scaduto. Usa key, summary, status e description dalla risposta.
```

Quella skill entra nel contesto a ogni invocazione. Se l'API cambia, o il filtro jq ha bisogno di un campo in più, aggiorni la skill e paghi di nuovo quei token a ogni caricamento. La versione mise ha una riga significativa. Tutto il resto è nello script, fuori dal contesto.

Questa è la divisione che la metafora descrive: lo chef non prepara la cucina e cucina allo stesso tempo. La mise en place avviene prima del servizio. L'AI si concentra sul giudizio e sulla generazione di codice; mise fornisce un execution layer testato e coerente per tutto ciò che è esterno.

## Cosa questo non risolve

Per essere onesti sul trade-off: i risparmi descritti qui riguardano principalmente i **token in output** e i cicli di retry falliti. Sono reali, ma non sono l'intera storia.

I token del risultato degli strumenti non sono interessati. Se `mise run jira:get PROJ-1234` restituisce un payload JSON da 2.000 token, quei 2.000 token entrano nel contesto come input al prossimo turno, esattamente come se l'AI avesse eseguito il curl completo direttamente. Ciò che hai invocato non ha alcun effetto su ciò che torna.

Se esponi i task mise come schema di strumenti formali con descrizioni dei parametri, quegli schema aggiungono token in input ad ogni chiamata API nella sessione, anche se il [prompt caching di Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) (che sconta il contesto ripetuto a circa il 10% del costo normale) riduce significativamente l'overhead ammortizzato per le sessioni lunghe.

Il risparmio netto è più significativo per comandi complessi, chiamati frequentemente e soggetti a errori di costruzione quando generati dinamicamente. Per semplici one-liner, l'overhead di avere la definizione dello strumento potrebbe superare il beneficio.

## Senza l'AI

Tutto ciò che è descritto qui funziona senza alcuna AI nel loop. `mise run jira:get PROJ-1234` funziona in un terminale, in CI, in uno script shell chiamato da un altro task. I comandi che definisci per un agent sono gli stessi che usi tu stesso, perché sono semplicemente comandi.

Ed è questo che vale la pena costruire indipendentemente dagli argomenti sull'efficienza dell'AI. Una configurazione mise che denomina chiaramente ogni dipendenza esterna, fissa ogni versione degli strumenti e esprime ogni workflow come un comando documentato è qualcosa di utile da avere indipendentemente da ciò che qualsiasi modello AI possa farci. Il risparmio di token è un effetto collaterale dell'essere ben organizzati.

La cucina è già allestita. L'AI deve solo mangiare senza fare disordine.

## La ricetta

L'esempio Jira in questo post è volutamente semplice: una chiamata curl, un filtro jq. Le integrazioni reali sono più complesse: paginazione, gestione degli errori, endpoint multipli, logica condizionale. Il punto era mostrare il pattern senza seppellirlo nel bash. Per qualsiasi cosa non banale, il workflow si scala allo stesso modo.

1. **Chiedi all'AI di scrivere lo script.** Descrivi l'obiettivo, non l'implementazione. "Scrivi uno script bash che recupera tutti gli issue Jira aperti assegnati a me nello sprint corrente e li formatta come checklist markdown." Il modello scrive bash senza difficoltà. Lascia che figuri i flag curl, il percorso jq, i casi d'errore. Eseguilo, verifica che funzioni, aggiusta se necessario.

2. **Consolida via mise.** Metti il risultato in `.mise/tasks/`. Aggiungi `set -euo pipefail`. Collega i segreti tramite `[env]` in `mise.toml` in modo che non appaiano mai nei prompt. Esegui `mise run` per confermare che funzioni in una shell pulita. Questo è il momento in cui il lavoro viene catturato: un comando testato e nominato che vive fuori da qualsiasi contesto agentivo.

3. **Scrivi una skill che lo chiama.** Poche righe di markdown. Quando l'agent dovrebbe invocarlo e quale comando eseguire. L'implementazione rimane nello script. La skill si limita a puntargli.

L'AI scrive la complessità una volta. mise la rende ripetibile. La skill la rende disponibile senza trascinarla nel contesto. Questa divisione del lavoro è l'intero punto.

Il tavolo è apparecchiato. Buona cena.
