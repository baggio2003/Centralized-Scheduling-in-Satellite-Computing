# Domande probabili della commissione

**Candidato:** Claudio Bagini | **Tesi:** *Centralized Scheduling in Satellite Computing: An LP-based Approach*

Venti domande, dalla più importante/probabile alla meno probabile, con spunti di risposta basati su tesi e slide. Gli spunti sono tracce da dire a parole tue, non risposte da recitare.

**Legenda:** ⚠ = la tesi non copre il punto (o lo copre in modo ambiguo): controlla nel codice o nei tuoi appunti prima di rispondere.

**Tre regole pratiche:** ammetti subito un limite quando è reale, e poi spiega cosa mostrano comunque i risultati; se non hai misurato qualcosa, dillo e indica come lo misureresti; ricollega sempre la risposta alla domanda di ricerca.

---

## Le più probabili (1-7)

### 1. Le euristiche distribuite vincono quasi ovunque: qual è il contributo della tesi, e perché dovrei usare un ILP centralizzato?

- Il contributo non è "l'ILP batte le euristiche", ma caratterizzare *fino a dove* regge la centralizzazione (la domanda di ricerca) e *perché* smette di reggere.
- Risultati concreti: il limite è architetturale (accodamento), non computazionale; la calibrazione di α e γ porta lo small-batching fino a +19,5 punti percentuali; la mappa prescrittiva dice quale paradigma usare per quale carico (euristiche per CPU-intensive e misto ad alto carico, ILP gerarchico per data-intensive, small-batching centralizzato per il misto fino a λ=4).
- Ammetti che sul puro success rate le euristiche restano in testa nel misto. Il vantaggio dell'ILP è un altro: visione globale, vincoli espliciti su batteria, eclissi e deadline, controllo centralizzato e diagnosi a cascata delle cause di scarto.

### 2. Cosa significa "il limite è architetturale, non computazionale"? Come lo dimostri?

- Il solutore impiega 0,30-0,60 ms con batch fino a 20 e circa 15 satelliti candidati, e l'esecuzione a bordo resta tra 0,17 e 0,48 s: nessuno dei due spiega il crollo.
- La prova più pulita: *stesso solver, stesso modello*, ma cambiando solo il batching cambia tutto. Con BS=2 le scadenze mancate scendono sotto il 2% degli scarti nel misto; con fast-dispatch o ILP gerarchico arrivano al 65-85% degli scarti.
- Quindi il collasso da λ ≥ 6 dipende da come e quanto si accoda, non da quanto è lento il solver. Rimanda alla domanda 8 per il caveat sul tempo del solver.

### 3. ⚠ Dov'è la formula della compensazione della deadline per il ritardo di accodamento?

- È il contributo 2 e la §1.3 annuncia una formula con T_batch, ma nel capitolo 3 non l'ho trovata definita esplicitamente. L'unica formula correlata è lo slack nella (5.1): Δt_slack = D_r − (T_wait + T_up + W_cpu,i + T_exec + T_down).
- Idea da spiegare: il tempo passato nel buffer riduce lo slack disponibile; nei retry si conserva t_arr, quindi l'attesa conta sempre dall'arrivo originale.
- **Prima della discussione:** apri Orchestrator.py, guarda come viene aggiustata D_r e preparati due righe esatte su cosa fa il codice.

### 4. Con BS=2 l'ILP è davvero "ottimizzazione globale"? Non è un greedy a coppie? E se anche il fast-dispatch lavora con 1-2 task, perché c'è un divario di circa 20 punti?

- Hai ragione: l'ottimalità è globale solo sul batch. |R|=2 è l'orizzonte minimo non banale, e il compromesso con l'attesa è proprio il "batching dilemma" delle conclusioni. Batch più grandi (BS ≥ 6) allargano l'orizzonte ma erodono le deadline.
- Il divario si spiega così: con BT=0,05 s e arrivi ogni 100-500 ms, il fast-dispatch spesso spedisce un task solo (|R|=1), quindi è miope. Il primo task prende i link e i nodi migliori e il secondo finisce su percorsi congestionati o viene rifiutato. Lo small-batching aspetta sempre la coppia e la alloca insieme.

