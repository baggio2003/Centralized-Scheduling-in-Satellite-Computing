# Discorso di Discussione Tesi (v3)

**Candidato:** Claudio Bagini | **Matricola:** 2045337
**Titolo:** *Centralized Scheduling in Satellite Computing: An LP-based Approach*

*Tempi indicativi per sezione, a circa 125 parole al minuto (totale circa 12 minuti, cambi slide compresi). Le sezioni sono richiamate per titolo di slide, non per numero; le indicazioni in corsivo tra parentesi quadre segnano il cambio slide e non vanno lette. Formule e simboli sono scritti come si leggono.*

---

## Titolo e Orbital Edge Computing (~1:05)

Buongiorno a tutti, gentile commissione, e buongiorno al mio relatore, il professor Emiliano Casalicchio. Presento la mia tesi, *Centralized Scheduling in Satellite Computing: An LP-based Approach*.

*[Slide: Orbital Edge Computing]*

Per anni i satelliti hanno lavorato come ripetitori passivi, il cosiddetto modello bent-pipe: inoltravano i dati grezzi a terra, con un collo di bottiglia sul downlink e ritardi lunghi. L'Orbital Edge Computing cambia paradigma: i satelliti elaborano i dati a bordo e trasmettono a terra solo i risultati.

Parliamo di megacostellazioni LEO, come Starlink, OneWeb e Kuiper: orbite tra i trecento e i duemila chilometri, a circa sette chilometri e mezzo al secondo. La topologia cambia quindi di continuo, e i collegamenti inter-satellite sono ottici o a radiofrequenza. Le applicazioni, come la gestione dei disastri, la navigazione marittima e la sorveglianza per la difesa, hanno tutte scadenze sotto il secondo.

---

## Problema e domanda di ricerca, obiettivi e contributi (~1:25)

*[Slide: Problem and research question]*

Orchestrare questi sistemi è difficile: energia limitata, hardware vincolato, collegamenti intermittenti e scadenze stringenti. La somma dei tempi di uplink, calcolo e downlink deve stare entro la deadline della richiesta.

Un modello ILP, di programmazione lineare intera, dà soluzioni globalmente ottime, ma è NP-hard: risolverne uno per ogni task costa troppo. Raggruppare le richieste in batch migliora le soluzioni, ma il tempo di attesa si mangia le deadline. La domanda di ricerca è quindi:

> Fino a che punto un'ottimizzazione centralizzata può mantenere un'allocazione efficiente in reti satellitari ad alta densità, prima che accodamento e tempi del solutore ne degradino le prestazioni?

*[Slide: Objective and contributions]*

La tesi porta cinque contributi. Primo: un modello ILP per instradamento e allocazione della CPU congiunti, in modalità Mapping Time o Mapping Energy. Secondo: un dynamic batching con dimensione e timeout, più una compensazione della deadline per il ritardo di accodamento. Terzo: un orchestratore in Python con SimPy, PuLP e HiGHS. Quarto: un'analisi di stress fino a dieci richieste al secondo. Quinto: un benchmark contro OrbitAware, DTS-base, DTS-Optimal e un ILP distribuito, che nei risultati chiamerò gerarchico.

---

## Modello di sistema, motore di batching e formulazione ILP (~2:10)

*[Slide: System model]*

Il sistema è una costellazione Walker-Delta regionale, con un Master Orchestrator che conosce lo stato globale: code, batterie e collegamenti. Ogni richiesta ha un carico di calcolo, una deadline, una dimensione e un punto di accesso di origine.

Il ciclo di vita ha tre fasi. L'uplink, un percorso multi-hop fino al satellite scelto, e l'esecuzione sulla coda di bordo sono pianificati insieme dall'ILP centralizzato. La consegna del risultato è invece delegata: i satelliti la instradano in autonomia, con un routing geografico greedy.

*[Slide: Dynamic batching engine]*

