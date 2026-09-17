---
title: "In Italia nessuno vede il prompt di tracking"
date: 2026-09-17
draft: false
ai: true
image: "images/posts/nobody-in-italy-sees-the-tracking-prompt.png"
tags: ["iOS", "App Tracking Transparency", "Privacy", "Apple", "EU"]
description: "Da iOS 27.0 il prompt di permesso di App Tracking Transparency non appare mai sui dispositivi loggati con un Apple Account italiano. Non solo nella mia app: in nessuna app. Sembra che un pezzo delle modifiche UE di iOS 27.2 sia arrivato tre release in anticipo."
---

Ho passato una serata convinto di aver rotto la richiesta di permesso per il tracking. Il prompt aveva smesso di apparire sul mio telefono. Stessa build che funzionava una settimana prima, nessuna modifica, e `ATTrackingManager.requestTrackingAuthorization()` che tornava `.notDetermined` in sette millisecondi, senza prompt e senza errori.

Poi ho aperto Instagram, che non me l'aveva mai chiesto nemmeno lui.

## Cosa dice davvero il sistema

L'app vede soltanto l'esito, quindi l'app è il posto sbagliato dove guardare. La decisione viene presa altrove, da un daemon chiamato `tccd`.

TCC sta per Transparency, Consent and Control. È la parte di iOS che possiede ogni permesso di privacy che hai mai accettato con un tap: fotocamera, microfono, foto, posizione, contatti. La tua app non disegna quegli alert e non l'ha mai fatto. Chiede a `tccd`, `tccd` decide se un alert vada mostrato o no, un processo di sistema lo disegna se la risposta è sì, e alla tua app viene infine consegnato un risultato. App Tracking Transparency è uno di quei permessi come tutti gli altri, archiviato sotto il nome di servizio `kTCCServiceUserTracking`.

Quindi il log interessante non è quello dell'app. È quello del dispositivo, e `tccd` dice ad alta voce cosa ha fatto:

```bash
log collect --device --last 5m --output att.logarchive
```

```
[ATTrackingManager] requestTrackingAuthorizationWithCompletionHandler API call invoked.
[ATTrackingManager] Performing TCC Access Request.
tccd  AUTHREQ_CTX: function=TCCAccessRequest, service=kTCCServiceUserTracking
tccd  #AuthorizationPromptServiceClient prompt is NOT eligibleToShow for kTCCServiceUserTracking <private>
tccd  AUTHREQ_RESULT: authValue=1, authReason=0, error=(null)
[ATTrackingManager] requestTrackingAuthorizationWithCompletionHandler returning - ATT not determined.
```

`NOT eligibleToShow`. Leggi la sequenza: la richiesta arriva, TCC la valuta, TCC rifiuta di presentare qualcosa, e il framework restituisce `.notDetermined`. Nessun errore da nessuna parte, perché dal punto di vista del sistema non è andato storto niente. Gli è stato chiesto se mostrare un alert e ha risposto no.

Questa è la parte con conseguenze. "Nessun prompt è apparso" è un esito legittimo di questa API, non uno stato di errore che puoi aggirare nel codice, ed è restituito con lo stesso identico valore che ottieni da un utente che semplicemente non ha ancora risposto. Le due situazioni sono indistinguibili per la tua app, per design.

Il motivo del rifiuto è su quella stessa riga, oscurato come `<private>`. Puoi togliere la censura con un configuration profile, ma quello generico rimuove la redazione per l'intero sistema: i dati privati di ogni app in chiaro, su un telefono che contiene la tua vita. Ho lasciato perdere.

## L'unica variabile che conta

