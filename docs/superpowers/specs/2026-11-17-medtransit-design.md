# MedTransit: design

Data: 2026-11-17. Stato: in attesa di approvazione.

## Intento e perché

MedTransit gestisce le richieste di trasporto interno in un ospedale. Un reparto chiede un trasporto (paziente, campione, materiale), il sistema lo assegna a un operatore in base a priorità e turno e tiene aggiornati reparto e operatore a ogni cambio di stato. I dati sono completamente fittizi.

Lo scopo del progetto è **imparare e dimostrare un'architettura a microservizi ed eventi con Kafka**. È il primo progetto MSA di Fabio, orientato a Java e backend, da presentare nel portfolio. Il dominio è scelto perché Fabio lavora in ambito ospedaliero e lo conosce. Il valore sta nelle decisioni di architettura e nella loro spiegazione, non nelle funzionalità.

## Obiettivi

- Tre servizi indipendenti che comunicano via Kafka, ciascuno con il proprio database.
- Un flusso end-to-end funzionante: creazione, assegnazione, presa in carico, completamento, con notifiche.
- Dimostrare gestione corretta dei guasti: consumer che muore a metà, messaggi duplicati (idempotenza), messaggi non elaborabili (dead-letter).
- Poter tracciare una richiesta attraverso i servizi tramite `correlationId`.
- Un README che spiega architettura, eventi e **perché Kafka e perché microservizi**, incluso quando non servirebbero.

## Non-obiettivi

- Autenticazione e autorizzazione (fuori dalla v1; eventuale estensione futura con Spring Security e JWT).
- Interfaccia grafica.
- Dati reali o integrazioni con sistemi ospedalieri.
- Deployment in produzione, Kubernetes, scalabilità reale.
- Notifiche vere (email, SMS, push): sono simulate.

## Anti-obiettivi (fallimento anche se "funziona")

- Codice o database condivisi tra servizi.
- Un servizio che chiama in modo sincrono un altro per completare il proprio lavoro principale.
- Kafka usato come semplice sostituto di una chiamata REST senza beneficio spiegabile.
- Un README che descrive cosa c'è senza spiegare perché.

## Vincoli

- Java, Spring Boot, Spring Kafka, Postgres, Flyway, Docker Compose, Testcontainers.
- Monorepo: una cartella per servizio, `docker-compose.yml` e `docs/` alla radice.
- Nessun codice condiviso: ogni servizio ha il proprio modello degli eventi. I contratti sono documentati in `docs/events/`.
- Repo: `fabiozagaria/med-transit-msa`. Nome prodotto: MedTransit.
- Non esiste un monolite precedente. Il confronto "monolite contro microservizi" resta argomentato nel README, non costruito.

## Approcci considerati

| | Approccio | Pro | Contro |
|---|---|---|---|
| **A (scelto)** | Microservizi e Kafka da subito, tre servizi | Percorso più corto verso ciò che si vuole imparare | I confini si decidono senza averli provati; l'argomento "perché spezzare" resta teorico |
| B | Monolite modulare piccolo, tag `v1-monolith`, poi estrazione | Confini scoperti sul codice; confronto prima/dopo reale | Lavoro in più (pochi giorni) |
| C | Due servizi subito, `notification` dopo | Flusso end-to-end presto | Stessi rischi di A, rimandati |

Fabio ha scelto A. Raccomandavo B per ridurre il rischio sui confini; il rischio è accettato. Si mitiga fissando i confini qui sotto e documentando nel README le decisioni di confine.

## Architettura

### Servizi

**request-service** (lato reparto)
- Espone REST per creare una richiesta e consultarne lo stato.
- Possiede le richieste. Il campo stato è una **proiezione** aggiornata dagli eventi di `dispatch-service`: è eventualmente consistente, e il README lo dice esplicitamente.
- Pubblica `richiesta.creata`.
- Consuma `richiesta.assegnata`, `richiesta.presa-in-carico`, `richiesta.completata` per aggiornare lo stato.

**dispatch-service** (lato operatore)
- Possiede **operatori, turni e assegnazioni**.
- Consuma `richiesta.creata` e assegna a un operatore libero e in turno. Se non c'è nessuno, la richiesta resta in coda ordinata per priorità e, a parità, per ordine di arrivo.
- La coda viene ripescata quando un operatore completa un trasporto o inizia un turno. L'inizio turno è guidato dal tempo: uno scheduler periodico confronta i turni con un `Clock` iniettabile, in modo che i test non dipendano dall'ora di sistema.
- Espone REST per l'operatore: `presa-in-carico` e `completata` su un'assegnazione. Espone anche un'API minima di gestione di operatori e turni (o dati seed) per poter provare il sistema.
- Pubblica `richiesta.assegnata`, `richiesta.presa-in-carico`, `richiesta.completata`.
- Punto di concorrenza: più richieste per lo stesso operatore libero non devono produrre doppie assegnazioni (da risolvere con un meccanismo esplicito, scelto in implementazione, per esempio lock pessimistico o versionamento ottimistico, e testato).

**notification-service**
- Nessun dominio proprio esposto. Consuma tutti gli eventi dei due topic e produce notifiche **simulate**: righe di log più una tabella `notifiche` nel proprio database, consultabile via REST.
- Possiede il proprio registro di eventi già elaborati per l'idempotenza.

### Eventi

