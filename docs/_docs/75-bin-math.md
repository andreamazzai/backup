---
title: "Aritmetica binaria"
permalink: /docs/math/
excerpt: "Aritmetica binaria"
---

## Tassonomia: Flag per parlare di registro o modulo, flag per parlare del singolo flag

[![Registro dei Flag del BEAM](../../assets/flags/30-flag-beam.png "Registro dei Flag del BEAM"){:width="100%"}](../../assets/flags/30-flag-beam.png)

Bisogna dire che più volte, leggendo la documentazione di Tom Nisbet, ho trovato delle idee molto clever; alcune di queste si trovano nel modulo dei Flag. Cercherò di riportare anche alcune fonti utilizzate da Tom indicandone i collegamenti alla fine della pagina.

[![Schema logico del modulo Flag di Tom Nisbet](../../assets/flags/30-flag-nqsap-schematics.png "Schema logico del modulo Flag di Tom Nisbet"){:width="100%"}](../../assets/flags/30-flag-nqsap-schematics.png)

*Schema logico del modulo Flag di Tom Nisbet, leggermente modificato al solo scopo di migliorarne la leggibilità.*

Il registro dei Flag dell'NQSAP emula i 4 flag **NVZC** del 6502:

- **N**egative (N)
- O**V**erflow (V)
- **Z**ero (Z)
- **C**arry (C)

E' completamente differente dal semplice registro dei Flag del computer SAP di Ben Eater, nel quale un unico Flip-Flop [74LS173](https://www.ti.com/lit/ds/sdls067a/sdls067a.pdf) memorizzava i soli 2 flag C e Z nello stesso momento: la gestione delle istruzioni necessitava di 4 set di microcode, cioè uno per ogni combinazione dei segnali di flag portati agli ingressi delle EEPROM; ogni set di microcode era infatti sviluppato su misura per attivare in output i corretti segnali per la gestione di C e/o Z. Questo è ben spiegato nel video di Ben Eater [Conditional jump instructions](https://www.youtube.com/watch?v=Zg1NdPKoosU).

Nell'approccio di Tom il microcode delle istruzioni non varia a seconda dello stato dei flag, che non sono più direttamente connessi agli indirizzi delle ROM che poi attivano diversi segnali di output in base all'indirizzo/flag presentato in ingresso!

Analizziamo ad esempio un'istruzione di salto condizionale legata al flag Z:

- il microcode dell'istruzione di salto attiva un segnale "Jump Enable" connesso al pin 7 del Selector/Multiplexer [74LS151](https://www.ti.com/lit/ds/symlink/sn54s151.pdf) visibile in basso a destra nello schema generale;

- importantissimo comprendere che la selezione del flag da mettere in uscita dipende dalla codifica dell'istruzione in esecuzione, poiché i 3 bit Select S2, S1 ed S0 del '151 sono direttamente collegati all'Instruction Register, cioè *hardwired* in maniera similare a quanto succede per la ALU;

 [![Output dell'IR verso il modulo Flag con evidenza dei 3 bit di selezione dell'istruzione di salto condizionale](../../assets/flags/30-flag-cl-ir-out.png "Output dell'Instruction Register verso il modulo Flag con evidenza dei 3 bit di selezione dell'istruzione di salto condizionale"){:width="50%"}](../../assets/flags/30-flag-cl-ir-out.png)

*Output dell'Instruction Register (IR) verso il modulo Flag con evidenza dei 3 bit di selezione dell'istruzione di salto condizionale.*

[![Ingressi di selezione dell'istruzione di salto condizionale del registro dei Flag e connessione "hardwired" con l'IR](../../assets/flags/30-flag-select-in.png "Ingressi di selezione dell'istruzione di salto condizionale del registro dei Flag e connessione "hardwired" con l'IR"){:width="50%"}](../../assets/flags/30-flag-select-in.png)

*Ingressi di selezione dell'istruzione di salto condizionale del registro dei Flag e connessione "hardwired" con l'IR.*