Qualcuno sugli [Apple Developer Forums](https://developer.apple.com/forums/thread/845135) aveva già fatto l'esperimento che io non potevo fare:

> Stesso dispositivo, stessa posizione fisica, stessa build, stesso progetto di test, nessun'altra modifica: con un Apple Account italiano il prompt non viene mai presentato e lo stato resta `notDetermined`, mentre con un Apple Account spagnolo viene presentato correttamente.

Non la region del dispositivo. Non il posto dove ti trovi. Il paese dell'Apple Account con cui il telefono è loggato. È registrato come [FB24689594](https://developer.apple.com/forums/thread/845135), e secondo quel thread torna a funzionare in iOS 27.2 beta 1. (I report su Feedback Assistant sono visibili solo a chi li ha aperti, quindi il thread è la fonte citabile.)

Che è un numero di versione molto specifico per correggere un bug molto specifico.

## Cosa ha annunciato Apple per la 27.2

Il [16 settembre](https://developer.apple.com/news/?id=idsft9ai) Apple ha pubblicato le modifiche in arrivo su App Tracking Transparency nell'Unione Europea, come parte degli accordi con le autorità europee della concorrenza. Da iOS 27.2 e iPadOS 27.2 gli sviluppatori hanno a disposizione una versione alternativa del prompt di sistema: formattazione diversa, linguaggio diverso e un pulsante opzionale "Informazioni aggiuntive" che può mostrare più dettagli su cosa intendi fare con i dati. Gli sviluppatori UE potranno anche richiedere di nuovo il permesso un anno dopo la risposta precedente dell'utente, qualunque essa sia stata. Le regole su *quando* serve il permesso non cambiano. I dettagli sono sulla pagina Apple [user privacy and data use](https://developer.apple.com/app-store/user-privacy-and-data-use/).

Poi c'è la frase che spiega la mia serata:

> Per requisiti di legge, solo la versione alternativa del prompt di sistema è disponibile per le app distribuite in Germania, Francia, Italia, Polonia e Romania.

Cinque paesi dove il vecchio alert non è più un'opzione. L'Italia è uno di quelli.

## Un cancello senza porta

Metti insieme le due metà e ottieni una teoria che spiega tutto quello che ho visto.

Nella 27.2, per gli account di quei cinque paesi, il prompt classico non deve essere mostrato. Qualcosa deve farlo rispettare. Sembra che l'applicazione della regola sia arrivata nella 27.0 e la schermata sostitutiva no, quindi su 27.0 e 27.1 il sistema rifiuta correttamente di presentare il vecchio alert a un account italiano e non ha niente da presentare al suo posto. Un cancello senza porta dietro. `NOT eligibleToShow` è esattamente come la cosa apparirebbe da fuori.

Voglio essere chiaro: questa è una lettura, non una scoperta. Non posso vedere il codice di Apple, e nemmeno tu. Quello che è verificabile: il rifiuto avviene, il paese dell'account è l'unica variabile che lo cambia, i cinque paesi nell'annuncio di Apple includono l'Italia, e la versione che corregge il problema è la stessa release che introduce il nuovo prompt.

Spiega anche come sia potuto arrivare in produzione. Per incontrarlo ti serve un dispositivo su 27.0 loggato con un account registrato in Germania, Francia, Italia, Polonia o Romania. Se i tuoi device di test sono americani, ogni prompt funziona perfettamente, per sempre.

## Sono rotte tutte, non solo la tua

Questa è la parte che vale la pena ripetere, perché è quella che ti risparmierà la serata che ho perso io: non è la tua app.

Facebook, Instagram e ogni altra app che chiede il permesso di tracking si comportano in modo identico su un account affetto dal problema. Non appare niente. Se hai un telefono italiano su 27.0 o 27.1 tra le mani in questo momento, apri un'app gratuita che non hai mai lanciato e guardala non chiederti niente.

Quindi se un tester apre una segnalazione "il prompt di tracking è scomparso", resisti alla tentazione di andare a leggere il tuo codice. Controlla se una qualunque app su quel telefono riesce a mostrarlo.

## Conviverci fino alla 27.2

Un paio di cose che farei comunque, indipendentemente da come si risolve.

Non far dipendere la tua UI dalla comparsa del prompt. Se hai uno step di onboarding la cui unica azione è richiedere il permesso di tracking, e che si rifiuta di avanzare finché lo stato non cambia, quella schermata diventa un vicolo cieco nel momento in cui il sistema smette di presentare qualsiasi cosa: un bottone che non può riuscire, niente a cui rispondere, nessun modo di andare avanti. Tratta "ancora non determinato" come un risultato perfettamente normale e lascia che le persone vadano oltre.

Dai alla richiesta una seconda via d'accesso, una riga nella schermata delle impostazioni o qualcosa di simile. Chi ha saltato lo step di onboarding, o non è mai stato interpellato, al momento non ha modo di tornarci.

Non fidarti del simulatore, qui. A me ha continuato a mostrare il prompt tutto il tempo, perché non è loggato con un Apple Account italiano. Implementa l'API. Non riproduce la policy, e la policy è tutto il bug.

E App Review fa meno paura di quanto sembri. I reviewer non usano Apple Account italiani, quindi il tuo prompt lo vedranno. E anche se non lo vedessero, iOS azzera l'IDFA senza autorizzazione ATT, quindi non si sta tracciando niente e non c'è niente su cui essere non conformi. [FB24689594](https://developer.apple.com/forums/thread/845135) è lì da citare se dovesse mai servire.

Le modifiche UE in sé vale la pena leggerle bene prima che arrivi la 27.2, soprattutto se distribuisci in quei cinque paesi, perché il prompt intorno a cui hai progettato per cinque anni sta per diventare un prompt diverso.

**Fonti:** [Updates to App Tracking Transparency in the European Union](https://developer.apple.com/news/?id=idsft9ai) (Apple, 16 settembre 2026), [User Privacy and Data Use](https://developer.apple.com/app-store/user-privacy-and-data-use/) (Apple), [thread 845135 dei Developer Forums](https://developer.apple.com/forums/thread/845135), [copertura di 9to5Mac](https://9to5mac.com/2026/09/16/ios-27-2-lets-developers-use-an-alternative-app-tracking-transparency-prompt-in-the-eu/).
