# Discorso di Discussione Tesi (v5)

**Candidato:** Claudio Bagini | **Matricola:** 2045337
**Titolo:** *Centralized Scheduling in Satellite Computing: An LP-based Approach*

*Tempi indicativi a circa 125 parole al minuto, cambi slide compresi. Il filo del discorso: ogni richiesta ha un budget di tempo, e ogni attesa lo consuma. Sulle slide dei risultati i dati sono visibili: a voce si spiega l'andamento e il perché, senza leggere i numeri. Le sezioni seguono l'ordine delle slide; le indicazioni in corsivo tra parentesi quadre segnano il cambio slide e non vanno lette. Formule e simboli sono scritti come si leggono.*

---

## Il problema (~2:20)

*[Slide: Titolo]*

Buongiorno a tutti, con il permesso della commissione, presento la mia tesi, *Centralized Scheduling in Satellite Computing: An LP-based Approach*.

*[Slide: Orbital Edge Computing]*

Per anni i satelliti sono stati semplici ripetitori: con il modello bent-pipe inoltravano a terra i dati grezzi, creando un collo di bottiglia sul downlink. L'Orbital Edge Computing ribalta questa logica: i satelliti elaborano i dati a bordo e trasmettono a terra  solo i risultati. Lo scenario principale per una possibile implementazione sono le megacostellazioni LEO, come Starlink, dove i satelliti viaggiano a circa sette chilometri e mezzo al secondo, di conseguenza  la topologia è in continuo cambiamento. Le applicazioni che possono trarre maggiore beneficio da questo nuovo approccio sono quelle che richiedono risposte in in tempi molto ridotti.

*[Slide: Problem and research question]*

Ogni richiesta ha quindi un budget di tempo,  uplink, calcolo e downlink devono rientrare nella deadline. Inoltre le risorse dei satelliti risultano molto stringenti: batteria limitata, hardware vincolato, collegamenti intermittenti. Decidere dove eseguire ogni task è quindi un problema di ottimizzazione: un modello ILP lo risolve in modo globalmente ottimo, ma è NP-hard, e risolverne uno per ogni task ha un costo troppo elevato. La soluzione naturale è raggruppare le richieste in batch, così che ogni decisione consideri più task insieme. Ma aspettare che il batch si riempia consuma proprio il budget di tempo delle richieste. L'obiettivo finale della mia tesi è quindi quello di rispondere a questa domanda:

> Fino a che punto un'ottimizzazione centralizzata può mantenere un'allocazione efficiente in reti satellitari ad alta densità, prima che accodamento e tempi del solutore ne degradino le prestazioni?

*[Slide: Objective and contributions]*

Per rispondere ho sviluppato un modello ILP per instradamento delle task e allocazione della CPU, un motore di batching con compensazione della deadline, e un orchestratore in Python con SimPy, PuLP e HiGHS. Ho successivamente svolto una campagna investigativa per analizzare e , confrontare le sue prestazioni contro altri 4 algoritmi: OrbitAware, DTS-base, DTS-Optimal e un ILP distribuito, che da qui in avanti chiamerò gerarchico.

---

## La soluzione proposta (~1:50)

*[Slide: System model]*

Il sistema è una costellazione Walker-Delta regionale, con un Master Orchestrator che è a conoscenza dello stato globale di code, batterie e collegamenti. Ogni richiesta attraversa tre fasi. L'uplink, ovvero un percorso multi-hop fino al satellite scelto e l'esecuzione sulla coda di bordo, sono entrambi pianificati insieme dall'ILP. Mentre la consegna del risultato è invece delegata: i satelliti la instradano in autonomia, con un routing geografico greedy.

*[Slide: Dynamic batching engine]*

Il componente che gestisce il compromesso tra qualità della decisione e tempo di attesa è il motore di batching. Le richieste entrano in un buffer, che viene svuotato in due casi: quando raggiunge la dimensione BS, il Batch Size, oppure quando scade  BT, il Batch Timeout. I due parametri hanno funzioni diverse: BS limita le variabili binarie, e quindi il tempo di risoluzione; BT limita l'attesa in coda, e quindi protegge le deadline anche quando il traffico è basso. Inoltre qualora il fallimento sia transitorio, la richiesta rientra in coda con il suo tempo di arrivo originale.

