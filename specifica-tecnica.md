# Specifica Tecnica — Prototipo Gestione Noleggi Oltre Il Giardino SRL

> **Versione:** 1.9  
> **Data:** 24/06/2026  
> **Stato:** Prototipo completato  
> **Prossimo passo:** Validazione prototipo → Sviluppo Fase 0

---

## 1. Introduzione

### 1.1 Contesto

Oltre Il Giardino SRL opera nel settore del noleggio di suppellettili, arredi, luci e allestimenti per eventi privati e aziendali (matrimoni, cerimonie, fiere, convegni). L'azienda dispone di un catalogo prodotti organizzato in categorie merceologiche e di un gestionale esistente che gestisce la contabilità, i movimenti fiscali e la valorizzazione di magazzino.

Si rende necessario un applicativo web dedicato che gestisca il flusso operativo: preventivazione, ordini, movimentazione fisica della merce (DDT di carico/scarico), tracciamento stato articoli e disponibilità.

### 1.2 Scopo del documento

Il presente documento costituisce la specifica tecnica per la realizzazione di un **prototipo funzionante** dell'applicativo. Sarà utilizzato come input per:

- La fase di **design** (UI/UX, wireframe)
- La successiva **specifica HTML**
- Lo **sviluppo** del frontend e backend

### 1.3 Glossario

| Termine | Descrizione |
|---|---|
| **Gestionale** | Sistema ERP/contabile esistente che gestisce fiscalità, contabilità e valorizzazione magazzino |
| **DDT Uscita** | Documento di Trasporto per carico merce in uscita verso evento |
| **DDT Ingresso** | Documento di Trasporto per scarico merce in rientro da evento |
| **Preventivo** | Ipotesi di noleggio con impegno non vincolante per il periodo indicato |
| **Ordine** | Trasformazione del preventivo in impegno definitivo |
| **Articolo** | Prodotto a catalogo (es. "Sedia Chiavari bianca") |
| **Disponibilità** | Quantità di un articolo non impegnata in un dato intervallo di date |

---

## 2. Architettura Generale

### 2.1 Stack Tecnologico

| Componente | Tecnologia |
|---|---|
| **Frontend** | React 18+ con TypeScript |
| **State management** | React Query + Context API |
| **UI Library** | Material UI (MUI) o Ant Design |
| **Backend** | Node.js + Express + TypeScript |
| **ORM** | Prisma (TypeScript-first ORM) |
| **Database** | PostgreSQL 15+ |
| **Autenticazione** | JWT (access + refresh token) + Sessioni web |
| **Documenti** | PDF generation (PDFMake o Puppeteer) |
| **Container** | Docker + docker-compose (sviluppo) |
| **Deploy** | VPS tradizionale (Node + PM2, Nginx reverse proxy) |

### 2.2 Schema Architetturale