### 5. Perché un ILP, che è NP-hard, e non altre tecniche (euristiche, metaeuristiche, reinforcement learning)? Come fa HiGHS a risolvere in 0,3 ms?

- L'ILP gestisce in modo esatto instradamento, CPU, energia e deadline insieme, e dà un riferimento di ottimalità. Le euristiche sono già nel benchmark come baseline; il reinforcement learning compare negli sviluppi futuri.
- NP-hard riguarda il caso peggiore (la tesi cita O(2^(|R|·|S|))). Qui l'istanza è piccola e strutturata: BS ≤ 20, circa 300 variabili binarie, vincoli sparsi (ogni vincolo di deadline ha un solo termine). HiGHS usa presolve, piani di taglio e simplex duale, e il rilassamento LP converge in meno di 50 pivot.
- Il batching serve proprio a tenere limitata la dimensione del problema, indipendentemente dal tasso d'arrivo.

### 6. Quanto è realistico il simulatore? Come hai riprodotto i baseline?

- Il simulatore usa orbite SGP4, ISL ottici, finestre di visibilità, code di CPU, batterie (40 o 80 kJ per passaggio) e payload da 1 a 50 MB. I profili di carico seguono la letteratura [6].
- I baseline (OrbitAware, DTS-base, DTS-Optimal, ILP gerarchico) vengono dal paper [6], che è del tuo relatore: aspettati attenzione su come li hai implementati. **⚠ Prepara:** cosa hai preso dal codice originale, cosa hai reimplementato e con quali parametri.
- Onestà sul limite: è una simulazione, quindi i risultati valgono come confronto relativo tra algoritmi, non come predizione assoluta. Per questo il prototipo hardware-in-the-loop è tra gli sviluppi futuri.

### 7. ⚠ Quante ripetizioni hai fatto per ogni punto? Ci sono intervalli di confidenza?

- Nella tesi non ho trovato né numero di run/seed né intervalli di confidenza.
- Se hai fatto più ripetizioni, dillo con il numero esatto. Se ogni punto è una singola run, ammettilo e argomenta la robustezza qualitativa: gli stessi andamenti si ripetono su 6 valori di λ, 2 batterie, 4 profili di payload e 3 tipi di carico.
- Un'estensione naturale è ripetere con più seed e riportare le bande di variabilità.

---

## Probabili (8-14)

### 8. ⚠ Il tempo del solver è contato nel tempo simulato? Include anche la costruzione del modello PuLP e Dijkstra?

- In SimPy il tempo reale del solver non avanza l'orologio simulato, a meno che tu non lo inietti. Chiarisci cosa fa il tuo codice e cosa misurano i 0,30-0,60 ms (solo HiGHS, o l'intera pipeline).
- Se non è incluso, riconoscilo: l'ordine di grandezza resta rilevante finché il costo reale è molto sotto lo slack (le deadline sono dell'ordine di 0,5-1 s), ma su un SoC spaziale sarà più lento. È uno dei motivi del prototipo hardware-in-the-loop.

### 9. ⚠ Il Mapping Energy minimizza l'energia, ma non mostri i consumi: dov'è il vantaggio? E perché va peggio del Mapping Time?

- Le metriche del benchmark sono success rate, tempo di risposta e cause di scarto. L'energia entra come vincolo di batteria e come causa di rifiuto, ma non c'è una metrica di consumo energetico, anche se il contributo 5 parla di "energy efficiency". Ammettilo: è una lacuna reale.
- Perché Energy ha un success rate più basso: penalizza i percorsi multi-hop e rifiuta task che il Time riuscirebbe a servire (§5.2.1.1); inoltre manda i task su satelliti efficienti ma con code più lunghe (§4.4.1). Per questo α=1000 non migliora nulla in modalità Energy.
- Quando ammette solo percorsi a singolo hop la latenza scende (0,45-0,63 s), ma a costo di scartare più task.

### 10. Il vincolo di deadline dell'ILP non include T_down: come garantisci la deadline end-to-end?

