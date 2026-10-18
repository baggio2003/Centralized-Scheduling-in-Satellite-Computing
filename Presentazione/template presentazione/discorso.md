# Discorso di Discussione Tesi (v2)

**Candidato:** Claudio Bagini | **Matricola:** 2045337
**Titolo:** *Centralized Scheduling in Satellite Computing: An LP-based Approach*

*Tempi indicativi per sezione, a circa 125 parole al minuto. Le sezioni sono richiamate per titolo di slide, non per numero. Formule e simboli sono scritti come si leggono.*

---

## Titolo e Orbital Edge Computing (~1:05)

Buongiorno a tutti, gentile commissione, e buongiorno al mio relatore, il Professor Emiliano Casalicchio. Presento la mia tesi, *Centralized Scheduling in Satellite Computing: An LP-based Approach*.

Per anni i satelliti hanno lavorato come ripetitori passivi, il cosiddetto modello bent-pipe: inoltravano i dati grezzi a terra, con un collo di bottiglia sul downlink e ritardi lunghi. L'Orbital Edge Computing cambia paradigma: i satelliti elaborano i dati a bordo e trasmettono a terra solo i risultati.

Parliamo delle megacostellazioni LEO, come Starlink, OneWeb e Kuiper: orbite tra i trecento e i duemila chilometri, circa sette chilometri e mezzo al secondo, quindi una topologia che cambia di continuo, con collegamenti inter-satellite ottici o a radiofrequenza. Le applicazioni sono la gestione dei disastri, la navigazione marittima e la sorveglianza: tutte con scadenze sotto il secondo.

---

## Problema e domanda di ricerca, obiettivi e contributi (~1:20)

Orchestrare questi sistemi è difficile: energia limitata, hardware vincolato, collegamenti intermittenti e deadline stringenti, per cui tempo di uplink, di calcolo e di downlink devono stare entro la scadenza della richiesta. Un modello ILP dà soluzioni globalmente ottime, ma è NP-hard, e risolverne uno per ogni task costa troppo. Raggruppare le richieste migliora le soluzioni, ma il tempo di attesa si mangia le deadline.

La domanda di ricerca è quindi:

> Fino a che punto un'ottimizzazione centralizzata può mantenere un'allocazione efficiente in reti satellitari ad alta densità, prima che accodamento e tempi del solutore ne degradino le prestazioni?

La tesi porta cinque contributi: un modello ILP per instradamento e allocazione della CPU congiunti, in modalità Mapping Time o Mapping Energy; un dynamic batching con dimensione e timeout, più una compensazione della deadline per il ritardo di accodamento; un orchestratore in Python con SimPy, PuLP e HiGHS; un'analisi di stress fino a dieci richieste al secondo; e un benchmark contro OrbitAware, DTS-base, DTS-Optimal e un ILP distribuito, che nei risultati chiamerò gerarchico.

---

## Modello di sistema, motore di batching e formulazione ILP (~1:50)

Il sistema è una costellazione Walker-Delta regionale con un Master Orchestrator, che conosce lo stato globale: code, batterie e collegamenti. Ogni richiesta ha tre fasi. L'uplink, un percorso multi-hop fino al satellite scelto, e l'esecuzione sulla coda di bordo sono pianificati insieme dall'ILP centralizzato. La consegna del risultato è delegata: i satelliti la instradano in autonomia, con un routing geografico greedy.

Il motore di batching è guidato da eventi. Le richieste entrano in un buffer, che si svuota quando raggiunge la dimensione BS, il Batch Size, oppure scade il timeout BT, il Batch Timeout. Una mappa della topologia, calcolata con Dijkstra multi-sorgente, alimenta l'ILP, e ogni richiesta viene instradata, riprovata o rifiutata. Le richieste che falliscono per cause transitorie, per esempio nessun percorso disponibile, rientrano in coda mantenendo il tempo di arrivo originale. BS limita le variabili binarie, e quindi il tempo di risoluzione; BT limita l'attesa, evitando la starvation quando il traffico è basso.

Il modello ILP assegna ogni richiesta a un satellite, oppure la rifiuta. L'obiettivo combina tempo, energia e una penalità di sunset, che allontana i task dai satelliti prossimi all'eclissi; la modalità Energy-first scambia i pesi di tempo ed energia. I vincoli sono unicità, deadline, batteria e bilanciamento del carico. Per ogni richiesta rifiutata, una diagnosi a cascata ne identifica la causa: la ritroveremo nell'analisi dei fallimenti.

---

## Setup sperimentale e calibrazione (~1:35)

Gli esperimenti sono simulazioni a eventi discreti con SimPy: costellazione Walker-Delta a seicento chilometri, orbite SGP4, collegamenti ottici, arrivi da due a dieci richieste al secondo e batteria da quaranta o ottanta chilojoule. Confronto nove algoritmi: tre euristiche distribuite, cioè DTS-base, DTS-Optimal e OrbitAware; due ILP gerarchici; e quattro configurazioni del nostro ILP centralizzato. I carichi sono di tre tipi: misto, CPU-intensive e data-intensive.