Flusso: `richiesta.creata` → `richiesta.assegnata` → `richiesta.presa-in-carico` → `richiesta.completata`.

- Topic: `request.events` (prodotto da request-service) e `dispatch.events` (prodotto da dispatch-service).
- Chiave del messaggio: `richiestaId`, per mantenere l'ordine degli eventi della stessa richiesta nella stessa partizione.
- Envelope comune (documentato in `docs/events/`, duplicato nei servizi): `eventId` (UUID), `type`, `version`, `occurredAt`, `correlationId`, `payload`. Serializzazione JSON.
- Evoluzione dei contratti: il campo `version` è presente fin dalla v1; la strategia di compatibilità è descritta in `docs/events/`.
- **Payload autosufficienti**: ogni evento porta i dati che servono ai consumer, così nessun servizio ne chiama un altro. Minimo: `richiestaId`, `reparto`, `tipo`, `priorita` in `richiesta.creata`; in più `operatoreId` e nome operatore in tutti gli eventi di `dispatch.events`. `notification-service` ricava i destinatari (reparto e operatore) solo dagli eventi.

### Affidabilità

- **Outbox pattern** sui produttori: la scrittura dello stato e dell'evento avvengono nella stessa transazione locale, un publisher separato inoltra a Kafka.
- **Idempotenza** sui consumer: ogni consumer registra gli `eventId` elaborati e ignora i duplicati. La registrazione dell'`eventId` e la scrittura di dominio avvengono **nella stessa transazione del database**: la consegna è at-least-once, l'effetto è effettivamente una volta sola. Il README spiega perché non è exactly-once.
- **Ordine e ritardi**: lo stato di una richiesta avanza solo in avanti (`CREATA → ASSEGNATA → IN_CARICO → COMPLETATA`). Un evento che porterebbe a uno stato precedente o uguale viene ignorato e registrato come elaborato.
- **Dead-letter topic** per messaggi non deserializzabili o che falliscono dopo i tentativi configurati. Un evento che riferisce una richiesta sconosciuta viene riprovato e poi va in dead-letter.
- **Outbox**: un solo publisher per servizio, che pubblica in ordine di inserimento (chiave `richiestaId`). Ogni servizio gira in una sola istanza nella v1. Con più istanze servirebbe un lock sul polling (per esempio `SKIP LOCKED`): è documentato nel README come limite noto.
- **Consumer crash**: l'offset si conferma solo dopo l'elaborazione. Il comportamento alla ripartenza è dimostrato con un test.
- **Tracciabilità**: `correlationId` creato alla ricezione della richiesta, salvato con la richiesta e poi con l'assegnazione in `dispatch-service`. Resta lo stesso per tutta la vita della richiesta, comprese le azioni dell'operatore e il ripescaggio dalla coda. Viene propagato nell'envelope e negli header e riportato in tutti i log.

### Infrastruttura

- Un'istanza Postgres in Compose con tre database separati, uno per servizio (scelta pragmatica: in produzione sarebbero istanze distinte; il README lo dice).
- Kafka in modalità KRaft (senza ZooKeeper).
- Migrazioni con Flyway, per servizio.

### Test

- Test unitari sulla logica di dominio (in particolare assegnazione e coda di `dispatch-service`).
- Test di integrazione con Testcontainers (Kafka e Postgres) per i flussi tra servizi.
- Test espliciti per: evento duplicato, consumer che si ferma e riparte, messaggio in dead-letter, doppia assegnazione concorrente.

## Struttura del repo

```
med-transit-msa/
  request-service/
  dispatch-service/
  notification-service/
  docs/
    events/           contratti degli eventi
    superpowers/specs/ questo documento
  docker-compose.yml
  README.md           architettura, eventi, perché Kafka, perché microservizi
```

## Parti decise da Fabio

- Tre servizi con le responsabilità sopra.
- L'operatore parla con `dispatch-service`, non con `request-service`.
- Approccio A: microservizi da subito, nessun monolite precedente.
- Nome: MedTransit.

## Scelte mie (da verificare)

- Stato di `request-service` come proiezione eventualmente consistente.
- Due topic per servizio produttore, chiave `richiestaId`.
- Envelope con `eventId`, `type`, `occurredAt`, `correlationId`, `version`.
- Outbox, idempotenza per `eventId`, dead-letter topic.
- Notifiche simulate con tabella e log.
- Un Postgres con tre database; Kafka in KRaft; Flyway; Testcontainers.
- JSON come formato dei messaggi.
- Autenticazione fuori dalla v1.
- Stato della richiesta solo in avanti, eventi fuori ordine ignorati.
- Idempotenza con registrazione dell'`eventId` nella stessa transazione della scrittura di dominio.
- Payload autosufficienti; `correlationId` unico per tutta la vita della richiesta.
- Una sola istanza per servizio nella v1; scheduler dei turni con `Clock` iniettabile.

## Lasciato al builder

- Versioni di Java, Spring Boot e delle librerie (da verificare con la documentazione ufficiale).
- Build tool (Maven o Gradle) e layout interno dei moduli.
- Meccanismo di concorrenza per l'assegnazione.
- Implementazione dell'outbox (polling o altro) e parametri di retry.
- Forma esatta delle API REST e dei payload.
- Modello dati dettagliato, priorità (valori e significato) e regole dei turni.
- Eventuali strumenti di osservabilità oltre al `correlationId`.