```
┌─────────────────────────────────────────────────────────────┐
│                     CLIENT (Browser)                         │
│  React SPA ─── React Query ─── Auth Context (JWT + Session) │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS / REST JSON
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   REVERSE PROXY (Nginx)                      │
│         HTTPS termination, static files, load balancing      │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   BACKEND (Node.js + Express)                │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐  │
│  │ Auth    │ │ Anagraf. │ │ Ordini   │ │ Integrazione   │  │
│  │ Module  │ │ Module   │ │ Module   │ │ Gestionale     │  │
│  └─────────┘ └──────────┘ └──────────┘ └────────────────┘  │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐  │
│  │ Catalogo│ │ DDT/     │ │ Calend.  │ │ Dashboard /    │  │
│  │ Prodotti│ │ Movim.   │ │ Disp.    │ │ Report         │  │
│  └─────────┘ └──────────┘ └──────────┘ └────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │ Prisma ORM
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   PostgreSQL Database                        │
│    anagrafiche, prodotti, preventivi, ordini, movimenti     │
└─────────────────────────────────────────────────────────────┘

                    INTEGRAZIONE FUTURA
┌─────────────────────────────────────────────────────────────┐
│              GESTIONALE ESISTENTE (da definire)              │
│  API REST (specifiche fornite dal vendor del gestionale)    │
│  Sincronizzazione: anagrafiche, movimenti magazzino,        │
│  ciclo attivo/passivo                                       │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Repository

- **frontend/**: `https://github.com/.../oltreilgiardino-frontend`
- **backend/**: `https://github.com/.../oltreilgiardino-backend`

### 2.4 Multi-tenancy

Architettura single-tenant con schema preparato per multi-tenancy futura:
- Tabella `tenant` con identificativo univoco
- Ogni tabella dati referenziata al `tenant_id`
- Middleware che filtra per tenant
- Attivabile in futuro senza rewriting

---

## 3. Modello Dati

### 3.1 Diagramma Entità-Relazioni (descrizione testuale)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              TENANT                                     │
│  id, nome, partita_iva, indirizzo, referente, email, telefono, attivo   │
└──────────┬──────────────────────────────────────────────────────────────┘
           │ 1
           │ *
┌──────────▼──────────────────┐       ┌───────────────────────────────────┐
│           USER              │       │           CATEGORIA               │
│  id, tenant_id, email,      │       │  id, tenant_id, nome, descrizione,│
│  password_hash, nome,       │       │  ordinamento, attivo              │
│  cognome, ruolo, attivo     │       └──────────┬────────────────────────┘
└─────────────────────────────┘                  │ 1
                                                  │ *
                                     ┌────────────▼──────────────────────┐
                                     │           PRODOTTO                │
                                     │ id, tenant_id, categoria_id,      │
                                     │ nome, codice_articolo,            │
                                     │ descrizione, quantita_totale,     │
                                     │ prezzo_giorno, prezzo_forfettario,│
                                     │ prezzo_3gg, prezzo_settimana,     │
                                     │ unita_misura, immagine_url,       │
                                     │ attivo, note                      │
                                     └────────────┬──────────────────────┘
                                                   │ 1
                                                   │ *
                          ┌────────────────────────┴──────────────────────┐
                          │               CLIENTE                        │
                          │ id, tenant_id, codice_gestionale,             │
                          │ ragione_sociale, partita_iva, cod_fiscale,    │
                          │ indirizzo_sede, città, cap, provincia,        │
                          │ paese, email, telefono, referente,            │
                          │ note, attivo, data_sincronizzazione           │
                          └────────────────────┬─────────────────────────┘
                                                │ 1
                                                │ *
                    ┌───────────────────────────┴──────────────────────┐
                    │               PREVENTIVO                        │
                    │ id, tenant_id, cliente_id, numero, anno,        │
                    │ data_emissione, data_inizio, data_fine,         │
                    │ data_scadenza, stato, subtotale, sconto_pct,    │
                    │ sconto_importo, totale, note, note_interne,     │
                    │ created_by, created_at, updated_at              │
                    └───────────────────────┬─────────────────────────┘
                                            │ 1
                                            │ *
                    ┌───────────────────────┴─────────────────────────┐
                    │         PREVENTIVO_RIGA                        │
                    │ id, preventivo_id, prodotto_id, quantita,      │
                    │ prezzo_unitario, tipo_prezzo (giorno/forfait/  │
                    │ durata), sconto_pct, sconto_importo,           │
                    │ totale_riga, note                              │
                    └───────────────────────┬─────────────────────────┘
                                            │
                                            │ (trasformazione)
                                            ▼
                    ┌────────────────────────────────────────────────┐
                    │               ORDINE                           │
                    │ id, tenant_id, preventivo_id?, cliente_id,     │
                    │ numero, anno, data_emissione,                  │
                    │ data_inizio_noleggio, data_fine_noleggio,      │
                    │ data_allestimento, data_smontaggio,            │
                    │ indirizzo_evento, citta_evento,               │
                    │ referente_nome, referente_telefono,            │
                    │ orario_allestimento, orario_smontaggio,        │
                    │ stato, subtotale, sconto_pct, sconto_importo,  │
                    │ totale, note, note_logistiche,                 │
                    │ created_by, created_at, updated_at             │
                    └───────────────────────┬─────────────────────────┘
                                            │ 1
                                            │ *
                    ┌───────────────────────┴─────────────────────────┐
                    │            ORDINE_RIGA                          │
                    │ id, ordine_id, prodotto_id, quantita,          │
                    │ prezzo_unitario, tipo_prezzo, sconto_pct,      │
                    │ sconto_importo, totale_riga, note              │
                    └───────────────────────┬─────────────────────────┘
                                            │ 1
                                            │ *
                     ┌──────────────────────┴──────────────────────┐
                     │      MOVIMENTO (DDT interno)               │
                     │ id, tenant_id, ordine_id, tipo (uscita/     │
                     │ ingresso), numero, anno,                    │
                     │ data_movimento, stato (bozza/confermato),   │
                     │ firma_consegna?, firma_ritiro?,             │
                     │ created_by, created_at, updated_at          │
                     └───────────────────────┬─────────────────────┘
                                             │ 1
                                             │ *
                    ┌────────────────────────┴─────────────────────┐
                    │         MOVIMENTO_RIGA                       │
                    │ id, movimento_id, ordine_riga_id,            │
                    │ prodotto_id, quantita,                       │
                    │ stato_destinazione (in_noleggio/reso/        │
                    │ danneggiato), quantita_danneggiata, note     │
                    └─────────────────────────────────────────────┘
```

### 3.2 Dettaglio Tabelle Principali

#### 3.2.1 tenant
- `id` UUID PK
- `nome` VARCHAR(255)
- `partita_iva` VARCHAR(20)
- `indirizzo` TEXT
- `referente` VARCHAR(255)
- `email` VARCHAR(255)
- `telefono` VARCHAR(50)
- `attivo` BOOLEAN DEFAULT true
- `created_at`, `updated_at`

#### 3.2.2 user
- `id` UUID PK
- `tenant_id` UUID FK → tenant
- `email` VARCHAR(255) UNIQUE
- `password_hash` VARCHAR(255)
- `nome` VARCHAR(100)
- `cognome` VARCHAR(100)
- `ruolo` ENUM('superadmin', 'admin', 'operatore', 'magazziniere')
- `refresh_token` TEXT (per sessioni JWT)
- `attivo` BOOLEAN DEFAULT true
- `created_at`, `updated_at`

#### 3.2.3 categoria
- `id` UUID PK
- `tenant_id` UUID FK → tenant
- `nome` VARCHAR(255)
- `descrizione` TEXT
- `ordinamento` INTEGER
- `attivo` BOOLEAN DEFAULT true
- `created_at`, `updated_at`

#### 3.2.4 prodotto
- `id` UUID PK
- `tenant_id` UUID FK → tenant
- `categoria_id` UUID FK → categoria
- `codice_articolo` VARCHAR(50) (codice interno / SKU)
- `nome` VARCHAR(255)
- `descrizione` TEXT
- `quantita_totale` INTEGER (giacenza totale di magazzino)
- `prezzo_giorno` DECIMAL(10,2) (prezzo noleggio per 1 giorno)
- `prezzo_forfettario` DECIMAL(10,2) (prezzo forfettario per periodo)
- `prezzo_3gg` DECIMAL(10,2) (prezzo per 3 giorni)
- `prezzo_settimana` DECIMAL(10,2) (prezzo settimanale)
- `unita_misura` VARCHAR(20) DEFAULT 'pezzi'
- `immagine_url` TEXT
- `attivo` BOOLEAN DEFAULT true
- `note` TEXT
- `created_at`, `updated_at`

#### 3.2.5 cliente
- `id` UUID PK
- `tenant_id` UUID FK → tenant
- `codice_gestionale` VARCHAR(50) (riferimento ID nel gestionale)
- `ragione_sociale` VARCHAR(255)
- `partita_iva` VARCHAR(20)
- `cod_fiscale` VARCHAR(20)
- `indirizzo_sede` TEXT
- `citta` VARCHAR(100)
- `cap` VARCHAR(10)
- `provincia` VARCHAR(10)
- `paese` VARCHAR(100) DEFAULT 'Italia'
- `email` VARCHAR(255)
- `telefono` VARCHAR(50)
- `referente` VARCHAR(255)
- `note` TEXT
- `attivo` BOOLEAN DEFAULT true
- `ultima_sincronizzazione` TIMESTAMP
- `created_at`, `updated_at`

#### 3.2.6 preventivo
- `id` UUID PK
- `tenant_id` UUID FK → tenant
- `cliente_id` UUID FK → cliente
- `numero` INTEGER (progressivo annuale per tenant)
- `anno` INTEGER
- `data_emissione` DATE
- `data_inizio` DATE (inizio noleggio previsto)
- `data_fine` DATE (fine noleggio previsto)
- `data_scadenza` DATE (scadenza validità preventivo)
- `stato` ENUM('bozza', 'inviato', 'accettato', 'scaduto', 'trasformato', 'archiviato')
- `subtotale` DECIMAL(10,2)
- `sconto_pct` DECIMAL(5,2) DEFAULT 0
- `sconto_importo` DECIMAL(10,2) DEFAULT 0
- `totale` DECIMAL(10,2)
- `note` TEXT (visibili al cliente)
- `note_interne` TEXT
- `created_by` UUID FK → user
- `created_at`, `updated_at`

#### 3.2.7 preventivo_riga
- `id` UUID PK
- `preventivo_id` UUID FK → preventivo
- `prodotto_id` UUID FK → prodotto
- `quantita` INTEGER
- `prezzo_unitario` DECIMAL(10,2)
- `tipo_prezzo` ENUM('giorno', 'forfait', 'durata')
- `sconto_pct` DECIMAL(5,2) DEFAULT 0
- `sconto_importo` DECIMAL(10,2) DEFAULT 0
- `totale_riga` DECIMAL(10,2)
- `note` TEXT
- `created_at`

#### 3.2.8 ordine
- `id` UUID PK
- `tenant_id` UUID FK → tenant
- `preventivo_id` UUID FK → preventivo (NULLABLE se ordine diretto)
- `cliente_id` UUID FK → cliente
- `numero` INTEGER (progressivo annuale per tenant)
- `anno` INTEGER
- `data_emissione` DATE
- `data_inizio_noleggio` DATE
- `data_fine_noleggio` DATE
- `data_allestimento` DATE
- `data_smontaggio` DATE
- `orario_allestimento` TIME
- `orario_smontaggio` TIME
- `indirizzo_evento` TEXT
- `citta_evento` VARCHAR(100)
- `referente_nome` VARCHAR(255)
- `referente_telefono` VARCHAR(50)
- `stato` ENUM('confermato', 'in_allestimento', 'in_corso', 'in_rientro', 'completato', 'annullato')
- `subtotale` DECIMAL(10,2)
- `sconto_pct` DECIMAL(5,2) DEFAULT 0
- `sconto_importo` DECIMAL(10,2) DEFAULT 0
- `totale` DECIMAL(10,2)
- `note` TEXT
- `note_logistiche` TEXT
- `created_by` UUID FK → user
- `created_at`, `updated_at`

#### 3.2.9 ordine_riga
- `id` UUID PK
- `ordine_id` UUID FK → ordine
- `prodotto_id` UUID FK → prodotto
- `quantita` INTEGER
- `prezzo_unitario` DECIMAL(10,2)
- `tipo_prezzo` ENUM('giorno', 'forfait', 'durata')
- `sconto_pct` DECIMAL(5,2) DEFAULT 0
- `sconto_importo` DECIMAL(10,2) DEFAULT 0
- `totale_riga` DECIMAL(10,2)
- `note` TEXT
- `created_at`

#### 3.2.10 movimento (DDT interno)
- `id` UUID PK
- `tenant_id` UUID FK → tenant
- `ordine_id` UUID FK → ordine
- `tipo` ENUM('uscita', 'ingresso')
- `numero` INTEGER (progressivo annuale per tenant, indipendente per tipo)
- `anno` INTEGER
- `data_movimento` TIMESTAMP
- `stato` ENUM('bozza', 'confermato')
- `firma_consegna` TEXT (base64 PNG, opzionale)
- `firma_ritiro` TEXT (base64 PNG, opzionale)
- `created_by` UUID FK → user
- `created_at`, `updated_at`

#### 3.2.11 movimento_riga
- `id` UUID PK
- `movimento_id` UUID FK → movimento
- `ordine_riga_id` UUID FK → ordine_riga
- `prodotto_id` UUID FK → prodotto
- `quantita` INTEGER
- `stato_destinazione` ENUM('in_noleggio', 'reso', 'danneggiato')
- `quantita_danneggiata` INTEGER DEFAULT 0
- `note` TEXT
- `created_at`

### 3.3 Indici Consigliati

- `prodotto(codice_articolo)` UNIQUE per tenant
- `preventivo(tenant_id, anno, numero)` UNIQUE
- `ordine(tenant_id, anno, numero)` UNIQUE
- `movimento(tenant_id, anno, numero, tipo)` UNIQUE
- `cliente(tenant_id, partita_iva)` UNIQUE
- `user(tenant_id, email)` UNIQUE
- Indici su tutte le FK per JOIN performance
- Indice su `prodotto(tenant_id, categoria_id)`
- Indice su `preventivo(cliente_id, stato)`
- Indice su `ordine(cliente_id, stato)`

---

## 4. API Design

### 4.1 Convenzioni

- **Base URL**: `/api/v1`
- **Autenticazione**: `Authorization: Bearer <jwt>` o cookie di sessione
- **Formato risposta**: JSON
- **Paginazione**: `?page=1&limit=20` — risposta con `{ data: [...], meta: { total, page, limit, totalPages } }`
- **Filtri**: `?campo=valore` o `?q=testo` per ricerca full-text
- **Ordinamento**: `?sort=campo&order=asc|desc`
- **Errori**: `{ error: { code, message, details } }`
- **Codici HTTP**: 200 (OK), 201 (Creato), 400 (Bad Request), 401 (Unauthorized), 403 (Forbidden), 404 (Not Found), 409 (Conflict), 422 (Validation Error), 500 (Server Error)

### 4.2 Endpoint Principali

#### 4.2.1 Auth

| Metodo | Path | Descrizione |
|---|---|---|
| POST | `/auth/login` | Login con email/password, restituisce JWT + imposta cookie |
| POST | `/auth/logout` | Logout, invalida refresh token |
| POST | `/auth/refresh` | Rinnova JWT con refresh token |
| GET | `/auth/me` | Profilo utente corrente |
| PUT | `/auth/password` | Cambio password |

#### 4.2.2 Anagrafiche Clienti

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/clienti` | Lista clienti (paginata, filtrabile) |
| GET | `/clienti/:id` | Dettaglio cliente |
| POST | `/clienti` | Crea nuovo cliente |
| PUT | `/clienti/:id` | Aggiorna cliente |
| DELETE | `/clienti/:id` | Disattiva cliente (logico) |
| GET | `/clienti/:id/ordini` | Storico ordini del cliente |
| GET | `/clienti/:id/preventivi` | Storico preventivi del cliente |

#### 4.2.3 Catalogo Prodotti

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/categorie` | Lista categorie |
| POST | `/categorie` | Crea categoria |
| PUT | `/categorie/:id` | Aggiorna categoria |
| DELETE | `/categorie/:id` | Disattiva categoria |
| GET | `/prodotti` | Lista prodotti (filtrabile per categoria, testo) |
| GET | `/prodotti/:id` | Dettaglio prodotto |
| POST | `/prodotti` | Crea prodotto |
| PUT | `/prodotti/:id` | Aggiorna prodotto |
| DELETE | `/prodotti/:id` | Disattiva prodotto (logico) |
| GET | `/prodotti/:id/disponibilita` | Disponibilità per intervallo date |

#### 4.2.4 Preventivi

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/preventivi` | Lista preventivi (filtrabile per stato, cliente, data) |
| GET | `/preventivi/:id` | Dettaglio preventivo con righe |
| GET | `/preventivi/:id/pdf` | Scarica PDF preventivo |
| POST | `/preventivi` | Crea preventivo (con righe) |
| PUT | `/preventivi/:id` | Aggiorna preventivo |
| PUT | `/preventivi/:id/stato` | Cambia stato (invia, archivia, ecc.) |
| POST | `/preventivi/:id/trasforma` | Trasforma preventivo in ordine |
| DELETE | `/preventivi/:id` | Elimina preventivo (solo se bozza) |

#### 4.2.5 Ordini

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/ordini` | Lista ordini (filtrabile per stato, cliente, data) |
| GET | `/ordini/:id` | Dettaglio ordine con righe |
| GET | `/ordini/:id/pdf` | Scarica PDF ordine / conferma |
| POST | `/ordini` | Crea ordine diretto (senza preventivo) |
| PUT | `/ordini/:id` | Aggiorna ordine |
| PUT | `/ordini/:id/stato` | Cambia stato ordine |
| POST | `/ordini/:id/ddt` | Crea DDT (uscita o ingresso) per questo ordine |
| DELETE | `/ordini/:id` | Annulla ordine |

#### 4.2.6 Movimenti (DDT)

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/movimenti` | Lista movimenti (filtrabile per tipo, ordine, data) |
| GET | `/movimenti/:id` | Dettaglio movimento con righe |
| POST | `/movimenti` | Crea movimento (con righe) |
| PUT | `/movimenti/:id` | Aggiorna movimento (solo bozza) |
| PUT | `/movimenti/:id/conferma` | Conferma movimento (cambio stato articoli) |
| GET | `/movimenti/:id/pdf` | Scarica PDF DDT |

#### 4.2.7 Calendario Eventi

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/disponibilita` | Matrice disponibilità per intervallo date e prodotti |
| GET | `/disponibilita/:prodottoId` | Disponibilità di un singolo prodotto per intervallo |
| GET | `/disponibilita/critica` | Prodotti con disponibilità sotto soglia (es. < 20%) |

#### 4.2.8 Dashboard

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/dashboard/riepilogo` | KPI: ordini attivi, preventivi in scadenza, impegni prossimi |
| GET | `/dashboard/preventivi-scaduti` | Preventivi scaduti/ in scadenza (per solleciti) |
| GET | `/dashboard/impegni-prossimi` | Ordini in partenza nei prossimi N giorni |
| GET | `/dashboard/disponibilita-critica` | Prodotti con disponibilità sotto soglia |

#### 4.2.9 Integrazione Gestionale (futura)

| Metodo | Path | Descrizione |
|---|---|---|
| POST | `/integrazione/sync-clienti` | Sincronizza clienti verso gestionale |
| POST | `/integrazione/sync-ordini` | Invia ordini confermati al gestionale |
| POST | `/integrazione/sync-movimenti` | Invia movimenti di magazzino al gestionale |
| GET | `/integrazione/log` | Log delle sincronizzazioni |

---

## 5. Flussi Funzionali Dettagliati

### 5.1 Flusso Preventivo → Ordine → DDT → Reso

```
1. CREAZIONE PREVENTIVO
   ─────────────────────
   Operatore seleziona cliente (o lo crea)
   → Aggiunge prodotti con quantità e periodo
   → Sistema calcola prezzi in base al tipo selezionato
     (giorno / forfait / durata)
   → Applica sconti di riga se necessario
   → Genera totale
   → Salva come BOZZA
   → Può inviare al cliente (cambia stato a INVIATO)
   → Data scadenza configurabile

2. TRASFORMAZIONE IN ORDINE
   ─────────────────────────
   Cliente accetta il preventivo
   → Operatore clicca "Trasforma in Ordine"
   → Sistema copia i dati in un nuovo ORDINE
   → Possibile applicare sconti aggiuntivi (%, importo, override)
   → Ordine creato in stato CONFERMATO
   → Preventivo passa a TRASFORMATO

3. DDT DI USCITA
   ──────────────
   In prossimità dell'evento:
   → Operatore/magazziniere crea MOVIMENTO tipo USCITA
   → Seleziona le righe dell'ordine e quantità caricate
   → Conferma il movimento
   → Stato articoli: Disponibile → In carico (transitorio)
   → Opzionale: firma digitale al momento del carico

4. CONSEGNA / ALLESTIMENTO
   ────────────────────────
   Merce trasportata all'evento e allestita
   → Ordine passa a IN_CORSO
   → Stato articoli: In carico → In noleggio

5. DDT DI INGRESSO
   ────────────────
   Termine evento / smontaggio:
   → Operatore/magazziniere crea MOVIMENTO tipo INGRESSO
   → Per ogni riga, indica quantità RESE e quantità DANNEGGIATE
   → Conferma il movimento
   → Stato articoli: In noleggio → Reso (o Danneggiato per alcune qty)
   → Ordine passa a COMPLETATO (se tutto reso)

6. RIALLINEAMENTO MAGAZZINO
   ─────────────────────────
   Periodicamente (o real-time via API):
   → I movimenti confermati vengono comunicati al gestionale
   → Il gestionale aggiorna la valorizzazione di magazzino
```

### 5.2 Gestione Prezzi

Il sistema supporta tre modalità di prezzo, selezionabili per ogni riga:

| Tipo | Calcolo |
|---|---|
| **giorno** | `quantità × prezzo_giorno × (data_fine - data_inizio)` |
| **forfait** | `quantità × prezzo_forfettario` (indipendentemente dalla durata) |
| **durata** | Prezzo in base alla durata: 1gg = prezzo_giorno, 2-3gg = prezzo_3gg, 4+gg = prezzo_settimana |

In fase di trasformazione in ordine è possibile:
- Sconto percentuale sulla riga o sul totale
- Sconto importo fisso sulla riga o sul totale
- Override completo del prezzo unitario

### 5.3 Gestione Stati Articolo

```
Disponibile
    │
    ▼
In carico (transito verso evento)
    │
    ▼
In noleggio (presso cliente)
    │
    ├──▶ Reso (rientrato in magazzino)
    │
    └──▶ Danneggiato (con quantitativo specifico)
          └──▶ (da valutare: riparazione / sostituzione / scarico)
```

Il sistema tiene traccia delle quantità danneggiate singolarmente (es. su 100 sedie, 3 danneggiate). Le quantità danneggiate sono dettagliate per movimento di ingresso.

### 5.4 Calendario Eventi

**Logica di visualizzazione:**
- Vista Gantt 2 mesi (Giugno–Luglio 2026) con eventi come barre colore
- Ogni evento mostra: numero ordine, cliente, date, qtà totale, stato
- Barre colore per stato: bozza (grigio), inviato (arancio), accettato/blu, confermato (blu), allestimento (verde), in corso (accent), completato (verde chiaro)
- Drag-to-pan sulla timeline, marker "OGGI"

**Modalità di visualizzazione:**
- Vista Gantt multi-articolo con timeline condivisa
- Per ogni prodotto, barre colore che mostrano gli impegni nel tempo
- Filtro per categoria, stato, ricerca testo
- Drag-to-pan sulla timeline, navigazione rapida
- Legenda interattiva con click per scrollare a un impegno specifico

---

## 6. Autenticazione e Autorizzazione

### 6.1 Ruoli Utente (configurabili)

| Ruolo | Permessi |
|---|---|
| **superadmin** | Accesso totale, configurazione sistema, gestione utenti, log, multi-tenant |
| **admin** | Tutto tranne gestione tenant. Crea/modifica utenti, catalogo, prezzi |
| **operatore** | Gestione clienti, preventivi, ordini, DDT. NON può modificare prezzi catalogo, utenti, impostazioni |
| **magazziniere** | Solo DDT (creazione/conferma), consultazione ordini, dashboard |

I ruoli e i permessi sono **configurabili** tramite tabella `permessi` e associazione `ruolo_permesso`, permettendo di creare ruoli custom in futuro.

### 6.2 Modalità Autenticazione

**Dual-mode (JWT + Sessioni):**

- **API**: Autenticazione via JWT (Authorization header). JWT breve durata (15 min) + refresh token (7 giorni) in httpOnly cookie.
- **Web app**: Autenticazione via sessioni (cookie httpOnly). Il backend gestisce la sessione lato server.
- Le API sono accessibili con entrambe le modalità.
- Logout invalida sia JWT refresh token che sessione.

---

## 7. Integrazione con Gestionale (progettazione futura)

### 7.1 Principi Generali

- Le API di integrazione saranno definite dal **fornitore del gestionale**
- L'applicativo gestisce in autonomia il proprio DB e il CRUD completo
- Periodicamente (o su evento) avviene la **sincronizzazione bidirezionale**
- La sincronizzazione copre:
  - **Anagrafiche** (clienti): create/modificate nell'app e inviate al gestionale per ciclo attivo
  - **Prodotti**: allineamento codici articolo e giacenze
  - **Ordini**: invio dati essenziali per fatturazione
  - **Movimenti**: comunicazione carichi/scarichi per valorizzazione magazzino

### 7.2 Punti di Integrazione Previsti

```
┌─────────────────────────┐         ┌──────────────────────────┐
│   App Noleggi            │         │   Gestionale             │
│                          │         │                          │
│  ┌───────────────────┐   │   API   │  ┌────────────────────┐  │
│  │ Anagrafica Clienti │──┼────────┼─▶│ Ciclo Attivo       │  │
│  └───────────────────┘   │         │  │ (fatture, note)    │  │
│                          │◀────────┼──┤                    │  │
│  ┌───────────────────┐   │         │  └────────────────────┘  │
│  │ Ordini Confermati  │──┼────────┼─▶│ Ciclo Passivo      │  │
│  └───────────────────┘   │         │  │ (ordini fornitori) │  │
│                          │         │  └────────────────────┘  │
│  ┌───────────────────┐   │         │                          │
│  │ Movimenti / DDT    │──┼────────┼─▶│ Magazzino            │  │
│  │ (carico/scarico)   │   │         │  │ (valorizzazione)   │  │
│  └───────────────────┘   │         │  └────────────────────┘  │
│                          │         │                          │
│  ┌───────────────────┐   │         │  ┌────────────────────┐  │
│  │ Catalogo Prodotti │◀────────────┼──│ Anagrafica         │  │
│  │ (allineamento)    │   │         │  │ Prodotti           │  │
│  └───────────────────┘   │         │  └────────────────────┘  │
└─────────────────────────┘         └──────────────────────────┘
```

### 7.3 Strategia di Sincronizzazione

1. **Doppia gestione**: l'app opera sul proprio DB in autonomia
2. **Batch periodico** (configurabile, default ogni 15 min) o **su evento** (alla conferma di un ordine/movimento)
3. **Campo `ultima_sincronizzazione`** su ogni entità sincronizzabile
4. **Log delle sincronizzazioni** per tracciare errori e riconciliazioni
5. **Meccanismo retry** con backoff esponenziale in caso di errore API

---

## 8. Interfaccia Utente

### 8.1 Struttura Generale

L'interfaccia è una **SPA (Single Page Application)** con sidebar laterale fissa (240px) e area contenuto dinamica. Il prototipo implementa **8 pagine** collegate da navigazione JavaScript `showPage()` + **1 modale** per il dettaglio ordine.

```
┌────────────────────────────────────────────────────────────────┐
│ ⚠ DEMO BANNER (giallo, monospace, sticky)                     │
├────────────────────────────────────────────────────────────────┤
│ ┌──────────┐ ┌──────────────────────────────────────────────┐  │
│ │ SIDEBAR  │ │  TOPBAR (titolo pagina + azioni contestuali) │  │
│ │ 240px    │ ├──────────────────────────────────────────────┤  │
│ │ [Logo]   │ │                                              │  │
│ │ OG       │ │         CONTENUTO PRINCIPALE                 │  │
│ │          │ │         (pagina attiva, scrollabile)         │  │
│ │ ────     │ │                                              │  │
│ │ Visuale  │ │                                              │  │
│ │ Dashboard│ │                                              │  │
│ │ Catalogo │ │                                              │  │
│ │ Calend.  │ │                                              │  │
│ │ ────     │ │                                              │  │
│ │ Operativo│ │                                              │  │
│ │ Prevent. │ │                                              │  │
│ │ Ordini   │ │                                              │  │
│ │ DDT      │ │                                              │  │
│ │ ────     │ │                                              │  │
│ │ Anagraf. │ │                                              │  │
│ │ Clienti  │ │                                              │  │
│ │ ──────── │ │                                              │  │
│ │ [Utente] │ │                                              │  │
│ └──────────┘ └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### 8.2 Schema Navigazione

| Sezione sidebar | Pagina | ID pagina | Navigate da |
|---|---|---|---|
| **Visuale** | Dashboard | `page-dashboard` | Sidebar → "Dashboard" |
| **Visuale** | Catalogo Prodotti | `page-catalogo` | Sidebar → "Catalogo" |
| **Visuale** | Dettaglio Prodotto | `page-dettaglio-prodotto` | Click card prodotto in Catalogo |
| **Visuale** | Calendario Eventi | `page-calendario` | Sidebar → "Calendario Eventi" |
| **Operativo** | Preventivi | `page-preventivi` | Sidebar → "Preventivi" |
| **Operativo** | Ordini | `page-ordini` | Sidebar → "Ordini" |
| **Operativo** | DDT / Movimenti | `page-ddt` | Sidebar → "DDT / Movimenti" |
| **Anagrafiche** | Clienti | `page-clienti` | Sidebar → "Clienti" |

### 8.3 Descrizione Pagina per Pagina

---

#### 8.3.1 Dashboard (`page-dashboard`)

**Purpose:** Vista operativa quotidiana — l'operatore apre la mattina e vede subito cosa è urgente.

**Layout:**
- Topbar: titolo "Dashboard" + timestamp ultimo aggiornamento + pulsante "Nuovo Preventivo"
- Contenuto: griglia KPI (4 colonne) → griglia 2:1 (tabella ordini + pannelli laterali) → tabella DDT

**Sezione A — KPI Cards (4 colonne):**

| KPI | Valore demo | Fonte dati | Colore |
|---|---|---|---|
| Ordini Attivi | 12 | `COUNT(ordini WHERE stato IN ('confermato','in_allestimento','in_corso'))` | `--accent` |
| Preventivi in Scadenza | 5 | `COUNT(preventivi WHERE data_scadenza <= TODAY+7 AND stato = 'inviato')` | `--warn` |
| Prossimi Impegni | 3 | `COUNT(ordini WHERE data_inizio <= TODAY+3 AND stato != 'completato')` | default |
| Disp. Critica | 7 | `COUNT(prodotti WHERE disponibilita_pct < 20)` | `--danger` |

Ogni KPI ha una sottoriga con variazione vs mese precedente o contesto aggiuntivo (es. "3 entro 3 giorni", "Prossimo: 22/06 — Rossi Srl").

**Sezione B — Griglia 2:1:**

*Colonna sinistra (2/3) — Tabella "Ordini Attivi per Stato":*

| Colonna | Tipo | Note |
|---|---|---|
| Ordine | `#YYYY/NNN` monospace, clickabile → dettaglio ordine | |
| Cliente | testo | |
| Evento | testo | |
| Dal | data `DD/MM` | |
| Al | data `DD/MM` | |
| Stato | pill colore | `pill-green` = In allestimento, `pill-blue` = Confermato, `pill-gray` = Bozza |

*Colonna destra (1/3) — 2 card empilate:*

1. **Preventivi in Scadenza** — lista compatta con: nome cliente, importo, merce, pill scadenza (rosso <3gg, giallo <7gg)
2. **Disponibilità Critica** — lista prodotti con disponibilità residua vs totale, colorata in base a soglia

**Sezione C — Tabella "Ultimi Movimenti (DDT)":**

| Colonna | Tipo | Note |
|---|---|---|
| DDT | `#YYYY/NNN` monospace | |
| Tipo | pill `pill-red` = Uscita, `pill-green` = Ingresso | |
| Ordine | `#YYYY/NNN` link | |
| Data | `DD/MM/YYYY` | |
| Articoli | testo compatto (es. "120 sedie, 15 tavoli") | |
| Stato | pill colore | |

**API necessarie:**
- `GET /dashboard/riepilogo` → KPI cards
- `GET /dashboard/impegni-prossimi` → tabella ordini imminenti
- `GET /dashboard/preventivi-scaduti` → lista preventivi in scadenza
- `GET /dashboard/disponibilita-critica` → prodotti sotto soglia
- `GET /movimenti?limit=5&sort=-data` → ultimi DDT

**Interazioni:**
- Click su riga ordine → apre modale dettaglio ordine
- Click su "Vedi tutti →" (DDT) → naviga a `page-ddt`
- Click "+ Nuovo Preventivo" → apre modale wizard multi-step (prototipato)

---

#### 8.3.2 Catalogo Prodotti (`page-catalogo`)

**Purpose:** Vista elenco prodotti con filtro rapido — il catalogo è il punto di partenza per creare preventivi/ordini e per verificare disponibilità.

**Layout:**
- Topbar: titolo "Catalogo Prodotti" + pulsanti "Esporta CSV" e "+ Nuovo Prodotto"
- Filtri: barra ricerca testo + dropdown categoria + dropdown stato
- Chips categoria: selezione rapida con conteggio prodotti per categoria
- Griglia card prodotto: layout auto-fill, min 220px per card

**Barra Filtri:**

| Filtro | Tipo | Opzioni | Note |
|---|---|---|---|
| Ricerca testo | `input text` | — | Cerca per nome, codice, descrizione |
| Categoria | `select` | Tutte, Sedute, Tavoli, Tovagliato, Luci, Mobili, Banconi, Piante e Vasi, Pavimenti, Mise en place, Coperture, Accessori e sicurezza | Filtraggio lato server |
| Stato | `select` | Tutti, Disponibile, In noleggio, Danneggiato | |

**Chips Categoria (filtro rapido):**

Ogni chip mostra nome + conteggio tra parentesi. Il chip "Tutti" è attivo di default. Click su un chip filtra la griglia e aggiorna il conteggio.

```
[Tutti (342)] [Sedute (128)] [Tavoli (45)] [Tovagliato (62)] [Luci (38)] [Mobili (22)] [Accessori (96)] [Mise en place (17)]
```

**Card Prodotto:**

Ogni card contiene:
- **Immagine**: `aspect-ratio: 4/3`, `object-fit: cover`. Usa reale da `assets/images/` oppure placeholder `.ph-img` se non disponibile
- **Nome**: grassetto, 13px
- **Meta**: codice articolo + quantità totale, 11px, colore muted (es. "SED-001 · 200 pz")
- **Prezzo**: monospace, colore accent, 13px (es. "€ 2,00 /giorno")

**Dati per card (da API `GET /prodotti`):**
- `id`, `nome`, `codice`, `categoria`, `quantita_totale`, `prezzo_giorno`, `immagine_url`

**Interazioni:**
- Click card → naviga a `page-dettaglio-prodotto` (passa `prodottoId` come param)
- Click chip categoria → filtra griglia
- Click "+ Nuovo Prodotto" → apre form creazione (Fase 1)
- Click "Esporta CSV" → download CSV del catalogo filtrato

**Responsive:**
- Desktop: griglia 4-5 colonne
- Tablet: 3 colonne
- Mobile (<920px): 2 colonne, sidebar nascosta, filtri in colonna

---

#### 8.3.3 Dettaglio Prodotto (`page-dettaglio-prodotto`)

**Purpose:** Vista completa di un prodotto con info anagrafiche, stato disponibilità e diagramma Gantt degli impegni. È la schermata più complessa del prototipo.

**Layout:**
- Topbar: breadcrumb "← Catalogo" + nome prodotto + pulsanti "Modifica" e "Verifica Disponibilità"
- Griglia 2:1 superiore (info + stats)
- Gantt chart interattivo (sezione piena larghezza)
- Legenda impegni + riepilogo disponibilità (griglia 2 colonne)

**Sezione A — Griglia 2:1 (Immagine + Info | Stats)**

*Colonna sinistra (2/3):*

1. **Card Immagine** — immagine prodotto a tutta larghezza, `aspect-ratio: 16/9`, `object-fit: cover`
2. **Card "Informazioni Prodotto"** — tabella a 2 colonne:

| Campo | Esempio | Note |
|---|---|---|
| Codice articolo | SED-001 | monospace |
| Categoria | Sedute → Sedie | gerarchia |
| Quantità totale | 200 pz | |
| Unità di misura | Pezzi | |
| Dimensioni | 45×50×h90 cm | |

Pill "Attivo" (verde) nell'header della card.

*Colonna destra (1/3):*

1. **Card "Stato Disponibilità"** — griglia 3 colonne con KPI mini:

| KPI | Valore demo | Colore |
|---|---|---|
| Disponibili | 152 | `--success` |
| In noleggio | 40 | `--warn` |
| Danneggiati | 8 | `--danger` |

2. **Card "Listino Prezzi"** — tabella tariffe:

| Tariffa | Prezzo |
|---|---|
| Per giorno | € 2,00 |
| 3 giorni | € 5,00 |
| Settimana | € 8,00 |
| Forfettario | € 10,00 |

**Sezione B — Diagramma Gantt (8.4 dettagliato sotto)**

Componente interattivo a tutta larghezza che mostra timeline degli impegni su 22 giorni.

**Sezione C — Legenda + Riepilogo (griglia 2 colonne)**

*Colonna sinistra — "Impegni Articolo":*
Lista clickabile di tutti gli impegni. Ogni riga:
- Punto colore (stato)
- Nome cliente + numero ordine
- Periodo + nome evento
- Quantità (monospace, grassetto)
- Percentuale su totale

Click su riga → scrolla il Gantt al periodo e fa highlight transiente della barra.

*Colonna destra — "Riepilogo Disponibilità":*
- Totale magazzino / Impegnati / Media disponibilità giornaliera
- Giorni a disponibilità piena (es. "12 / 21")
- Picco impegni (data + quantità)
- Giorno più libero (data + quantità)
- **Sparkline** — mini-grafico a barre verticali colorate per giorno (verde >70%, giallo 30-70%, rosso <30%)

**API necessarie:**
- `GET /prodotti/:id` → info prodotto
- `GET /prodotti/:id/disponibilita?from=YYYY-MM-DD&to=YYYY-MM-DD` → disponibilità giornaliera
- `GET /ordini?prodottoId=:id&stato=in_corso,confermato,in_allestimento` → impegni per il Gantt

**Interazioni:**
- Drag-to-pan sul Gantt (mouse down + move)
- Pulsanti ◀ 2 sett / Oggi / 2 sett ▶ per scrollare
- Click legenda → scroll + highlight barra
- Click "Modifica" → form modifica prodotto (Fase 1)
- Click "Verifica Disponibilità" → apre Calendario Eventi pre-filtrato su questo ordine

---

#### 8.3.4 Calendario Eventi (`page-calendario`)

**Purpose:** Vista Gantt 2 mesi — mostra tutti gli eventi/ordini su una timeline condivisa con barre colore per stato, permettendo di visualizzare rapidamente gli impegni del magazzino.

**Layout:**
- Topbar: titolo + barra navigazione Gantt (◀ 2 sett / Oggi / 2 sett ▶ + label periodo)
- Filtri: ricerca testo per evento/cliente, dropdown stato
- Griglia principale: Gantt eventi a sinistra, pannelli legenda a destra

**Sezione A — Gantt Eventi**

Layout a griglia con due colonne:
- **Colonna etichette (220px):** pallino colore stato, nome evento, cliente · qtà pz, pill stato
- **Colonna timeline:** header con giorni (Lun-Dom, evidenziati weekend e oggi), righe evento con barre colore

Ogni riga evento mostra:
- **Pallino colore** dello stato
- **Nome evento** e **cliente** + qtà totale
- **Pill stato** con colore corrispondente
- **Barre colore** sulla timeline con numero ordine + cliente come tooltip
- **Marker "OGGI"** con linea verticale arancio

**Timeline:**
- 42px per giorno, 2 mesi (1 Giu — 31 Lug 2026)
- Drag-to-pan con mouse
- Navigazione rapida: ◀ 2 sett / Oggi / 2 sett ▶
- Scroll automatico al giorno odierno al caricamento
- Weekend evidenziati con sfondo leggero
- Cambio mese segnalato con bordo sinistro

**Sezione B — Pannelli Legenda (colonna destra, 300px)**

1. **Card "Riepilogo":**
   - Eventi totali
   - In questo periodo (filtrati)
   - Clienti coinvolti

2. **Card "Prossimi Eventi":** lista ordinata per data degli eventi futuri con:
   - Pallino colore stato
   - Nome evento
   - Cliente · periodo
   - Qtà totale
   - Click = scrolla il Gantt a quel periodo

3. **Card "Legenda Stati":** conteggio eventi per stato

**Filtri:**
- **Ricerca testo:** filtra per nome evento, cliente o numero ordine
- **Stato:** Bozza, Inviato, Accettato, Confermato, In allestimento, In corso, Completato

**Interazioni:**
- Drag-to-pan sulla timeline
- Click su voce legenda → scrolla Gantt al periodo dell'evento
- Click su riga etichetta → apre modale dettaglio ordine
- Click su barra Gantt → apre modale dettaglio ordine
- Filtri combinabili (testo + stato)
- Navigazione ◀ ▶ con scroll fluido
- Click "Oggi" → centra la vista sulla data corrente

**API necessarie:**
- `GET /ordini` → lista ordini con date, cliente, evento, stato, qtà
- `GET /ordini?da=YYYY-MM-DD&a=YYYY-MM-DD` → ordini nel periodo
- `GET /ordini/:id` → dettaglio singolo ordine (per modale)

---

#### 8.3.5 Preventivi (`page-preventivi`)

**Purpose:** Elenco preventivi con filtro e stato — il preventivo è il documento commerciale di apertura del rapporto con il cliente.

**Layout:**
- Topbar: titolo "Preventivi" + pulsante "+ Nuovo Preventivo"
- Filtri: ricerca testo + select stato
- Tabella paginata

**Filtri:**

| Filtro | Tipo | Opzioni |
|---|---|---|
| Ricerca | `input text` | Cerca per cliente, numero preventivo |
| Stato | `select` | Tutti, Bozza, Inviato, Accettato, Trasformato, Scaduto |

**Tabella Preventivi:**

| Colonna | Tipo | Formato | Note |
|---|---|---|---|
| Numero | monospace | `#YYYY/NNN` | |
| Cliente | testo | — | |
| Periodo | testo | `DD-DD/MM/YYYY` | Data inizio — fine evento |
| Totale | monospace, allineato a destra | `€ X.XXX,XX` | Formato italiano |
| Stato | pill colore | — | Vedi codice colore sotto |
| Scadenza | monospace | `DD/MM` | |
| Azioni | — | — | Pulsante "Apri →" |

**Codice Stati Preventivo:**

| Stato | Pill | Colore | Significato |
|---|---|---|---|
| Bozza | `pill-gray` | Grigio | Preventivo in creazione, non inviato |
| Inviato | `pill-amber` | Giallo | Inviato al cliente, in attesa risposta |
| Accettato | `pill-green` | Verde | Cliente ha accettato, da trasformare |
| Trasformato | `pill-blue` | Blu | Trasformato in ordine |
| Scaduto | `pill-red` | Rosso | Scaduto senza risposta |

**API necessarie:**
- `GET /preventivi?page=N&limit=20&stato=X&q=testo` → lista paginata

**Interazioni:**
- Click "Apri →" → naviga a dettaglio preventivo (non prototipato, Fase 2)
- Click "+ Nuovo Preventivo" → apre modale wizard multi-step (prototipato)
- Click colonna "Numero" → dettaglio preventivo

---

##### 8.3.5a Modale "Nuovo Preventivo" (overlay multi-step)

**Purpose:** Wizard di creazione preventivo in 3 step, accessibile da Dashboard e Preventivi.

**Struttura:**
- Overlay fullscreen con backdrop blur, chiusura con ✕, click fuori, o Esc
- Header: titolo "Nuovo Preventivo" + pulsante ✕
- Indicatore step: 3 pill cerchiati (1 Dati ordine → 2 Righe ordine → 3 Riepilogo)
- Body scrollabile + footer con pulsanti "← Indietro" / "Continua →"

**Step 1 — Dati ordine:**

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| Cliente | `select` | Sì | Dropdown con anagrafica clienti |
| Data evento | `date` | Sì | Data inizio evento |
| Nome evento | `text` | Sì | Descrizione breve |
| Indirizzo venue | `text` | No | Via, numero, città |
| Referente | `text` | No | Nome · Telefono |
| Giorni noleggio | `number` | Sì | Default 1, min 1, max 30 |
| Note | `textarea` | No | Accesso, parcheggio, note logistiche |

**Step 2 — Righe ordine:**

- **Pulsante "+ Aggiungi articoli"** → apre modale secondaria "Righe ordine"
- **Griglia articoli selezionati:** tabella con colonne:
  - Thumbnail, Nome + Codice + badge disponibilità (`V N / T N / P N / Tot: N`), Qtà (input number con warning disponibilità), Tariffa (select: Giorno/3gg/Settimana/Forfait), Prezzo unitario, Sconto % (input), Totale riga, Pulsante rimuovi ✕
- **Badge disponibilità per riga:** sotto nome e codice, 4 badge inline:
  - `V N` (verde) = pezzi liberi considerando solo impegnato vero
  - `T N` (arancio) = pezzi liberi considerando impegnato teorico
  - `P N` (blu) = pezzi liberi considerando impegnato probabile
  - `/ Tot: N` = quantità totale a magazzino
- **Warning disponibilità sulla qtà:** quando la quantità inserita supera una soglia di disponibilità, l'input mostra:
  - Bordo + sfondo colorato (verde/arancio/blu) in base al livello superato
  - Pilloletta sotto l'input: "▲ Supera vero/teorico/probabile (N liberi)"
  - Si aggiorna dinamicamente quando cambia la data evento o i giorni
- **Sconto globale:** input percentuale applicato a tutte le righe
- **Riepilogo:** Subtotale, Sconto righe, Sconto globale, Totale
- **Conteggio articoli** visibile nella testata della griglia

##### 8.3.5b Modale "Righe ordine" (overlay secondaria)

**Purpose:** Ricerca e selezione articoli dal catalogo, aperta dal wizard preventivo.

**Struttura:**
- Overlay `z-index: 250` (sopra il wizard a `z-index: 200`), backdrop blur
- Header: titolo "Righe ordine" + pulsante ✕
- Body scrollabile + footer con pulsante "Chiudi"

**Contenuto:**
- **Ricerca catalogo:** input testo + dropdown categoria, filtra in tempo reale
- **Legenda disponibilità:** 3 indicatori + totale
- **Lista risultati:** ogni riga catalogo contiene:
  - Thumbnail + Nome + Codice + Categoria
  - **Riga disponibilità:** badge `V N` / `T N` / `P N` / `Tot: N`
  - **Input qtà** direttamente nella riga (default: 1, min: 1, max: totale)
  - **Warning disponibilità:** se la qtà inserita supera una soglia, sotto l'input appare una pilloletta arancio/rossa con l'indicazione di quale impegno si sta superando
  - Pulsante "+ Aggiungi" / "✓ Aggiunto" (toggle: se già aggiunto, lo rimuove)
- La qtà inserita viene riportata nella griglia step 2 quando l'articolo viene aggiunto
- Chiusura con ✕, Esc, o pulsante "Chiudi"

**Step 3 — Riepilogo:**
- Card "Dati Evento": data, evento, cliente, venue, referente, giorni, note
- Card "Riepilogo Costi": subtotali, sconti, totale, conteggio articoli/pezzi
- Tabella righe complete: articolo, qtà, tariffa, prezzo ud., sconto, totale riga

**Logica disponibilità:**
- La disponibilità si calcola in base alla data evento selezionata e ai giorni di noleggio
- Se la data non è impostata, mostra i totali generali senza filtro temporale
- `Impegnato vero` = somma qty ordini con status `confirmed`, `allestimento`, `in-corso` nel periodo
- `Impegnato teorico` = vero + somma qty preventivi nel periodo
- `Impegnato probabile` = vero + somma (qty × probabilità) preventivi nel periodo
- La probabilità è un valore 0–1 associato a ogni preventivo (simulato nel prototipo)

**API necessarie (Fase 2):**
- `GET /catalogo/disponibilita?articoloId=X&dataInizio=YYYY-MM-DD&giorni=N` → disponibilità 3 livelli
- `POST /preventivi` → crea preventivo con righe e sconti

---

#### 8.3.6 Ordini (`page-ordini`)

**Purpose:** Elenco ordini attivi e storico — ogni ordine deriva da un preventivo accettato o è creato direttamente.

**Layout:**
- Topbar: titolo "Ordini" + pulsante "+ Nuovo Ordine"
- Filtri: ricerca testo + select stato
- Tabella ordini

**Tabella Ordini:**

| Colonna | Tipo | Formato | Note |
|---|---|---|---|
| Ordine | monospace | `#YYYY/NNN` | |
| Cliente | testo | — | |
| Evento | testo | — | Nome evento |
| Inizio | monospace | `DD/MM` | Data inizio noleggio |
| Fine | monospace | `DD/MM` | Data fine noleggio |
| Totale | monospace, right | `€ X.XXX,XX` | |
| Stato | pill colore | — | Vedi sotto |
| Azioni | — | — | "Apri →" |

**Codice Stati Ordine:**

| Stato | Pill | Colore | Transizione successiva |
|---|---|---|---|
| Bozza | `pill-gray` | Grigio | → Confermato |
| Confermato | `pill-blue` | Blu | → DDT Uscita |
| In allestimento | `pill-green` | Verde | → In corso |
| In corso | `pill-amber` | Arancio | → DDT Ingresso |
| In rientro | `pill-amber` | Arancio | → Completato |
| Completato | `pill-green` | Verde | (finale) |

**API necessarie:**
- `GET /ordini?page=N&limit=20&stato=X&q=testo` → lista paginata

**Interazioni:**
- Click su riga ordine → apre modale dettaglio ordine (`openOrderModal()`)
- Click "+ Nuovo Ordine" → creazione diretta (Fase 2)

---

#### 8.3.7 Dettaglio Ordine (modale `order-modal`)

**Purpose:** Vista completa di un ordine — timeline stato, dati evento, righe ordine, riepilogo costi, DDT associati, foto allestimento, questionario post-evento per AI. Aperta come modale sovrapposta alla pagina corrente.

**Layout:**
- Overlay con backdrop blur, chiusura con ✕, click fuori o tasto Esc
- Header: numero ordine + pill stato + pulsanti "Scarica PDF" e "Crea DDT Uscita"
- Timeline stato (componente a tutta larghezza)
- Griglia 2 colonne (dati evento + riepilogo costi)
- Tabella righe ordine
- Tabella DDT associati
- Griglia foto allestimento (con zoom)
- Questionario post-evento AI

**Sezione A — Timeline Stato**

Componente visivo a 6 step con linee connesse:

```
●───────────●───────────●───────────○───────────○───────────○
Confermato   DDT Uscita  Allestimento In corso   DDT Ingresso Completato
(done)       (done)      (active)     (pending)  (pending)    (pending)
```

| Step | Stati corrispondenti | Colore when done | Colore when active |
|---|---|---|---|
| Confermato | `confermato` | Verde | — |
| DDT Uscita | `in_allestimento` (dopo DDT) | Verde | — |
| In allestimento | `in_allestimento` | — | Accent + glow |
| In corso | `in_corso` | — | Accent |
| DDT Ingresso | `in_rientro` | — | Accent |
| Completato | `completato` | Verde | — |

**Sezione B — Griglia 2 colonne**

*Colonna sinistra — "Dati Evento":*

| Campo | Esempio |
|---|---|
| Cliente | Rossi Srl |
| Evento | Matrimonio Verdi |
| Indirizzo | Via Roma 15, Bergamo |
| Referente | Laura Rossi · 393 037 3440 |
| Periodo noleggio | 22/06/2026 — 23/06/2026 |
| Allestimento | 22/06/2026 ore 08:00 |
| Smontaggio | 23/06/2026 ore 18:00 |

*Colonna destra — "Riepilogo Costi":*

| Campo | Valore |
|---|---|
| Subtotale | € 6.022,00 |
| Sconto 10% | − € 602,00 (rosso) |
| **Totale** | **€ 5.420,00** (grassetto, 16px) |

Sotto: card "Note logistiche" con testo libero su sfondo `--fg-soft`.

**Sezione C — Tabella Righe Ordine:**

| Colonna | Tipo | Formato |
|---|---|---|
| Prodotto | testo + codice | "Sedia Chiavari bianca `SED-001`" |
| Qtà | monospace | 50 |
| Tipo prezzo | testo | "Giorno × 2gg" / "Forfait" |
| Prezzo unit. | monospace | € 2,00 |
| Sconto | monospace | — / −10% |
| Totale riga | monospace | € 200,00 |

**Sezione D — DDT Associati:**

Tabella con DDT collegati all'ordine (uscita e ingresso):

| Colonna | Tipo |
|---|---|
| DDT | monospace `#YYYY/NNN` |
| Tipo | pill `pill-red` Uscita / `pill-green` Ingresso |
| Data | monospace |
| Articoli | testo compatto |
| Stato | pill colore |
| Azioni | "PDF →" |

**API necessarie:**
- `GET /ordini/:id` → dati ordine + righe
- `GET /movimenti?ordineId=:id` → DDT associati

**Sezione E — Foto Allestimento:**

Griglia responsive di thumbnail (min 120px) con caption sovrapposta. Ogni foto è cliccabile per zoom fullscreen.

| Elemento | Comportamento |
|---|---|
| Thumbnail | `aspect-ratio: 4/3`, `object-fit: cover`, caption in basso su sfondo semi-trasparente |
| Click su foto | Apre overlay zoom fullscreen con navigazione ◀/▶ |
| Zoom overlay | `z-index: 300`, backdrop blur, foto max 90vw × 85vh |
| Navigazione | Freccia sinistra/destra, contatore "1 / N — caption" |
| Chiusura zoom | ✕, click fuori, Esc |
| Pulsante "+ Aggiungi foto" | Placeholder upload (dashed border) |

| Campo dati | Tipo |
|---|---|
| `photos` | Array di `{ src, caption }` |
| Thumbnail | `aspect-ratio: 4/3`, `object-fit: cover` |

**Sezione F — Questionario Post-Evento AI:**

Form strutturato per raccogliere feedback post-evento e addestrare l'AI sull'idoneità articoli/venue.

| Campo | Tipo | Opzioni/Valori |
|---|---|---|
| Qualità percepita | Stelle 1–5 | ★ clickable con stato filled |
| Venue | Input testo | Es. "Sala privata", "Villa con giardino" |
| Adattabilità venue | Select | Perfetto / Ottimo / Buono / Difficile |
| Accesso / carico | Input testo | Descrizione accesso |
| Puntualità consegna | Select | Puntuale / Ritardo < 1h / Ritardo > 1h |
| Danni riscontrati | Input testo | Es. "Nessuno", "2 sedie con graffio" |
| Feedback cliente | Input testo (full width) | Testo libero |
| Miglioramenti suggeriti | Textarea (full width) | Note per eventi futuri |
| Tag AI | pills + pulsante "+ tag" | Tag categorici (es. "matrimonio", "villa", "lusso") |

**Comportamento:**
- Quando il questionario è già compilato: mostra tutti i campi con i valori salvati
- Quando non è compilato: messaggio "Questionario non ancora compilato"
- Pulsante "Salva questionario" → conferma visiva "✓ Salvato" (2s)
- I tag AI alimentano il calcolo dell'impegnato probabile nei preventivi
- Le stelle sono interattive: click aggiorna immediatamente lo stato visivo

**API necessarie:**
- `GET /ordini/:id` → dati ordine + righe
- `GET /movimenti?ordineId=:id` → DDT associati
- `GET /ordini/:id/foto` → lista foto allestimento
- `POST /ordini/:id/foto` → upload foto (multipart/form-data)
- `GET /ordini/:id/questionario` → dati questionario AI
- `PUT /ordini/:id/questionario` → salva/aggiorna questionario

**Interazioni:**
- Click ✕ / click fuori / Esc → chiudi modale
- Click "Scarica PDF" → download PDF ordine
- Click "Crea DDT Uscita" → wizard DDT (Fase 3)
- Click "PDF →" su DDT → download PDF DDT
- Click su foto → zoom overlay con navigazione ◀/▶
- Click ✕ / fuori / Esc (zoom) → chiudi zoom
- Click stelle → aggiorna rating
- Click "+ Aggiungi foto" → placeholder upload
- Click "Salva questionario" → conferma "✓ Salvato"
- Click "+ tag" → placeholder aggiunta tag
- La modale è accessibile da: tabella ordini, barre Gantt Dashboard, sidebar ordini Gantt

---

#### 8.3.8 DDT / Movimenti (`page-ddt`)

**Purpose:** Elenco documenti di trasporto (DDT di uscita e ingresso) — traccia ogni movimento fisico della merce.

**Layout:**
- Topbar: titolo "DDT / Movimenti" + pulsante "+ Nuovo DDT"
- Filtri: ricerca testo + select tipo (Uscita/Ingresso) + select stato
- Tabella DDT

**Tabella DDT:**

| Colonna | Tipo | Formato |
|---|---|---|
| DDT | monospace | `#YYYY/NNN` |
| Tipo | pill | `pill-red` = Uscita, `pill-green` = Ingresso |
| Ordine | monospace link | `#YYYY/NNN` |
| Cliente | testo | |
| Data | monospace | `DD/MM/YYYY` |
| Articoli | testo | "6 righe · 89 pz" |
| Stato | pill | `pill-green` = Confermato, `pill-gray` = Bozza |
| Azioni | — | "Apri →" |

**Filtri:**

| Filtro | Tipo | Opzioni |
|---|---|---|
| Ricerca | `input text` | Per numero DDT, numero ordine |
| Tipo | `select` | Tutti, Uscita, Ingresso |
| Stato | `select` | Tutti, Bozza, Confermato |

**API necessarie:**
- `GET /movimenti?page=N&limit=20&tipo=X&stato=X&q=testo` → lista paginata

**Interazioni:**
- Click "Apri →" → dettaglio DDT (non prototipato, Fase 3)
- Click "+ Nuovo DDT" → wizard creazione DDT (Fase 3)
- Click su "Ordine" link → naviga a dettaglio ordine

---

#### 8.3.9 Clienti (`page-clienti`)

**Purpose:** Anagrafica clienti con sync gestionale — elenco di tutti i clienti con dati di contatto e stato sincronizzazione.

**Layout:**
- Topbar: titolo "Clienti" + pulsanti "Sincronizza Gestionale" e "+ Nuovo Cliente"
- Filtri: ricerca testo + select stato (Tutti/Attivi/Inattivi)
- Tabella clienti

**Tabella Clienti:**

| Colonna | Tipo | Formato |
|---|---|---|
| Codice | monospace | `CLT-NNN` |
| Ragione Sociale | grassetto | |
| P.IVA | monospace | `XXXXXXXXXXX` |
| Città | testo | |
| Referente | testo | Nome Cognome |
| Telefono | monospace | |
| Ultima Sync | monospace, 11px | `DD/MM HH:MM` |
| Azioni | — | "Apri →" |

**Filtri:**

| Filtro | Tipo | Opzioni |
|---|---|---|
| Ricerca | `input text` | Per ragione sociale, P.IVA, referente |
| Stato | `select` | Tutti, Attivi, Inattivi |

**API necessarie:**
- `GET /clienti?page=N&limit=20&q=testo&stato=X` → lista paginata
- `POST /integrazione/sync-clienti` → sincronizzazione gestionale

**Interazioni:**
- Click "Apri →" → dettaglio cliente (non prototipato)
- Click "Sincronizza Gestionale" → avvia sync, mostra spinner/esito
- Click "+ Nuovo Cliente" → form creazione (Fase 1)

---

## 9. Dashboard e KPI

### 9.1 KPI Principali

| Indicatore | Fonte | Aggiornamento |
|---|---|---|
| **Ordini attivi** | Ordini in stato confermato/in_corso | Real-time |
| **Preventivi in scadenza** | Preventivi con data_scadenza ≤ 7gg | Real-time |
| **Prossimi impegni** | Ordini con data_inizio ≤ 3gg | Real-time |
| **Disponibilità critica** | Prodotti con disponibilità < 20% in date future | Real-time |
| **Totale fatturato (mense)** | Ordini completati nel periodo | Mese corrente |
| **Tasso trasformazione** | Preventivi trasformati / totale inviati | Mese corrente |

### 9.2 Sezione Solleciti (AI future)

- **Dashboard dedicata** con vista preventivi in scadenza / scaduti
- **Priorità** basata su: data scadenza, valore preventivo, storico cliente
- **Azioni rapide**: invia sollecito email, proroga scadenza, archivia
- **AI futura**: generazione automatica testo sollecito, suggerimento sconto, predizione probabilità trasformazione

---

## 10. Considerazioni AI (Evolutive)

L'argomento AI è aperto e verrà sviluppato in una fase successiva. Aree di applicazione identificate:

1. **Creazione offerta assistita**: suggerimento prodotti in base a tipologia evento, storico cliente, stagionalità
2. **Solleciti automatici**: generazione testo reminder preventivi scaduti, prioritizzazione
3. **Previsione disponibilità**: stima picchi di domanda, suggerimento acquisti
4. **Analisi danneggiamenti**: pattern su prodotti più soggetti a danneggiamento
5. **Chatbot / assistente**: interfaccia conversazionale per operatori

Il database e l'architettura sono progettati per raccogliere i dati necessari (storico ordini, preventivi, danneggiamenti, tempi noleggio) per alimentare questi modelli in futuro.

---

## 11. Piano di Sviluppo

### 11.1 Fasi

| Fase | Durata | Attività |
|---|---|---|
| **Fase 0 — Setup** | 1 settimana | Setup repo, ambiente dev (Docker), CI/CD base, scheletro frontend/backend, Prisma schema, migrazioni DB |
| **Fase 1 — Core** | 3 settimane | Auth + ruoli, CRUD clienti, CRUD catalogo (categorie + prodotti) |
| **Fase 2 — Noleggio** | 3 settimane | Preventivi (CRUD + wizard + PDF), Ordini (trasformazione + CRUD), gestione sconti |
| **Fase 3 — Movimenti** | 2 settimane | DDT (uscita/ingresso), cambio stato articoli, calendario disponibilità |
| **Fase 4 — Dashboard** | 1 settimana | KPI, report, vista solleciti, integrazione calendario in dashboard |
| **Fase 5 — Gestionale** | 2 settimane | Integrazione API gestionale (adapter pattern), sincronizzazione, log |
| **Fase 6 — Rifiniture** | 1 settimana | Testing E2E, responsive design, performance tuning, deploy VPS |
| **Totale** | **13 settimane** | |

### 11.2 Milestone

| # | Milestone | Criterio |
|---|---|---|
| M0 | Prototipo UI | Prototipo HTML interattivo con 8 schermate, Gantt, dati reali (✅ completato) |
| M1 | Setup completato | Frontend e backend in esecuzione in dev, DB migrato |
| M2 | Catalogo e clienti | CRUD completo clienti e prodotti con UI |
| M3 | Preventivi e ordini | Wizard preventivo funzionante, trasformazione in ordine |
| M4 | Movimenti | DDT uscita/ingresso confermano e cambiano stato articoli |
| M5 | Calendario eventi e dashboard | Vista eventi Gantt aggiornata, KPI corretti |
| M6 | Integrazione | Sync con gestionale funzionante (end-to-end) |
| M7 | Go-live | Deploy su VPS, test utente, documentazione |

---

## 12. Requisiti Non Funzionali

| Requisito | Specifica |
|---|---|
| **Performance** | API risponde in <500ms per il 95% delle richieste |
| **Disponibilità** | 99.5% uptime (fermo programmato notturno) |
| **Sicurezza** | HTTPS, JWT con refresh rotation, password bcrypt, rate limiting, input sanitization, CORS configurato |
| **Backup** | Backup DB giornaliero (PostgreSQL pg_dump), retention 30 giorni |
| **Logging** | Log strutturato (winston/pino), rotazione giornaliera |
| **Responsive** | UI fruibile da tablet (minimo 1024px di larghezza) |

---

## 13. Note Finali

- La specifica è **viva** e potrà essere raffinata durante lo sviluppo
- Le API del gestionale, quando definite, determineranno l'implementazione esatta del modulo di integrazione
- Il design UI/UX è stato validato tramite prototipo HTML interattivo (vedi sezione 14)
- L'architettura multi-tenant-ready permette di ospitare più società dello stesso gruppo su una singola istanza
- I PDF (preventivi, ordini, DDT) seguiranno il template grafico del brand aziendale

---

## 14. Prototipo Realizzato

### 14.1 File e Struttura

| File | Descrizione |
|---|---|
| `index.html` | Pagina launcher/overview con 8 card che linkano alle singole schermate del prototipo + note demo + footer |
| `app.html` | Prototipo principale — SPA completa con 8 pagine + 2 modali, supporto parametri URL (`?page=X`) + footer |
| `oltre-il-giardino-gestione-noleggi.html` | Copia di backup della SPA (stessa struttura di app.html) |
| `assets/images/` | 44 immagini prodotto/categoria/hero/logo scaricate da oltreilgiardino.biz |

**Parametri URL supportati:** `app.html?page=dashboard` | `catalogo` | `dettaglio-prodotto` | `calendario` | `preventivi` | `ordini` | `ddt` | `clienti`

### 14.2 Stack Prototipo

| Componente | Tecnologia |
|---|---|
| Rendering | HTML + CSS + JavaScript vanilla (nessuna dipendenza esterna) |
| Layout | CSS Grid + Flexbox |
| Icone | SVG inline nella sidebar (8 vettoriali: dashboard, catalogo, calendario, preventivi, ordini, DDT, clienti) |
| Navigazione | SPA con `showPage(id)` e classi `.page.active` — mappa nav index nel JS per evidenziare la voce sidebar corretta |
| Gantt | Engine JavaScript puro: costruzione dinamica DOM, drag-to-pan, scroll-by, scroll-to-commit, sparkline |
| Design system | CSS custom properties (`:root` tokens) con palette OKLch |

### 14.3 Design System Applicato

| Token | Valore | Uso |
|---|---|---|
| `--bg` | `oklch(97% 0.006 145)` | Sfondo applicazione |
| `--surface` | `oklch(100% 0 0)` | Card, sidebar, superfici |
| `--fg` | `oklch(18% 0.015 145)` | Testo primario |
| `--muted` | `oklch(50% 0.015 145)` | Testo secondario, label, intestazioni tabella |
| `--border` | `oklch(91% 0.008 145)` | Bordi, separatori riga |
| `--accent` | `oklch(52% 0.14 155)` | Colore azione primario (verde erba, richiama il brand) |
| `--accent-soft` | `color-mix(in oklch, var(--accent) 12%, transparent)` | Sfondo hover, today marker |
| `--fg-soft` | `color-mix(in oklch, var(--fg) 5%, transparent)` | Hover righe tabella, sfondo header Gantt |
| `--success` | `oklch(55% 0.15 145)` | Stato confermato / disponibile / in allestimento |
| `--warn` | `oklch(65% 0.15 80)` | Stato in corso / attenzione |
| `--danger` | `oklch(55% 0.18 25)` | Stato critico / esaurito / uscita DDT |

**Font stack:**
- Display: `'Iowan Old Style', 'Charter', Georgia, serif` — usato in topbar title, titoli card, sidebar brand
- Body: `-apple-system, BlinkMacSystemFont, 'Inter', 'Segoe UI', system-ui, sans-serif` — usato ovunque
- Mono: `'JetBrains Mono', 'IBM Plex Mono', ui-monospace, Menlo, monospace` — usato per numeri, KPI, label tabella, codici, pill, breadcrumbs

**Componenti CSS riusabili (definiti nel prototipo):**
- `.btn` / `.btn-primary` / `.btn-secondary` / `.btn-ghost` / `.btn-sm`
- `.card` / `.card-header`
- `.kpi-grid` / `.kpi-card` / `.kpi-label` / `.kpi-value`
- `.ds-table` (tabella dati standard)
- `.pill` / `.pill-green` / `.pill-amber` / `.pill-red` / `.pill-gray` / `.pill-blue`
- `.filter-bar` / `.input`
- `.cat-chips` / `.cat-chip`
- `.product-grid` / `.product-card` / `.product-img` / `.product-info`
- `.grid-2` / `.grid-3` / `.grid-2-1` / `.stack` / `.row` / `.row-between`
- `.cal-grid` / `.cal-head` / `.cal-day` / `.cal-event` / `.cal-qty`
- `.timeline` / `.timeline-step` (stato ordine)
- `.gantt-wrap` / `.gantt-toolbar` / `.gantt-viewport` / `.gantt` / `.gantt-bar` / `.gantt-legend`
- `.om-photos-grid` / `.om-photo-thumb` / `.om-photo-upload` / `.om-photo-zoom-overlay`
- `.om-ai-section` / `.om-ai-header` / `.om-ai-grid` / `.om-ai-field` / `.om-ai-stars`

### 14.4 Mappa Schermate

| # | Schermata | ID pagina | Riga sidebar | File prototype (righe) |
|---|---|---|---|---|
| 1 | Dashboard | `page-dashboard` | Principale → Dashboard | `oltre-il-giardino-gestione-noleggi.html:557-635` |
| 2 | Catalogo Prodotti | `page-catalogo` | Principale → Catalogo | `:638-675` |
| 3 | Dettaglio Prodotto | `page-dettaglio-prodotto` | (click card prodotto) | `:678-851` |
| 4 | Calendario Eventi (Gantt) | `page-calendario` | Principale → Calendario Eventi | `oltre-il-giardino-gestione-noleggi.html:1010-1100` |
| 5 | Preventivi | `page-preventivi` | Operativo → Preventivi | `:931-950` |
| 6 | Ordini | `page-ordini` | Operativo → Ordini | `:953-971` |
| 7 | DDT / Movimenti | `page-ddt` | Operativo → DDT / Movimenti | `:1038-1055` |
| 8 | Clienti | `page-clienti` | Anagrafiche → Clienti | `:1058-1077` |

### 14.5 Dati Demo

Il prototipo contiene dati realistici estratti da oltreilgiardino.biz:

| Entità | Quantità | Esempi |
|---|---|---|
| Categorie | 13 | Sedute, Tavoli, Tovagliato, Luci, Mobili, Banconi, Piante e Vasi, Pavimenti, Mise en place, Coperture, Accessori e sicurezza, Bimbi, Natale |
| Prodotti | 20 | Sedia Chiavari, Sedia Bristol, Sedia Ball, Poltrona Amal, Divano Acapulco, Tavolo rett. 180cm, Tavolo tondo, Tavolo Holland, Tavolo Bigjoy, Lampada Calobra, Nuvola, Neon, Lampadario Cristal, Candelabro, Lanterna Celine, Cornice LED, Libreria Scala, Valigia Vintage, Specchio Reflex, Cuscini damascati |
| Clienti | 7 | Rossi Srl, Bianchi Spa, Verdi & Figli, Neri Spa, Blu Events, Gialli Party, Arancio Catering |
| Ordini | 6 | #2026/040 — #2026/047 con stati diversi |
| Preventivi | 7 | #2026/026 — #2026/032 con 5 stati diversi |
| DDT | 5 | #2026/085 — #2026/089, uscita e ingresso |
| Impegni Gantt | 6 | Su 22 giorni (15 Giu → 6 Lug), con 4 stati |
| Immagini | 33 | 13 categorie + 15 prodotti + 2 hero + 1 logo + 2 grafiche |

### 14.6 Note per Sviluppo

**Cosa funziona nel prototipo:**
- Navigazione SPA completa tra 8 pagine + modale dettaglio ordine
- Gantt interattivo con drag-to-pan, scroll-by, scroll-to-commit, sparkline
- Filtri ricerca e selezione stato su Catalogo e Calendario Eventi
- Pill colore per stati (ordini, preventivi, DDT, prodotti)
- Timeline stato ordine a 6 step
- Responsive base (sidebar collapse a 920px)
- Foto allestimento con griglia thumbnail e zoom overlay con navigazione ◀/▶
- Questionario post-evento AI con stelle, select, input, textarea, tag AI

**Cosa manca e va implementato:**

| Componente | Fase | Note |
|---|---|---|
| Wizard "Nuovo Preventivo" | Prototipato | Modale 3 step: dati ordine → righe ordine (modale secondaria per ricerca catalogo con disponibilità 3 livelli) → riepilogo costi. Sconto per riga e globale.
| Foto allestimento | Prototipato | Griglia thumbnail + zoom overlay. Manca: upload reale, eliminazione, didascalia editabile, storage backend |
| Questionario AI | Prototipato | Form post-evento con stelle, select, input, tag. Manca: persistenza backend, calcolo AI reale, training model |
| Wizard "Nuovo Ordine" | Fase 2 | Trasformazione da preventivo o creazione diretta |
| Dettaglio Preventivo | Fase 2 | Vista completa preventivo con azioni (invia, archivia, trasforma) |
| Dettaglio Cliente | Fase 1 | Vista anagrafica con storico ordini e preventivi |
| Form creazione Cliente | Fase 1 | Inserimento anagrafica |
| Form modifica Prodotto | Fase 1 | Modifica info, prezzi, quantità |
| Wizard DDT (Uscita/Ingresso) | Fase 3 | Creazione movimento con selezione righe ordine e quantità |
| Dettaglio DDT | Fase 3 | Vista completa DDT con righe e stato |
| Conferma DDT | Fase 3 | Cambio stato articoli (Disponibile → In carico → In noleggio → Reso) |
| Paginazione tabella | Fase 1 | Componente paginazione client-side/server-side |
| Export CSV/PDF | Fase 4 | Download dati da tabella |
| Sidebar mobile | Fase 6 | Toggle hamburger per schermi <920px |
| Autenticazione | Fase 1 | Login page, JWT, refresh token, ruoli |
| Integrazione gestionale | Fase 5 | Adapter pattern per sync anagrafiche/ordini/movimenti |

---

## 15. Note Demo

Il prototipo include un **footer** in tutte le pagine (index e app) con:
- Nota "Demo dimostrativa" con avviso che i dati sono fittizi e alcune funzionalità potrebbero non essere operative
- Crediti: "Oltre Il Giardino — Noleggi Creativi · Prototipo ad alta fedeltà · Realizzato da Ugo Volpato AI Consultant"

Nell'index il footer è in fondo alla pagina dopo la griglia di card. Nell'app è posizionato sotto la sidebar e il contenuto principale.

---

*Documento generato il 24/06/2026 — Specifica Tecnica v1.9 — Prototipo Gestione Noleggi Oltre Il Giardino SRL*