- Nell'ILP, R(r,i) comprende uplink, code di rete, attesa e calcolo; il ritorno del risultato è delegato a un routing geografico greedy. La slide 2 scrive invece T_up + T_cpu + T_down ≤ D_r: tieni presente la differenza.
- Motivazione (§3.1.1): i task lunghi (W_cpu + d_cpu ≫ 10 s) superano la finestra di stabilità dei collegamenti, quindi pianificare il ritorno al tempo t_arr richiederebbe previsioni inaffidabili.
- Il costo: nessuna garanzia formale su T_down. **⚠ Verifica:** come è contato T_down nelle metriche di successo (la tesi conta come successo la consegna entro la deadline).

### 11. Perché a basso carico nel misto consigli lo small-batching se le euristiche hanno un success rate più alto (95-99% contro 82-92%)?

- Non lo consigli per il success rate: lo scegli quando servono coordinamento globale e controllo centralizzato, con latenza contenuta (0,72-0,76 s). È la riga "Latency-Throughput Balanced" della Tabella 5.1.
- Da λ=6 in su, o se conta soprattutto il throughput, la raccomandazione è per le euristiche (79-83%).
- Il messaggio finale è proprio che l'ottimalità dipende dal carico.

### 12. Se il fast-dispatch aspetta solo 50 ms, come fa l'ingress queuing a spiegare il suo crollo?

- Non lo spiega, e la tesi lo dice a modo suo (§4.2.2 e §4.4.2): con BT=0,05 s i task sono allocati quasi singolarmente e senza visione congiunta, quindi finiscono sugli stessi nodi "migliori". Le code di CPU a bordo si gonfiano e il completamento supera la deadline: il 65-85% degli scarti è per deadline superata.
- L'ingress queuing vero e proprio spiega invece i batch grandi e l'ILP gerarchico (ritardi di aggregazione e coordinamento). Usa "accodamento, in ingresso e sui nodi di bordo", come nel discorso.

### 13. Come hai scelto α=1000 e γ=10? Sono robusti o ottimizzati su un solo scenario?

- Ricerca in due stadi: prima BS×BT (28 configurazioni), poi α∈{10, 50, 1000} e γ∈{2, ..., 10000}, sotto stress (λ=8 e 10), poi validazione su λ da 2 a 10.
- Gli effetti sono strutturali: con γ ≥ 1000 il solver allunga i percorsi per evitare le eclissi (tempo di risposta +55%); con γ tra 6 e 10 ci sono zero drop da eclissi, e γ=8 è equivalente. α=1000 aiuta solo in modalità Time con small-batching; il fast-dispatch è insensibile.
- Il peso va letto in relazione a P_drop=10^6, che garantisce che si rifiuti solo quando serve.
- Limite: calibrazione e validazione usano lo stesso simulatore e lo stesso scenario, quindi con un'altra costellazione i pesi andrebbero ricalibrati.

### 14. ⚠ Nel data-intensive vince l'ILP gerarchico e non il tuo centralizzato (81-89% contro 59-68%): perché?

- Il collo di bottiglia sono i link e l'energia dei transceiver (oltre il 95% degli scarti è "CPU+NET"), non le code. A 40 kJ i centralizzati soffrono di esaurimento anticipato della batteria (48-63%).
- Ipotesi (dillo come ipotesi, perché la tesi non isola la causa): la gestione dei link a livello di cluster è più efficace, mentre il tuo centralizzato con |R| ≤ 2 non ha un orizzonte abbastanza ampio per bilanciare l'uso dei link.
- Collega la risposta agli sviluppi futuri: l'architettura a due livelli manda i task data-intensive a un livello ILP.

---

## Possibili (15-20)

### 15. Cosa succede se il Master si guasta? Come mitigheresti il single point of failure?

- È un limite dichiarato (§6.2.2): radiazioni, riavvii, manovre di collision avoidance bloccano l'intera pipeline. Nella simulazione i guasti del Master non sono modellati.
- Mitigazioni: Master di riserva con elezione e replica dello stato, partizionamento regionale, fallback sull'euristica a latenza zero (l'architettura ibrida a due livelli), oppure MADRL completamente distribuito, che elimina il problema alla radice.

### 16. ⚠ Come modelli telemetria obsoleta e latenza verso il Master? Dov'è il Master?

