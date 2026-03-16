---
title: "Il blog che quasi non è mai esistito"
date: 2026-03-01
draft: false
ai: true
tags: ["Hugo", "Blogging", "AI", "Workflow"]
description: "Avevo Payload CMS, VS Code e cose da dire. Non pubblicavo comunque. Questo articolo parla di cosa è cambiato, e del perché ci ha pensato un'AI."
image: "images/posts/from-payload-to-hugo.png"
---

Per circa un anno il mio sito personale ha girato su [Payload CMS](https://payloadcms.com/), un CMS headless potente e orientato agli sviluppatori, costruito su Node.js. Funzionava bene, ma col tempo ho iniziato a sentire attrito nel mio workflow. Così sono passato a [Hugo](https://gohugo.io/), un generatore di siti statici dove ogni post è semplicemente un file Markdown in una cartella.

Questo articolo parla del perché ho fatto quel cambio, e di come ha sbloccato qualcosa che non mi aspettavo: un ciclo di feedback molto più stretto con strumenti AI come [Claude](https://claude.ai).

## Cosa non andava con Payload?

Niente, in realtà. Payload è un ottimo CMS. Ti dà un pannello admin completo, modelli di contenuto flessibili, e tutto vive nel codice. Per anni sono stato affascinato dai sistemi CMS, e nell'era pre-AI era naturale esplorare qualcosa di popolare, open-source e tecnicamente interessante: Payload spuntava tutte le caselle. Ma per un blog personale era troppo:

- **Overhead di deployment**: un server Node, un database, variabili d'ambiente da gestire. Onestamente Vercel gestisce tutto brillantemente, quindi questo non era il vero problema.
- **Context switching**: scrivere un post significava aprire un browser, navigare al pannello admin e usare un editor rich-text. Anzi, era quasi una *feature*: potevo scrivere da qualsiasi dispositivo. Non una lamentela.

Il vero problema è emerso quando ho cercato di integrare l'AI nel mio processo di scrittura. Aggiungere assistenza AI a Payload non è banale: richiede plugin personalizzati, API call cablate nell'editor e navigare un ecosistema Node.js piuttosto complesso. Ci sono andato a fondo, e anche [Claude](https://claude.ai) non riusciva a trovare un percorso pulito. Non perché Claude non fosse utile (lo era), ma perché il problema in sé è genuinamente difficile, e semplicemente non ho le competenze tecniche avanzate per portarlo a termine. Inoltre, qualsiasi AI incorporata in un CMS è intrinsecamente limitata al contesto: vede un campo alla volta, senza consapevolezza del resto del post, del tono del sito, o di nulla al di là del cursore.

È lì che ho capito che il problema non era lo strumento: stavo chiedendo allo strumento sbagliato di fare qualcosa per cui non era stato progettato. Payload è costruito per contenuti strutturati su larga scala, non per un blogger solitario che vuole pensare ad alta voce con un co-pilota AI. Il disallineamento non era colpa di Payload. **Era mia.**

## Perché Hugo?

Non starò molto sulla parte generica. Hugo non ha database, né server, né pannello admin: il tuo contenuto sono file Markdown, la configurazione è un file TOML, esegui un comando e ottieni un sito statico. È documentato ampiamente e predicato ovunque.

Ciò che mi ha convinto davvero è qualcosa di più ovvio in retrospettiva: passo nove ore al giorno in Xcode o VS Code. Ogni problema che risolvo al lavoro vive in un file da qualche parte. Ogni modifica che faccio è un commit. Il mio intero modello mentale del "fare le cose" gira su file di testo e Git. Nel momento in cui scrivere un post è diventato lo stesso gesto di scrivere un'estensione Swift (apri file, scrivi, salva, committa) ha smesso di sembrare un'attività separata con il suo overhead. Sembrava lavoro, che può sembrare una strana cosa da consigliare, ma per me è proprio il punto.

Il pannello admin non era il problema. Era il context switch. I quindici secondi per aprire un browser, navigare da qualche parte, aspettare che un editor rich-text si carichi: non è molto tempo, ma era abbastanza attrito da farmi scegliere di non farlo. Rimuovere quell'attrito non ha cambiato la mia intenzione di pubblicare. **Ha cambiato se lo facessi davvero.**

## La migrazione: da una cartella vuota a un sito funzionante

La migrazione vera e propria è stata una delle parti più interessanti dell'intero esperimento, perché ne ho fatta pochissima manualmente.

Ho iniziato con una cartella vuota e ho chiesto a Claude di configurare un sito Hugo da zero. Ho scelto [Mana](https://github.com/Livour/hugo-mana-theme), un tema minimalista orientato agli sviluppatori che corrispondeva all'estetica che avevo in mente. Claude ha gestito lo scaffolding, la configurazione e la configurazione del tema.

Poi gli ho chiesto di andare sul mio vecchio sito Payload, recuperare i due post che avevo pubblicato lì e migrarli su Hugo: contenuto, struttura e immagini incluse. Ha preso tutto, convertito in Markdown, posizionato le immagini nelle cartelle giuste e configurato il front matter. Quello che mi avrebbe preso un pomeriggio ha richiesto qualche minuto. Ha anche preservato esattamente gli stessi slug URL del vecchio sito Payload, quindi i link esistenti e il posizionamento nei motori di ricerca non si sono rotti.

La parte più complicata è stata la sezione dei talk. Ho una manciata di conferenze registrate su YouTube e volevo elencarle sul sito. La soluzione ovvia (incorporare iframe YouTube) era fuori discussione. Gli embed YouTube impostano cookie di terze parti, che secondo le normative europee richiederebbero un banner per il consenso ai cookie. Non voglio un banner cookie. Voglio un sito pulito, veloce e rispettoso della privacy.

Invece, ho chiesto a Claude di recuperare ogni URL YouTube che gli fornivo, estrarne il thumbnail e visualizzarlo come immagine statica con link al video. Nessun iframe, nessun tracking, nessun cookie. L'utente clicca il thumbnail e atterra direttamente su YouTube. Scelta sua, contesto suo. Il sito resta cookie-free.

È una piccola decisione, ma riflette qualcosa a cui tengo: il web non deve spiarti solo perché hai guardato un talk a una conferenza.

L'unica analisi su questo sito è [GoatCounter](https://www.goatcounter.com/), uno strumento gratuito e open-source che raccoglie conteggi di visualizzazioni di pagina completamente anonimi. Nessun fingerprinting, nessun tracking cross-sito, nessun dato personale. Posso vedere che qualcuno ha visitato una pagina; non ho idea di chi sia.

## Il vero cambiamento: AI ovunque io scriva

Questa è la parte che non avevo del tutto anticipato, e che ancora fatico a descrivere bene.

Quando ho cercato di incorporare AI in Payload, il massimo che avrei potuto ottenere era un campo di testo più intelligente. L'AI avrebbe visto qualunque cosa si trovasse davanti al cursore. Non avrebbe saputo il titolo, la struttura, gli altri post, il tono che cercavo, né il fatto che avevo già fatto questo stesso punto tre paragrafi prima. **Sarebbe stato autocomplete con un vocabolario più ampio.**

Con Hugo, il post è solo un file in un progetto. Claude può aprirlo nello stesso modo in cui apre qualsiasi altro file. Può leggere tutto, dirmi dove l'argomentazione si inceppa, notare quando mi ripeto, controllare se il tono cambia goffamente a metà. Può confrontare con gli altri post e dirmi se sto coprendo lo stesso terreno. Può guardare la configurazione del sito e capire che questo è un blog personale con un certo tipo di lettore in mente, non una knowledge base aziendale. Il contesto non è più un singolo campo di testo: è tutto, nello stesso modo in cui sarebbe per un collaboratore che avesse davvero letto il tuo lavoro.

Quel passaggio, da "AI dentro uno strumento" a "AI accanto a tutto", si è rivelato essere la cosa che mi mancava davvero. Non un editor più intelligente. Un collaboratore che poteva vedere il quadro completo.

## Il workflow ora

Il mio processo di blogging è diventato quasi indistinguibile dal mio workflow di codice:

1. Dò a Claude l'argomento, qualche appunto grezzo, o solo un'idea vaga
2. Claude crea il file, scrive il front matter e bozza l'intero post: struttura, prosa, tutto
3. Revisiono, aggiusto il tono, correggo ciò che non suona come me
4. Anteprima in locale con `hugo server -D`
5. Commit, push, fatto

Nessun pannello admin su cui fare login, nessun deploy da babysitterare. Solo file, Git e il mio editor.

## Compromessi

Per essere onesti, questa configurazione non è per tutti:

- **Nessun editor visivo.** Se preferisci il WYSIWYG, l'approccio solo-Markdown di Hugo potrebbe sembrare limitante.
- **Nessuna gestione visiva dei media.** Hugo può ridimensionare, ritagliare e ottimizzare le immagini, ma non c'è UI per farlo. Configuri l'elaborazione delle immagini nei template o negli shortcode, il che è potente ma richiede una configurazione iniziale.
- **La personalizzazione del tema richiede i template Go.** Il linguaggio di templating di Hugo ha una curva di apprendimento se hai bisogno di andare oltre ciò che il tema fornisce.
- **Il contenuto generato da AI rischia di sembrare generico.** Questa è vera. Più deleghi a un'AI, più alto è il rischio che la scrittura perda il suo spigolo: la specificità, le opinioni, la voce che la rende *tua*. La prosa AI tende ad essere competente ma sicura. Se non stai attivamente spingendo indietro, aggiustando e iniettando il tuo punto di vista, il risultato può sembrare rifinito ma vuoto. Lo strumento non sa cosa pensi davvero. Quella parte è ancora tua.

Per me, questi compromessi ne valgono la pena. Preferisco avere il pieno controllo nel mio editor rispetto alla comodità di un browser.

Nel mio caso specifico c'è una dimensione extra: non sono un madrelingua inglese, e Claude semplicemente scrive in inglese meglio di me. Non è una cosa da poco. Ma revisiono attentamente ogni sezione, non per correggere la grammatica, ma per assicurarmi che le idee comunicate siano effettivamente le mie. Le parole sono di Claude; le opinioni, l'enfasi, le cose che scelgo di tenere o tagliare: **quelle sono mie.**

E onestamente? Senza AI non pubblicherei nulla. Non perché non abbia cose da dire, ma perché il tempo è il vero collo di bottiglia. L'AI non migliora solo la scrittura: la fa *accadere*.

## Considerazioni finali

Il passaggio da Payload a Hugo non riguardava davvero quale strumento sia "migliore". Riguardava trovare la configurazione in cui riesco davvero a fare le cose, dove l'attrito è abbastanza basso da far sembrare la pubblicazione un'estensione naturale di come già lavoro, piuttosto che uno sforzo separato che richiede il suo overhead mentale.

Non sono sicuro che questo workflow funzionerebbe per tutti. Ma per un non-madrelingua inglese che ha cose da dire, poco tempo per dirle, e un'AI che scrive in inglese meglio di lui: funziona incredibilmente bene.

Nell'interesse della trasparenza: questo stesso post è stato scritto da Claude. Gli ho dato le idee, le opinioni e la direzione editoriale. Lui ha gestito le parole. Penso valga la pena essere chiari, motivo per cui ogni post assistito da AI su questo sito è contrassegnato con un banner in cima alla pagina, gestito da un campo personalizzato `ai: true` nel front matter. Lo trovo leggermente scomodo, se sono onesto. C'è qualcosa di strano nel rileggere i propri pensieri in una prosa più pulita di qualunque cosa scriveresti tu stesso. Ma l'alternativa (non pubblicare, non condividere, restare in silenzio perché l'attrito era troppo alto) sembrava peggio. Così ci ho fatto pace.
E il minimo che posso fare è essere trasparente.