La calibrazione ha due livelli. Sul batching, in ventotto configurazioni: con timeout da 0,2 secondi o batch da sei in su, il 70-85% dei task salta la deadline; con timeout zero, un task alla volta, il greedy è miope e si ferma al 41-48% a lambda dieci. Restano due archetipi: il fast-dispatch, con batch da venti e timeout di 0,05 secondi, e lo small-batching, con batch da due e timeout di due secondi.

Sui pesi, scelgo alfa mille e gamma dieci: con gamma da mille in su i percorsi si allungano e il tempo di risposta sale del 55%; con gamma dieci i drop da eclissi sono zero. Come mostra il grafico, questa ottimizzazione porta lo small-batching in modalità Time fino a 19,5 punti percentuali di success rate in più.

---

## Benchmark sul carico misto, tempo di risposta e cause di scarto (~1:50)

Veniamo ai risultati, partendo dal carico misto a ottanta chilojoule. Le euristiche distribuite sono le più robuste: dal 98-99% al 79-83% a lambda dieci, perché non hanno coda d'ingresso. Lo small-batching in modalità Time è il migliore tra i centralizzati: dal 90-92% al 65-67%, con 12-20 punti percentuali in più del fast-dispatch a lambda dieci. L'ILP gerarchico crolla invece al 39-45%, per i ritardi di coordinamento tra cluster.

Per capire perché, scomponiamo il tempo di risposta. L'esecuzione a bordo è costante: tra 0,17 e 0,48 secondi, in tutti gli algoritmi e a tutti i carichi. Per la latenza, quindi, la CPU del singolo satellite non è il collo di bottiglia: lo sono rete e code. L'ILP gerarchico sale fino a 1,28 secondi. Lo small-batching, invece, mostra una latency compression: la latenza scende da circa 0,76 a 0,56 secondi all'aumentare del carico, perché i burst riempiono subito il batch da due e l'attesa reale resta ben sotto il timeout. È così che un timeout di due secondi convive con deadline sotto il secondo.

Il diagramma delle cause di scarto conferma il quadro. Fast-dispatch e ILP gerarchico, sotto stress, perdono il 65-85% dei task per deadline superata: le code d'ingresso erodono lo slack. Lo small-batching porta i drop per deadline sotto il 2% nel carico misto, e all'8-12% nel CPU-intensive: i task vengono scartati quasi solo per esaurimento fisico di CPU o banda.

---

## Carichi data-intensive e CPU-intensive, linee guida (~1:15)

Il ranking, però, dipende dal carico. Nel data-intensive il collo di bottiglia sono i link ottici e l'energia dei transceiver, oltre il 95% degli scarti, e vince l'ILP gerarchico in modalità Time: dal 99-100% all'81-89%, contro il 77-78% delle euristiche. Qui il vincolo sono i link e non le code, e la gestione ottimizzata dei link conta più dell'attesa. Passare da quaranta a ottanta chilojoule vale fino a 9-15 punti.

Nel CPU-intensive i dati sono trascurabili e tornano in testa le euristiche, con il 76-77% a lambda dieci; lo small-batching tiene il 56-58%, fino a 25 punti sopra il fast-dispatch, che si ferma al 31-33%.

Ne derivano le linee guida. CPU-intensive: euristiche distribuite. Data-intensive: ILP gerarchico in modalità Time. Carico misto: small-batching centralizzato fino a lambda quattro, con l'82-92% di success rate; da lambda sei in su, di nuovo le euristiche, con il 79-83%. L'ottimalità dell'orchestrazione dipende dunque dal carico.

---

## Complessità, conclusioni e limiti (~1:20)

Sulla complessità: la mappa della topologia, con nodi di grado al massimo quattro, è quasi lineare nel numero di satelliti. Con batch fino a venti e circa quindici satelliti candidati, l'ILP ha poche centinaia di variabili binarie e HiGHS lo risolve in 0,30-0,60 millisecondi, indipendentemente dal tasso di arrivo.

Rispondo quindi alla domanda di ricerca: il limite dell'orchestrazione centralizzata è architetturale, non computazionale. Il solutore è rapidissimo e l'esecuzione a bordo è limitata; il collasso da lambda sei in su dipende dall'accodamento in ingresso, che consuma lo slack delle deadline. Tra i centralizzati, il migliore è lo small-batching con alfa mille e gamma dieci.

I limiti sono tre. Il batching resta un compromesso per servizi sotto il secondo. Il Master è un single point of failure, esposto a radiazioni, riavvii e manovre. E la telemetria, che pesa meno dello 0,1% della banda ISL fino a cinquanta satelliti, diventa un limite con migliaia di satelliti, quando lo stato si invecchia alla velocità LEO.

---

## Sviluppi futuri e chiusura (~0:40)

Come sviluppi futuri: un'orchestrazione ibrida a due livelli, con una via veloce euristica per i task compute-intensive e un livello ILP per quelli data-intensive; un batching adattivo con reinforcement learning, che regola il timeout in base a traffico, slack residuo e code; un approccio multi-agente distribuito, MADRL, per eliminare il single point of failure; e un prototipo hardware-in-the-loop, con HiGHS su SoC di classe spaziale.

Vi ringrazio per l'attenzione e sono a vostra disposizione per le domande.