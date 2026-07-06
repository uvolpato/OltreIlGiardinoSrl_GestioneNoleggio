# Specifica Tecnica — Prototipo Gestione Noleggi Oltre Il Giardino SRL

> **Versione:** 2.2  
> **Data:** 06/07/2026  
> **Stato:** Specifica aggiornata v2.2 — IVA, sconto fisso, compositi, logistica, pagamenti, anagrafica/movimenti prodotto  
> **Prossimo passo:** Validazione prototipo → Sviluppo Fase 1

---

## 1. Introduzione

### 1.1 Contesto

Oltre Il Giardino SRL opera nel settore del noleggio di suppellettili, arredi, luci e allestimenti per eventi privati e aziendali (matrimoni, cerimonie, fiere, convegni). L'azienda dispone di un catalogo prodotti organizzato in categorie merceologiche.

**Architettura:**
- **Questo sistema (Gestione Noleggi)** gestisce il flusso operativo: preventivazione, ordini, tracciamento stato, disponibilità articoli, logistica fisica (contenitori, volumi, documenti logistici). Comunica lo stato degli ordini al gestionale esterno per la generazione dei documenti contabili.
- **Gestionale esterno** è il sistema contabile/fiscale (es. Zucchetti, TeamSystem) che genera i documenti contabili (DDT, fatture, note di credito) sulla base degli stati ordine comunicati da questo sistema, e restituisce le giacenze di magazzino aggiornate e valorizzate.

Si rende necessario un applicativo web dedicato che gestisca il flusso operativo completo: preventivazione, ordini, disponibilità, logistica fisica della merce, tracciamento stato articoli e comunicazione col gestionale per giacenze e documenti contabili.

### 1.2 Scopo del documento

Il presente documento costituisce la specifica tecnica per la realizzazione di un **prototipo funzionante** dell'applicativo. Sarà utilizzato come input per:

- La fase di **design** (UI/UX, wireframe)
- La successiva **specifica HTML**
- Lo **sviluppo** del frontend e backend

### 1.3 Glossario

| Termine | Descrizione |
|---|---|---|
| **Gestionale esterno** | Sistema ERP/contabile esistente che gestisce fiscalità, contabilità generale, valorizzazione magazzino e genera DDT/fatture/note credito a partire dagli stati ordine comunicati da questo sistema |
| **Questo sistema** | Gestione Noleggi: gestione di preventivi, ordini, disponibilità, logistica fisica. Comunica stati ordine al gestionale |
| **DDT Uscita** | Documento di Trasporto per carico merce — generato dal gestionale esterno, visibile in questo sistema come riferimento |
| **DDT Ingresso** | Documento di Trasporto per rientro merce — generato dal gestionale esterno, visibile in questo sistema come riferimento |
| **Fattura** | Documento fiscale emesso dal gestionale esterno. Questo sistema registra solo il riferimento (numero, data, importo) |
| **Preventivo** | Ipotesi di noleggio con impegno non vincolante per il periodo indicato |
| **Ordine** | Trasformazione del preventivo in impegno definitivo |
| **Articolo** | Prodotto a catalogo (es. "Sedia Chiavari bianca") |
| **Disponibilità** | Quantità di un articolo non impegnata in un dato intervallo di date |
| **Prodotto composito** | Articolo composto da più sotto-articoli (es. Bancone Pallet = piano + cavalletti × 2) |

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
| **Documenti sistema** | PDF generation per preventivi e conferme (PDFMake o Puppeteer) |  
| **Documenti contabili** | Generati dal gestionale esterno — questo sistema riceve riferimenti (codice, data, pdf_url) |
| **Container** | Docker + docker-compose (sviluppo) |
| **Deploy** | VPS tradizionale (Node + PM2, Nginx reverse proxy) |

### 2.2 Schema Architetturale

```
┌───────────────────────────────────────────────────────────────┐
│                     CLIENT (Browser)                           │
│  React SPA ─── React Query ─── Auth Context (JWT + Session)   │
└──────────────────────────┬────────────────────────────────────┘
                           │ HTTPS / REST JSON
                           ▼
┌───────────────────────────────────────────────────────────────┐
│                   REVERSE PROXY (Nginx)                        │
│         HTTPS termination, static files, load balancing       │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌───────────────────────────────────────────────────────────────┐
│                   BACKEND (Node.js + Express)                  │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐ │
│  │ Auth    │ │ Anagraf. │ │ Ordini   │ │ Generazione       │ │
│  │ Module  │ │ Module   │ │ Module   │ │ Documenti (PDF)   │ │
│  └─────────┘ └──────────┘ └──────────┘ └───────────────────┘ │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐ │
│  │ Catalogo│ │ DDT /    │ │ Calend.  │ │ Integrazione      │ │
│  │ Prodotti│ │ Fatture  │ │ Disp.    │ │ Gestionale Esterno│ │
│  └─────────┘ └──────────┘ └──────────┘ └───────────────────┘ │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Dashboard / Report / AI (Impegnato Probabile)           │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────┬────────────────────────────────────┘
                           │ Prisma ORM
                           ▼
┌───────────────────────────────────────────────────────────────┐
│                   PostgreSQL Database                          │
│  anagrafiche, prodotti, preventivi, ordini,                   │
│  DDT, fatture, movimenti, pagamenti, logistica               │
└───────────────────────────────────────────────────────────────┘

                    INTEGRAZIONE CON GESTIONALE ESTERNO
┌───────────────────────────────────────────────────────────────┐
│           GESTIONALE ESTERNO (es. Zucchetti/TeamSystem)       │
│                                                               │
│  ↓ Riceve da questo sistema (stati ordine + anagrafiche):    │
│    - Ordine confermato (con righe, date, cliente)            │
│    - Ordine in allestimento                                  │
│    - Ordine in corso (merce consegnata)                      │
│    - Ordine completato (merce rientrata)                     │
│    - Variazioni anagrafiche clienti                          │
│                                                               │
│  ↑ Invia a questo sistema:                                   │
│    - Giacenze aggiornate (quantità fisiche a magazzino)      │
│    - DDT generati (numero, data, tipo, pdf_url)              │
│    - Fatture emesse (numero, data, importo, xml_url)         │
│    - Note di credito                                         │
│    - Saldo cliente / situazione contabile                    │
│    - Piano dei conti e aliquote IVA                          │
└───────────────────────────────────────────────────────────────┘
```

### 2.3 Repository

