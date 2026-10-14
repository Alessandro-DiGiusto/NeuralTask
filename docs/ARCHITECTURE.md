# Architettura di NeuralTask™

Documento pubblico. Descrive **come è strutturato il sistema**, non come è implementato.
Nessun dettaglio proprietario, nessun codice.

---

## 1. Principio guida

> **L'app deve funzionare in aereo.**

Ogni decisione architetturale discende da questa frase. Non è una posizione ideologica sulla privacy: è un vincolo di prodotto.

Un'app che pianifica la tua giornata non può dipendere dalla rete, perché la usi nei momenti in cui la rete non c'è — in metropolitana, in aereo, in montagna, in un seminterrato. E non può mandare i tuoi progetti, le tue abitudini e i tuoi spostamenti a un server che non controlli.

Da qui: **locale per default, cloud come eccezione opzionale e consensuale.**

---

## 2. Strati

```
┌─────────────────────────────────────────────────────────┐
│  INTERFACCIA                                            │
│  SwiftUI · il modello dati è la fonte di verità delle   │
│  view, non uno stato duplicato in memoria               │
├─────────────────────────────────────────────────────────┤
│  INTELLIGENZA                                           │
│  parsing deterministico · modelli on-device · LLM       │
│  di sistema · escalation cloud opzionale                │
├─────────────────────────────────────────────────────────┤
│  DOMINIO E PERSISTENZA                                  │
│  19 entità · regole di pianificazione · economia        │
│  interna · apprendimento dal comportamento              │
├─────────────────────────────────────────────────────────┤
│  INTEGRAZIONI DI SISTEMA                                │
│  Calendario · Promemoria · Posizione · Mappe ·          │
│  Widget · Live Activity · Notifiche · Attività in       │
│  background · Salute · Haptics · Voce                   │
└─────────────────────────────────────────────────────────┘
```

**Il vincolo strutturale:** lo strato di intelligenza non ha accesso diretto alla persistenza. Produce **proposte**, non scritture. Solo lo strato di dominio decide cosa diventa dato — ed è l'unico punto in cui i dati cambiano.

Questo è ciò che rende possibile la regola §3.1.

---

## 3. Come è organizzata l'intelligenza

### 3.1 Determinismo dove serve, AI dove serve

Il sistema è costruito su una separazione netta:

| Tipo di problema | Chi lo risolve |
|---|---|
| Capire **cosa vuole** l'utente | Modello linguistico |
| Decidere **quando** e **con che ricorrenza** | Parsing deterministico |
| Scegliere **in quale slot** metterlo | Algoritmo di pianificazione |
| Capire **come sta andando** | Analisi statistica locale |

Un modello linguistico è eccellente nel primo caso e inaffidabile nel secondo e nel terzo. Chiedergli di compilare campi strutturati significa accettare che prima o poi li inventi — e una routine sbagliata è peggio di nessuna routine.

**Regola:** i campi strutturati hanno una sola fonte di verità, e non è un modello probabilistico.

### 3.2 Tre livelli, dal più privato al più capace

1. **Modello proprietario on-device** (Core ML, Neural Engine). Generazione autoregressiva con tokenizer dedicato. Sempre disponibile, mai connesso.
2. **LLM di sistema** (`FoundationModels`). Per i ragionamenti che il modello compatto non copre. Sempre on-device.
3. **Cloud opzionale.** Attivabile solo con consenso esplicito ed esplicito per l'uso, con sanitizzazione prima dell'invio.

Il sistema **verifica la disponibilità di ogni livello prima di usarlo** e degrada con grazia verso il basso. Il rifiuto del cloud non rompe nessuna funzionalità critica: rimuove un'opzione, non un percorso.

### 3.3 La pianificazione

Il pianificatore è un algoritmo, non un modello. Riceve:
- le finestre libere reali, lette dal calendario di sistema
- la durata stimata di ogni task
- il profilo comportamentale dell'utente (quando rende meglio, cosa tende a rimandare)
- i vincoli rigidi (impegni con orario fisso, spostamenti)

e produce un'assegnazione task → slot che rispetta i vincoli e massimizza la probabilità di completamento.

