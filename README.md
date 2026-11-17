# MedTransit

MedTransit gestisce le richieste di trasporto interno in un ospedale: un reparto chiede un trasporto, il sistema lo assegna a un operatore secondo priorità e turno e tiene tutti aggiornati sui cambi di stato. I dati sono completamente fittizi.

È un progetto di studio e portfolio per imparare e dimostrare un'architettura **a microservizi ed eventi con Kafka**.

## Stato

**Fase di design.** Il design doc è approvato; non c'è ancora codice. Questo README descrive il progetto previsto, non funzionalità già disponibili.

Design completo: [`docs/superpowers/specs/2026-11-17-medtransit-design.md`](docs/superpowers/specs/2026-11-17-medtransit-design.md).

## Architettura prevista

Tre servizi indipendenti, ciascuno con il proprio database e nessun codice condiviso:

- **request-service** (lato reparto): crea le richieste e ne mostra lo stato.
- **dispatch-service** (lato operatore): possiede operatori, turni e assegnazioni. Assegna la richiesta a un operatore libero e in turno; se non c'è nessuno, la richiesta resta in coda per priorità.
- **notification-service**: consuma gli eventi e produce notifiche simulate.

```text
request-service  --request.events-->  dispatch-service  --dispatch.events-->  request-service
                                                       \--> notification-service (consuma entrambi i topic)
```

## Eventi

```text
richiesta.creata -> richiesta.assegnata -> richiesta.presa-in-carico -> richiesta.completata
```

I contratti degli eventi saranno documentati in `docs/events/`.

## Cosa si vuole dimostrare

- Comunicazione tra servizi via Kafka, senza chiamate sincrone per il lavoro principale.
- Outbox pattern sui produttori.
- Idempotenza dei consumer (messaggi duplicati).
- Consumer che si ferma e riparte a metà elaborazione.
- Dead-letter topic per i messaggi non elaborabili.
- Tracciare una richiesta tra i servizi tramite `correlationId`.

## Perché Kafka e perché microservizi

Per un singolo ospedale un monolite con una coda basterebbe: questa architettura è più pesante del problema. È una scelta didattica deliberata. Il progetto serve a mostrare cosa Kafka e i microservizi richiedono davvero (consistenza eventuale, idempotenza, gestione dei guasti) e dove portano benefici (disaccoppiamento di `notification-service`, più consumer sugli stessi eventi, ordine per richiesta tramite chiave). Questa sezione verrà ampliata con le decisioni reali man mano che il codice esiste.

## Stack previsto

Java, Spring Boot, Spring Kafka, PostgreSQL, Flyway, Kafka (KRaft), Docker Compose, Testcontainers. Versioni da verificare sulla documentazione ufficiale.

## Struttura prevista

```text
med-transit-msa/
  request-service/
  dispatch-service/
  notification-service/
  docs/
  docker-compose.yml
  README.md
```

## Fuori scope della v1

Autenticazione e autorizzazione, interfaccia grafica, dati reali, produzione e Kubernetes, notifiche vere.