- **frontend/**: `https://github.com/.../oltreilgiardino-frontend`
- **backend/**: `https://github.com/.../oltreilgiardino-backend`

### 2.4 Integrazione con Gestionale Esterno

Il sistema Gestione Noleggi **traccia gli stati dell'ordine** e li comunica al gestionale esterno, che è il **sistema autore dei documenti contabili** (DDT, fatture, note di credito). Il gestionale restituisce i riferimenti dei documenti generati e le giacenze aggiornate.

#### Flusso di scambio dati

```
┌──────────────────────────────────────────────────────────┐
│               QUESTO SISTEMA (Gestione Noleggi)           │
│                                                           │
│  Preventivo → Ordine → [stato: allestimento]             │
│                         [stato: in corso]                 │
│                         [stato: completato]               │
│                                                           │
│  Comunica al gestionale lo stato e le righe ordine       │
│  Riceve dal gestionale: giacenze, DDT, fatture           │
└──────────────┬────────────────────────┬───────────────────┘
               │  ↑ Stati ordine         │  ↓ Documenti + giacenze
               │  ↑ Anagrafiche          │  ↓ Riferimenti DDT/fatture
               ▼ API / Webhook / Export  ▼
┌──────────────────────────────────────────────────────────┐
│              GESTIONALE ESTERNO (contabilità/fiscale)     │
│                                                           │
│  Riceve: ordine con righe → genera DDT Uscita            │
│          ordine completato → genera DDT Ingresso         │
│          fine mese → genera Fattura / Nota Credito       │
│                                                           │
│  Invia: giacenze aggiornate, riferimenti DDT/fatture,    │
│         saldo cliente, aliquote IVA                      │
└──────────────────────────────────────────────────────────┘
```

#### Modalità di integrazione (da definire con il vendor del gestionale)

1. **API REST** — questo sistema notifica cambiamenti stato; il gestionale risponde con documenti generati e giacenze
2. **Webhook** — questo sistema invia POST su URL configurato del gestionale a ogni cambio stato
3. **CSV/JSON strutturato** — fallback: export/import manuale programmato

#### Tabella scambio dati

| Direzione | Dato | Trigger |
|---|---|---|
| Questo sistema → Gestionale | Ordine confermato (cliente, righe, date, importi) | Trasformazione preventivo → ordine |
| Questo sistema → Gestionale | Ordine in allestimento (merce in carico) | Cambio stato a `in_allestimento` |
| Questo sistema → Gestionale | Ordine in corso (merce consegnata) | Cambio stato a `in_corso` |
| Questo sistema → Gestionale | Ordine completato (merce rientrata, con qta danneggiate) | Cambio stato a `completato` |
| Questo sistema → Gestionale | Anagrafiche clienti (nuove/modificate) | Sincronizzazione periodica |
| Gestionale → Questo sistema | Giacenze magazzino aggiornate (prodotto → qta disponibile) | Batch giornaliero o on-demand |
| Gestionale → Questo sistema | DDT generato (numero, data, tipo, pdf_url) | Dopo conferma ordine in allestimento |
| Gestionale → Questo sistema | Fattura emessa (numero, data, importo, xml_url) | Dopo completamento ordine |
| Gestionale → Questo sistema | Saldo cliente / partitario | Su richiesta |

### 2.5 Multi-tenancy

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
                     ┌──────────────────────┴──────────────────────────┐
                     │   DOCUMENTO_RIFERIMENTO (documenti esterni)   │
                     │ id, ordine_id, tipo (ddt_uscita/ddt_ingresso/│
                     │ fattura/nota_credito), codice_doc, data_doc, │
                     │ importo?, pdf_url?, xml_url?,                │
                     │ note, created_at, updated_at                 │
                     └──────────────────────────────────────────────┘

                      │ 1
                      │ *
             ┌────────┴─────────────────────────────────────────────┐
             │            PAGAMENTO                                 │
             │ id, ordine_id, tipo (bonifico/assegno/contanti/     │
             │ carta/rid), data_scadenza, data_incasso?,           │
             │ importo, stato (da_incassare/incassato/parziale/    │
             │ scaduto), nota, created_at, updated_at              │
             └──────────────────────────────────────────────────────┘
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
- `quantita_indisponibile` INTEGER DEFAULT 0 (rotti/manutenzione)
- `prezzo_giorno` DECIMAL(10,2) (prezzo noleggio per 1 giorno)
- `prezzo_forfettario` DECIMAL(10,2) (prezzo forfettario per periodo)
- `prezzo_3gg` DECIMAL(10,2) (prezzo per 3 giorni)
- `prezzo_settimana` DECIMAL(10,2) (prezzo settimanale)
- `unita_misura` VARCHAR(20) DEFAULT 'pezzi'
- `peso_kg` DECIMAL(8,2) (per calcolo volumi e logistica)
- `dimensioni` VARCHAR(100) (es. "50×50×80 cm")
- `volume_m3` DECIMAL(8,3) (calcolato: L×P×H in m³)
- `fornitore_nome` VARCHAR(255)
- `fornitore_contatto` TEXT
- `data_acquisto` DATE
- `costo_acquisto` DECIMAL(10,2)
- `immagine_url` TEXT
- `immagini_galleria` JSON (array URL immagini extra)
- `composito` BOOLEAN DEFAULT false
- `attivo` BOOLEAN DEFAULT true
- `note` TEXT
- `positione_magazzino` VARCHAR(100) (scaffale/corsia/piano)
- `created_at`, `updated_at`

#### 3.2.4a prodotto_composito_riga (per prodotti composti)
- `id` UUID PK
- `prodotto_composito_id` UUID FK → prodotto (il prodotto composito principale)
- `prodotto_singolo_id` UUID FK → prodotto (il sotto-articolo)
- `quantita` INTEGER (es. 2 cavalletti per bancone)
- `note` TEXT
- `created_at`, `updated_at`

#### 3.2.4b contenitore (casse, ceste, palette, carrelli)
- `id` UUID PK
- `tenant_id` UUID FK → tenant
- `codice` VARCHAR(50)
- `tipo` ENUM('cassa', 'cesta', 'baule', 'pallet', 'carrello')
- `descrizione` TEXT
- `dimensioni_esterne` VARCHAR(100) (es. "120×80×100 cm")
- `peso_vuoto_kg` DECIMAL(8,2)
- `portata_kg` DECIMAL(8,2)
- `volume_utile_m3` DECIMAL(8,3)
- `attivo` BOOLEAN DEFAULT true
- `created_at`, `updated_at`

#### 3.2.4c prodotto_contenitore (associazione contenitori usati per un prodotto)
- `id` UUID PK
- `prodotto_id` UUID FK → prodotto
- `contenitore_id` UUID FK → contenitore
- `quantita_per_contenitore` INTEGER (es. 20 sedie per cassa)
- `note` TEXT

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
- `totale_imponibile` DECIMAL(10,2)
- `totale_iva` DECIMAL(10,2)
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
- `prezzo_manuale` BOOLEAN DEFAULT false (override manuale del prezzo)
- `tipo_prezzo` ENUM('giorno', 'forfait', 'durata')
- `sconto_pct` DECIMAL(5,2) DEFAULT 0
- `sconto_importo` DECIMAL(10,2) DEFAULT 0
- `aliquota_iva` DECIMAL(5,2) DEFAULT 22.00 (percentuale — 22, 10, 4, 0 per esenti)
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
- `totale_imponibile` DECIMAL(10,2)
- `totale_iva` DECIMAL(10,2)
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
- `prezzo_manuale` BOOLEAN DEFAULT false (override manuale del prezzo)
- `tipo_prezzo` ENUM('giorno', 'forfait', 'durata')
- `sconto_pct` DECIMAL(5,2) DEFAULT 0
- `sconto_importo` DECIMAL(10,2) DEFAULT 0
- `aliquota_iva` DECIMAL(5,2) DEFAULT 22.00 (percentuale — 22, 10, 4, 0 per esenti)
- `totale_riga` DECIMAL(10,2)
- `note` TEXT
- `created_at`

#### 3.2.10 documento_riferimento
Tabella che memorizza i riferimenti ai documenti contabili (DDT, fatture, note credito) **generati dal gestionale esterno**. Questo sistema non genera documenti contabili, ma ne riceve i riferimenti per esposizione nell'interfaccia utente.

- `id` UUID PK
- `ordine_id` UUID FK → ordine
- `tipo` ENUM('ddt_uscita', 'ddt_ingresso', 'fattura', 'nota_credito')
- `codice_doc` VARCHAR(50) (numero documento assegnato dal gestionale, es. "DDT-2026-085")
- `data_doc` DATE (data di emissione del documento)
- `importo` DECIMAL(10,2) (importo del documento, se disponibile)
- `pdf_url` TEXT (URL al PDF generato dal gestionale, se accessibile)
- `xml_url` TEXT (URL al file XML fattura elettronica, solo per fatture)
- `note` TEXT (note sul documento)
- `created_at`, `updated_at`

#### 3.2.11 logistica_movimento
Tabella per il tracciamento logistico interno (non fiscale) — casse, pallet, carrelli caricati per l'evento. Non sostituisce il DDT (generato dal gestionale) ma serve per la gestione operativa del magazzino fisico.

- `id` UUID PK
- `ordine_id` UUID FK → ordine
- `tipo` ENUM('carico', 'scarico')
- `data_movimento` TIMESTAMP
- `contenitore_id` UUID FK → contenitore (cassa, pallet, carrello usato)
- `operatore_id` UUID FK → user
- `note` TEXT
- `created_at`

#### 3.2.11a logistica_movimento_riga
- `id` UUID PK
- `movimento_id` UUID FK → logistica_movimento
- `prodotto_id` UUID FK → prodotto
- `quantita` INTEGER
- `condizione` ENUM('ottimo', 'buono', 'danneggiato') DEFAULT 'ottimo'
- `quantita_danneggiata` INTEGER DEFAULT 0
- `note` TEXT

#### 3.2.12 pagamento
- `id` UUID PK
- `ordine_id` UUID FK → ordine
- `tipo` ENUM('bonifico', 'assegno', 'contanti', 'carta', 'rid')
- `data_scadenza` DATE
- `data_incasso` DATE (NULLABLE se non ancora incassato)
- `importo` DECIMAL(10,2)
- `stato` ENUM('da_incassare', 'incassato', 'parziale', 'scaduto')
- `nota` TEXT
- `created_at`, `updated_at`

#### 3.2.13 documento_logistico
- `id` UUID PK
- `ordine_id` UUID FK → ordine
- `consegna_ztl` BOOLEAN DEFAULT false
- `ztl_orari` VARCHAR(255) (es. "Lun-Ven 7:30-9:30, 11:00-13:00")
- `area_c` BOOLEAN DEFAULT false
- `mulino_in_sede` BOOLEAN DEFAULT false
- `personale_carico_scarico` BOOLEAN DEFAULT false
- `n_persone_necessarie` INTEGER
- `piano_consegna` VARCHAR(50) (es. "piano terra", "1° piano con ascensore")
- `ascensore_presente` BOOLEAN DEFAULT false
- `dimensioni_ascensore` VARCHAR(100)
- `note_trasportatore` TEXT
- `percorso_consigliato` TEXT
- `pdf_url` TEXT (PDF esportabile)
- `created_at`, `updated_at`

#### 3.2.14 ordine_foto
- `id` UUID PK
- `ordine_id` UUID FK → ordine
- `url` TEXT
- `caption` VARCHAR(255)
- `tipo` ENUM('allestimento', 'particolare', 'danno')
- `created_by` UUID FK → user
- `created_at`

#### 3.2.15 ordine_questionario
- `id` UUID PK
- `ordine_id` UUID FK → ordine UNIQUE
- `qualita_stelle` INTEGER CHECK (1-5)
- `venue_adatto` BOOLEAN
- `adattabilita_venue` TEXT
- `accesso_carico` TEXT
- `puntualita_consegna` INTEGER CHECK (1-5)
- `danni_riscontrati` TEXT
- `feedback_cliente` TEXT
- `miglioramenti_suggeriti` TEXT
- `tag_ai` TEXT[] (array di tag per training AI)
- `compilato_da` UUID FK → user
- `compilato_il` TIMESTAMP

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
| PUT | `/ordini/:id/stato` | Cambia stato ordine (comunicato al gestionale) |
| DELETE | `/ordini/:id` | Annulla ordine |

#### 4.2.6 Logistica (movimenti interni)

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/logistica/movimenti` | Lista movimenti logistici interni (carico/scarico contenitori) |
| GET | `/logistica/movimenti/:id` | Dettaglio movimento logistico con righe |
| POST | `/logistica/movimenti` | Crea movimento logistico (carico/scarico casse/pallet) |
| GET | `/logistica/contenitori` | Anagrafica contenitori (casse, pallet, carrelli) |
| POST | `/logistica/contenitori` | Crea contenitore |

#### 4.2.6b Documenti Riferimento (dal gestionale)

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/ordini/:id/documenti` | Documenti associati a un ordine (DDT, fatture — ricevuti dal gestionale) |
| POST | `/integrazione/ricevi-documento` | Webhook per ricevere documento contabile dal gestionale (DDT/fattura/nota credito) |

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

### 5.1 Flusso Preventivo → Ordine → Stati → Completato

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
   → Questo sistema comunica l'ordine confermato al gestionale
   → Il gestionale genera DDT Uscita e lo comunica a questo sistema
     (riferimento salvato in documento_riferimento)

3. ALLESTIMENTO / CARICO MERCE
   ────────────────────────────
   In prossimità dell'evento:
   → Operatore/magazziniere registra carico logistico in logistica_movimento
     (casse, pallet, carrelli — non fiscale, solo tracciamento interno)
   → Ordine passa a IN_ALLESTIMENTO
   → Stato comunicato al gestionale

4. CONSEGNA / EVENTO IN CORSO
   ───────────────────────────
   Merce trasportata all'evento e allestita
   → Ordine passa a IN_CORSO
   → Stato comunicato al gestionale
   → Il gestionale conferma DDT Uscita (aggiorna riferimento)

5. RIENTRO / SMONTAGGIO
   ─────────────────────
   Termine evento / smontaggio:
   → Ordine passa a IN_RIENTRO
   → Operatore registra rientro logistico in logistica_movimento
   → Per ogni articolo, indica quantità RESE e DANNEGGIATE

6. COMPLETATO
   ───────────
   → Ordine passa a COMPLETATO
   → Stato comunicato al gestionale
   → Il gestionale:
     • Genera DDT Ingresso (riferimento → documento_riferimento)
     • Aggiorna giacenze di magazzino valorizzate
     • Se necessario, emette fattura / nota credito
   → Questo sistema riceve giacenze aggiornate via integrazione
   → La disponibilità per nuovi preventivi si basa su:
     impegni_ordini + giacenze_gestionale - soglia_sicurezza
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
| **operatore** | Gestione clienti, preventivi, ordini, logistica. NON può modificare prezzi catalogo, utenti, impostazioni |
| **magazziniere** | Solo logistica (movimenti carico/scarico), consultazione ordini, dashboard |

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
│ │ Logistica│ │                                              │  │
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
| **Operativo** | Logistica | `page-logistica` | Sidebar → "Logistica" |
| **Anagrafiche** | Clienti | `page-clienti` | Sidebar → "Clienti" |

### 8.3 Descrizione Pagina per Pagina

---

#### 8.3.1 Dashboard (`page-dashboard`)

**Purpose:** Vista operativa quotidiana — l'operatore apre la mattina e vede subito cosa è urgente.

**Layout:**
- Topbar: titolo "Dashboard" + timestamp ultimo aggiornamento + pulsante "Nuovo Preventivo"
- Contenuto: griglia KPI (4 colonne) → griglia 2:1 (tabella ordini + pannelli laterali) → tabella Prossime Spedizioni

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

**Sezione C — Tabella "Prossime Spedizioni":**

| Colonna | Tipo | Note |
|---|---|---|
| Ordine | `#YYYY/NNN` monospace link | |
| Cliente | testo | |
| Evento | testo | |
| Allestimento | data `DD/MM` | |
| Qtà | monospace | "89 pz · 320 kg" |
| Stato | pill colore | |

**API necessarie:**
- `GET /dashboard/riepilogo` → KPI cards
- `GET /dashboard/impegni-prossimi` → tabella ordini imminenti
- `GET /dashboard/preventivi-scaduti` → lista preventivi in scadenza
- `GET /dashboard/disponibilita-critica` → prodotti sotto soglia
- `GET /dashboard/prossime-spedizioni?limit=5` → prossime uscite

**Interazioni:**
- Click su riga ordine → apre modale dettaglio ordine
- Click su "Vedi tutti →" (spedizioni) → naviga a `page-logistica`
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

**Purpose (v2.2):** Vista completa di un prodotto con schede multiple: Dettaglio (disponibilità, Gantt), Anagrafica (fornitore, posizione, dimensioni), Movimenti (entrate/uscite per evento con saldo progressivo).

**Layout:**
- Topbar: breadcrumb "← Catalogo" + nome prodotto + pulsanti "Modifica" e "Verifica Disponibilità"
- Griglia 2:1 superiore (info + stats)
- Gantt chart interattivo (sezione piena larghezza)
- Legenda impegni + riepilogo disponibilità (griglia 2 colonne)

**Sezione A — Tab Navigator (v2.2):** Il dettaglio prodotto ora ha 3 tab selezionabili nella parte superiore:
- **Dettaglio** (active default) — vista originale con immagine, info, stats, Gantt
- **Anagrafica** — scheda fornitore (nome, contatti, data acquisto, costo acquisto), posizione magazzino (fila, scaffale, corsia), dimensioni (L×P×H cm, peso unitario), note logistiche
- **Movimenti** — tabella cronologica entrate/uscite per evento con data, tipo (Uscita/Rientro/Acquisto/Manutenzione), quantità, saldo progressivo

**Sezione B — Griglia 2:1 (Immagine + Info | Stats)**

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

**Purpose (v2.2):** Wizard di creazione preventivo in **4 step**, accessibile da Dashboard e Preventivi.

**Struttura:**
- Overlay fullscreen con backdrop blur, chiusura con ✕, click fuori, o Esc
- Header: titolo "Nuovo Preventivo" + pulsante ✕
- Indicatore step: 4 pill cerchiati (1 Dati ordine → 2 Righe ordine → 3 Logistica → 4 Riepilogo)
- Body scrollabile + footer con pulsanti "← Indietro" / "Continua →" / "Crea preventivo"

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

**Step 2 — Righe ordine (v2.2 arricchito):**

- **Pulsante "+ Aggiungi articoli"** → apre modale secondaria "Righe ordine"
- **Griglia articoli selezionati:** tabella con colonne:
  - Thumbnail, Nome + Codice + badge disponibilità (`V N / T N / P N / Tot: N`), Qtà (input number con warning disponibilità), Tariffa (select: Giorno/3gg/Settimana/Forfait), Prezzo unitario (con toggle manual ✏️), IVA (select: 22% / 4% / Esente), Sconto % (input), Sconto importo fisso (input), Note prodotto (input text), Totale riga, Pulsante rimuovi ✕
- **Badge disponibilità per riga:** sotto nome e codice, 4 badge inline:
  - `V N` (verde) = pezzi liberi considerando solo impegnato vero
  - `T N` (arancio) = pezzi liberi considerando impegnato teorico
  - `P N` (blu) = pezzi liberi considerando impegnato probabile
  - `/ Tot: N` = quantità totale a magazzino
- **Prodotti composti (v2.2):** se l'articolo è un `composito` (es. Bancone Pallet), sotto la riga principale appaiono sotto-righe indentate con codice, nome, qty, prezzo unitario, sotto-totale
- **Warning disponibilità sulla qtà:** quando la quantità inserita supera una soglia di disponibilità, l'input mostra:
  - Bordo + sfondo colorato (verde/arancio/blu) in base al livello superato
  - Pilloletta sotto l'input: "▲ Supera vero/teorico/probabile (N liberi)"
  - Si aggiorna dinamicamente quando cambia la data evento o i giorni
- **Sconto globale:** input percentuale applicato a tutte le righe
- **Riepilogo:** Subtotale, Sconto righe, Sconto globale, Totale (con dettaglio IVA)
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

**Step 3 — Logistica (nuovo v2.2):**
- **Selezione contenitori:** checkbox + input qty per tipi predefiniti (Cassa media/grande, Baule cristallo, Pallet EU/industriale, Carrello appendiabiti/piatto)
- **Calcolatore automatico:** volume totale, peso totale, colli, pallet necessari, viaggi stimati (formula: volume / portata / capienza furgone)
- **Vincoli consegna:** ZTL (testo libero), serve muletto? (Sì/No toggle), personale carico/scarico (input), note trasportatore (textarea)

**Step 4 — Riepilogo (v2.2 arricchito):**
- Card "Dati Evento": data, evento, cliente, venue, referente, giorni, note
- Card "Riepilogo Costi": subtotali per aliquota IVA + totale IVA + totale finale
- Card "Logistica": contenitori selezionati, volume, peso, colli, pallet, viaggi, vincoli consegna
- Tabella righe complete: articolo, qtà, tariffa, prezzo ud., IVA, sconto %, sconto fisso, totale riga

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
| Confermato | `pill-blue` | Blu | → Allestimento |
| In allestimento | `pill-green` | Verde | → In corso |
| In corso | `pill-amber` | Arancio | → Rientro |
| In rientro | `pill-amber` | Arancio | → Completato |
| Completato | `pill-green` | Verde | (finale) |

**API necessarie:**
- `GET /ordini?page=N&limit=20&stato=X&q=testo` → lista paginata

**Interazioni:**
- Click su riga ordine → apre modale dettaglio ordine (`openOrderModal()`)
- Click "+ Nuovo Ordine" → creazione diretta (Fase 2)

---

#### 8.3.7 Dettaglio Ordine (modale `order-modal`)

**Purpose:** Vista completa di un ordine — timeline stato, dati evento, righe ordine, riepilogo costi, documenti contabili dal gestionale, foto allestimento, questionario post-evento per AI. Aperta come modale sovrapposta alla pagina corrente.

**Layout:**
- Overlay con backdrop blur, chiusura con ✕, click fuori o tasto Esc
- Header: numero ordine + pill stato + pulsante "Scarica PDF"
- Timeline stato (componente a tutta larghezza)
- Griglia 2 colonne (dati evento + riepilogo costi)
- Tabella righe ordine
- Tabella documenti contabili (dal gestionale)
- Griglia foto allestimento (con zoom)
- Questionario post-evento AI

**Sezione A — Timeline Stato**

Componente visivo a 6 step con linee connesse:

```
●───────────●───────────●───────────○───────────○
Confermato   Allestimento In corso   Rientro    Completato
(done)       (done)      (active)     (pending)  (pending)
```

| Step | Stati corrispondenti | Colore when done | Colore when active |
|---|---|---|---|---|
| Confermato | `confermato` | Verde | — |
| Allestimento | `in_allestimento` (comunicato al gestionale → DDT Uscita) | Verde | Accent + glow |
| In corso | `in_corso` | — | Accent |
| Rientro | `in_rientro` (comunicato al gestionale → DDT Ingresso) | — | Accent |
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

**Sezione D — Documenti Contabili (dal Gestionale):**

Riferimenti ai documenti generati dal gestionale esterno per questo ordine:

| Colonna | Tipo | Note |
|---|---|---|
| Documento | monospace `DDT-2026-085` | Codice assegnato dal gestionale |
| Tipo | pill | Uscita / Ingresso / Fattura |
| Data | monospace | Data emissione documento |
| Importo | monospace | Solo per fatture |
| PDF | link | "PDF →" se disponibile (hostato dal gestionale) |

**API necessarie:**
- `GET /ordini/:id` → dati ordine + righe
- `GET /ordini/:id/documenti` → documenti ricevuti dal gestionale

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

**Sezione G — Pagamenti (nuovo v2.2):**

Tabella storico pagamenti per l'ordine:

| Colonna | Tipo | Formato |
|---|---|---|
| Tipo | testo | Bonifico / Assegno / RID / Contanti |
| Data pagamento | monospace | `DD/MM/YYYY` o `—` |
| Importo | monospace grassetto | `€ X.XXX,XX` |
| Stato | pill | `pagato` (verde) / `in_attesa` (giallo) |
| Scadenza | monospace | `DD/MM/YYYY` |

Pulsanti azione: "+ Registra pagamento", "Storico cliente" (entrambi placeholder).

**Struttura dati:** ogni ordine ha `payments: [{ type, date, amount, status, statusCls, scadenza }]`

**Sezione H — Documento Logistico (nuovo v2.2):**

Form con campi editabili (input):

| Campo | Tipo |
|---|---|
| ZTL / Aree regolamentate | input text |
| Serve muletto | input text (Sì/No) |
| Personale carico/scarico | input text |
| Note trasportatore | input text |

Pulsante "📄 Scarica PDF" (placeholder).

**Struttura dati:** ogni ordine ha `docLog: { ztl, muletto, personale, note }`

**API necessarie:**
- `GET /ordini/:id` → dati ordine + righe
- `GET /ordini/:id/documenti` → documenti contabili dal gestionale
- `GET /ordini/:id/foto` → lista foto allestimento
- `POST /ordini/:id/foto` → upload foto (multipart/form-data)
- `GET /ordini/:id/questionario` → dati questionario AI
- `PUT /ordini/:id/questionario` → salva/aggiorna questionario
- `GET /ordini/:id/pagamenti` → storico pagamenti ordine
- `POST /ordini/:id/pagamenti` → registra nuovo pagamento
- `GET /ordini/:id/documento-logistico` → documento logistico
- `PUT /ordini/:id/documento-logistico` → aggiorna documento logistico

**Interazioni:**
- Click ✕ / click fuori / Esc → chiudi modale
- Click "Scarica PDF" → download PDF ordine
- Click "PDF →" su documento → apre PDF documento (hostato dal gestionale)
- Click su foto → zoom overlay con navigazione ◀/▶
- Click ✕ / fuori / Esc (zoom) → chiudi zoom
- Click stelle → aggiorna rating
- Click "+ Aggiungi foto" → placeholder upload
- Click "Salva questionario" → conferma "✓ Salvato"
- Click "+ tag" → placeholder aggiunta tag
- La modale è accessibile da: tabella ordini, barre Gantt Dashboard, sidebar ordini Gantt

---

#### 8.3.8 Logistica (`page-logistica`)

**Purpose:** Gestione logistica fisica — tracciamento contenitori (casse, pallet, carrelli), volumi, pesi e documentazione per il trasporto. I documenti contabili (DDT, fatture) sono generati dal gestionale esterno e visibili in questa pagina come riferimenti.

**Layout:**
- Topbar: titolo "Logistica" + pulsante "+ Movimento Carico/Scarico"
- Riquadro "Prossime Spedizioni" — ordini imminenti (stato `in_allestimento`) con badge peso/volume
- Sezione "Contenitori" — anagrafica casse/pallet/carrelli
- Sezione "Documenti dal Gestionale" — tabella riferimenti DDT/fatture ricevuti

**Sezione A — Prossime Spedizioni:**

| Colonna | Tipo | Formato |
|---|---|---|
| Ordine | monospace link | `#YYYY/NNN` |
| Cliente — Evento | testo | |
| Allestimento | monospace | `DD/MM HH:MM` |
| Periodo noleggio | monospace | `DD/MM → DD/MM` |
| Articoli | testo | "6 righe · 89 pz" |
| Peso stimato | monospace | "320 kg" |
| Volume stimato | monospace | "4,2 m³" |
| ZTL / Note log. | testo | "ZTL Lun-Ven 7:30-9:30" |
| Azioni | — | "Piano di Carico" → |

**Sezione B — Anagrafica Contenitori:**

| Colonna | Tipo | Formato |
|---|---|---|
| Codice | monospace | `CST-001` |
| Tipo | pill | Cassa / Pallet / Carrello / Cesta |
| Dimensioni | testo | `120×80×70 cm` |
| Portata max | monospace | "500 kg" |
| Peso | monospace | "15 kg" |
| Assegnato a | link | Ordine `#YYYY/NNN` |
| Stato | pill | `pill-blue` In uso / `pill-green` Libero |

**Sezione C — Documenti dal Gestionale:**

Tabella che mostra i riferimenti ai documenti contabili generati dal gestionale:

| Colonna | Tipo | Note |
|---|---|---|
| Documento | monospace | `DDT-2026-085` (codice_doc dal gestionale) |
| Tipo | pill | `pill-red` Uscita / `pill-green` Ingresso / `pill-blue` Fattura |
| Ordine | monospace link | `#YYYY/NNN` |
| Data | monospace | `DD/MM/YYYY` |
| Importo | monospace | Solo per fatture |
| PDF | link | "PDF →" se pdf_url disponibile |

**API necessarie:**
- `GET /ordini?stato=in_allestimento` → prossime spedizioni
- `GET /logistica/contenitori` → anagrafica contenitori
- `GET /ordini/:id/documenti` → documenti contabili ricevuti dal gestionale

**Interazioni:**
- Click su "Piano di Carico" → apre modale documento logistico (ZTL, orari, personale)
- Click su "PDF →" → apre PDF del documento (se presente, hostato dal gestionale)
- Click "+ Movimento Carico/Scarico" → registra carico/scarico contenitori per ordine

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

**Sezione aggiuntiva v2.2 — Storico Pagamenti Cliente:**

Tabella con gli ultimi 6 movimenti di pagamento del cliente selezionato (Rossi Srl nel prototipo):

| Colonna | Tipo | Formato |
|---|---|---|
| Data | monospace | `DD/MM/YYYY` o `—` |
| Tipo | testo | Bonifico / Assegno / RID / Contanti |
| Ordine | monospace link | `#YYYY/NNN` |
| Evento | testo | Nome evento |
| Importo | monospace grassetto | `€ X.XXX,00` |
| Stato | pill | `pagato` (verde) / `in_attesa` (giallo) |
| Scadenza | monospace | `DD/MM/YYYY` |

**API necessarie:**
- `GET /clienti?page=N&limit=20&q=testo&stato=X` → lista paginata
- `POST /integrazione/sync-clienti` → sincronizzazione gestionale
- `GET /clienti/:id/pagamenti?limit=N` → storico pagamenti cliente

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

## 11. Funzionalità Richieste dal Cliente (emerse dalla review prototipo)

### 11.1 Preventivi — Integrazioni richieste

| Funzionalità | Stato attuale | Modifica necessaria |
|---|---|---|
| **Logistica nel preventivo** | ✅ **Prototipato v2.2** | Step 3 dedicato: selezione contenitori (casse/pallet/carrelli), calcolatore volume/peso/colli, form vincoli consegna (ZTL, orari, muletti, personale) |
| **Modifica manuale importi** | ✅ **Prototipato v2.2** | Toggle ✏️ per riga → sblocca input prezzo manuale e disabilita calcolo automatico da tariffa |
| **Modifica immagini prodotto** | Non permessa | Le immagini sono dell'anagrafica prodotto, modificabili solo in anagrafica (non in fase di preventivo) |
| **Prodotti composti** | ✅ **Prototipato v2.2** | Nuovo tipo `composito`: articolo padre con sotto-righe (es. Bancone Pallet = piano legno + cavalletti × 2). In preventivo si espande con proprie qtà, prezzi unitari e sotto-totale. Aggiunto Bancone Pallet a catalogo |
| **Note ai singoli prodotti** | ✅ **Prototipato v2.2** | Campo `note` testuale per ogni riga, visibile in tabella e riepilogo |
| **Sconto per riga** | ✅ **Prototipato v2.2** | % + **importo fisso** per riga |
| **Sconto globale** | Già presente | Nessuna modifica |
| **IVA / non IVA** | ✅ **Prototipato v2.2** | Nuova colonna "IVA" per riga (22%, 4%, esente). Riepilogo IVA con dettaglio per aliquota (imponibile + IVA per ciascuna) + totali finali |
| **Pagamenti e storico** | ✅ **Prototipato v2.2** | Nuova sezione Pagamenti nell'ordine + storico pagamenti nell'anagrafica cliente (tabella con data, tipo, importo, stato, scadenza) + pulsanti "Registra pagamento" e "Storico cliente" |
| **Generazione DDT/Fattura** | **Non di competenza di questo sistema** | I documenti contabili (DDT, fatture, note credito) sono **generati dal gestionale esterno**. Questo sistema comunica lo stato ordine, il gestionale genera i documenti e li restituisce come riferimenti (codice, data, pdf_url). Vedi §2.4 per lo schema d'integrazione. |

### 11.2 Catalogo Prodotti — Integrazioni richieste

| Funzionalità | Stato attuale | Modifica necessaria |
|---|---|---|
| **Anagrafica dettagliata** | ✅ **Prototipato v2.2** | Nuova scheda "Anagrafica" nel dettaglio prodotto: fornitore (nome, contatti, data acquisto, costo acquisto), posizione magazzino, dimensioni/peso, note logistiche |
| **Tabella entrate/uscite per evento** | ✅ **Prototipato v2.2** | Nuova tab "Movimenti": tabella cronologica con evento/qtà/tipo (uscita/rientro/acquisto/manutenzione), saldo progressivo |

### 11.3 Logistica — Nuove funzionalità

| Funzionalità | Descrizione |
|---|---|
| **Casse / Ceste / Pallet** | ✅ **Prototipato v2.2** — Nuova anagrafica "Contenitori" nella pagina Logistica: 7 tipi (cassa media/grande, baule cristallo, pallet EU/industriale, carrello appendiabiti/piatto) con codice, tipo, dimensioni, volume, peso, carico max, quantità. Associabili ai prodotti in fase di preventivo (Step 3) |
| **Calcolo volumi** | ✅ **Prototipato v2.2** — Calcolatore automatico nel wizard: volume totale, peso totale, colli, pallet necessari, viaggi stimati. Mostrato nel riepilogo preventivo |
| **Documento logistico** | ✅ **Prototipato v2.2** — Form nell'ordine: ZTL (libero testo), serve muletto (Sì/No), personale carico/scarico, note trasportatore. Pulsante "Scarica PDF" (placeholder). Dati via `docLog` nell'oggetto ordine |

### 11.4 Offerta / Carrello Sito

| Funzionalità | Descrizione |
|---|---|
| **Volume totale spedizione** | Mostrato nel riepilogo preventivo e nella stampa PDF |
| **Codice univoco da carrello sito** | Nuova pagina "Offerte da Sito": lista preventivi generati da carrello web con codice univoco identificativo. Da qui si apre il wizard preventivo precompilato con i dati del carrello |

### 11.5 Note Operative

| Funzionalità | Descrizione |
|---|---|
| **Materiali lasciati al cliente** | Nuovo campo "Materiali lasciati in sede" nella sezione Logistica: imballi, accessori, carrellini, quantità, data previsto rientro |
| **Foto allestimento** | Già presente nella modale ordine. Estendere con upload e cattura da dispositivo mobile |
| **Questionario AI** | Già presente nella modale ordine. I dati raccolti alimentano il calcolo impegnato probabile |

### 11.6 Priorità Implementazione

| Priorità | Funzionalità | Fase |
|---|---|---|
| P0 | Logistica (Step 3 + Volume) | ✅ **Prototipato v2.2** |
| P0 | IVA / esenzione per riga | ✅ **Prototipato v2.2** |
| P0 | Sconto importo fisso per riga | ✅ **Prototipato v2.2** |
| P1 | Prodotti composti (anagrafica + espansione) | ✅ **Prototipato v2.2** |
| P1 | Pagamenti e storico | ✅ **Prototipato v2.2** |
| P1 | Anagrafica dettagliata + Movimenti catalogo | ✅ **Prototipato v2.2** |
| P2 | Documento logistico PDF | Fase 2 (placeholder presente) |
| P2 | Documenti contabili (ricezione riferimenti dal gestionale) | Fase 2 |
| P2 | Integrazione gestionale (sync stati ordine) | Fase 2 |
| P2 | Note operative materiali lasciati | Fase 2 |
| P3 | Codice univoco carrello sito → offerta | Fase 3 |

---

## 12. Piano di Sviluppo

### 12.1 Fasi

| Fase | Durata | Attività |
|---|---|---|
| **Fase 0 — Setup** | 1 settimana | Setup repo, ambiente dev (Docker), CI/CD base, scheletro frontend/backend, Prisma schema, migrazioni DB |
| **Fase 1 — Core** | 3 settimane | Auth + ruoli, CRUD clienti, CRUD catalogo (categorie + prodotti + anagrafica + compositi + contenitori) |
| **Fase 2 — Preventivi** | 3 settimane | Wizard completo (Step 1-2-2b-3), IVA, sconti per riga/globali, override prezzo, note riga, PDF |
| **Fase 3 — Ordini e Logistica** | 3 settimane | Trasformazione preventivo→ordine, stati ordine, logistica (contenitori, volumi, documento logistico), pagamenti, materiali lasciati, integrazione gestionale (sync stati ordine) |
| **Fase 4 — Dashboard** | 1 settimana | KPI, report, Gantt eventi, vista solleciti |
| **Fase 5 — Gestionale** | 2 settimane | Integrazione API gestionale (adapter pattern), sincronizzazione, log |
| **Fase 6 — Rifiniture** | 1 settimana | Testing E2E, responsive design, performance tuning, deploy VPS |
| **Totale** | **14 settimane** | |

### 12.2 Milestone

| # | Milestone | Criterio |
|---|---|---|
| M0 | Prototipo UI | Prototipo HTML interattivo con 8 schermate, Gantt, dati reali (✅ completato) |
| M1 | Setup completato | Frontend e backend in esecuzione in dev, DB migrato |
| M2 | Catalogo e clienti | CRUD completo clienti, prodotti, categorie, compositi, contenitori |
| M3 | Preventivi | Wizard preventivo completo con IVA, sconti, logistica, PDF stampabile |
| M4 | Ordini e Logistica | Trasformazione ordine, cambio stato, logistica contenitori/volumi, documento logistico |
| M5 | Pagamenti e documenti | Gestione pagamenti, ricezione riferimenti DDT/fatture dal gestionale |
| M6 | Dashboard e calendario | Vista eventi Gantt aggiornata, KPI corretti, solleciti |
| M7 | Integrazione gestionale | Sync stati ordine, ricezione giacenze e documenti, adapter pattern |
| M8 | Go-live | Deploy su VPS, test utente, documentazione |

---

## 13. Requisiti Non Funzionali

| Requisito | Specifica |
|---|---|
| **Performance** | API risponde in <500ms per il 95% delle richieste |
| **Disponibilità** | 99.5% uptime (fermo programmato notturno) |
| **Sicurezza** | HTTPS, JWT con refresh rotation, password bcrypt, rate limiting, input sanitization, CORS configurato |
| **Backup** | Backup DB giornaliero (PostgreSQL pg_dump), retention 30 giorni |
| **Logging** | Log strutturato (winston/pino), rotazione giornaliera |
| **Responsive** | UI fruibile da tablet (minimo 1024px di larghezza) |

---

## 14. Note Finali

- La specifica è **viva** e potrà essere raffinata durante lo sviluppo
- Le API del gestionale, quando definite, determineranno l'implementazione esatta del modulo di integrazione
- Il design UI/UX è stato validato tramite prototipo HTML interattivo (vedi sezione 15)
- L'architettura multi-tenant-ready permette di ospitare più società dello stesso gruppo su una singola istanza
- I PDF (preventivi, ordini) seguiranno il template grafico del brand aziendale. I documenti contabili (DDT, fatture) sono generati dal gestionale esterno con il suo template.

---

## 15. Prototipo Realizzato

### 15.1 File e Struttura

| File | Descrizione |
|---|---|---|
| `index.html` | Pagina launcher/overview con 8 card che linkano alle singole schermate del prototipo + note demo + footer *(non aggiornato in v2.2 — puntare direttamente ad app.html)* |
| `app.html` | Prototipo principale — SPA completa con 9 pagine (Dashboard, Catalogo, Dettaglio Prodotto, Calendario Eventi, Preventivi, Ordini, Logistica, Clienti) + 4 aree modali (dettaglio ordine, wizard preventivo, modale prodotto, zoom foto), supporto parametri URL (`?page=X`) + footer |
| `assets/images/` | 44 immagini prodotto/categoria/hero/logo scaricate da oltreilgiardino.biz |

**Parametri URL supportati:** `app.html?page=dashboard` | `catalogo` | `dettaglio-prodotto` | `calendario` | `preventivi` | `ordini` | `logistica` | `clienti`

### 15.2 Stack Prototipo

| Componente | Tecnologia |
|---|---|
| Rendering | HTML + CSS + JavaScript vanilla (nessuna dipendenza esterna) |
| Layout | CSS Grid + Flexbox |
| Icone | SVG inline nella sidebar (8 vettoriali: dashboard, catalogo, calendario, preventivi, ordini, logistica, clienti) |
| Navigazione | SPA con `showPage(id)` e classi `.page.active` — mappa nav index nel JS per evidenziare la voce sidebar corretta |
| Gantt | Engine JavaScript puro: costruzione dinamica DOM, drag-to-pan, scroll-by, scroll-to-commit, sparkline |
| Design system | CSS custom properties (`:root` tokens) con palette OKLch |

### 15.3 Design System Applicato

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
| `--danger` | `oklch(55% 0.18 25)` | Stato critico / esaurito |

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
- `.pag-storico-table` / `.pag-badge` / `.pagato` / `.in_attesa`
- `.doc-log-grid` / `.doc-log-field`
- `.comp-sub-row` / `.comp-sub-label`
- `.qs-volcalc` / `.qs-volcalc-row` / `.qs-volcalc-label` / `.qs-volcalc-value`
- `.qs-vincoli` / `.qs-vincoli-row` / `.qs-vincoli-label`

### 15.4 Mappa Schermate

| # | Schermata | ID pagina | Riga sidebar | File prototype (righe) |
|---|---|---|---|---|
| 1 | Dashboard | `page-dashboard` | Principale → Dashboard | `oltre-il-giardino-gestione-noleggi.html:557-635` |
| 2 | Catalogo Prodotti | `page-catalogo` | Principale → Catalogo | `:638-675` |
| 3 | Dettaglio Prodotto | `page-dettaglio-prodotto` | (click card prodotto) | `:678-851` |
| 4 | Calendario Eventi (Gantt) | `page-calendario` | Principale → Calendario Eventi | `oltre-il-giardino-gestione-noleggi.html:1010-1100` |
| 5 | Preventivi | `page-preventivi` | Operativo → Preventivi | `:931-950` |
| 6 | Ordini | `page-ordini` | Operativo → Ordini | `:953-971` |
| 7 | Logistica | `page-logistica` | Operativo → Logistica | `app.html` — anagrafica contenitori + documenti di riferimento (fatture, DDT) |
| 8 | Clienti | `page-clienti` | Anagrafiche → Clienti | `app.html` — elenco clienti + storico pagamenti Rossi Srl |

### 15.5 Dati Demo

Il prototipo contiene dati realistici estratti da oltreilgiardino.biz:

| Entità | Quantità | Esempi |
|---|---|---|---|
| Categorie | 13 | Sedute, Tavoli, Tovagliato, Luci, Mobili, Banconi, Piante e Vasi, Pavimenti, Mise en place, Coperture, Accessori e sicurezza, Bimbi, Natale |
| Prodotti | 21 | 20 originali + Bancone Pallet (composito) |
| Clienti | 7 | Rossi Srl, Bianchi Spa, Verdi & Figli, Neri Spa, Blu Events, Gialli Party, Arancio Catering |
| Ordini | 6 | #2026/040 — #2026/047 con stati diversi — ogni ordine include `payments` (1-2 rate) e `docLog` (ZTL, muletto, personale, note) |
| Preventivi | 7 | #2026/026 — #2026/032 con 5 stati diversi |
| Documenti riferimento | 5 | DDT-2026-085 — DDT-2026-089 + FATT-001 (dal gestionale) |
| Contenitori | 7 | Cassa media/grande, Baule cristallo, Pallet EU/industriale, Carrello appendiabiti/piatto |
| Movimenti prodotto | 6 | Per Sedia Chiavari: 4 uscite + 2 rientri con saldo progressivo |
| Impegni Gantt | 6 | Su 22 giorni (15 Giu → 6 Lug), con 4 stati |
| Immagini | 33 | 13 categorie + 15 prodotti + 2 hero + 1 logo + 2 grafiche |

### 15.6 Note per Sviluppo

**Cosa funziona nel prototipo v2.2:**
- Navigazione SPA completa tra 8 pagine + modale dettaglio ordine
- Gantt interattivo con drag-to-pan, scroll-by, scroll-to-commit, sparkline
- Filtri ricerca e selezione stato su Catalogo e Calendario Eventi
- Pill colore per stati (ordini, preventivi, logistica, prodotti)
- Timeline stato ordine a 6 step
- Responsive base (sidebar collapse a 920px)
- Foto allestimento con griglia thumbnail e zoom overlay con navigazione ◀/▶
- Questionario post-evento AI con stelle, select, input, textarea, tag AI
- **Wizard Preventivo 4 step** (v2.2): Dati ordine → Righe ordine (con IVA, sconto %, sconto fisso, prezzo manuale, note, compositi, disponibilità) → Logistica (contenitori, volume, peso, colli, pallet, viaggi, vincoli consegna) → Riepilogo (dettaglio IVA per aliquota, logistica, sconti)
- **Anagrafica dettagliata prodotto** (v2.2): 2 nuovi tab — Anagrafica (fornitore, costi, posizione, dimensioni) + Movimenti (entrate/uscite per evento, saldo progressivo)
- **Pagamenti nell'ordine** (v2.2): Sezione nella modale ordine con tabella storico pagamenti e pulsanti azione
- **Storico pagamenti cliente** (v2.2): Tabella in pagina Clienti con ultimi 6 movimenti Rossi Srl
- **Documento Logistico** (v2.2): Sezione nella modale ordine con campi ZTL, muletto, personale, note; pulsante PDF placeholder
- **Logistica** (v2.2): Pagina con anagrafica contenitori (7 tipi) e documenti di riferimento (DDT + fatture)
- **Bancone Pallet** (composito): Primo prodotto composito nel catalogo con 3 sotto-righe espandibili

**Cosa manca e va implementato:**

| Componente | Fase | Note |
|---|---|---|
| Wizard "Nuovo Preventivo" | ✅ **Prototipato v2.2** | 4 step completi. Manca: persistenza backend, invio email |
| Foto allestimento | Prototipato | Griglia thumbnail + zoom overlay. Manca: upload reale, eliminazione, didascalia editabile, storage backend |
| Questionario AI | Prototipato | Form post-evento con stelle, select, input, tag. Manca: persistenza backend, calcolo AI reale, training model |
| Documento logistico PDF | Fase 2 | Pulsante presente (placeholder alert). Generazione PDF reale |
| Ricezione documenti da gestionale | Fase 2 | Webhook/API per ricevere DDT, fatture, note credito dal gestionale esterno |
| Integrazione gestionale (sync stati) | Fase 2 | Comunicazione stato ordine → gestionale per fatturazione |
| Wizard "Nuovo Ordine" | Fase 2 | Trasformazione da preventivo o creazione diretta |
| Dettaglio Preventivo | Fase 2 | Vista completa preventivo con azioni (invia, archivia, trasforma) |
| Dettaglio Cliente | Fase 1 | Vista anagrafica completa con storico ordini e preventivi |
| Form creazione Cliente | Fase 1 | Inserimento anagrafica |
| Form modifica Prodotto | Fase 1 | Modifica info, prezzi, quantità |
| Cambio stato articoli | Fase 3 | Disponibile → In carico → In noleggio → Reso/Danneggiato |
| Paginazione tabella | Fase 1 | Componente paginazione client-side/server-side |
| Export CSV/PDF | Fase 4 | Download dati da tabella |
| Sidebar mobile | Fase 6 | Toggle hamburger per schermi <920px |
| Autenticazione | Fase 1 | Login page, JWT, refresh token, ruoli |

---

## 16. Note Demo

Il prototipo include un **footer** in tutte le pagine (index e app) con:
- Nota "Demo dimostrativa" con avviso che i dati sono fittizi e alcune funzionalità potrebbero non essere operative
- Crediti: "Oltre Il Giardino — Noleggi Creativi · Prototipo ad alta fedeltà · Realizzato da Ugo Volpato AI Consultant"

Nell'index il footer è in fondo alla pagina dopo la griglia di card. Nell'app è posizionato sotto la sidebar e il contenuto principale.

---

*Documento generato il 24/06/2026 — Specifica Tecnica v2.1 — Prototipo Gestione Noleggi Oltre Il Giardino SRL*

## Changelog v2.0 → v2.1

| Sezione | Modifica |
|---|---|
| §1.1 | Architettura documentale riscritta: questo sistema **non genera** documenti contabili |
| §1.3 | Glossario aggiornato: DDT/fatture non più "generati da questo sistema" |
| §2.2 | Stack: rimosso PDF generation per DDT/fatture |
| §2.4 | Flusso integrazione riscritto: questo sistema → stati ordine → gestionale → DDT/fatture/giacenze |
| §3.1 | ER diagramma: rimosse entità `ddt`, `ddt_riga`, `fattura`, `fattura_riga` → sostituite con `documento_riferimento` (riferimenti a documenti del gestionale) + `logistica_movimento` (tracciamento interno non fiscale) |
| §3.2.10–11a | Sezioni riscritte con `documento_riferimento` e `logistica_movimento` |
| §4.2.6 | API rimosse per DDT → API per Logistica (contenitori, movimenti interni) + ricezione documenti |
| §5.1 | Flusso riscritto: non più "DDT di uscita/ingresso" ma stati ordine comunicati al gestionale |
| §8.3.7 | Timeline ordine semplificata (5 step, senza DDT). Rimosso "Crea DDT Uscita" |
| §8.3.8 | `page-ddt` → `page-logistica` con nuovo contenuto |
| §11.1 | "Generazione DDT/Fattura" non è di competenza di questo sistema |
| §12 | Fasi e milestone aggiornate rimuovendo generazione documenti contabili |
| §15.4 | Mappa schermate: `page-ddt` → `page-logistica` |
| Varie | Oltre 30 riferimenti a DDT/fattura aggiornati in tutto il documento |