*[Slide: Centralized ILP formulation]*

All'interno di ciascun batch, l'ILP assegna ogni richiesta a un satellite, oppure la rifiuta. La funzione obiettivo bilancia tempo di risposta, energia totale spesa ed una penalità di sunset, che tiene i task lontani dai satelliti vicini all'eclissi; in modalità Energy-first i pesi di tempo ed energia si scambiano. I vincoli inseriti nell ILP sono di unicità, deadline, batteria e bilanciamento del carico. Per ogni task rifiutata, una diagnosi a cascata ne registra la causa.

---

## La calibrazione (~1:55)

*[Slide: Experimental setup]*

Resta da stabilire come scegliere i parametri del batching. Gli esperimenti svolti sono simulazioni a eventi discreti tramite SimPy con una costellazione a seicento chilometri di altitudine, orbite SGP4, collegamenti ottici, un tasso di arrivo da due a dieci richieste al secondo e batteria da quaranta o ottanta chilojoule. Confrontando nove algoritmi: tre euristiche distribuite, due ILP gerarchici distribuiti ed infine  quattro configurazioni del nostro ILP centralizzato, su tre tipi di carico: misto, CPU-intensive e data-intensive.

*[Slide: Two-tier parameter calibration]*

La calibrazione iniziale effettuata ha due livelli. Il primo riguarda il batching, e qui abbiamo notato che entrambi gli estremi falliscono. Se si attende troppo, con timeout lunghi o batch grandi, l'attesa consuma lo slack e gran parte degli scarti è dovuta alla deadline superata. Qualora non si attendesse affatto, quindi con un timeout di zero secondi: l'allocazione diventa miope e le prestazioni peggiorano nettamente sotto carico. Di conseguenza abbiamo estratto due configurazioni valide: il fast-dispatch, con batch size pari a venti e timeout di 0,05 secondi, e lo small-batching, con batch size da due e timeout di due secondi.

Il secondo livello riguarda i pesi della funzione obiettivo, dopo un'estensiva campagna di simulazioni sono stati scelti alfa pari a mille e gamma pari a dieci. è emerso che  gamma troppo alto spinge l'orchestratore ad allungare i percorsi pur di evitare le eclissi, peggiorando il tempo di risposta; con gamma pari a dieci, invece, le eclissi non causano più scarti.

*[Slide: Calibration result]*

Il grafico mostra i risultati ottenuti: la calibrazione migliora in modo evidente lo small-batching in modalità Time, fino a circa venti punti percentuali di success rate, mentre il fast-dispatch e la modalità Energy risultati quasi del tutto invariate.

---

## I risultati (~2:00)

*[Slide: Benchmark: mixed workload]*

Spostiamocu ora sull'analisi del confronto tra i vari algoritmi, partendo dal carico misto. Il risultato è meno ovvio del previsto. Le euristiche distribuite sono le più robuste in quanto decidono subito, senza coda d'ingresso, e degradano solo gradualmente all'aumentare del carico. Tra gli approcci centralizzati il migliore è lo small-batching in modalità Time, che a carico alto supera il fast-dispatch di circa venti punti. L'ILP gerarchico, invece, crolla inesorabilmente non appena il traffico inizia a crescere.

*[Slide: Response time]*

Per capire il motivo di questi risultati, scomponiamo il tempo di risposta in esecuzione a bordo e ritardo di rete. L'esecuzione a bordo è contenuta, nell'ordine di pochi decimi di secondo, di conseguenza a limitare la latenza non è la velocità del singolo satellite, ma la rete e le code. Nell'ILP gerarchico il ritardo cresce molto all'aumentare del tasso di arrivo. Lo small-batching mostra invece un andamento opposto: la latenza diminuisce quando il carico aumenta, perché le raffiche di richieste riempiono subito il batch da due e l'attesa reale resta ben sotto il timeout. In questa maniera  un timeout di due secondi riesce a convivere con deadline sotto il secondo.

*[Slide: Failure diagnostics]*