Il risultato è **deterministico e ispezionabile**: lo stesso input dà lo stesso piano, e si può spiegare perché un task è finito lì. Un LLM non darebbe né l'uno né l'altro.

---

## 4. Il modello dati

19 entità, organizzate per dominî:

- **Progetto** → obiettivi → macro-task → micro-task (gerarchia, non lista piatta)
- **Comportamento**: profilo attentivo, tendenza alla procrastinazione, preferenze di aiuto
- **Pianificazione**: slot occupati, finestre libere, spostamenti, promemoria
- **Conversazione**: storico del coach e contesto attivo
- **Economia**: saldo e transazioni della moneta interna
- **Sistema**: preferenze, consensi, stato dei permessi

**Regola di migrazione:** ogni campo introdotto dopo il primo rilascio è **opzionale**, con un fallback in lettura. Un'app con dati degli utenti sul dispositivo non può permettersi una migrazione distruttiva.

---

## 5. Integrazioni di sistema

| Integrazione | Uso |
|---|---|
| **Calendario** | Lettura delle finestre libere, scrittura dei task pianificati. Bidirezionale. |
| **Promemoria** | Sincronizzazione in due direzioni, con riconciliazione quando l'utente modifica fuori dall'app. |
| **Posizione** | Calcolo degli spostamenti e rilevamento di quando sei a casa. |
| **Mappe** | Stima del tempo di percorrenza sul **traffico reale**, per il promemoria di partenza. |
| **Notifiche** | Canali separati per categoria, con azioni rapide e un contratto di azionabilità. |
| **Widget / Live Activity** | Il piano visibile fuori dall'app, aggiornato in tempo reale. |
| **Attività in background** | Rivalutazione periodica degli spostamenti, ottimizzazione notturna. |
| **Salute, Movimento, Haptics, Voce** | Segnali di contesto (energia, attività) e input. |

### Il principio local-first sulle integrazioni

**Nessuna integrazione è un prerequisito.** Se un permesso manca:

- il calendario interno funziona comunque, con le sue date
- la scrittura su Calendario Apple viene ritentata automaticamente quando il permesso arriva
- le funzionalità che dipendono dal permesso semplicemente non si attivano, senza rompere il resto

Il caso peggiore non è "l'app non funziona": è "una funzione in meno".

---

## 6. Backend

Il backend esiste per ciò che **non può** stare sul dispositivo:

- identità dell'utente e continuità tra dispositivi
- autorità sull'economia interna (il saldo non può essere locale, o sarebbe falsificabile)
- verifica lato server degli acquisti in app
- relay per il livello AI cloud opzionale

Il resto — cioè quasi tutto — resta on-device.

**Sicurezza:** attestazione del dispositivo su ogni chiamata, nessuna fiducia nel client per le decisioni economiche, infrastruttura in regione europea.

**Degrado:** se il backend non risponde, l'app continua a funzionare in locale. Le operazioni che richiedono autorità restano in coda invece di fallire in modo silenzioso.

---

## 7. Qualità

- **83 test automatici** tra unit test e test di interfaccia, sui percorsi critici: pianificazione, completamento, notifiche, migrazioni.
- **Documentazione come artefatto di prima classe:** 675 KB in 19 documenti. Il documento di architettura dichiara per ogni componente lo stato reale — implementato, parziale, dormiente — e viene corretto quando il codice lo smentisce.
- **Regola di verità runtime:** quando documento e codice sono in disaccordo, ha ragione il codice, e il documento va riscritto. Non il contrario.

---

## 8. Cosa non è in questo documento

Per scelta, non sono descritti: gli algoritmi di pianificazione, il modello AI e il suo addestramento, lo schema del database, il protocollo di comunicazione con il backend, le regole anti-abuso dell'economia interna, e le scelte di design visivo.

Non per segretezza fine a se stessa, ma perché sono **il prodotto**. Quello che resta qui è il *come è pensato*, che è la parte che si può raccontare senza regalare il lavoro.

---

<div align="center">
<sub>© 2026 Alessandro Di Giusto · NeuralTask™ è un prodotto proprietario.</sub>
</div>
