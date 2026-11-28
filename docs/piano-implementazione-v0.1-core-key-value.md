# Piano di implementazione V0.1 — Core e Key-Value

## Scopo

Realizzare la prima versione funzionante di Qynvex come libreria Rust con un
motore Key-Value interamente in memoria. La milestone deve fornire un nucleo
minimo utilizzabile e verificabile, su cui aggiungere la persistenza in V0.2.

Il piano deriva dalla [baseline architetturale](./baseline_architetturale1.0.md),
in particolare dalle sezioni 4, 5, 6.1, 11.1, 12 e 13.1.

## Risultato atteso

Un'applicazione Rust può creare un motore, inserire una coppia chiave-valore,
leggerla, aggiornarla ed eliminarla. Il motore mantiene i dati solo durante la
vita dell'istanza: chiusura o riavvio comportano la perdita dei dati.

## Ambito

### Incluso

- Struttura minima del crate e API pubblica Rust.
- Core responsabile della creazione e del ciclo di vita del motore.
- Storage in memoria basato inizialmente su `HashMap`.
- Operazioni `insert`, `get`, `update` e `delete`.
- Errori espliciti per chiavi duplicate e chiavi inesistenti.
- Test unitari e test d'integrazione delle operazioni fondamentali.
- Documentazione essenziale per compilare e usare la libreria.

### Escluso

- WAL, snapshot, recupero e qualsiasi persistenza su disco (V0.2).
- Documenti, collezioni e metadati (V0.3).
- Indici, ricerca testuale o vettoriale.
- Query language, parser, planner ed executor.
- Server, protocollo di rete e API HTTP.
- Concorrenza ottimizzata, transazioni e benchmark prestazionali.

## Proposta per il contratto iniziale

Per mantenere la prima API semplice e coerente con gli esempi della baseline:

- Le chiavi sono stringhe UTF-8.
- I valori sono stringhe UTF-8 nella V0.1.
- `insert` fallisce se la chiave esiste già.
- `update` fallisce se la chiave non esiste.
- `insert` e `update` restituiscono `Result<(), QynvexError>`.
- `get` restituisce `Option<String>`: `None` indica una chiave assente.
- `delete` restituisce `bool`: `false` indica che la chiave non esisteva.
- I metodi accettano chiavi e valori in prestito (`&str`) e lo storage ne
  conserva copie possedute.
- Un'istanza nuova parte vuota; non esiste un metodo `open(path)` in questa
  milestone, perché suggerirebbe una persistenza non ancora implementata.

Queste scelte vanno confermate e documentate prima di stabilizzare l'API
pubblica. Se si prevede di supportare valori binari, scegliere `Vec<u8>` prima
di pubblicare la prima versione, anziché introdurre una migrazione API dopo.

## Piano di lavoro

### 1. Verificare e predisporre il crate

- Verificare lo stato del repository e la toolchain Rust disponibile.
- Se il crate non è ancora inizializzato, predisporre il package Cargo come
  libreria (`src/lib.rs`) e la struttura minima dei moduli.
- Definire il nome del crate, la versione iniziale e i comandi standard di
  build e test.
- Evitare dipendenze esterne finché le funzionalità della libreria standard
  sono sufficienti.

**Verifica:** il crate compila senza funzionalità applicative.

### 2. Definire tipi ed errori del Core

- Definire il tipo pubblico del motore (ad esempio `Qynvex`).
- Definire un errore tipizzato per almeno:
  - chiave già presente durante `insert`;
  - chiave assente durante `update`.
- Stabilire il risultato di `get` e `delete` per una chiave assente.
- Implementare `Display` e `std::error::Error` per gli errori pubblici.
- Mantenere i tipi indipendenti dai dettagli interni della `HashMap`.

**Verifica:** i chiamanti possono gestire gli errori senza dover interpretare
stringhe o conoscere lo storage.

### 3. Implementare lo storage in memoria

- Creare il modulo storage e un'implementazione memory privata al crate.
- Conservare le coppie in una `HashMap<String, String>`.
- Implementare le operazioni fondamentali preservando le semantiche definite
  nel contratto.
- Non aggiungere persistenza, serializzazione o astrazioni di backend non
  necessarie alla V0.1.

**Verifica:** test unitari dello storage coprono inserimento, lettura,
aggiornamento, eliminazione, duplicati e chiavi assenti.

### 4. Esporre l'API dal Core

- Creare il motore con un costruttore esplicito per l'uso in memoria, ad
  esempio `Qynvex::new()`.
- Delegare le operazioni dell'API allo storage senza esporre la `HashMap`.
- Usare riferimenti per gli argomenti in lettura e mutabilità esplicita per le
  scritture, evitando di introdurre sincronizzazione concorrente prematura.
- Esportare dall'interfaccia pubblica soltanto il motore e gli errori necessari.

**Verifica:** un esempio esterno al modulo storage usa il motore esclusivamente
tramite l'API pubblica.

### 5. Completare test e documentazione

- Aggiungere test d'integrazione per il flusso completo di utilizzo.
- Verificare che due istanze siano indipendenti e che un'istanza appena creata
  sia vuota.
- Verificare esplicitamente il comportamento con valori vuoti e chiavi
  inesistenti.
- Aggiungere un esempio minimo nel README o nella documentazione del crate.
- Eseguire formattazione, test e lint disponibili nel progetto.

**Verifica:** i test descrivono il contratto osservabile e passano con i comandi
standard del repository.

## Criteri di accettazione

- Un'istanza nuova può essere creata senza configurazione di percorso.
- `insert` salva una coppia e `get` restituisce il valore corrispondente.
- `insert` non sovrascrive silenziosamente una chiave già presente.
- `update` modifica una chiave esistente e segnala una chiave assente.
- `delete` rimuove il valore; una lettura successiva non lo trova.
- Chiavi e valori distinti non si sovrascrivono a vicenda.
- I casi limite definiti nel contratto hanno test automatici.
- I dati non vengono dichiarati persistenti e non sopravvivono alla chiusura
  dell'istanza/processo.
- Il crate compila, i test passano e l'API pubblica è documentata.

## Dipendenze e ordine

Il lavoro è sequenziale: il contratto di dati ed errori precede lo storage;
lo storage precede l'API del motore; l'API precede test d'integrazione ed
esempio documentato. Non sono previste dipendenze da servizi esterni o da
componenti delle milestone successive.

## Rischi e decisioni da non rimandare

- **Tipo dei valori:** `String` è coerente con l'esempio architetturale e
  mantiene semplice l'API iniziale; `Vec<u8>` evita di limitare il database a
  dati testuali. Scegliere prima di pubblicare l'API.
- **Semantica degli errori:** duplicati, chiavi mancanti e risultato di
  `delete` devono essere uniformi tra API, test e documentazione.
- **Confine con V0.2:** evitare di introdurre `open(path)` o promesse di
  durabilità nella V0.1. Il design dello storage dovrebbe restare sostituibile,
  ma senza costruire in anticipo un framework di backend.

## Definizione di completamento

La V0.1 è completata quando tutti i criteri di accettazione sono soddisfatti,
i test automatici passano e una piccola applicazione Rust può usare le
operazioni Key-Value attraverso l'API pubblica. Il risultato costituisce la
base per progettare WAL e recupero dati nella V0.2, senza includerli in questa
milestone.