Il motore di batching è guidato da eventi. Le richieste entrano in un buffer, che si svuota quando raggiunge la dimensione BS, il Batch Size, oppure scade il timeout BT, il Batch Timeout. BS limita le variabili binarie, e quindi il tempo di risoluzione; BT limita l'attesa ed evita la starvation a basso traffico. Prima di leggere la telemetria, una barriera a ritardo zero fa completare gli eventi simultanei, per avere uno stato coerente. Poi una mappa della topologia, calcolata con Dijkstra multi-sorgente, alimenta l'ILP, e ogni richiesta viene instradata, riprovata o rifiutata. Se il fallimento è transitorio, per esempio per mancanza di un percorso, la richiesta rientra in coda con il tempo di arrivo originale.

*[Slide: Centralized ILP formulation]*

Il modello ILP assegna ogni richiesta a un satellite, oppure la rifiuta. L'obiettivo combina tempo, energia e una penalità di sunset, che allontana i task dai satelliti vicini all'eclissi; la modalità Energy-first scambia i pesi di tempo ed energia. I vincoli sono quattro: unicità dell'assegnazione, deadline, batteria e bilanciamento del carico. Per ogni richiesta rifiutata, una diagnosi a cascata ne individua la causa: la ritroveremo nell'analisi dei fallimenti.

---

## Setup sperimentale e calibrazione (~1:45)

*[Slide: Experimental setup]*

Gli esperimenti sono simulazioni a eventi discreti con SimPy: costellazione Walker-Delta a seicento chilometri, orbite SGP4, collegamenti ottici, arrivi da due a dieci richieste al secondo e batteria da quaranta o ottanta chilojoule. Confronto nove algoritmi: tre euristiche distribuite, cioè DTS-base, DTS-Optimal e OrbitAware; due ILP gerarchici, in modalità Time ed Energy; e quattro configurazioni del nostro ILP centralizzato. I carichi sono di tre tipi: misto, CPU-intensive e data-intensive.

*[Slide: Two-tier parameter calibration]*

La calibrazione ha due livelli. Prima il batching, su ventotto configurazioni. Con timeout da 0,2 secondi in su, o con batch da sei in su, l'attesa in coda consuma lo slack: tra il 70 e l'85% degli scarti è per deadline superata. All'estremo opposto, con timeout zero, quindi un task alla volta, l'allocazione diventa greedy e miope, e a lambda dieci il success rate si ferma al 41-48%. Restano due archetipi: il fast-dispatch, con batch da venti e timeout di 0,05 secondi, e lo small-batching, con batch da due e timeout di due secondi.

Poi i pesi dell'obiettivo: scelgo alfa mille e gamma dieci. Con gamma da mille in su i percorsi si allungano e il tempo di risposta sale del 55%; con gamma dieci, invece, i drop da eclissi sono zero.

*[Slide: Calibration result]*

Come mostra il grafico, questa calibrazione porta lo small-batching in modalità Time fino a 19,5 punti percentuali di success rate in più.

---

## Benchmark sul carico misto, tempo di risposta e cause di scarto (~2:05)

*[Slide: Benchmark: mixed workload]*

Partiamo dal carico misto con batteria da ottanta chilojoule. Le euristiche distribuite sono le più robuste: scendono dal 98-99% al 79-83% a lambda dieci, perché non hanno una coda d'ingresso. Tra i centralizzati, il migliore è lo small-batching in modalità Time: dal 90-92% al 65-67%, cioè fino a 20 punti percentuali in più del fast-dispatch a lambda dieci. L'ILP gerarchico, invece, crolla al 39-45%, per i ritardi di coordinamento tra i cluster.

*[Slide: Response time]*

Per capire perché, scomponiamo il tempo di risposta. L'esecuzione a bordo è contenuta: tra 0,17 e 0,48 secondi, con rare punte appena oltre il mezzo secondo. Quindi la velocità di calcolo del singolo satellite non limita la latenza: la limitano la rete e le code. L'ILP gerarchico arriva fino a 1,28 secondi. Lo small-batching mostra invece una latency compression: la latenza scende da circa 0,76 a 0,56 secondi all'aumentare del carico, perché i burst riempiono subito il batch da due e l'attesa reale resta ben sotto il timeout. È così che un timeout di due secondi convive con deadline sotto il secondo.

*[Slide: Failure diagnostics]*

