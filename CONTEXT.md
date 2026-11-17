# MedTransit — Context

Ultimo aggiornamento: 2026-11-17 (design approvato, su main, presente nel portfolio)

## Obiettivo

MedTransit gestisce le richieste di trasporto interno in un ospedale: un reparto chiede un trasporto, il sistema lo assegna a un operatore secondo priorità e turno e tiene aggiornati reparto e operatore. Dati completamente fittizi.

Scopo: imparare e dimostrare un'architettura a microservizi ed eventi con Kafka. È il primo progetto MSA di Fabio e parte dopo JobFlow.

## Stato reale verificato

- Il repository contiene LICENSE, README, AGENTS.md, CLAUDE.md, questo file e il design doc.
- **Nessun codice, nessun test, nessuna build.**
- Design doc scritto, riletto da un revisore indipendente e approvato da Fabio: `docs/superpowers/specs/2026-11-17-medtransit-design.md`.
- Tutto questo è su `main` dal 17 novembre 2026 (PR #1 mergiata con squash, commit `fd6d565`).
- Scheda Notion nel database "Progetti software" (priorità Secondaria, stato Pianificato), con lo stato verificato e la sezione sul portfolio.
- Portfolio (`fabio-zagaria-portfolio`, PR #65 mergiata il 17 novembre 2026): scheda MedTransit nella sezione Backend, dopo JobFlow e prima di AccessDesk, con stato «In progettazione», schema del flusso con tutti i passaggi tratteggiati e tecnologie marcate «(previsto)». Nessun «ultimo aggiornamento» né badge «Commit recente».

## Decisioni architetturali stabili

- Approccio A: microservizi e Kafka da subito, nessun monolite precedente. Rischio accettato: i confini si decidono senza averli provati.
- Tre servizi: `request-service`, `dispatch-service`, `notification-service`. Un database per servizio, nessun codice condiviso.
- L'operatore parla con `dispatch-service`, non con `request-service`.
- `dispatch-service` possiede operatori, turni e assegnazioni; se nessun operatore è libero la richiesta resta in coda per priorità.
- Lo stato in `request-service` è una proiezione eventualmente consistente e avanza solo in avanti.
- Topic `request.events` e `dispatch.events`, chiave `richiestaId`, envelope con `eventId`, `type`, `version`, `occurredAt`, `correlationId`, `payload`.
- Outbox sui produttori; idempotenza dei consumer con `eventId` nella stessa transazione della scrittura di dominio; dead-letter topic; `correlationId` unico per tutta la vita della richiesta.
- Una sola istanza per servizio nella v1; scheduler dei turni con `Clock` iniettabile.
- Autenticazione, interfaccia grafica e notifiche reali sono fuori dalla v1.

## Stack previsto (non ancora usato)

Java, Spring Boot, Spring Kafka, PostgreSQL (un'istanza con tre database), Flyway, Kafka in KRaft, Docker Compose, Testcontainers. Versioni da verificare sulla documentazione ufficiale.

## Lasciato al builder

Build tool, layout dei moduli, meccanismo di concorrenza per l'assegnazione, implementazione dell'outbox e retry, forma delle API REST e dei payload, modello dati, valori di priorità, regole dei turni, osservabilità oltre al `correlationId`.

## WIP / punto preciso di ripresa

Dopo la chiusura di JobFlow: invocare `writing-plans` sul design doc per ottenere il piano di implementazione, poi scrivere il codice insieme (Fabio scrive, Claude affianca).

## Metodo di lavoro

Tutor mode: prima ragionamento di Fabio, poi indizi progressivi. Evitare implementazioni complete copia-incolla salvo richiesta esplicita. Una sola attività importante in corso. Repository reale > Notion > memoria. Nessun commit o push senza autorizzazione.