- Il Master prende uno snapshot di code, batterie e ISL al momento del flush, dopo la barriera a ritardo zero (stato coerente). La telemetria costa O(|S|) messaggi, meno dello 0,1% della banda ISL fino a 50 satelliti.
- Limite dichiarato (§6.2.3): a velocità LEO lo stato invecchia mentre viene raccolto e risolto; l'instradamento greedy di emergenza copre i collegamenti che cadono in volo.
- **Verifica:** se il simulatore modella un ritardo di raccolta o di accesso al Master. Se no, dillo e indicalo come estensione.

### 17. La soluzione scala a migliaia di satelliti, come Starlink?

- Il lavoro è regionale (|S| fino a circa 50, circa 15 candidati nell'ILP). Dijkstra costa O(|AP|·|S| log |S|) perché il grado è al massimo 4.
- Con migliaia di satelliti la telemetria e lo stato invecchiato diventano il limite (§3.6.4); la strada indicata è il partizionamento regionale o la decomposizione gerarchica.
- Il numero di variabili cresce linearmente con i candidati: servirebbe potare i candidati per ogni richiesta.

### 18. ⚠ L'effetto "latency compression" non dipende da traffico a raffiche? E con timeout di 2 s e deadline sotto il secondo, un task isolato non scade?

- Sì, dipende dai burst che riempiono il batch da 2. A basso carico (λ=2, un arrivo ogni 0,5 s) il task aspetta: latenza 0,72-0,76 s e scadenze mancate al 10-15% degli scarti. È il "batching dilemma" dei limiti.
- Rimedio: batching adattivo con BT(t) funzione di traffico, slack residuo e code (§6.3.2).
- **⚠ Prepara:** come generi gli arrivi (la tesi dà solo i tempi medi di inter-arrivo, 0,5 s e 0,1 s). Se sono Poisson, dillo; con traffico regolare o molto sparso il vantaggio dello small-batching cala.

### 19. ⚠ Perché Walker-Delta a 600 km e 53°? Quanti satelliti, quanti piani, quanti access point?

- 53° e circa 550-600 km sono parametri tipici delle shell Starlink, scelti per uno scenario realistico di costellazione regionale.
- **Prepara i numeri esatti:** la tesi non dichiara quanti satelliti ha la costellazione simulata (solo |S| ≈ 15 candidati e |S| ≤ 50 per scenari regionali), né piani e access point. Chi ascolta può chiederlo.

### 20. Sviluppi futuri: come useresti il reinforcement learning per BT? È fattibile far girare HiGHS su un SoC spaziale?

- RL per BT(t) (§6.3.2): stato = tasso di arrivo stimato, slack residuo minimo nel buffer, code di CPU; azione = regolare il timeout (o flush immediato); ricompensa = success rate meno penalità di latenza. Un contextual bandit o Q-learning è abbastanza leggero da girare sul Master.
- MADRL con CTDE: ogni satellite decide con osservazioni a un hop, per togliere il single point of failure.
- Hardware-in-the-loop: HiGHS su Cortex-R52/A53 o NOEL-V per misurare istruzioni, memoria, cache e potenza. L'istanza è piccola, ma solo la misura dirà se i tempi restano compatibili con le deadline.

---

## Definizioni da avere pronte

- **ΔD:** rilassamento della deadline (10%, 20%, 30% nello studio; 10% nella validazione finale). Verifica con il codice la definizione esatta.
- **P_drop = 10^6:** penalità di rifiuto, così il solver scarta un task solo se è infeasible.
- **Vincolo di carico:** ⌊|R|/|S|⌋ + 5 task per satellite per batch; il +5 lascia margine ai nodi migliori.
- **Penalità sunset:** Ω_i cresce quando il satellite si avvicina all'eclissi; γ pesa questo termine.
- **Cascata di diagnosi:** No Route, Deadline Violation, Sunset, Energy Exhaustion, Load Balancing, Global ILP Conflict.
- **Attenzione ai simboli:** α e β sono sia i pesi dell'obiettivo (α=1000) sia i vettori dei carichi (β = tipi di task; α = dimensione dei payload nei grafici "Alpha (0.3, 0.5, 0.2)"). Se qualcuno si confonde, chiarisci subito quale dei due stai usando.
- **Archetipi:** fast-dispatch (BS=20, BT=0,05 s), small-batching (BS=2, BT=2 s).