Infine l'analisi dei fallimenti conferma questa lettura. Nel fast-dispatch e nell'ILP gerarchico, sotto stress, la gran parte degli scarti è dovuta alla deadline superata: sono le code a erodere lo slack, quelle di coordinamento nel gerarchico e quelle di CPU sui singoli nodi nel fast-dispatch, dove ogni task è allocato senza una visione d'insieme. Con lo small-batching la deadline diventa una causa marginale: i task vengono rifiutati quasi solo per esaurimento fisico di CPU o di banda.  

---

## Il ranking dipende dal carico (~0:45)

*[Slide: Benchmark: the ranking depends on the workload]*

Questi risultati, però, dipendono molto dal tipo di carico. Nelle simulazioni data-intensive il vincolo non sono più le code ma i link ottici stessi e l'energia dei transceiver, che causano quasi tutti gli scarti. In questo scenario vince l'ILP gerarchico in modalità Time, perché la gestione dei link conta più dell'attesa.

*[Slide: Benchmark: CPU-intensive workload]*

Nello scenario CPU-intensive, invece, i dati sono trascurabili e tornano in testa le euristiche, mentre lo small-batching tiene molto meglio del fast-dispatch, che collassa per le scadenze mancate. In questo caso a esaurirsi è la CPU: è un limite di capacità, non di velocità del singolo satellite.

---

## La risposta alla domanda di ricerca (~2:00)

*[Slide: Complexity and scalability]*

Resta da verificare se il collo di bottiglia possa essere il solutore. Con batch fino a venti e circa quindici satelliti candidati, l'ILP ha circa trecento variabili binarie, e HiGHS lo risolve in meno di un millisecondo, indipendentemente dal tasso di arrivo. Il solutore, quindi, non è il problema.

*[Slide: Deployment guidelines]*

Mettendo insieme i tre scenari, si ottengono le seguenti linee guida per una possibile implementazion. Per il carichi CPU-intensive, le euristiche distribuite. Per data-intensive, l'ILP gerarchico in modalità Time. Nel carico misto invece dipende dall'intensità: da lambda sei in su, di nuovo le euristiche; fino a lambda quattro, lo small-batching centralizzato. A basso carico l'alto tasso di successo e la bassa latenza rendono lo small batching una valida opzione per le applicazioni che richiedono brevi tempi di risposta. Come si può vedere nessun paradigma vince ovunque: la scelta dell'algoritmo ottimale risulta strettamente legata al tipo di carico che deve essere gestito.

*[Slide: Conclusions]*

Posso quindi rispondere alla domanda di ricerca. Il limite dell'orchestrazione centralizzata è architetturale, non computazionale: il solutore è rapidissimo e l'esecuzione a bordo è limitata, mentre il collasso da lambda sei in su nasce dall'accodamento, in ingresso e sui nodi di bordo, che consuma lo slack delle deadline. Tra le 28 configurazioni analizzatei, la configurazione migliore è lo small-batching con alfa pari a mille e gamma pari a dieci.

I limiti rilevati sono tre. Il batching resta un compromesso per i servizi sotto il secondo. Il Master è un single point of failure, in un abiente costantemente esposto a radiazioni, riavvii e spostamenti orbitali. Infine la telemetria: pesa meno dello 0,1% della banda ISL fino a cinquanta satelliti, ma con migliaia di satelliti lo stato invecchia alla velocità dei satelliti LEO.

---

## Sviluppi futuri (~0:45)

*[Slide: Future work]*

Ed è proprio dai limiti che emergono delle nuove possibilità. Poiché nessun paradigma vince ovunque, una nuova direzione potrebbe essere quella di un'orchestrazione ibrida a due livelli: una via veloce euristica per i task compute-intensive e un livello ILP per quelli data-intensive. Per il dilemma del batching si potrebbe implementare un batching adattivo con reinforcement learning, che regoli il timeout in base a traffico, slack residuo e code. Per il single point of failure invece, la soluzione potrebbe essere un approccio multi-agente distribuito, il MADRL. 

*[Slide: Thank you]*

Con questo concludo la mia presentazione, vi ringrazio per l'attenzione e sono a vostra completa disposizione per eventuali domande.