- se per esempio l'istruzione *Jump on Zero* è codificata come 010 sui 3 segnali S2, S1 ed S0 comuni tra IR e registro dei Flag, questa andrà ad attivare il pin I2 di ingresso del '151 che, se troverà 1 al suo ingresso (vale a dire che l'uscita Q del Flip-Flop del flag Zero ha valore logico HI), andrà ad abilitare il segnale /PC-LOAD sul Program Counter (**PC**), attivando il caricamento del nuovo indirizzo calcolato a partire dal valore dell'operando dell'istruzione di salto.

![Selector/Multiplexer 74LS151](../../assets/flags/30-flag-151-table.png){:width="33%"}

*Tabella funzioni Selector/Multiplexer 74LS151.*

Detto in altre parole: la logica del salto condizionale del SAP era implementata nel microcode, utilizzando linee di indirizzamento delle ROM. Poiché i flag dell'NQSAP sono invece implementati in hardware, non c'è bisogno di consumare preziose linee di indirizzamento delle EEPROM (e di raddoppiare la dimensione del microcode ad ogni aggiunta di un flag). 

I miglioramenti derivanti da questa architettura sono:

- possibilità di settare i flag anche singolarmente;
- risparmio di linee di indirizzamento delle EEPROM;
- l'output delle EEPROM non si modifica durante l'esecuzione della singola istruzione (**ma nel SAP-1 come si comportava? 04/10/2022 l'ho compreso andando a rileggere gli appunti** del BE 8 bit computer). Teoricamente, e l'avevo letto anche altrove, questo potrebbe essere un problema perché causa "glitching".

## Componenti e funzionamento

Un multiplexer (MUX) [74LS157](https://www.ti.com/lit/ds/symlink/sn74ls157.pdf) prende in input i valori dei flag V, Z e C selezionandone la provenienza:

1. dal bus; quando il '157 legge dal bus, è possibile caricare i registri dei flag leggendo valori arbitrari dalla memoria del computer (similarmente al funzionamento dell'istruzione Pull Processor Status **PLP** del 6502; inoltre, come nell'architettura del 6502, una parte della memoria del computer è dedicata allo **Stack**);

2. computandoli opportunamente:
    - **C** attraverso un Data Selector / Multiplexer '151 che permette di selezionare la sorgente del Carry;
    - **Z** come risultato del comparatore [74LS688](https://www.ti.com/lit/ds/symlink/sn74ls688.pdf);
    - **V** attraverso un altro '151 che permette di ricreare la funzione logica dell'Overflow per verificare un eventuale cambio di segno nel risultato delle operazioni di somma o sottrazione con segno; **questo aspetto verrà ulteriormente evidenziato** nella sezione apposita dedicato alla comprensione dell'Overflow.

Il flag **N**egative viene letto direttamente dalla linea D7 del bus e caricato su uno dei due Flip-Flop disponibili in un [74LS74](https://www.ti.com/lit/ds/symlink/sn54ls74a.pdf). NB: il flag viene gestito sulla linea D7 del bus in quanto i numeri Signed utilizzano proprio l'MSB per indicare il proprio segno.

V, Z e C escono dal MUX '157 e sono presentati ad altrettanti Flip-Flop.

Quattro porte AND permettono il caricamento dei FF in presenza del segnale di clock e della contemporanea attivazione degli opportuni segnali F**N**, F**V**, F**Z** ed F**C** provenienti dalla CL (il caricamento dei registri viene sempre effettuato durante il Rising Edge del Clock). Ogni istruzione, grazie alla personalizzazione del microcode, può settare anche più di un flag alla volta (come accade ad esempio per l'operazione ADC, che sul 6502 influisce contemporaneamente su tutti i 4 flag **NVZC**).

![Caricamento dei Flip-Flop C e Z con segnali Clock AND FC / FZ](../../assets/flags/30-flag-ff-cz.png){:width="50%"}

*Caricamento dei Flip-Flop C e Z con segnali Clock AND FC / FZ.*

- Si noti che i FF non vengono mai pre-settati, pertanto /Preset resta fisso a Vcc (e dunque mai attivo); è presente invece una connessione al segnale di reset generale del computer (/RST).

- Le uscite dei FF sono connesse a un '151 per poter gestire, come si vedrà in seguito in questa pagina, i salti condizionali.

- Il flag C viene esportato anche verso i 74LS181 e il registro H del modulo ALU. **Ulteriori dettagli a seguire**.

Un transceiver 74LS245 permette infine di esportare i 4 flag NVZC sul bus per salvarli in memoria, o più precisamente nello Stack, similarmente a quanto accade nel 6502 con l'istruzione Push Processor Status (PHP).

Interessante notare che le istruzioni CLC, CLV e SEC non hanno bisogno di segnali dedicati della CL per settare od azzerare i flag C e V: si utilizza la ALU per mettere 0 o 1 sul bus e si modifica il solo flag di interesse attivando opportunamente il segnale di controllo FC o FV.

## I salti condizionali

Ogni variazione di un flag nel computer SAP di Ben Eater generava una variazione degli indirizzi delle EEPROM, così da poter presentare una logica opportunamente diversa in uscita in conseguenza delle diverse combinazioni degli stati dei flag.

L'approccio dell'NQSAP è molto diverso, in quanto tutti i segnali dei flag (presenti sulle uscite Q e /Q dei FF '74) vengono presentati a un '151; la sua funzione è quella di selezionare uno dei flag da mettere sulla sua uscita per eventualmente permettere l'attivazione del segnale /PC-LOAD, che abilita il caricamento del contenuto del bus sul PC (e dunque i salti). E' sufficiente infatti caricare un nuovo valore nel PC perché questo diventi il nuovo indirizzo a partire dal quale saranno caricate ed eseguite (fetch/execute) le successive istruzioni del programma caricato nella memoria del computer.  

Come avviene la selezione del flag da portare all'uscita Z del '151? I segnali IR-Q5 (S0), IR-Q6 (S1) ed IR-Q7 (S2) provenienti dall'IR selezionano quale tra gli input del '151 (I0-I7) si debba portare sull'output Z, come anticipato precedentemente nella *Tabella funzioni Selector/Multiplexer 74LS151*. IR-Q5, Q6 e Q7 sono infatti *hardwired* con l'IR: le istruzioni di branch, ognuna con la sua codifica specifica, determinano quale ingresso Ix del '151 sarà attivato ed esposto in output; a seconda dello stato del flag, /PC-LOAD sarà eventualmente attivo e l'istruzione di salto relativo potrà essere eseguita.

Prendiamo come esempio l'istruzione BCS (Branch on Carry Set) ipotizzando che l'istruzione precedente abbia generato un Carry e che dunque il corrispondente FF presenti lo stato logico HI sull'uscita Q:

[![Esempio istruzione Branch on Carry Set](../../assets/flags/30-flag-bcs.png "Esempio istruzione Branch on Carry Set"){:width="50%"}](../../assets/flags/30-flag-bcs.png)

*Esempio istruzione Branch on Carry Set.*

- l'istruzione sarà codificata nel microcode in modo che S0, S1 ed S2 selezionino l'ingresso I0, che è collegato al FF del Carry:

  - S0 = LO
  - S1 = LO
  - S2 = LO

- poiché il Carry è attivo, l'output della NOR connessa all'uscita Z è certamente LO (/PC-LOAD = NOT (1+x) = 0), pertanto il valore presente nel bus viene caricato nel PC (il segnale /PC-LOAD è attivo LO).

Un aspetto che inizialmente sfuggiva alla mia comprensione era come poter includere le istruzioni di branch (8 combinazioni = 3 bit), le istruzioni dell'ALU (5 bit) e tutte le altre istruzioni (ad esempio caricamento, trasferimento) in soli 8 bit: come era possibile gestire tutte le combinazioni e costruire una matrice di istruzioni? Nel momento un cui eseguivo una qualsiasi altra istruzione, cosa sarebbe successo nella gestione dei salti relativi, visto che erano direttamente funzione della codifica dell'istruzione correntemente in esecuzione? Non rischiavo di eseguire un salto condizionale non voluto se mi fossi trovato ad esempio in una situazione in cui (come descritto poco sopra) IR-Q5, Q6 e Q7 fossero LO e il Carry fosse attivo?

In seguito avevo notato che nel '151 addetto alla selezione dei flag era presente anche un segnale di Enable JE, presente nel microcode delle sole istruzioni di salto condizionale, pertanto:

- se JE è HI **e** se l'output del FF selezionato è HI, l'uscita del '151 sarà essa stessa HI, permettendo il caricamento del Program Counter;

- se JE è LO, l'uscita del '151 sarà disattivata, pertanto nessun segnale di caricamento verrà inviato al Program Counter.

Dunque le istruzioni - e solo quelle - il cui microcode comprendeva il segnale JE erano considerate istruzioni di salto condizionale; tutte le altre ignoravano il salto, in quanto il segnale /PC-LOAD non sarebbe mai stato attivato.

Tom evidenziava che "*questo metodo semplifica il microcode, perché tutte le operazioni di salto utilizzeranno lo stesso microcode*".

Perché tutte le istruzioni di salto dovrebbero essere "uguali"? La spiegazione, semplice solo dopo averla ben compresa, sta nel fatto che la scelta del flag da utilizzare per il salto condizionale non dipende più dal microcode dell'istruzione, bensì dalla codifica dell'istruzione stessa: essendo *hardwired* con l'IR, i segnali IR-Q5, Q6 e Q7 vengono automaticamente applicati agli ingressi Select del '151; è nella realizzazione del set di istruzioni che si deve tenere conto di quale codifica associare alle varie istruzioni BCS, BCC, BNE, BPL eccetera. Nessuna variazione è richiesta nel microcode.

Prendiamo come ulteriore esempio l'istruzione BVC (Branch on OVerflow Clear) ipotizzando che non ci sia Overflow e che dunque il segnale /V sia attivo:

[![Esempio istruzione Branch on OVerflow Clear](../../assets/flags/30-flag-bvc.png "Esempio istruzione Branch on OVerflow Clear"){:width="50%"}](../../assets/flags/30-flag-bvc.png)

*Esempio istruzione Branch on OVerflow Clear.*

- l'istruzione dovrà essere codificata nel microcode in modo che S0, S1 ed S2 selezionino l'ingresso I5, che è collegato all'uscita /Q del FF dell'Overflow:

  - S0 = HI
  - S1 = LO
  - S2 = HI

Tenendo ora in considerazione l'esistenza del segnale Jump Enable, evidenziato in giallo nello schema precedente, si evince che l'attivazione di /PC-LOAD e una conseguente esecuzione condizionale di un salto determinata dalla eventuale assenza di Overflow richiedono un'istruzione:

- la cui codifica porti ad avere S2/S1/S0 = 101 agli ingressi di selezione del '151, **e**
- il cui microcode attivi il segnale JE.

A questo punto della spiegazione si sarà notato che i FF '74 espongono i flag sia in logica normale, sia in logica invertita, con lo scopo di poter più facilmente determinare se un determinato flag sia attivo o no; se ad esempio il Carry non fosse presente e si desiderasse eseguire un salto verificando la condizione "Carry non presente" (Branch on Carry Clear, BCC), sarebbe più semplice verificare se l'inverso del Carry fosse presente, così da attivare opportunamente il segnale di salto /PC-LOAD.

L'utilizzo di una NOR all'uscita Z del '151 permette di gestire sia i salti condizionali (dunque da validare con una apposita verifica logica, cioè quella della presenza / assenza di un determinato flag), sia i salti incondizionati:

- In caso di salto condizionale (BCS, BVC, BEQ eccetera), una verifica positiva di presenza / assenza del flag selezionato (normale o invertito) genera un'uscita HI sul '151 --> la NOR presenta dunque output LO ed attiva il caricamento sul Program Counter del valore presente sul bus (che altri non è che l'operando dell'istruzione di branch condizionale).

  - Notare che l'operando delle istruzioni di salto condizionale è un valore relativo, che viene addizionato al valore attuale del PC. **Long Story Short: carico in D il PC attuale, mentre carico in X l'operando dell'istruzione di salto; il nuovo valore da caricare nel PC corissponde all'indirizzo del byte successivo a quello dell'operando dell'istruzione Branch + l'operando stesso**, ad esempio:

- In caso di salto incondizionato (JMP, JSR), il microcode dell'istruzione di salto attiverà /WP (in logica invertita) che a sua volta attiverà il caricamento sul PC del valore presente sul bus: (/PC-LOAD = NOT (x+1) = 0), pertanto il PC caricherà dal bus il suo nuovo valore.

![NOR per l'attivazione di /PC-LOAD con salti condizionali ed incondizionati](../../assets/flags/30-flag-je-wp.png){:width="50%"}

*NOR per l'attivazione di /PC-LOAD con salti condizionali ed incondizionati.*

In definitiva, il microcode delle istruzioni di salto condizionato e di salto incondizionato sono molto simili e differiscono per i segnali di controllo:

- JE per attivare il '151 di selezione del flag del quale verificare presenza / assenza;
- WP per caricare senza altri controlli un determinato indirizzo sul PC.

Vi è un effetto non desiderato: "le istruzioni di salto condizionato non eseguite sprecano cicli di clock"… non si potrebbe semplicemente usare N per terminare anticipatamente l'istruzione? Lui sembra renderla un po' complicata

**29/01/2023 leggendo bene dice che dovrebbe essere possibile fare in modo che la logica elettronica dell'istruzione Jump vada ad attivare N se il salto non deve esserci… da verificare**

## Calcolo dei Flag - Utilizzo del Carry da ALU e H

### Negative

Come detto precedentemente, il flag **N**egative è semplicemente il MSB del bus 😁. Interessante notare che essendo N mappato sul bus e non direttamente sull'ALU, è possibile rilevare un numero Negative anche in contesti esterni ai '181, ad esempio nel risultato di una rotazione fatta con lo Shift-Register o in un trasferimento di dato da un registro a un altro: tutto ciò che transita sul bus può essere oggetto di verifica del suo stato di numero Signed.

### Zero

Il flag **Z**ero si attiva se il valore presente nel bus è zero; invece di usare una serie di AND per verificare se tutte le linee siano LO (come nel SAP), un singolo comparatore '688 può svolgere lo stesso lavoro. Anche questo opera sul bus e non sulla sola ALU.

![Comparatore 74LS688 per verifica dello stato zero del bus](../../assets/flags/30-flag-688.png){:width="50%"}

*Comparatore 74LS688 per verifica dello stato zero del bus.*

### Overflow

- The Overflow flag is calculated using a 74LS151 8-to-1 selector as described in "74181 with V_Flag" on 6502.org http://6502.org/users/dieter/v_flag/v_4.htm.
- The MSB of the ALU operands in H and B, as well as the MSB of the ALU result, are used as inputs.
  - 30/10/2022 Dunque, come sto iniziando a capire un pochino, l'overflow è un calcolo molto semplice e preciso di bit… 
- Two of the ALU operation select lines are used to differentiate between addition and subtraction. 06/01/2023 ho capito il riferimento:
  - IR-Q1 HI e IR-Q3 LO = addizione
  - IR-Q1 LO e IR-Q3 HI = sottrazione
  - 20/06/2023 attenzione anche qui alla congruenza tra istruzioni e collegamenti

  - 23/10/2022 oggi ho approfondito l'Overflow: se nella somma di due numeri signed noto un cambiamento di segno, allora ho un overflow
  - però l'NQSAP non lavora in complemento di due, dunque attenzione a cosa diciamo… qui non mi sembra di poter applicare il caso precedente… 27/11/2022 e in effetti rileggendo la questione è che stiamo lavorando non in complemento di due, ma con numeri signed… 06/01/2023 rileggendo ulteriormente direi che non è proprio corretto. Il complemento di 2 è semplicemente il modo di rappresentare i numeri signed, dove MSB = LO indica numero positivo e MSB = HI indica numero negativo.

Overflow

- Nel Flags Register ho un Carry Input che scrivo nel registro C e che può derivare da diverse  operazioni:
  - per i calcoli matematici corrisponde al Carry Output dell'ALU '181.
  - per le operazioni Shift, si tratta dell'LSB (pin H-Q0) o MSB (pin H-Q7) del registro H.
    - Verificare se MSB = risultato di Shift Left o Right
    - Un Multiplexer / Data Selector '151, a seconda dei suoi input C0 e C1, determina la sorgente del Carry:
    - ALU (sia normale sia invertito), oppure
      - MSB del registro H, oppure
      - LSB del registro H.

- Il valore in input del registro Carry Input dipende anche dall'istruzione che è stata eseguita: può arrivare da ALU o da H, CLC e SEC. Vedi tabella per l'uso del Carry nelle varie situazioni:

- Suppongo che il significato sia:
  - Se il registro sorgente dell'operazione è l'ALU
  - per istruzioni somma/sottrazione passo il Carry esistente negato
	                ○ per istruzioni INC o DEC passo il Carry "normale"
	        • Se il registro sorgente dell'operazione è H (usato per le varie rotazioni) prendo MSB per rotazione a sinistra e LSB per rotazione a destra… ma questo non mi convince… 26/09/2022 ma ora che ci penso mi pare ok: prendo il MSB  e poi faccio shift a sinistra, dunque "salvo" il MSB e viceversa quando faccio shift a destra
	
	L'ultimo caso perché noi pensiamo in logica positiva col Carry che, se presente come conseguenza del risultato dell'operazione, è HI per l'addizione e LO per il prestito, come nel 6502, mentre la ALU '181 lavora in logica negativa, con LO che indica che il Carry  è presente nell'addizione e con HI che indica che c'è un prestito nella sottrazione.
	        • ma non mi è chiaro… mi pare che lavori in entrambi i modi a seconda degli input che le vengono passati. 06/10/2022 credo di aver capito. Praticamente il 181 nella modalità High-Active Data utilizza HI per indicare un Carry assente e LO per indicare il Carry presente, come vedo nel datasheet…
	        • Però poi non mi è chiaro davvero cosa significa che la ALU lavora in logica positiva o negativa… perché anche gli input sono in logica negativa, ma un semplice esercizio sul quaderno cercando di invertire tutto non mi ha dato risultato…
	        • Dunque bisogna provare a fare un circuito 😊 per capire
	
	• Come già detto, i flag possono essere anche letti (PLP) e scritti (PSP) dal / verso il bus.
	
	Carry Input
	
	
	Il '151 opera così, cioè a seconda degli input S0, S1 ed S2 seleziono cosa portare in uscita da I0 a I7:
	
	
	
	• Oltre a essere usato dal 151 per i salti condizionali, il Carry in uscita, cioè Carry Output, è anche input dell'ALU e di H; in questo ultimo caso, a seconda dell'istruzione, utilizzando CC e CS può:
	        • essere hard-coded 0
	        • essere hard-coded 1
	        • essere il valore presente nel registro C
	
	Ecco come settare il Carry Output fixed HI o LO oppure semplicemente lasciarlo passare (riferimento ai segnali dell'NQSAP).
	        • Set LO, Clear LO: Flag-In passa normale
	        • SET HI, Clear LO: Flag-Out HI
	        • SET LO, Clear HI: Flag-Out LO
	        
	Nota:
	        • Clear è LC (ALU Clear)
	        • Set è LS (ALU Set)
	        • 05/10/2022 nei vecchi schemi li chiamava LC e LS, ma questi sono diventati CC e CS perché avevo chiesto spiegazioni in quanto non mi trovavo e Tom mi ha risposto
	
	L'output del Carry a ALU e H è controllato da LC ed LS CC e CS (01/10/2022 l'autore ha aggiornato i nomi sul blog, ma non sullo schema). Questi due segnali possono semplicemente passare il Carry attuale presente in C, oppure passare HI o LO.
	Per il motivo già descritto sopra, nel caso di uso del Carry da parte dell'ALU (che lavora in logica "negativa", ma ancora da chiarire il senso, 27/11/2022) usiamo il valore invertito.
	
	
	
	Signal
	Description
	FC
	write Carry flag
	FZ
	write Zero flag
	FV
	write oVerflow flag
	FN
	write Negative flag
	FB
	load flags from the bus
	JC
	jump conditional
	C0
	carry source select 0
	C1
	carry source select 1
	LC -> CC
	ALU carry input clear
	LS -> CS
	ALU carry input set
	
	Normale:
	
	De Morgan (l'ho capito 😁):
	
	
	
• Flag e Microcode	Molte delle istruzioni modificano i flag.
	
	Per fare il microcode sto usando:
	        • https://www.masswerk.at/6502/6502_instruction_set.html
	        • https://www.masswerk.at/6502/ che è utile per simulare le istruzioni e capire quali Flag esse modifichino durante la loro esecuzione.
	        • Ad esempio inizialmente ho trovato difficoltà a capire quali Flag fossero modificati da CPY, che viene indicata come:
	                
	        • In quali combinazioni si attivano i vari flag N, Z e C?
	        • Ho trovato supporto su http://www.6502.org/tutorials/compare_beyond.html nel quale si spiega che fare un confronto equivale a settare il carry e fare la differenza, ma senza effettivamente scrivere il risultato nel registro di partenza:
	                CMP NUM
	                        is very similar to:
	                SEC
SBC NUM
	                
	        • If the Z flag is 0, then A <> NUM and BNE will branch
	        • If the Z flag is 1, then A = NUM and BEQ will branch
	        • If the C flag is 0, then A (unsigned) < NUM (unsigned) and BCC will branch
	        • If the C flag is 1, then A (unsigned) >= NUM (unsigned) and BCS will branch
	
	Facciamo le prove:
	Codice:
	        LDY #$40
	        CPY #$30
	Viene attivato il C, coerentemente con quanto spiegato sopra… direi perché nell'equivalenza si fa il SEC prima di SBC; essendo il numero da comparare inferiore, non faccio "il prestito" (borrow) del Carry e dunque alla fine dell'istruzione me lo ritrovo attivo come in partenza.
	
	
	Codice:
	        LDY #$40
	        CPY #$40
	Vengono attivati sia Z sia C: Z perché 40 - 40 = 0 e dunque il risultato è Zero e il contenuto del registro e del confronto numeri sono uguali; essendo il numero da comparare inferiore, non faccio "il prestito" (borrow) del Carry.
	
	
	
	Codice:
	        LDY #$40
	        CPY #$50
	No Z e C, coerentemente con quanto spiegato sopra, ma N, perché il numero risultante è negativo: in 2C il primo bit è 1 ☺️. C è diventato Zero perché l'ho "preso in prestito".
	
	
	
	
	Su BEAM: LDY #$40; CPY #$30 e ottengo nessun Flag, mentre dovrei avere C.
	 La ALU presenta il COUT acceso, dunque la sua uscita è a livello logico basso. DA CAPIRE!!! Cosa volevo dire?
	
	Teoricamente dunque dovrei attivare l’ingresso di uno del 151 di Carry Input settando opportunamente i segnali C0 e C1.
	
	In conseguenza di questo, verifico sul BEAM il comportamento del Carry Out dell'ALU nei 3 casi descritti e poi modifico il microcode di conseguenza. In effetti, il comportamento non era quello desiderato da teoria e ho fatto le modifiche necessarie:
	
	        • Aggiunti i segnali C0 e C1, che non avevo ancora cablato, che permettono al 151 di scelta del Carry Input di selezionare cosa prendere in ingresso. L'ALU emette un Carry invertito (0 = Attivo), dunque, per poter settare a 1 il Flag del Carry Input, lo devo prendere in ingresso dall'ALU attraverso una NOT su uno dei 4 ingressi attivi del 151, che seleziono appunto con i segnali C0 e C1 attivando il solo C0.
	        • Ho poi incluso nel microcode anche LF, in quanto ho definito l'utilizzo di LF su tutte le istruzioni di comparazione, tranne CPX abs.
	        • Considerare anche che tipo di Carry devo iniettare nella ALU… In realtà, poiché per fare il confronto utilizzo l’istruzione SBC, devo utilizzare il normale LHHL con Carry, cioè CIN = LO, che nel microcode corrisponde ad attivare il segnale CS.
	
	Ho posizionato in uscita sul Carry dell'ALU un LED (ricordare che l'uscita è negata, dunque anodo a Vcc e catodo verso il pin del chip). Anche l’ingresso Carry è negato e dunque attivo a zero, pertanto anche qui ho un LED con anodo a Vcc e catodo sul Pin.
	
	Dopo queste modifiche, le istruzioni di comparazione sembrano funzionare correttamente.
	
	TO DO: finire http://www.6502.org/tutorials/compare_beyond.html da "In fact, many 6502 assemblers will allow BLT (Branch on Less Than) "
	
	        • Vedere bene quali istruzioni CP* hanno bisogno di LF, anche sul file XLS
	
Altre referenze Tom Nisbet per Flags	• Question for all 74ls181 alu people on reddit led to the design of the oVerflow flag.
	• How to add a decremental and incremental circuit to the ALU ? on reddit inspired the idea to drive the PC load line from the flags instead of running the flags through the microcode.
	• Opcodes and Flag decoding circuit on reddit has a different approach to conditional jumps using hardware. Instead of driving the LOAD line of the PC, the circuit sits between the Instruction Register and the ROM and conditionally jams a NOP or JMP instruction to the microcode depending on the state of the flags. One interesting part of the design is that the opcodes of the jump instructions are arranged so that the flag of interest can be determined by bits from the IR. NQSAP already did something similar with the ALU select lines, so the concept was used again for the conditional jump select lines.