Nel diagramma delle cause di scarto, per il fast-dispatch e per l'ILP gerarchico, sotto stress, il 65-85% degli scarti è dovuto alla deadline superata: sono le code a erodere lo slack, cioè i ritardi di coordinamento nel gerarchico, e le code di CPU sui singoli nodi nel fast-dispatch, dove ogni task è allocato senza visione d'insieme. Con lo small-batching, invece, la quota di scarti per deadline scende sotto il 2% nel carico misto e all'8-12% nel CPU-intensive: i task vengono scartati quasi solo per esaurimento fisico di CPU o di banda.

---

## Carichi data-intensive e CPU-intensive (~1:00)

*[Slide: Benchmark: the ranking depends on the workload]*

Il ranking, però, dipende dal carico. Nel data-intensive i colli di bottiglia sono i link ottici e l'energia dei transceiver, che causano oltre il 95% degli scarti, e vince l'ILP gerarchico in modalità Time: dal 99-100% all'81-89%, contro il 77-78% delle euristiche. Qui il vincolo sono i link, non le code. Anche la batteria pesa: raddoppiarla, da quaranta a ottanta chilojoule, vale fino a 9-15 punti per l'ILP gerarchico, contro circa 4 per le euristiche.

*[Slide: Benchmark: CPU-intensive workload]*

Nel CPU-intensive, invece, i dati sono trascurabili e tornano in testa le euristiche, con il 76-77% a lambda dieci. Lo small-batching tiene il 56-58%, fino a 25 punti sopra il fast-dispatch, che si ferma al 31-33%. Qui la CPU si esaurisce davvero: è un limite di capacità, non di velocità.

---

## Complessità e linee guida (~1:05)

*[Slide: Complexity and scalability]*

Veniamo alla complessità. Poiché ogni satellite ha al massimo quattro collegamenti, la mappa della topologia è quasi lineare nel numero di satelliti. Con batch fino a venti e circa quindici satelliti candidati, l'ILP ha circa trecento variabili binarie, e HiGHS lo risolve in 0,30-0,60 millisecondi, indipendentemente dal tasso di arrivo.

*[Slide: Deployment guidelines]*

Ecco le linee guida di deployment. CPU-intensive: euristiche distribuite. Data-intensive: ILP gerarchico in modalità Time. Nel carico misto la scelta cambia con l'intensità. Da lambda sei in su, di nuovo le euristiche, con il 79-83%. Fino a lambda quattro, lo small-batching centralizzato, con l'82-92% di success rate e una latenza contenuta, tra 0,72 e 0,76 secondi: a basso carico le euristiche restano più alte in success rate, e lo small-batching si sceglie quando servono coordinamento globale e controllo centralizzato. L'ottimalità dell'orchestrazione dipende, dunque, dal carico.

---

## Conclusioni e limiti (~1:00)

*[Slide: Conclusions]*

Rispondo quindi alla domanda di ricerca: il limite dell'orchestrazione centralizzata è architetturale, non computazionale. Il solutore è rapidissimo e l'esecuzione a bordo è limitata; il collasso da lambda sei in su dipende dall'accodamento, in ingresso e sui nodi di bordo, che consuma lo slack delle deadline. Tra i centralizzati, la configurazione migliore è lo small-batching con alfa mille e gamma dieci.

I limiti sono tre. Primo: il batching resta un compromesso per i servizi sotto il secondo. Secondo: il Master è un single point of failure, esposto a radiazioni, riavvii e manovre. Terzo: la telemetria, che pesa meno dello 0,1% della banda ISL fino a cinquanta satelliti, ma con migliaia di satelliti diventa un limite, perché lo stato invecchia alla velocità dei satelliti LEO.

---

## Sviluppi futuri e chiusura (~0:40)

*[Slide: Future work]*

Come sviluppi futuri vedo quattro direzioni. Un'orchestrazione ibrida a due livelli, con una via veloce euristica per i task compute-intensive e un livello ILP per quelli data-intensive. Un batching adattivo con reinforcement learning, che regoli il timeout in base a traffico, slack residuo e code. Un approccio multi-agente distribuito, il MADRL, per eliminare il single point of failure. E un prototipo hardware-in-the-loop, con HiGHS su SoC di classe spaziale.

*[Slide: Thank you]*

Vi ringrazio per l'attenzione e sono a vostra disposizione per le domande.