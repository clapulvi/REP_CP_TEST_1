U v7.70+** introduce:
```
<TIPOLOGIA_ACCESSO_UBB> → valori S|F
```
### 7.4 Nuovi Campi Sincronizzazione NNI
**Interfaccia NNI (Network-to-Network) v1.6+** introduce:
```
ANAGRAFICA_GPON
├── IDGPON (chiave)
├── TIPOLOGIA_ACCESSO_UBB
├── CARATTERIZZAZIONE_GEOGRAFICA
├── CLLI_CRO
└── [altri dati infrastruttura GPON]
```
---
## 8. FLUSSO DI ACQUISIZIONE: COSA FA NOW QUANDO RICEVE IL FILE
### 8.1 Fasi di Elaborazione Generale (Canale FILE)
```
┌─────────────────────────────────────────────────────────────────┐
│ FASE 1: RICEZIONE FILE                                          │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • File Manager NOW monitora directory ingresso
       • Rileva file xml con naming convention corretto
       • Salva in tabella RICHIESTA_FILE_INGRESSO
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 2: VALIDAZIONE STRUTTURA XSD                              │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Valida file XML contro XSD v5.2
       • Controllo: encoding UTF-8
       • Controllo: well-formed XML
       • Controllo: conformità schema elementi/attributi
       ↓
       ├─ ✅ VALIDA XSD → FASE 3
       └─ ❌ FALLISCE → Scarto file, notifica errore a OAO
           (Status: FILE_SCARTATO, Email notifica)
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 3: ACQUISIZIONE RICHIESTE BATCH (C01)                     │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Legge file XML elemento per elemento <Richiesta>
       • Per ogni richiesta nel file:
         - Estrae dati testata e ordine
         - Crea record RICHIESTA_OLO in DB
         - Assegna ID_RICHIESTA univoco NOW
         - Stato iniziale: ACCOLTA
       ↓
       • Registra:
         ├─ CodiceOlo (operatore)
         ├─ CodiceOrdineOlo (ID OAO)
         ├─ Timestamp ricezione
         ├─ Tipo lavorazione (ATT/CESS/VAR)
         └─ Tipo offerta (GPON, ecc.)
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 4: VALIDAZIONI LOGICHE (C04 - Gestione Verifiche)         │
└─────────────────────────────────────────────────────────────────┘
       ↓
       Per ogni richiesta acquisita:
       
       A. CONTROLLI DI BASE
          1. Verifica CodiceOlo valido ✓ (funz. C04M01F09)
          2. Verifica OLO Accorpato ✓ (funz. C04M01F30)
          3. Verifica OLO Richiesta ✓ (funz. C04M03F22)
          4. Verifica Servizio in Consistenza ✓ (funz. C04M03F04)
       
       B. CONTROLLI SPECIFICI GPON (ST_354378)
          1. Verifica IdGPON valorizzato e valido
          2. Verifica IdGPON in ANAGRAFICA_GPON
          3. Verifica TipologiaAccessoUBB (S|F)
          4. Verifica coerenza TipologiaAccessoUBB ↔ Tipo Contratto
             └─ "S" → Contratto SEMIGPON1BPNRR
             └─ "F" → Contratto FULLGPON1BPNRR
          5. Verifica indirizzo cliente in NetMap DB
          6. Verifica Referente almeno 1
       
       C. CONTROLLI TERRITORIALI
          1. Verifica coordinate geografiche (CodiceComune, CodiceIstat)
          2. Verifica disponibilità infrastruttura GPON (NNI sync)
          3. Query NetMap per disponibilità tecnica
       
       D. CONTROLLI DI CONSISTENZA
          1. Se attivazione: verifica non doppio ordine
          2. Se cessazione: verifica linea attiva
          3. Se variazione: verifica linea esistente
          4. Verifica compatibilità offerta+tecnologia
       
       ↓
       ├─ ✅ TUTTE VALIDAZIONI OK → FASE 5 (Processing)
       └─ ❌ FALLISCE VALIDAZIONE → Status RICHIESTA: "SCARTATA_VALIDAZIONE"
                                      Crea task utente per analisi
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 5: CREAZIONE ORDINE NOW (C02)                              │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Crea record ORDINE da RICHIESTA_OLO validata
       • Valorizza componenti ordine:
         ├─ ACCESSO (GPON)
         ├─ SERVIZIO (SEMIGPON1BPNRR/FULLGPON1BPNRR)
         ├─ CLIENTE (da DatiCliente XML)
         ├─ REFERENTE (contatti)
         ├─ SEDE (indirizzo)
         └─ DATI_TECNICI (IdGPON, TipologiaAccessoUBB, ecc.)
       
       • Stato ordine: "IN_LAVORAZIONE"
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 6: ORCHESTRAZIONE ORDINE (BPM Process)                    │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Invia ordine a BPM per orchestrazione
       • BPM esegue workflow:
         1. Pre-Delivery checks (inventario, qualità, ecc.)
         2. Planning tecnico (assegnazione risorse)
         3. Provisioning HW (configurazione GPON, IP, ecc.)
         4. Attivazione servizio
         5. Testing (Qualificazione + Test2 per FTTH)
       
       • Stato ordine durante elaborazione: "PROCESSING"
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 7: PARCHEGGIO ORDINI GPON (ST_354378 - FASE TRANSITORIA)  │
└─────────────────────────────────────────────────────────────────┘
       ↓
       ⚠️ SOLO DURANTE C1-2026 (fino a 30/03/2026):
       
       • Se ordine di tipo ATTIVAZIONE con contratto:
         - SEMIGPON1BPNRR → PARCHEGGIO
         - FULLGPON1BPNRR → PARCHEGGIO
       
       • Logica parcheggio:
         1. Ordine creato normalmente
         2. Dopo orchestrazione OK → Status "ESPLETAMENTO_TECNICO_OK"
         3. Invece di inviare a JPUB (Fatturazione)
         4. Inserisce record in tabella: ORDINE_PARCHEGGIATO_JPUB
            ├─ ID_ORDINE
            ├─ STATUS: "IN_ATTESA"
            ├─ IDGPON
            ├─ TIPOLOGIA_ACCESSO_UBB
            ├─ TIMESTAMP_PARCHEGGIO
            └─ [NOTE]
       
       • Ordine rimane visibile in NOW ma NON va a JPUB
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 8: INVIO A JPUB (C08M01)                                   │
└─────────────────────────────────────────────────────────────────┘
       ↓
       ⚠️ SCENARIO A: FASE A REGIME (da C3-2026)
       
       • Dopo bonifica ANAGRAFICA_GPON:
         ├─ Legge IDGPON da ORDINE_PARCHEGGIATO_JPUB
         ├─ Recupera CaratterizzazioneGeografica da ANAGRAFICA_GPON
         ├─ Recupera CLLI_CRO da ANAGRAFICA_GPON
         └─ Prepara payload JPUB con V8 XSD (NEW)
       
       • Payload JPUB v8 include:
         ├─ TipologiaAccessoUBB (S|F)
         ├─ CaratterizzazioneGeografica (PNRR|AREA_NERA|AREA_GRIGIA)
         ├─ IDGPON
         ├─ CLLI_CRO
         └─ [Dati standard ordine]
       
       • Invio a JPUB via WebService (SOAP)
       • JPUB riceve → Fatturazione ordine
       ↓
       ├─ ✅ JPUB OK → Status "FATTURATO", Ordine CHIUSO
       └─ ❌ JPUB KO → Status "ERRORE_JPUB", Task utente per recovery
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 9: NOTIFICA FINALE A OAO                                   │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Genera file notifica con stato finale ordine
       • Formato: ISP_GGMMAAAA_HH_NGA.xml.gz
       • Contenuto: ID ordine NOW, stato, date, eventuali errori
       • Invia via sftp/http a operatore
```
### 8.2 Tempo di Elaborazione Atteso
- **Validazione XSD:** < 1 minuto
- **Acquisizione batch:** 1-5 minuti (dipende numero richieste)
- **Validazioni logiche:** 2-10 minuti per richiesta
- **Orchestrazione BPM:** Ore/Giorni (dipende complessità lavori)
- **Invio JPUB:** < 1 minuto post-espletamento tecnico
- **Total E2E:** Ore fino a giorni
---
## 9. VALIDAZIONI ESEGUITE DA NOW SUL FILE RICEVUTO
### 9.1 Validazioni di Primo Livello (Sintassi)
| # | Validazione | Esito Positivo | Esito Negativo | Azione |
|---|---|---|---|---|
| 1 | **File exists** | ✅ File presente | ❌ File non trovato | Retry schedulato |
| 2 | **File naming** | ✅ Segue naming convention | ❌ Nome non conforme | Scarto file |
| 3 | **Encoding** | ✅ UTF-8 | ❌ Encoding diverso | Scarto file |
| 4 | **XML Well-formed** | ✅ XML valido | ❌ XML malformed | Scarto file |
| 5 | **XSD conformance** | ✅ Conforme schema v5.2 | ❌ Non conforme | Scarto file |
| 6 | **Root element** | ✅ `<TracciatoOLONGA>` | ❌ Elemento root diverso | Scarto file |
| 7 | **Multiple requests** | ✅ Max 1000 `<Richiesta>` | ❌ > 1000 richieste | Scarto batch request |
| 8 | **Character length** | ✅ Rispetta limiti XSD | ❌ Campo eccessivamente lungo | Scarto richiesta |
### 9.2 Validazioni di Secondo Livello (Logica)
#### A. VALIDAZIONI TESTATA
| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | CodiceOlo | Obbligatorio | 3 car esatti | Scarto richiesta |
| 2 | CodiceOlo | Codice valido | Presente in anagrafica OLO | Scarto richiesta (C04M01F09) |
| 3 | CodiceOlo | Accorpamento | Verifica OLO accorpati | Scarto richiesta (C04M01F30) |
| 4 | CodiceOrdineOlo | Unicità | Se presente, unico in batch | Warning (non scarta) |
| 5 | CodiceProgetto | Validazione | Se presente, valido | Scarto richiesta |
| 6 | TestMode | Valore | Se presente, solo "0" | Scarto richiesta |
| 7 | Note | Lunghezza | Max 255 caratteri | Scarto richiesta |
| 8 | semiVULA / semiBitstreamNGA / semiEasyIPNGA / FullGPONPLUS | CHOICE (uno solo) | Massimo 1 valore tra questi | Scarto richiesta |
| 9 | IdReteSecondariaOAO | Validazione | Se presente, max 19 car | Scarto richiesta |
| 10 | ReferentiOAOProgettiSpeciali | Lunghezza | Max 255 car | Scarto richiesta |
| 11 | CodiceOperatoreSecondario | Lunghezza | 1-3 car | Scarto richiesta |
#### B. VALIDAZIONI DATI CLIENTE
| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | CognomeRagSoc | Obbligatorio | Non vuoto | Scarto richiesta |
| 2 | CF / PIVA | Mutua esclusività | Uno almeno obbligatorio | Scarto richiesta |
| 3 | CF | Formato | Valido per Italia (16 car) | Scarto richiesta |
| 4 | PIVA | Formato | Valido per Italia (11 car) | Scarto richiesta |
| 5 | Nome | Lunghezza | Max 50 car (stimato) | Scarto richiesta |
#### C. VALIDAZIONI SEDE
| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | ParticellaToponomastica | Obbligatorio | Presente in NetMap | Scarto richiesta |
| 2 | Via | Obbligatorio | Presente in NetMap | Scarto richiesta |
| 3 | Civico | Obbligatorio | Presente in NetMap | Scarto richiesta |
| 4 | Localita | Obbligatorio | Presente in NetMap | Scarto richiesta |
| 5 | CodiceIstatIndirizzo | Obbligatorio | Formato "NN_NNN_NNNNN_NN" | Scarto richiesta |
| 6 | CodiceIstatIndirizzo | Validazione | Presente in NetMap | Scarto richiesta |
| 7 | CodiceEGON | Obbligatorio | 15 car numerici | Scarto richiesta |
| 8 | CodiceComune | Obbligatorio | 8 car, valido ISTAT | Scarto richiesta |
| 9 | Coerenza Sede | Validazione | ParticellaToponomastica + Via + Civico + Localita == NetMap | Scarto richiesta |
#### D. VALIDAZIONI REFERENTE
| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | Progressivo | Obbligatorio | 1, 2, 3 (max 3 per FTTH) | Scarto richiesta |
| 2 | Progressivo | Sequenzialità | Sempre 1,2,3 senza buchi | Scarto richiesta |
| 3 | QualificaRef | Obbligatorio | Non vuoto | Scarto richiesta |
| 4 | CognomeReferente | Obbligatorio | Non vuoto | Scarto richiesta |
| 5 | NomeReferente | Obbligatorio | Non vuoto | Scarto richiesta |
| 6 | TelefonoFisso / Mobile | Mutua obbligatorietà | Almeno 1 valorizzato | Scarto richiesta |
| 7 | TelefonoFisso / Mobile | Formato | Numerico 10 car | Scarto richiesta |
| 8 | Email | Obbligatorio per FTTH X1/X2 | Validazione RFC5321 se presente | Scarto richiesta |
| 9 | Numero Referenti | Obbligatorio | Almeno 1 referente | Scarto richiesta |
#### E. VALIDAZIONI ACCESSO GPON (ST_354378)
| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | IdGPON | **Obbligatorio** | **Presente e valorizzato** | **Scarto richiesta (C02M78F05)** |
| 2 | IdGPON | Validazione | **Presente in ANAGRAFICA_GPON** | **Scarto richiesta** |
| 3 | IdGPON | Compatibilità SemiGPON | **Se TipologiaAccessoUBB="S" → IDGPON deve supportare SemiGPON** | **Scarto richiesta** |
| 4 | IdGPON | Compatibilità FullGPON | **Se TipologiaAccessoUBB="F" → IDGPON deve supportare FullGPON** | **Scarto richiesta** |
| 5 | **TipologiaAccessoUBB** | **Obbligatorio** | **Valore: "S" (SemiGPON) oppure "F" (FullGPON)** | **Scarto richiesta (C02M78F05)** |
| 6 | **TipologiaAccessoUBB** | Coerenza Contratto | **Se="S" → Contratto SEMIGPON1BPNRR**<br>**Se="F" → Contratto FULLGPON1BPNRR** | **Scarto richiesta** |
| 7 | DataDAI | Formato | GG/MM/AAAA | Scarto richiesta |
| 8 | DataDAI | Giorno lavorativo | Lun-Ven, escluse festività infrasettimanali | Scarto richiesta |
| 9 | DataDAI | Coerenza | DataDAI ≥ data odierna (data ordine) | Scarto richiesta |
| 10 | DAD | Formato | GG/MM/AAAA | Scarto richiesta |
| 11 | DAD | Giorno lavorativo | Lun-Ven, escluse festività infrasettimanali | Scarto richiesta |
#### F. VALIDAZIONI DATI TECNICI
| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | CodiceQualita | Enumerazione | "X", "Y", "Z" se presente | Scarto richiesta |
| 2 | SlaAssurance | Valore | 0-3 o default 0 | Scarto richiesta |
| 3 | CodiceDelivery | Lunghezza | Max 10 car se presente | Scarto richiesta |
| 4 | FasciaOraria | Enumerazione | Valori definiti se presente | Scarto richiesta |
| 5 | CampoServizioIT | Uso esclusivo | Solo per FiberCop (opzionale per OAO) | Warning |
| 6 | Qualifica | Valore | "SI" se presente (default SI se omesso) | — |
| 7 | Test2 | Valore | "SI" se presente (default SI se omesso) | — |
### 9.3 Validazioni di Terzo Livello (Business Logic)
#### A. VALIDAZIONI ORDINE PER TIPO LAVORAZIONE
**Attivazione:**
```
1. Verifica non esista già ordine attivo per stessa linea
2. Verifica IdGPON disponibile (non saturato)
3. Verifica non doppio ordine stesso OAO/Data/Cliente
4. Verifica tipo contratto supportato (SEMIGPON1BPNRR/FULLGPON1BPNRR)
5. Verifica capacità network GPON per nuovo cliente
6. Verifica disponibilità tecnici per Data Desiderata Attivazione
```
**Cessazione:**
```
1. Verifica esista ordine attivo da cessare
2. Verifica linea in stato attivo (non già in cessazione)
3. Verifica coerenza cliente
4. Se con Rientro (NP): verifica OAO destinatario valido
```
**Variazione:**
```
1. Verifica ordine attivo da variare
2. Verifica compatibilità nuovi parametri vs profilo GPON
3. Verifica non sia variazione impossibile (es. downgrade che violerebbe SLA)
4. Verifica non sia duplicata di variazione precedente
5. Verifica non sia variazione illecita (es. cambio cliente)
```
#### B. VALIDAZIONI TERRITORIALI
```
1. Copertura GPON disponibile per indirizzo cliente
2. IdGPON disponibile nella geographic area
3. CodiceComune coerente con area di copertura dell'IdGPON
4. Se progetto PNRR: verifica area dentro perimetro PNRR
5. Se area_nera: verifica mercato competitivo attivo
6. Se area_grigia: verifica limitazione operatori
```
#### C. VALIDAZIONI REGULATORY
```
1. Delibera 82/19/CIR (per migrazioni wholesale)
2. Requisiti di non discriminazione (non preferenze OAO)
3. Separazione contabile (non cross-selling FiberCop)
4. Compliance Data Protection (GDPR per dati cliente)
```
### 9.4 Output Validazioni
**Esito POSITIVO (File Acquisito):**
```
Status FILE:           ACQUISITO_OK
Status RICHIESTE:      ACCOLTA (per ogni richiesta valida)
Email Notifica OAO:    "Pratiche acquisite con esito OK = N"
                       "Pratiche acquisite con esito KO = 0"
                       "Pratiche scartate = 0"
Timestamp:             GGMMAAAA HH:MM:SS
```
**Esito NEGATIVO (File Scartato):**
```
Status FILE:           FILE_SCARTATO / RICHIESTE_SCARTATE
Email Notifica OAO:    "Pratiche acquisite con esito OK = 0"
                       "Pratiche acquisite con esito KO = N"  (se parziale)
                       "Pratiche scartate = M" (se totale)
Errori Dettagliati:    [Lista causali scarto con numeri richiesta]
Task Utente:           Se scarto parziale o errori recuperabili
```
---
## 10. RISPOSTE/NOTIFICHE NOW → OAO DOPO RICEZIONE
### 10.1 Tipologie di Notifiche
#### NOTIFICA 1: ESITO RICEZIONE FILE (Email)
**Timing:** Entro 5 minuti da ricezione file
**Oggetto Email:**
```
File: <isp_filename> - Codice: <code_lavorazione_file>
```
**Corpo Email (caso OK - File acquisito):**
```
DATI IDENTIFICAZIONE
Codice ISP = MTW
Descrizione = Fastweb Metavisione
DATI FILE
Nome file ISP = NEWGPONMTW2026021102.xml
DATI ELABORAZIONE
Totale pratiche presenti = 1
Pratiche acquisite con esito OK = 1
Pratiche acquisite con esito KO = 0
Pratiche scartate = 0
Data elaborazione = 11/02/2026 02:15:30
```
**Corpo Email (caso KO - File scartato):**
```
DATI IDENTIFICAZIONE
Codice ISP = MTW
Descrizione = Fastweb Metavisione
DATI FILE
Nome file ISP = NEWGPONMTW2026021102.xml
DATI ELABORAZIONE
Totale pratiche presenti = 1
Pratiche acquisite con esito OK = 0
Pratiche acquisite con esito KO = 0
Pratiche scartate = 1
Data elaborazione = 11/02/2026 02:15:30
ERRORI
[Lista errori per ogni richiesta scartata]
Richiesta #1: IdGPON non valido - non presente in ANAGRAFICA_GPON
```
#### NOTIFICA 2: FILE DI NOTIFICA (XML compresso)
**Naming:**
```
<ISP>_<GGMMAAAA>_<HH>_NGA.xml.gz
```
**Esempi:**
- `MTW_20260211_00_NGA.xml.gz` - parziale ore 00:00
- `MTW_20260211_RG_NGA.xml.gz` - riepilogo giornaliero (entro 10:00 giorno succ.)
- `MTW_20260211_EX_NGA.xml.gz` - file straordinario (su richiesta)
**Contenuto XML Notifica (semplificato):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<NotificaOLO>
    <Testata>
        <CodiceOlo>MTW</CodiceOlo>
        <DataGenerazione>11/02/2026 10:00:00</DataGenerazione>
        <TipoBatch>RG</TipoBatch> <!-- RG=Riepilogo, EX=Straordinario, 00-23=Parziale -->
    </Testata>
    <Ordini>
        <Ordine>
            <CodiceOrdineOlo>FCC00000001</CodiceOrdineOlo>
            <CodiceOrdineNow>ORD_MTW_2026021101</CodiceOrdineNow>
            <StatoOrdine>ACCOLTA</StatoOrdine> <!-- ACCOLTA, ELABORAZIONE, ESPLETATO_OK, ERRORE -->
            <DataUltimoAggiornamento>11/02/2026 08:30:15</DataUltimoAggiornamento>
            <TipoLavorazione>ATTIVAZIONE</TipoLavorazione>
            <Offerta>SEMIGPON1BPNRR</Offerta>
            <IdGPON>FWE0000000000083517</IdGPON>
            <TipologiaAccessoUBB>S</TipologiaAccessoUBB>
            <CaratterizzazioneGeografica>PNRR</CaratterizzazioneGeografica>
            <DescrizioneStato>Ordine in lavorazione - fase di provisioning</DescrizioneStato>
            <CodiceErrore></CodiceErrore>
            <DescrizioneErrore></DescrizioneErrore>
        </Ordine>
        <Ordine>
            <CodiceOrdineOlo>FCC00000002</CodiceOrdineOlo>
            <CodiceOrdineNow>ORD_MTW_2026021102</CodiceOrdineNow>
            <StatoOrdine>ERRORE</StatoOrdine>
            <TipoLavorazione>ATTIVAZIONE</TipoLavorazione>
            <CodiceErrore>E001</CodiceErrore>
            <DescrizioneErrore>TipologiaAccessoUBB non coerente con tipo contratto FULLGPON1BPNRR (valorizzato 'S' invece di 'F')</DescrizioneErrore>
        </Ordine>
    </Ordini>
</NotificaOLO>
```
#### NOTIFICA 3: NOTIFICA DI AVANZAMENTO (Asincrona durante elaborazione)
**Timing:** Periodica durante elaborazione (es. ogni 6 ore)
**Evento Generato:**
- Pre-Delivery completato
- Provisioning iniziato
- Provisioning completato
- Testing in corso
- Testing completato
- Espletamento tecnico positivo
- Espletamento tecnico negativo
**Formato:** File XML o Email con stato e timestamp
#### NOTIFICA 4: NOTIFICA FINALE (Quando ordine entra in stato terminale)
**Timing:** Quando ordine raggiunge uno stato stabile
**Stati Possibili:**
```
✅ ESPLETAMENTO_TECNICO_OK
   ↓ Attesa invio JPUB o Parcheggio
   ↓
💾 ORDINE_PARCHEGGIATO (ST_354378 - FASE TRANSITORIA)
   ↓ In attesa bonifica da ANAGRAFICA_GPON
   ↓
📤 INVIATO_JPUB (da C3-2026)
   ↓
✅ FATTURATO (JPUB OK)
❌ ESPLETAMENTO_TECNICO_KO
   → Causa + Descrizione errore
   → Task utente per recovery
❌ SCARTATO
   → Motivo scarto
   → Azioni correttive consigliate
```
### 10.2 Canali di Notifica
| Tipo Notifica | Canale | Timing | Compressione | Allegati |
|---|---|---|---|---|
| Esito Ricezione File | Email | 5 min | No | — |
| File Notifica Parziale | SFTP | Ogni ora | GZ | XML |
| File Notifica Riepilogo Giornaliero | SFTP | Entro 10:00 giorno succ. | GZ | XML |
| File Notifica Straordinaria | SFTP | Su richiesta | GZ | XML |
| Notifica Avanzamento Ordine | Email | Periodica | No | — |
| Notifica Finale Ordine | Email + File | A fine ordine | No/GZ | XML |
### 10.3 Credenziali e Destinatari
```
Email Operatore:       ReferentiOAOProgettiSpeciali o contatto di default
SFTP Server:           ftp.now.fibercop.it
Directory Ingresso:    /in/olo_ordini/
Directory Notifiche:   /out/olo_notifiche/
Autenticazione:        Certificate-based o User/Pass
```
---
## 11. ESEMPIO CONCRETO: FILE XML ORDINE OAO (DA DOCUMENTAZIONE)
### Esempio 1: ATTIVAZIONE SemiGPON (da NEWGPONMTW2026021102.xml)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<RichiestaOAO>
<Richiesta>
    <Testata>
        <CodiceOlo>MTW</CodiceOlo>
        <CodiceOrdineOlo>FCC00000001</CodiceOrdineOlo>
    </Testata>
    <Ordine>
        <Attivazione>
            <AccessoGPON>
                <DataDAI>18/02/2026</DataDAI>
                <DatiCliente>
                    <CognomeRagSoc>BERNINI</CognomeRagSoc>
                    <Nome>MASSIMO</Nome>
                    <CF>BRNMSM83S06E801P</CF>
                </DatiCliente>
                <Sede>
                    <ParticellaToponomastica>PIAZZA</ParticellaToponomastica>
                    <Via>GIUSEPPE MAGGIOLINI</Via>
                    <Civico>29</Civico>
                    <Localita>PARABIAGO</Localita>
                    <CodiceIstatIndirizzo>15_168_19344_29</CodiceIstatIndirizzo>
                    <CodiceEGON>380100023791543</CodiceEGON>
                    <CodiceComune>03015168</CodiceComune>
                </Sede>
                <Referente>
                    <Progressivo>1</Progressivo>
                    <QualificaRef>CF</QualificaRef>
                    <CognomeReferente>BERNINI</CognomeReferente>
                    <NomeReferente>MASSIMO</NomeReferente>
                    <TelefonoMobileReferente>3493659110</TelefonoMobileReferente>
                </Referente>
                <ReferenteOnField>
                    <CognomeReferente>FASTWEB</CognomeReferente>
                    <NomeReferente>TIPOG</NomeReferente>
                    <TelefonoFissoReferente>0245400106</TelefonoFissoReferente>
                </ReferenteOnField>
                <!-- NUOVO ST_354378 - Campi obbligatori per GPON -->
                <IdGPON>FWE0000000000083517</IdGPON>
                <TipologiaAccessoUBB>S</TipologiaAccessoUBB>
                <!-- NUOVO ST_354378 (da JPUB V8) -->
                <!-- Nota: CaratterizzazioneGeografica e CLLI_CRO NON presenti nel file OAO
                         ma verranno aggiunti da NOW leggedoli da ANAGRAFICA_GPON durante elaborazione -->
                <ReferentiOAOProgettiSpeciali>GC24748301</ReferentiOAOProgettiSpeciali>
            </AccessoGPON>
        </Attivazione>
    </Ordine>
</Richiesta>
</RichiestaOAO>
```
### Esempio 2: ATTIVAZIONE FullGPON (con tutti i campi)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<RichiestaOAO>
<Richiesta>
    <Testata>
        <CodiceOlo>MTW</CodiceOlo>
        <CodiceOrdineOlo>FCC00000002</CodiceOrdineOlo>
        <CodiceProgetto>GC24748302</CodiceProgetto>
        <Note>Ordine prioritario PNRR</Note>
    </Testata>
    <Ordine>
        <Attivazione>
            <AccessoGPON>
                <DataDAI>20/02/2026</DataDAI>
                <DatiCliente>
                    <CognomeRagSoc>ROSSI S.R.L.</CognomeRagSoc>
                    <CF>IT03015168123456</CF>
                </DatiCliente>
                <Sede>
                    <ParticellaToponomastica>VIA</ParticellaToponomastica>
                    <Via>XXIV MAGGIO</Via>
                    <Civico>50</Civico>
                    <Localita>MILANO</Localita>
                    <CodiceIstatIndirizzo>15_146_03111_50</CodiceIstatIndirizzo>
                    <CodiceEGON>380100031111543</CodiceEGON>
                    <CodiceComune>03015146</CodiceComune>
                </Sede>
                <Referente>
                    <Progressivo>1</Progressivo>
                    <QualificaRef>REP</QualificaRef>
                    <CognomeReferente>ROSSI</CognomeReferente>
                    <NomeReferente>FABIO</NomeReferente>
                    <TelefonoFissoReferente>0248124567</TelefonoFissoReferente>
                </Referente>
                <ReferenteOnField>
                    <CognomeReferente>DITTA INSTALLER</CognomeReferente>
                    <NomeReferente>POSA FIBRA</NomeReferente>
                    <TelefonoMobileReferente>3388123456</TelefonoMobileReferente>
                </ReferenteOnField>
                <!-- NUOVO ST_354378 -->
                <IdGPON>FWE0000000000093520</IdGPON>
                <TipologiaAccessoUBB>F</TipologiaAccessoUBB>
                <ReferentiOAOProgettiSpeciali>GC24748302;Support@fastweb.it</ReferentiOAOProgettiSpeciali>
            </AccessoGPON>
        </Attivazione>
    </Ordine>
</Richiesta>
</RichiestaOAO>
```
**Differenze tra Esempio 1 e 2:**
- Es1: TipologiaAccessoUBB = **"S"** (SemiGPON - rete secondaria)
- Es2: TipologiaAccessoUBB = **"F"** (FullGPON - rete primaria+secondaria)
- Es2: IdGPON diverso (nodo GPON diverso)
- Es2: CodiceProgetto valorizzato (progetto PNRR)
---
## 12. SCHEMA XSD TRACCIATO (Definizioni Elementi)
### 12.1 Struttura XSD TracciatoOLONGA v5.2
**Definizione radice:**
```xml
<xs:element name="TracciatoOLONGA">
    <xs:complexType>
        <xs:sequence maxOccurs="1000">
            <xs:element name="Richiesta">
                <xs:complexType>
                    <xs:sequence>
                        <xs:element name="Testata">
                            <!-- Definizioni Testata (vedi sopra) -->
                        </xs:element>
                        <xs:element name="TipoRichiesta">
                            <xs:complexType>
                                <xs:choice>
                                    <xs:element name="Attivazione">...</xs:element>
                                    <xs:element name="Cessazione">...</xs:element>
                                    <xs:element name="Variazione">...</xs:element>
                                </xs:choice>
                            </xs:complexType>
                        </xs:element>
                    </xs:sequence>
                </xs:complexType>
            </xs:element>
        </xs:sequence>
    </xs:complexType>
</xs:element>
```
**Definizione CodiceOlo:**
```xml
<xs:element name="CodiceOlo">
    <xs:simpleType>
        <xs:restriction base="xs:string">
            <xs:maxLength value="3"/>
            <xs:minLength value="3"/>
        </xs:restriction>
    </xs:simpleType>
</xs:element>
```
**Definizione CodiceOrdineOlo:**
```xml
<xs:element name="CodiceOrdineOlo" minOccurs="0">
    <xs:simpleType>
        <xs:restriction base="xs:string">
            <xs:maxLength value="30"/>
            <xs:minLength value="1"/>
        </xs:restriction>
    </xs:simpleType>
</xs:element>
```
**Definizione TestMode:**
```xml
<xs:element name="TestMode" minOccurs="0">
    <xs:simpleType>
        <xs:restriction base="xs:string">
            <xs:maxLength value="1"/>
            <xs:enumeration value="0"/>
        </xs:restriction>
    </xs:simpleType>
</xs:element>
```
**Definizione Choice Servizi Speciali:**
```xml
<xs:choice>
    <xs:element name="semiVULA" minOccurs="0">
        <xs:simpleType>
            <xs:restriction base="xs:string">
                <xs:enumeration value="SI"/>
            </xs:restriction>
        </xs:simpleType>
    </xs:element>
    <xs:element name="semiBitstreamNGA" minOccurs="0">...</xs:element>
    <xs:element name="semiEasyIPNGA" minOccurs="0">...</xs:element>
    <xs:element name="FullGPONPLUS" minOccurs="0">...</xs:element>
</xs:choice>
```
### 12.2 Nuovo Tracciato V8 JPUB (ST_354378)
**Evoluzione schema JPUB:**
```
Versione Attuale:  OrdiniCRMWS_Reg_2.23.xsd (2.23)
                   ↓
Versione Nuova:    OrdiniCRMWS_Reg_V8.xsd (8.0)
Campi Aggiunti:
├── TipologiaAccessoUBB (CHAR 1)
│   ├── Enumerazione: S | F
│   └── Obbligatorio per Semi/Full GPON
├── CaratterizzazioneGeografica (VARCHAR)
│   ├── Enumerazione: PNRR | AREA_NERA | AREA_GRIGIA
│   └── Obbligatorio per Semi/Full GPON
├── IDGPON (VARCHAR)
│   ├── Identificativo univoco albero GPON
│   └── Obbligatorio
└── CLLI_CRO (VARCHAR)
    ├── Codice CLLI Cabinet Ripartizione Ottica
    └── Obbligatorio
```
**Distribuzione XSD V8 nei moduli TIBCO:**
```
Moduli interessati:
├── tim.now.billingmanager
├── tim.now.timerbilling
├── tim.now.tool.billing
├── tim.now.bpm.localBOM
└── tim.now.proxytoexternal
```
---
## 13. FASE TRANSITORIA vs FASE A REGIME (ST_354378)
### 13.1 FASE TRANSITORIA (C1-2026) - DA OGGI FINO 30/03/2026
**Periodo:** Gennaio 2026 → 30 Marzo 2026
**Caratteristiche:**
- JPUB **NON** pronto a ricevere nuovo tracciato V8
- Ordini SemiGPON/FullGPON vengono acquisiti da NOW normalmente
- Dopo espletamento tecnico → **PARCHEGGIO** anzichè invio a JPUB
- Tabella `ORDINE_PARCHEGGIATO_JPUB` con status `IN_ATTESA`
**Flusso:**
```
OAO invia ordine
    ↓
NOW acquisisce + elabora
    ↓
Espletamento tecnico OK
    ↓
Ordine marcato con status "ESPLETAMENTO_TECNICO_OK"
    ↓
❌ NON invia a JPUB (V8 non pronto)
    ↓
Inserisce in ORDINE_PARCHEGGIATO_JPUB
    ├─ STATUS: "IN_ATTESA"
    ├─ IDGPON: da ordine
    ├─ TIPOLOGIA_ACCESSO_UBB: S|F da ordine
    └─ TIMESTAMP_PARCHEGGIO: ora attuale
```
**Gestione Errori Fase Transitoria:**
- Se IdGPON mancante → Ordine scartato (non parcheggiato)
- Se TipologiaAccessoUBB non coerente → Ordine scartato
- Se validazioni falliscono → Task utente per recovery manuale
### 13.2 FASE A REGIME (C3-2026) - DAL 31 MARZO 2026 IN POI
**Periodo:** 31 Marzo 2026 → ...
**Prerequisiti:**
- JPUB versione 8 disponibile e stabile
- ANAGRAFICA_GPON sincronizzata con NNI
- Tutti i test A/B completati
**Caratteristiche:**
- Ordini SemiGPON/FullGPON **inviati direttamente** a JPUB senza parcheggio
- Tracciato JPUB V8 operativo
- Bonifica ordini parcheggiati in fase transitoria
**Flusso Bonifica Ordini Parcheggiati:**
```
Batch process schedulato (es. giornaliero)
    ↓
Seleziona ordini ORDINE_PARCHEGGIATO_JPUB status "IN_ATTESA"
    ↓
Per ogni ordine:
    1. Legge IDGPON
    2. Query ANAGRAFICA_GPON
    3. Recupera:
       ├─ CaratterizzazioneGeografica
       ├─ CLLI_CRO
       └─ [dati aggiornati infrastruttura]
    ↓
    4. Aggiorna payload JPUB con campi V8
    5. Valida coerenza dati
    6. Invia a JPUB via WS
    ↓
    7. Attende risposta JPUB
    ↓
    ├─ ✅ OK → Status "INVIATO" + "FATTURATO"
    └─ ❌ KO → Status "ERRORE", Task utente
```
**Flusso Nuovi Ordini Fase A Regime:**
```
OAO invia ordine
    ↓
NOW acquisisce + elabora
    ↓
Espletamento tecnico OK
    ↓
Preleva da ANAGRAFICA_GPON:
├─ CaratterizzazioneGeografica
├─ CLLI_CRO
└─ [dati geolocalizzazione]
    ↓
Prepara payload JPUB V8 con TUTTI i campi
    ↓
✅ Invia direttamente a JPUB (NO parcheggio)
    ↓
├─ JPUB OK → Fatturazione OK
└─ JPUB KO → Task utente, Retry
```
---
## SINTESI ESECUTIVA
### Punti Chiave
1. **Modalità invio:** FILE XML (primaria) + WS (opzionale)
2. **Naming file:** `<OLO>_<GGMMAAAA>_<HH>_NGA.xml`
3. **Formato dati:** TracciatoOLONGA XSD v5.2 (NOW-OLO interface)
4. **Nuovi campi ST_354378:**
   - IdGPON (obbligatorio)
   - TipologiaAccessoUBB: "S" (SemiGPON) | "F" (FullGPON)
   - CaratterizzazioneGeografica (verso JPUB v8): PNRR / AREA_NERA / AREA_GRIGIA
   - CLLI_CRO (verso JPUB v8)
5. **Validazioni:** 3 livelli (sintassi XSD → logica DB → business rules)
6. **Notifiche risposta:** Email (5 min) + File XML compresso SFTP (orario/giornaliero/straordinario)
7. **Flusso:** Ricezione → Acquisizione → Validazione → Orchestrazione BPM → [PARCHEGGIO transitorio] → Invio JPUB → Fatturazione
8. **Fase Transitoria:** Parcheggio fino 30/03/2026 (JPUB V8 non pronto)
9. **Fase A Regime:** Invio diretto a JPUB da 31/03/2026 (JPUB V8 operativo)
---
**Data Documento:** 26 Febbraio 2026  
**Versione:** 1.0  
**Scope:** Analisi completa flusso ST_354378 SemiGPON/FullGPON OAO→NOW→JPUB
ANALISI_EOF
cat /tmp/ANALISI_ORDINE_OAO_ST354378.md
                EC=$?
                echo "___BEGIN___COMMAND_DONE_MARKER___$EC"
            }
___BEGIN___COMMAND_OUTPUT_MARKER___
# ANALISI COMPLETA: FLUSSO ORDINE OAO → NOW (ST_354378 - SemiGPON/FullGPON)

**Data Analisi:** 26 Febbraio 2026  
**Scope:** Tracciamento completo del flusso ordine da OAO (Operatore Wholesale) verso sistema NOW per offerte SemiGPON e FullGPON  
**Versione Interfaccia:** NOW-OLO v5.2, Tracciato V8 NOW-JPUB, ST_354378

---

## 1. MODALITÀ D'INVIO ORDINE (OAO → NOW)

### Canale di Trasmissione
**MODALITÀ: FILE XML (principalmente) + Opzione WS secondaria**

#### 1.1 Trasmissione Primaria: FILE XML
- **Protocollo:** File-based transfer (upload tramite FTP/SFTP o interfaccia web)
- **Formato:** XML conforme a schema XSD NOW_Tracciato_OLO_ServiziNGA_v5.2.xsd
- **Charset:** UTF-8
- **Validazione:** Schemi XSD versione 5.2

#### 1.2 Trasmissione Secondaria: WebService (WS)
- **Canale alternativo:** Integrazione WS per operatori con elevato volume transazionale
- **Standard:** SOAP/REST (in base a configurazione operatore)
- **Velocità:** Invio unitario ordine vs batch file

#### 1.3 Punti di Ingresso Fisici nel Sistema NOW
- **File Manager:** Acquisizione file schedulata (time-based o event-based)
- **GUI NOW:** Interfaccia web per OAO (se presente)
- **API/WS:** Endpoint dedicati per integrazioni B2B

---

## 2. NAMING CONVENTION FILE ORDINE OAO

### 2.1 Formato Generale
```
<OPERATORE_CODE>_<GGMMAAAA>_<HH>_<TIPO>.xml
```

Dove:
- **<OPERATORE_CODE>:** Codice ISP a 3 caratteri (es. MTW = Fastweb/Metavisione)
- **<GGMMAAAA>:** Data invio (GG=giorno, MM=mese, AAAA=anno) - 8 caratteri numerici
- **<HH>:** Ora estrazione - '00'-'23' (parziale ogni ora), 'RG' (riepilogo giornaliero), 'EX' (straordinario)
- **<TIPO>:** Tipologia richiesta (es. NGA, MIGRAZIONI, ecc.)

### 2.2 Esempio Concreto da Documentazione
```
NEWGPONMTW2026021102.xml
├─ NEW          = Prefisso ordine GPON
├─ GPON         = Servizio GPON
├─ MTW          = Codice OLO Fastweb
├─ 20260211     = Data 11 febbraio 2026
└─ 02           = Ora 02:00

Oppure:
MTW_20260211_02_NGA.xml
├─ MTW          = Codice OLO
├─ 20260211     = Data
├─ 02           = Ora
└─ NGA          = Servizio NGA
```

### 2.3 File di Notifica (Risposta NOW → OAO)
```
<ISP>_<GGMMAAAA>_<HH>_NGA.xml.gz
```
Dove HH:
- **'00'-'23':** Parziale ogni ora
- **'RG':** Riepilogo giornaliero (entro le 10:00 del giorno successivo)
- **'EX':** File straordinario

**Esempio:** `MTW_20260211_RG_NGA.xml.gz` (riepilogo giornaliero compresso)

---

## 3. STRUTTURA COMPLETA FILE XML ORDINE (RichiestaOAO)

### 3.1 Elemento Radice
```xml
<?xml version="1.0" encoding="UTF-8"?>
<RichiestaOAO>
    <Richiesta>
        <!-- STRUTTURA ORDINE -->
    </Richiesta>
</RichiestaOAO>
```

### 3.2 Struttura Completa (Schema XSD)
```
RichiestaOAO (elemento radice)
│
└── Richiesta (0..1000 occorrenze - batch di ordini)
    │
    ├── Testata (OBBLIGATORIO)
    │   ├── CodiceOlo (OBBLIGATORIO, 3 car, es. "MTW")
    │   ├── CodiceOrdineOlo (OPZ, max 30 car, ID ordine operatore)
    │   ├── CodiceOrdineOloPadre (OPZ, max 30 car, per ordini padre)
    │   ├── CodiceProgetto (OPZ, max 15 car, es. "GC24748301")
    │   ├── Note (OPZ, max 255 car)
    │   ├── TestMode (OPZ, valore "0" = test)
    │   ├── [CHOICE - uno tra i seguenti]
    │   │   ├── semiVULA (OPZ, valore "SI")
    │   │   ├── semiBitstreamNGA (OPZ, valore "SI")
    │   │   ├── semiEasyIPNGA (OPZ, valore "SI")
    │   │   └── FullGPONPLUS (OPZ, valore "SI")
    │   ├── IdReteSecondariaOAO (OPZ, max 19 car, ID rete secondaria operatore)
    │   ├── ReferentiOAOProgettiSpeciali (OPZ, max 255 car)
    │   └── CodiceOperatoreSecondario (OPZ, max 3 car)
    │
    └── Ordine (OBBLIGATORIO - una scelta tra i tipi sottostanti)
        │
        ├── Attivazione (per ordini di ATTIVAZIONE nuovo servizio)
        │   └── AccessoGPON (NUOVO ST_354378 - SemiGPON/FullGPON)
        │       ├── DataDAI (es. "18/02/2026")
        │       ├── DatiCliente (contenitore dati cliente)
        │       │   ├── CognomeRagSoc
        │       │   ├── Nome
        │       │   └── CF (Codice Fiscale)
        │       ├── Sede (indirizzo installazione)
        │       │   ├── ParticellaToponomastica (es. "PIAZZA")
        │       │   ├── Via (es. "GIUSEPPE MAGGIOLINI")
        │       │   ├── Civico (es. "29")
        │       │   ├── Localita (es. "PARABIAGO")
        │       │   ├── CodiceIstatIndirizzo (format: "15_168_19344_29")
        │       │   ├── CodiceEGON (identificativo geografico)
        │       │   └── CodiceComune (es. "03015168")
        │       ├── Referente (contatti cliente)
        │       │   ├── Progressivo (es. "1")
        │       │   ├── QualificaRef (es. "CF")
        │       │   ├── CognomeReferente
        │       │   ├── NomeReferente
        │       │   └── TelefonoMobileReferente
        │       ├── ReferenteOnField (supporto in campo)
        │       │   ├── CognomeReferente
        │       │   ├── NomeReferente
        │       │   └── TelefonoFissoReferente
        │       ├── IdGPON (OBBLIGATORIO ST_354378, es. "FWE0000000000083517")
        │       ├── TipologiaAccessoUBB (OBBLIGATORIO ST_354378)
        │       │   └── Valori: "S" (SemiGPON) | "F" (FullGPON)
        │       └── ReferentiOAOProgettiSpeciali (codice progetto es. "GC24748301")
        │
        ├── Cessazione (per ordini di CESSAZIONE servizio)
        │   └── AccessoGPON
        │       ├── [Struttura simile ad Attivazione]
        │       └── [Motivi cessazione, date, ecc.]
        │
        └── Variazione (per ordini di VARIAZIONE parametri servizio)
            └── AccessoGPON
                ├── [Parametri da variare]
                ├── [Profili banda modificati]
                └── [VLAN, CoS, ecc.]
```

---

## 4. DETTAGLIO CAMPO PER CAMPO - ELEMENTO <AccessoGPON1BPNRR> (ST_354378)

### 4.1 Sezione TESTATA

| # | Campo | Tipo | Obbligatorietà | Descrizione | Valori Ammessi | Esempio | Note |
|---|-------|------|---|---|---|---|---|
| 1 | CodiceOlo | CHAR(3) | OBB | Codice ISP/OAO identificativo dell'operatore | Es. MTW, TIM, WND | "MTW" | Fisso 3 caratteri |
| 2 | CodiceOrdineOlo | VARCHAR(30) | OPZ | Numero ordine assegnato dall'OAO | Alfanumerico | "FCC00000001" | Max 30 car, può essere usato come riferimento |
| 3 | CodiceOrdineOloPadre | VARCHAR(30) | OPZ | ID ordine padre (relazioni) | Alfanumerico | — | Per ordini correlati |
| 4 | CodiceProgetto | VARCHAR(15) | OPZ | Codice progetto/incentivo | Alfanumerico | "GC24748301" | Progetti speciali PNRR, CNO |
| 5 | Note | VARCHAR(255) | OPZ | Note libere dell'operatore | Testo libero | — | Max 255 car |
| 6 | TestMode | CHAR(1) | OPZ | Modalità di test | "0" solo | "0" | Valore fisso per test |
| 7 | ReferentiOAOProgettiSpeciali | VARCHAR(255) | OPZ | Referenti per progetti speciali | Testo | "GC24748301" | Contatti operatore |
| 8 | IdReteSecondariaOAO | VARCHAR(19) | OPZ | ID rete secondaria operatore (Semi-VULA) | Codice rete | — | Solo se OAO ha rete secondaria |
| 9 | CodiceOperatoreSecondario | VARCHAR(3) | OPZ | Codice operatore secondario correlato | 1-3 caratteri | — | Operatori corrispondenti |

### 4.2 Sezione DATI CLIENTE

| # | Campo | Tipo | Obbligatorietà | Descrizione | Valori Ammessi | Esempio | Note |
|---|-------|------|---|---|---|---|---|
| 1 | CognomeRagSoc | VARCHAR | OBB | Cognome cliente o Ragione Sociale | Testo | "BERNINI" | Obbligatorio per identificazione |
| 2 | Nome | VARCHAR | OPZ | Nome cliente (se persona fisica) | Testo | "MASSIMO" | Non richiesto per aziende |
| 3 | CF | VARCHAR(16) | COND | Codice Fiscale | Formato CF italiano | "BRNMSM83S06E801P" | Obbligatorio se non PIVA |
| 4 | PIVA | VARCHAR(11) | COND | Partita IVA | Formato PIVA italiano | — | Alternativa CF |

**Condizioni:** CF e PIVA sono **mutuamente esclusivi** - almeno uno deve essere presente

### 4.3 Sezione SEDE/INDIRIZZO

| # | Campo | Tipo | Obbligatorietà | Descrizione | Valori Ammessi | Esempio | Note |
|---|-------|------|---|---|---|---|---|
| 1 | ParticellaToponomastica | VARCHAR | OBB | Tipologia via | PIAZZA, VIA, VIALE, CORSO, ecc. | "PIAZZA" | Da NetMap DB Toponomastica |
| 2 | Via | VARCHAR | OBB | Nome strada | Testo | "GIUSEPPE MAGGIOLINI" | Da NetMap |
| 3 | Civico | VARCHAR | OBB | Numero civico | Numerico/alfanumerico | "29" | Da NetMap |
| 4 | Localita | VARCHAR | OBB | Nome città/comune | Testo | "PARABIAGO" | Da NetMap |
| 5 | CodiceIstatIndirizzo | VARCHAR | OBB | Codice ISTAT indirizzo | Format: NN_NNN_NNNNN_NN | "15_168_19344_29" | Identificativo univoco indirizzo da ISTAT |
| 6 | CodiceEGON | VARCHAR | OBB | Codice EGON | Numerico, 15 car | "380100023791543" | Identificativo geografico NetMap |
| 7 | CodiceComune | VARCHAR | OBB | Codice Istat Comune | 8 caratteri numerici | "03015168" | Per validazione geografica |

**Validazione:** Tutti i dati SEDE devono corrispondere esattamente ai valori riportati nel DB "Banca Dati di Copertura Tecnica e Toponomastica NetMap"

### 4.4 Sezione REFERENTE CLIENTE

| # | Campo | Tipo | Obbligatorietà | Descrizione | Valori Ammessi | Esempio | Note |
|---|-------|------|---|---|---|---|---|
| 1 | Progressivo | VARCHAR | OBB | Numero sequenziale referente | 1, 2, 3, ... | "1" | Per profili FTTH può arrivare a 3 |
| 2 | QualificaRef | VARCHAR | OBB | Qualifica referente | "CF", "REP", "LEGALE", ecc. | "CF" | Cf = Cliente Finale |
| 3 | CognomeReferente | VARCHAR | OBB | Cognome referente | Testo | "BERNINI" | — |
| 4 | NomeReferente | VARCHAR | OBB | Nome referente | Testo | "MASSIMO" | — |
| 5 | TelefonoMobileReferente | VARCHAR(10) | COND | Telefono mobile | Numerico | "3493659110" | Uno tra Fisso/Mobile obbligatorio |
| 6 | TelefonoFissoReferente | VARCHAR(10) | COND | Telefono fisso | Numerico | — | Uno tra Fisso/Mobile obbligatorio |
| 7 | Email | VARCHAR | OPZ | Email referente | RFC5321 | — | Solo per FTTH profili X1/X2 |

**Regola:** Almeno uno tra TelefonoMobileReferente e TelefonoFissoReferente deve essere valorizzato

### 4.5 Sezione REFERENTE ON-FIELD (Tecnico in campo)

| # | Campo | Tipo | Obbligatorietà | Descrizione | Valori Ammessi | Esempio | Note |
|---|-------|------|---|---|---|---|---|
| 1 | CognomeReferente | VARCHAR | OBB | Cognome tecnico | Testo | "FASTWEB" | Nome azienda tecnica |
| 2 | NomeReferente | VARCHAR | OBB | Nome tecnico/tipo intervento | Testo | "TIPOG" | Tipologia intervento |
| 3 | TelefonoFissoReferente | VARCHAR(10) | OBB | Contatto tecnico | Numerico | "0245400106" | Contatto operatore lavori |

### 4.6 Sezione DATI ACCESSO GPON (NUOVO ST_354378)

| # | Campo | Tipo | Obbligatorietà | Descrizione | Valori Ammessi | Esempio | Note |
|---|-------|------|---|---|---|---|---|
| 1 | DataDAI | VARCHAR(10) | OPZ | Data Desiderata Attivazione (formato GG/MM/AAAA) | Data | "18/02/2026" | Solo giorni lavorativi Lun-Ven |
| 2 | **IdGPON** | VARCHAR | **OBB** | **Identificativo univoco albero GPON** | **Codice IDGPON standard** | **"FWE0000000000083517"** | **NUOVO ST_354378 - Id nodo GPON dove collegarsi** |
| 3 | **TipologiaAccessoUBB** | CHAR(1) | **OBB** | **Tipologia accesso infrastruttura GPON** | **S = SemiGPON**<br>**F = FullGPON** | **"S"** | **NUOVO ST_354378 - Determina se accesso su rete secondaria (S) o primaria+secondaria (F)** |
| 4 | **CaratterizzazioneGeografica** | VARCHAR | **OBB** (da v8) | **Classificazione geografica area** | **PNRR = Area PNRR**<br>**AREA_NERA = Mercato competitivo**<br>**AREA_GRIGIA = Operatori limitati** | **"PNRR"** | **NUOVO ST_354378 - Acquisita da ANAGRAFICA_GPON via IDGPON** |
| 5 | **CLLI_CRO** | VARCHAR | **OBB** (da v8) | **Codice CLLI Cabinet Ripartizione Ottica** | **Codice CLLI standard** | **—** | **NUOVO ST_354378 - Estratto da anagrafica GPON** |

### 4.7 Sezione DATI TECNICI ACCESSO

| # | Campo | Tipo | Obbligatorietà | Descrizione | Valori Ammessi | Esempio | Note |
|---|-------|------|---|---|---|---|---|
| 1 | CodiceQualita | CHAR(1) | OPZ | Livello qualità SLA | "X", "Y", "Z" | "X" | Profili SLA differenziati |
| 2 | SlaAssurance | VARCHAR | OPZ | Tipo SLA | 0, 1, 2, 3, ecc. | "0" | Default "0" |
| 3 | CodiceDelivery | VARCHAR(10) | OPZ | Codice delivery/modalità consegna | Alfanumerico | — | Parametri consegna |
| 4 | DAD | DATE | OPZ | Data Desiderata Attivazione | GG/MM/AAAA | "18/02/2026" | Solo giorni lavorativi Lun-Ven, escluse festività |
| 5 | FasciaOraria | VARCHAR | OPZ | Fascia oraria intervento | Valori definiti | — | Es. mattina, pomeriggio |
| 6 | CampoServizioIT | VARCHAR(10) | OPZ | Campo servizio (uso FiberCop) | Alfanumerico | — | Uso esclusivo FiberCop |
| 7 | Qualifica | VARCHAR | OPZ | Richiesta qualificazione collaudo | "SI"/"" (vuoto=SI) | "SI" | Prestazione ex Del. 87/18/CIR |
| 8 | Test2 | VARCHAR | OPZ | Richiesta test PPP | "SI"/"" (vuoto=SI) | — | Prestazione ex Del. 87/18/CIR |

**Regole Speciali:**
- Se **Qualifica** omesso o vuoto → si intende **"SI"** (collaudo obbligatorio)
- Se **Test2** omesso o vuoto → si intende **"SI"** (test PPP obbligatorio)
- **DAD** può essere valorizzato solo con **giorni lavorativi** (Lun-Ven, escluse festività infrasettimanali)

---

## 5. DISTINZIONE TRA TIPOLOGIE ORDINE (Attivazione, Cessazione, Variazione)

### 5.1 ORDINE DI ATTIVAZIONE

**Elemento XML:**
```xml
<Richiesta>
    <Testata>...</Testata>
    <Ordine>
        <Attivazione>
            <AccessoGPON>
                <!-- Campi Attivazione GPON -->
            </AccessoGPON>
        </Attivazione>
    </Ordine>
</Richiesta>
```

**Caratteristiche:**
- **Obiettivo:** Attivare nuovo servizio GPON su punto cliente
- **Campo TipologiaAccessoUBB:** Valorizzato con "S" (SemiGPON) o "F" (FullGPON)
- **IdGPON:** Obbligatorio - identifica nodo GPON di collegamento
- **DAI/DAD:** Data desiderata inizio servizio
- **Referente:** Almeno 1 referente cliente presente (per profili X1/X2 FTTH fino a 3)
- **Email:** Opzionale (obbligatorio solo per FTTH X1/X2)
- **Regola SLA:** Se `SlaAssurance` escluso o empty → default "0"

**Controlli specifici attivazione:**
1. Verifica codice OLO/OAO valido
2. Verifica compatibilità contratto SEMIGPON1BPNRR o FULLGPON1BPNRR
3. Verifica IdGPON presente in ANAGRAFICA_GPON
4. Verifica TipologiaAccessoUBB coerente con tipo contratto
5. Verifica indirizzo cliente da NetMap
6. Verifica disponibilità infrastruttura GPON


### 5.2 ORDINE DI CESSAZIONE

**Elemento XML:**
```xml
<Richiesta>
    <Testata>...</Testata>
    <Ordine>
        <Cessazione>
            <AccessoGPON>
                <!-- Campi Cessazione GPON -->
                <MotivoCessazione>...</MotivoCessazione>
                <DataCessazione>...</DataCessazione>
            </AccessoGPON>
        </Cessazione>
    </Ordine>
</Richiesta>
```

**Caratteristiche:**
- **Obiettivo:** Disattivare servizio GPON esistente
- **MotivoCessazione:** Motivo disattivazione (cambio operatore, fine contratto, ecc.)
- **DataCessazione:** Data richiesta disattivazione
- **IdGPON:** Riferimento al nodo GPON da cui staccare
- **Referente:** Sempre richiesto (almeno 1)

**Scenari Cessazione:**
- **Cessazione Pura:** Disattivazione senza rientro in altro operatore
- **Cessazione con Rientro:** Disattivazione con Number Portability (NP)
- **Cessazione con Variazione:** Cambio profilo/velocità prima disattivazione

---

### 5.3 ORDINE DI VARIAZIONE

**Elemento XML:**
```xml
<Richiesta>
    <Testata>...</Testata>
    <Ordine>
        <Variazione>
            <AccessoGPON>
                <!-- Campi che cambiano -->
                <ProfiloTecnicoLinea>V2</ProfiloTecnicoLinea>
                <Banda_Picco_Down>50</Banda_Picco_Down>
                <Banda_Profilo_Up>25</Banda_Profilo_Up>
                <!-- Opzionali: riconfigurazioni VLAN, CoS, ecc. -->
            </AccessoGPON>
        </Variazione>
    </Ordine>
</Richiesta>
```

**Caratteristiche:**
- **Obiettivo:** Modificare parametri servizio attivo
- **Campi modificabili:**
  - Profilo tecnico linea (V0-V14 per Bitstream NGA FTTCAB)
  - Banda profilo (download/upload)
  - Banda di picco (download/upload) - NEW: valori fino a 25 (max)
  - Configurazione VLAN tagging/untagged
  - Profili CoS (Class of Service)
  - Parametri SLA
- **Regola:** Possono essere modificati SOLO campi compatibili con offerta
- **Vicoli:** Alcune combinazioni di banda non ammesse

**Tipi di Variazione:**
1. **Riconfigurazione profilatoria:** cambio velocità/profilo
2. **Riconfigurazione VLAN:** modifica VLAN tagging
3. **Riconfigurazione CoS:** cambio QoS/priorità traffico
4. **Variazione Multi-CoS:** per offerte multiservice

---

## 6. DIFFERENZE TRACCIATO VECCHIO (AccessoGPON legacy UBB) vs NUOVO (AccessoGPON1BPNRR ST_354378)

### 6.1 Comparazione Versioni

| Aspetto | VECCHIO (Pre-ST_354378) | NUOVO (ST_354378) | Cambio |
|--------|---|---|---|
| **Nome Elemento** | `<AccessoGPON>` | `<AccessoGPON1BPNRR>` | ✅ Rinomato |
| **Contratti Supportati** | UBBLEGACY, UBBITALIA, ecc. | SEMIGPON1BPNRR, FULLGPON1BPNRR | ✅ Nuovo dominio contratti |
| **IdGPON** | Opzionale | ✅ **OBBLIGATORIO** | ✅ Reso obbligatorio |
| **TipologiaAccessoUBB** | ❌ Non presente | ✅ **NUOVO - OBBLIGATORIO** | ✅ Aggiunto |
| **CaratterizzazioneGeografica** | ❌ Non presente | ✅ **NUOVO - OBBLIGATORIO (da V8)** | ✅ Aggiunto |
| **CLLI_CRO** | ❌ Non presente | ✅ **NUOVO - OBBLIGATORIO (da V8)** | ✅ Aggiunto |
| **Interfaccia JPUB** | XSD v2.23 | ✅ **XSD v8 (NUOVO)** | ✅ Evoluto |
| **Tracciato DTU** | v7.60 | ✅ **v7.70+ (con TipologiaAccessoUBB)** | ✅ Esteso |
| **Georeferenziazione** | Basic (Comune) | ✅ **Avanzata (CRO/CNO + PNRR/AREA_NERA/AREA_GRIGIA)** | ✅ Potenziata |
| **Rete GPON** | Accesso singolo | ✅ **Accesso primaria+secondaria (FullGPON) o solo secondaria (SemiGPON)** | ✅ Differenziato |

### 6.2 Visualizzazione Strutturale

**VECCHIO (Legacy):**
```xml
<AccessoGPON>
    <DataDAI>18/02/2026</DataDAI>
    <DatiCliente>...</DatiCliente>
    <Sede>...</Sede>
    <Referente>...</Referente>
    <ReferenteOnField>...</ReferenteOnField>
    <!-- IdGPON: OPZIONALE -->
    <IdGPON>FWE0000000000083517</IdGPON>
    <!-- NO: TipologiaAccessoUBB -->
    <!-- NO: CaratterizzazioneGeografica -->
    <!-- NO: CLLI_CRO -->
</AccessoGPON>
```

**NUOVO (ST_354378):**
```xml
<AccessoGPON1BPNRR>
    <DataDAI>18/02/2026</DataDAI>
    <DatiCliente>...</DatiCliente>
    <Sede>...</Sede>
    <Referente>...</Referente>
    <ReferenteOnField>...</ReferenteOnField>
    <!-- NEW: IdGPON OBBLIGATORIO -->
    <IdGPON>FWE0000000000083517</IdGPON>
    <!-- NEW: OBBLIGATORIO - S|F -->
    <TipologiaAccessoUBB>S</TipologiaAccessoUBB>
    <!-- NEW (da V8 JPUB): OBBLIGATORIO -->
    <CaratterizzazioneGeografica>PNRR</CaratterizzazioneGeografica>
    <!-- NEW (da V8 JPUB): OBBLIGATORIO -->
    <CLLI_CRO>CRO123456</CLLI_CRO>
    <ReferentiOAOProgettiSpeciali>GC24748301</ReferentiOAOProgettiSpeciali>
</AccessoGPON1BPNRR>
```

---

## 7. CAMPI NUOVI INTRODOTTI DA ST_354378

### 7.1 Nuovi Campi nel Tracciato OLO (NOW ↔ OAO)

| # | Campo | Tipo | Obbligatorietà | Descrizione | Valori | Fase Introduzione |
|---|-------|------|---|---|---|---|
| 1 | **IdGPON** | VARCHAR | OBB (per GPON) | Identificativo univoco nodo GPON | Codice IDGPON | C1-2026 (attivo) |
| 2 | **TipologiaAccessoUBB** | CHAR(1) | OBB (per GPON) | Tipologia accesso GPON | S=SemiGPON, F=FullGPON | C1-2026 (attivo) |
| 3 | **CaratterizzazioneGeografica** | VARCHAR | OBB (V8 JPUB) | Caratterizzazione area geografica | PNRR, AREA_NERA, AREA_GRIGIA | C3-2026 (Fase A Regime) |
| 4 | **CLLI_CRO** | VARCHAR | OBB (V8 JPUB) | Codice CLLI Cabinet Ripartizione | Codice standard CLLI | C3-2026 (Fase A Regime) |

### 7.2 Nuovi Campi nel Tracciato JPUB (NOW → JPUB v8)

**XSD Tracciato V8 (NEW)** - introdotti campi per billing:

```
OrdiniCRMWS_Reg_2.23.xsd (versione attuale)
    ↓ EVOLVE
OrdiniCRMWS_Reg_V8.xsd (NUOVO ST_354378)
    ├── TIPOLOGIA_ACCESSO_UBB (NEW)
    ├── CARATTERIZZAZIONE_GEOGRAFICA (NEW)
    ├── IDGPON (NEW)
    └── CLLI_CRO (NEW)
```

### 7.3 Nuovi Campi nel Tracciato DTU (NOW → DTU)

**Versione DTU v7.70+** introduce:
```
<TIPOLOGIA_ACCESSO_UBB> → valori S|F
```

### 7.4 Nuovi Campi Sincronizzazione NNI

**Interfaccia NNI (Network-to-Network) v1.6+** introduce:
```
ANAGRAFICA_GPON
├── IDGPON (chiave)
├── TIPOLOGIA_ACCESSO_UBB
├── CARATTERIZZAZIONE_GEOGRAFICA
├── CLLI_CRO
└── [altri dati infrastruttura GPON]
```

---

## 8. FLUSSO DI ACQUISIZIONE: COSA FA NOW QUANDO RICEVE IL FILE

### 8.1 Fasi di Elaborazione Generale (Canale FILE)

```
┌─────────────────────────────────────────────────────────────────┐
│ FASE 1: RICEZIONE FILE                                          │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • File Manager NOW monitora directory ingresso
       • Rileva file xml con naming convention corretto
       • Salva in tabella RICHIESTA_FILE_INGRESSO
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 2: VALIDAZIONE STRUTTURA XSD                              │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Valida file XML contro XSD v5.2
       • Controllo: encoding UTF-8
       • Controllo: well-formed XML
       • Controllo: conformità schema elementi/attributi
       ↓
       ├─ ✅ VALIDA XSD → FASE 3
       └─ ❌ FALLISCE → Scarto file, notifica errore a OAO
           (Status: FILE_SCARTATO, Email notifica)
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 3: ACQUISIZIONE RICHIESTE BATCH (C01)                     │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Legge file XML elemento per elemento <Richiesta>
       • Per ogni richiesta nel file:
         - Estrae dati testata e ordine
         - Crea record RICHIESTA_OLO in DB
         - Assegna ID_RICHIESTA univoco NOW
         - Stato iniziale: ACCOLTA
       ↓
       • Registra:
         ├─ CodiceOlo (operatore)
         ├─ CodiceOrdineOlo (ID OAO)
         ├─ Timestamp ricezione
         ├─ Tipo lavorazione (ATT/CESS/VAR)
         └─ Tipo offerta (GPON, ecc.)
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 4: VALIDAZIONI LOGICHE (C04 - Gestione Verifiche)         │
└─────────────────────────────────────────────────────────────────┘
       ↓
       Per ogni richiesta acquisita:
       
       A. CONTROLLI DI BASE
          1. Verifica CodiceOlo valido ✓ (funz. C04M01F09)
          2. Verifica OLO Accorpato ✓ (funz. C04M01F30)
          3. Verifica OLO Richiesta ✓ (funz. C04M03F22)
          4. Verifica Servizio in Consistenza ✓ (funz. C04M03F04)
       
       B. CONTROLLI SPECIFICI GPON (ST_354378)
          1. Verifica IdGPON valorizzato e valido
          2. Verifica IdGPON in ANAGRAFICA_GPON
          3. Verifica TipologiaAccessoUBB (S|F)
          4. Verifica coerenza TipologiaAccessoUBB ↔ Tipo Contratto
             └─ "S" → Contratto SEMIGPON1BPNRR
             └─ "F" → Contratto FULLGPON1BPNRR
          5. Verifica indirizzo cliente in NetMap DB
          6. Verifica Referente almeno 1
       
       C. CONTROLLI TERRITORIALI
          1. Verifica coordinate geografiche (CodiceComune, CodiceIstat)
          2. Verifica disponibilità infrastruttura GPON (NNI sync)
          3. Query NetMap per disponibilità tecnica
       
       D. CONTROLLI DI CONSISTENZA
          1. Se attivazione: verifica non doppio ordine
          2. Se cessazione: verifica linea attiva
          3. Se variazione: verifica linea esistente
          4. Verifica compatibilità offerta+tecnologia
       
       ↓
       ├─ ✅ TUTTE VALIDAZIONI OK → FASE 5 (Processing)
       └─ ❌ FALLISCE VALIDAZIONE → Status RICHIESTA: "SCARTATA_VALIDAZIONE"
                                      Crea task utente per analisi
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 5: CREAZIONE ORDINE NOW (C02)                              │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Crea record ORDINE da RICHIESTA_OLO validata
       • Valorizza componenti ordine:
         ├─ ACCESSO (GPON)
         ├─ SERVIZIO (SEMIGPON1BPNRR/FULLGPON1BPNRR)
         ├─ CLIENTE (da DatiCliente XML)
         ├─ REFERENTE (contatti)
         ├─ SEDE (indirizzo)
         └─ DATI_TECNICI (IdGPON, TipologiaAccessoUBB, ecc.)
       
       • Stato ordine: "IN_LAVORAZIONE"
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 6: ORCHESTRAZIONE ORDINE (BPM Process)                    │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Invia ordine a BPM per orchestrazione
       • BPM esegue workflow:
         1. Pre-Delivery checks (inventario, qualità, ecc.)
         2. Planning tecnico (assegnazione risorse)
         3. Provisioning HW (configurazione GPON, IP, ecc.)
         4. Attivazione servizio
         5. Testing (Qualificazione + Test2 per FTTH)
       
       • Stato ordine durante elaborazione: "PROCESSING"
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 7: PARCHEGGIO ORDINI GPON (ST_354378 - FASE TRANSITORIA)  │
└─────────────────────────────────────────────────────────────────┘
       ↓
       ⚠️ SOLO DURANTE C1-2026 (fino a 30/03/2026):
       
       • Se ordine di tipo ATTIVAZIONE con contratto:
         - SEMIGPON1BPNRR → PARCHEGGIO
         - FULLGPON1BPNRR → PARCHEGGIO
       
       • Logica parcheggio:
         1. Ordine creato normalmente
         2. Dopo orchestrazione OK → Status "ESPLETAMENTO_TECNICO_OK"
         3. Invece di inviare a JPUB (Fatturazione)
         4. Inserisce record in tabella: ORDINE_PARCHEGGIATO_JPUB
            ├─ ID_ORDINE
            ├─ STATUS: "IN_ATTESA"
            ├─ IDGPON
            ├─ TIPOLOGIA_ACCESSO_UBB
            ├─ TIMESTAMP_PARCHEGGIO
            └─ [NOTE]
       
       • Ordine rimane visibile in NOW ma NON va a JPUB
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 8: INVIO A JPUB (C08M01)                                   │
└─────────────────────────────────────────────────────────────────┘
       ↓
       ⚠️ SCENARIO A: FASE A REGIME (da C3-2026)
       
       • Dopo bonifica ANAGRAFICA_GPON:
         ├─ Legge IDGPON da ORDINE_PARCHEGGIATO_JPUB
         ├─ Recupera CaratterizzazioneGeografica da ANAGRAFICA_GPON
         ├─ Recupera CLLI_CRO da ANAGRAFICA_GPON
         └─ Prepara payload JPUB con V8 XSD (NEW)
       
       • Payload JPUB v8 include:
         ├─ TipologiaAccessoUBB (S|F)
         ├─ CaratterizzazioneGeografica (PNRR|AREA_NERA|AREA_GRIGIA)
         ├─ IDGPON
         ├─ CLLI_CRO
         └─ [Dati standard ordine]
       
       • Invio a JPUB via WebService (SOAP)
       • JPUB riceve → Fatturazione ordine
       ↓
       ├─ ✅ JPUB OK → Status "FATTURATO", Ordine CHIUSO
       └─ ❌ JPUB KO → Status "ERRORE_JPUB", Task utente per recovery
       ↓
┌─────────────────────────────────────────────────────────────────┐
│ FASE 9: NOTIFICA FINALE A OAO                                   │
└─────────────────────────────────────────────────────────────────┘
       ↓
       • Genera file notifica con stato finale ordine
       • Formato: ISP_GGMMAAAA_HH_NGA.xml.gz
       • Contenuto: ID ordine NOW, stato, date, eventuali errori
       • Invia via sftp/http a operatore
```

### 8.2 Tempo di Elaborazione Atteso
- **Validazione XSD:** < 1 minuto
- **Acquisizione batch:** 1-5 minuti (dipende numero richieste)
- **Validazioni logiche:** 2-10 minuti per richiesta
- **Orchestrazione BPM:** Ore/Giorni (dipende complessità lavori)
- **Invio JPUB:** < 1 minuto post-espletamento tecnico
- **Total E2E:** Ore fino a giorni

---

## 9. VALIDAZIONI ESEGUITE DA NOW SUL FILE RICEVUTO

### 9.1 Validazioni di Primo Livello (Sintassi)

| # | Validazione | Esito Positivo | Esito Negativo | Azione |
|---|---|---|---|---|
| 1 | **File exists** | ✅ File presente | ❌ File non trovato | Retry schedulato |
| 2 | **File naming** | ✅ Segue naming convention | ❌ Nome non conforme | Scarto file |
| 3 | **Encoding** | ✅ UTF-8 | ❌ Encoding diverso | Scarto file |
| 4 | **XML Well-formed** | ✅ XML valido | ❌ XML malformed | Scarto file |
| 5 | **XSD conformance** | ✅ Conforme schema v5.2 | ❌ Non conforme | Scarto file |
| 6 | **Root element** | ✅ `<TracciatoOLONGA>` | ❌ Elemento root diverso | Scarto file |
| 7 | **Multiple requests** | ✅ Max 1000 `<Richiesta>` | ❌ > 1000 richieste | Scarto batch request |
| 8 | **Character length** | ✅ Rispetta limiti XSD | ❌ Campo eccessivamente lungo | Scarto richiesta |

### 9.2 Validazioni di Secondo Livello (Logica)

#### A. VALIDAZIONI TESTATA

| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | CodiceOlo | Obbligatorio | 3 car esatti | Scarto richiesta |
| 2 | CodiceOlo | Codice valido | Presente in anagrafica OLO | Scarto richiesta (C04M01F09) |
| 3 | CodiceOlo | Accorpamento | Verifica OLO accorpati | Scarto richiesta (C04M01F30) |
| 4 | CodiceOrdineOlo | Unicità | Se presente, unico in batch | Warning (non scarta) |
| 5 | CodiceProgetto | Validazione | Se presente, valido | Scarto richiesta |
| 6 | TestMode | Valore | Se presente, solo "0" | Scarto richiesta |
| 7 | Note | Lunghezza | Max 255 caratteri | Scarto richiesta |
| 8 | semiVULA / semiBitstreamNGA / semiEasyIPNGA / FullGPONPLUS | CHOICE (uno solo) | Massimo 1 valore tra questi | Scarto richiesta |
| 9 | IdReteSecondariaOAO | Validazione | Se presente, max 19 car | Scarto richiesta |
| 10 | ReferentiOAOProgettiSpeciali | Lunghezza | Max 255 car | Scarto richiesta |
| 11 | CodiceOperatoreSecondario | Lunghezza | 1-3 car | Scarto richiesta |

#### B. VALIDAZIONI DATI CLIENTE

| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | CognomeRagSoc | Obbligatorio | Non vuoto | Scarto richiesta |
| 2 | CF / PIVA | Mutua esclusività | Uno almeno obbligatorio | Scarto richiesta |
| 3 | CF | Formato | Valido per Italia (16 car) | Scarto richiesta |
| 4 | PIVA | Formato | Valido per Italia (11 car) | Scarto richiesta |
| 5 | Nome | Lunghezza | Max 50 car (stimato) | Scarto richiesta |

#### C. VALIDAZIONI SEDE

| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | ParticellaToponomastica | Obbligatorio | Presente in NetMap | Scarto richiesta |
| 2 | Via | Obbligatorio | Presente in NetMap | Scarto richiesta |
| 3 | Civico | Obbligatorio | Presente in NetMap | Scarto richiesta |
| 4 | Localita | Obbligatorio | Presente in NetMap | Scarto richiesta |
| 5 | CodiceIstatIndirizzo | Obbligatorio | Formato "NN_NNN_NNNNN_NN" | Scarto richiesta |
| 6 | CodiceIstatIndirizzo | Validazione | Presente in NetMap | Scarto richiesta |
| 7 | CodiceEGON | Obbligatorio | 15 car numerici | Scarto richiesta |
| 8 | CodiceComune | Obbligatorio | 8 car, valido ISTAT | Scarto richiesta |
| 9 | Coerenza Sede | Validazione | ParticellaToponomastica + Via + Civico + Localita == NetMap | Scarto richiesta |

#### D. VALIDAZIONI REFERENTE

| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | Progressivo | Obbligatorio | 1, 2, 3 (max 3 per FTTH) | Scarto richiesta |
| 2 | Progressivo | Sequenzialità | Sempre 1,2,3 senza buchi | Scarto richiesta |
| 3 | QualificaRef | Obbligatorio | Non vuoto | Scarto richiesta |
| 4 | CognomeReferente | Obbligatorio | Non vuoto | Scarto richiesta |
| 5 | NomeReferente | Obbligatorio | Non vuoto | Scarto richiesta |
| 6 | TelefonoFisso / Mobile | Mutua obbligatorietà | Almeno 1 valorizzato | Scarto richiesta |
| 7 | TelefonoFisso / Mobile | Formato | Numerico 10 car | Scarto richiesta |
| 8 | Email | Obbligatorio per FTTH X1/X2 | Validazione RFC5321 se presente | Scarto richiesta |
| 9 | Numero Referenti | Obbligatorio | Almeno 1 referente | Scarto richiesta |

#### E. VALIDAZIONI ACCESSO GPON (ST_354378)

| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | IdGPON | **Obbligatorio** | **Presente e valorizzato** | **Scarto richiesta (C02M78F05)** |
| 2 | IdGPON | Validazione | **Presente in ANAGRAFICA_GPON** | **Scarto richiesta** |
| 3 | IdGPON | Compatibilità SemiGPON | **Se TipologiaAccessoUBB="S" → IDGPON deve supportare SemiGPON** | **Scarto richiesta** |
| 4 | IdGPON | Compatibilità FullGPON | **Se TipologiaAccessoUBB="F" → IDGPON deve supportare FullGPON** | **Scarto richiesta** |
| 5 | **TipologiaAccessoUBB** | **Obbligatorio** | **Valore: "S" (SemiGPON) oppure "F" (FullGPON)** | **Scarto richiesta (C02M78F05)** |
| 6 | **TipologiaAccessoUBB** | Coerenza Contratto | **Se="S" → Contratto SEMIGPON1BPNRR**<br>**Se="F" → Contratto FULLGPON1BPNRR** | **Scarto richiesta** |
| 7 | DataDAI | Formato | GG/MM/AAAA | Scarto richiesta |
| 8 | DataDAI | Giorno lavorativo | Lun-Ven, escluse festività infrasettimanali | Scarto richiesta |
| 9 | DataDAI | Coerenza | DataDAI ≥ data odierna (data ordine) | Scarto richiesta |
| 10 | DAD | Formato | GG/MM/AAAA | Scarto richiesta |
| 11 | DAD | Giorno lavorativo | Lun-Ven, escluse festività infrasettimanali | Scarto richiesta |

#### F. VALIDAZIONI DATI TECNICI

| # | Campo | Validazione | Regola | Azione KO |
|---|---|---|---|---|
| 1 | CodiceQualita | Enumerazione | "X", "Y", "Z" se presente | Scarto richiesta |
| 2 | SlaAssurance | Valore | 0-3 o default 0 | Scarto richiesta |
| 3 | CodiceDelivery | Lunghezza | Max 10 car se presente | Scarto richiesta |
| 4 | FasciaOraria | Enumerazione | Valori definiti se presente | Scarto richiesta |
| 5 | CampoServizioIT | Uso esclusivo | Solo per FiberCop (opzionale per OAO) | Warning |
| 6 | Qualifica | Valore | "SI" se presente (default SI se omesso) | — |
| 7 | Test2 | Valore | "SI" se presente (default SI se omesso) | — |

### 9.3 Validazioni di Terzo Livello (Business Logic)

#### A. VALIDAZIONI ORDINE PER TIPO LAVORAZIONE

**Attivazione:**
```
1. Verifica non esista già ordine attivo per stessa linea
2. Verifica IdGPON disponibile (non saturato)
3. Verifica non doppio ordine stesso OAO/Data/Cliente
4. Verifica tipo contratto supportato (SEMIGPON1BPNRR/FULLGPON1BPNRR)
5. Verifica capacità network GPON per nuovo cliente
6. Verifica disponibilità tecnici per Data Desiderata Attivazione
```

**Cessazione:**
```
1. Verifica esista ordine attivo da cessare
2. Verifica linea in stato attivo (non già in cessazione)
3. Verifica coerenza cliente
4. Se con Rientro (NP): verifica OAO destinatario valido
```

**Variazione:**
```
1. Verifica ordine attivo da variare
2. Verifica compatibilità nuovi parametri vs profilo GPON
3. Verifica non sia variazione impossibile (es. downgrade che violerebbe SLA)
4. Verifica non sia duplicata di variazione precedente
5. Verifica non sia variazione illecita (es. cambio cliente)
```

#### B. VALIDAZIONI TERRITORIALI

```
1. Copertura GPON disponibile per indirizzo cliente
2. IdGPON disponibile nella geographic area
3. CodiceComune coerente con area di copertura dell'IdGPON
4. Se progetto PNRR: verifica area dentro perimetro PNRR
5. Se area_nera: verifica mercato competitivo attivo
6. Se area_grigia: verifica limitazione operatori
```

#### C. VALIDAZIONI REGULATORY

```
1. Delibera 82/19/CIR (per migrazioni wholesale)
2. Requisiti di non discriminazione (non preferenze OAO)
3. Separazione contabile (non cross-selling FiberCop)
4. Compliance Data Protection (GDPR per dati cliente)
```

### 9.4 Output Validazioni

**Esito POSITIVO (File Acquisito):**
```
Status FILE:           ACQUISITO_OK
Status RICHIESTE:      ACCOLTA (per ogni richiesta valida)
Email Notifica OAO:    "Pratiche acquisite con esito OK = N"
                       "Pratiche acquisite con esito KO = 0"
                       "Pratiche scartate = 0"
Timestamp:             GGMMAAAA HH:MM:SS
```

**Esito NEGATIVO (File Scartato):**
```
Status FILE:           FILE_SCARTATO / RICHIESTE_SCARTATE
Email Notifica OAO:    "Pratiche acquisite con esito OK = 0"
                       "Pratiche acquisite con esito KO = N"  (se parziale)
                       "Pratiche scartate = M" (se totale)
Errori Dettagliati:    [Lista causali scarto con numeri richiesta]
Task Utente:           Se scarto parziale o errori recuperabili
```

---

## 10. RISPOSTE/NOTIFICHE NOW → OAO DOPO RICEZIONE

### 10.1 Tipologie di Notifiche

#### NOTIFICA 1: ESITO RICEZIONE FILE (Email)

**Timing:** Entro 5 minuti da ricezione file

**Oggetto Email:**
```
File: <isp_filename> - Codice: <code_lavorazione_file>
```

**Corpo Email (caso OK - File acquisito):**
```
DATI IDENTIFICAZIONE
Codice ISP = MTW
Descrizione = Fastweb Metavisione

DATI FILE
Nome file ISP = NEWGPONMTW2026021102.xml

DATI ELABORAZIONE
Totale pratiche presenti = 1
Pratiche acquisite con esito OK = 1
Pratiche acquisite con esito KO = 0
Pratiche scartate = 0
Data elaborazione = 11/02/2026 02:15:30
```

**Corpo Email (caso KO - File scartato):**
```
DATI IDENTIFICAZIONE
Codice ISP = MTW
Descrizione = Fastweb Metavisione

DATI FILE
Nome file ISP = NEWGPONMTW2026021102.xml

DATI ELABORAZIONE
Totale pratiche presenti = 1
Pratiche acquisite con esito OK = 0
Pratiche acquisite con esito KO = 0
Pratiche scartate = 1
Data elaborazione = 11/02/2026 02:15:30

ERRORI
[Lista errori per ogni richiesta scartata]
Richiesta #1: IdGPON non valido - non presente in ANAGRAFICA_GPON
```

#### NOTIFICA 2: FILE DI NOTIFICA (XML compresso)

**Naming:**
```
<ISP>_<GGMMAAAA>_<HH>_NGA.xml.gz
```

**Esempi:**
- `MTW_20260211_00_NGA.xml.gz` - parziale ore 00:00
- `MTW_20260211_RG_NGA.xml.gz` - riepilogo giornaliero (entro 10:00 giorno succ.)
- `MTW_20260211_EX_NGA.xml.gz` - file straordinario (su richiesta)

**Contenuto XML Notifica (semplificato):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<NotificaOLO>
    <Testata>
        <CodiceOlo>MTW</CodiceOlo>
        <DataGenerazione>11/02/2026 10:00:00</DataGenerazione>
        <TipoBatch>RG</TipoBatch> <!-- RG=Riepilogo, EX=Straordinario, 00-23=Parziale -->
    </Testata>
    <Ordini>
        <Ordine>
            <CodiceOrdineOlo>FCC00000001</CodiceOrdineOlo>
            <CodiceOrdineNow>ORD_MTW_2026021101</CodiceOrdineNow>
            <StatoOrdine>ACCOLTA</StatoOrdine> <!-- ACCOLTA, ELABORAZIONE, ESPLETATO_OK, ERRORE -->
            <DataUltimoAggiornamento>11/02/2026 08:30:15</DataUltimoAggiornamento>
            <TipoLavorazione>ATTIVAZIONE</TipoLavorazione>
            <Offerta>SEMIGPON1BPNRR</Offerta>
            <IdGPON>FWE0000000000083517</IdGPON>
            <TipologiaAccessoUBB>S</TipologiaAccessoUBB>
            <CaratterizzazioneGeografica>PNRR</CaratterizzazioneGeografica>
            <DescrizioneStato>Ordine in lavorazione - fase di provisioning</DescrizioneStato>
            <CodiceErrore></CodiceErrore>
            <DescrizioneErrore></DescrizioneErrore>
        </Ordine>
        <Ordine>
            <CodiceOrdineOlo>FCC00000002</CodiceOrdineOlo>
            <CodiceOrdineNow>ORD_MTW_2026021102</CodiceOrdineNow>
            <StatoOrdine>ERRORE</StatoOrdine>
            <TipoLavorazione>ATTIVAZIONE</TipoLavorazione>
            <CodiceErrore>E001</CodiceErrore>
            <DescrizioneErrore>TipologiaAccessoUBB non coerente con tipo contratto FULLGPON1BPNRR (valorizzato 'S' invece di 'F')</DescrizioneErrore>
        </Ordine>
    </Ordini>
</NotificaOLO>
```

#### NOTIFICA 3: NOTIFICA DI AVANZAMENTO (Asincrona durante elaborazione)

**Timing:** Periodica durante elaborazione (es. ogni 6 ore)

**Evento Generato:**
- Pre-Delivery completato
- Provisioning iniziato
- Provisioning completato
- Testing in corso
- Testing completato
- Espletamento tecnico positivo
- Espletamento tecnico negativo

**Formato:** File XML o Email con stato e timestamp

#### NOTIFICA 4: NOTIFICA FINALE (Quando ordine entra in stato terminale)

**Timing:** Quando ordine raggiunge uno stato stabile

**Stati Possibili:**
```
✅ ESPLETAMENTO_TECNICO_OK
   ↓ Attesa invio JPUB o Parcheggio
   ↓
💾 ORDINE_PARCHEGGIATO (ST_354378 - FASE TRANSITORIA)
   ↓ In attesa bonifica da ANAGRAFICA_GPON
   ↓
📤 INVIATO_JPUB (da C3-2026)
   ↓
✅ FATTURATO (JPUB OK)

❌ ESPLETAMENTO_TECNICO_KO
   → Causa + Descrizione errore
   → Task utente per recovery

❌ SCARTATO
   → Motivo scarto
   → Azioni correttive consigliate
```

### 10.2 Canali di Notifica

| Tipo Notifica | Canale | Timing | Compressione | Allegati |
|---|---|---|---|---|
| Esito Ricezione File | Email | 5 min | No | — |
| File Notifica Parziale | SFTP | Ogni ora | GZ | XML |
| File Notifica Riepilogo Giornaliero | SFTP | Entro 10:00 giorno succ. | GZ | XML |
| File Notifica Straordinaria | SFTP | Su richiesta | GZ | XML |
| Notifica Avanzamento Ordine | Email | Periodica | No | — |
| Notifica Finale Ordine | Email + File | A fine ordine | No/GZ | XML |

### 10.3 Credenziali e Destinatari

```
Email Operatore:       ReferentiOAOProgettiSpeciali o contatto di default
SFTP Server:           ftp.now.fibercop.it
Directory Ingresso:    /in/olo_ordini/
Directory Notifiche:   /out/olo_notifiche/
Autenticazione:        Certificate-based o User/Pass
```

---

## 11. ESEMPIO CONCRETO: FILE XML ORDINE OAO (DA DOCUMENTAZIONE)

### Esempio 1: ATTIVAZIONE SemiGPON (da NEWGPONMTW2026021102.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<RichiestaOAO>
<Richiesta>
    <Testata>
        <CodiceOlo>MTW</CodiceOlo>
        <CodiceOrdineOlo>FCC00000001</CodiceOrdineOlo>
    </Testata>
    <Ordine>
        <Attivazione>
            <AccessoGPON>
                <DataDAI>18/02/2026</DataDAI>
                <DatiCliente>
                    <CognomeRagSoc>BERNINI</CognomeRagSoc>
                    <Nome>MASSIMO</Nome>
                    <CF>BRNMSM83S06E801P</CF>
                </DatiCliente>
                <Sede>
                    <ParticellaToponomastica>PIAZZA</ParticellaToponomastica>
                    <Via>GIUSEPPE MAGGIOLINI</Via>
                    <Civico>29</Civico>
                    <Localita>PARABIAGO</Localita>
                    <CodiceIstatIndirizzo>15_168_19344_29</CodiceIstatIndirizzo>
                    <CodiceEGON>380100023791543</CodiceEGON>
                    <CodiceComune>03015168</CodiceComune>
                </Sede>
                <Referente>
                    <Progressivo>1</Progressivo>
                    <QualificaRef>CF</QualificaRef>
                    <CognomeReferente>BERNINI</CognomeReferente>
                    <NomeReferente>MASSIMO</NomeReferente>
                    <TelefonoMobileReferente>3493659110</TelefonoMobileReferente>
                </Referente>
                <ReferenteOnField>
                    <CognomeReferente>FASTWEB</CognomeReferente>
                    <NomeReferente>TIPOG</NomeReferente>
                    <TelefonoFissoReferente>0245400106</TelefonoFissoReferente>
                </ReferenteOnField>
                <!-- NUOVO ST_354378 - Campi obbligatori per GPON -->
                <IdGPON>FWE0000000000083517</IdGPON>
                <TipologiaAccessoUBB>S</TipologiaAccessoUBB>
                <!-- NUOVO ST_354378 (da JPUB V8) -->
                <!-- Nota: CaratterizzazioneGeografica e CLLI_CRO NON presenti nel file OAO
                         ma verranno aggiunti da NOW leggedoli da ANAGRAFICA_GPON durante elaborazione -->
                <ReferentiOAOProgettiSpeciali>GC24748301</ReferentiOAOProgettiSpeciali>
            </AccessoGPON>
        </Attivazione>
    </Ordine>
</Richiesta>
</RichiestaOAO>
```

### Esempio 2: ATTIVAZIONE FullGPON (con tutti i campi)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<RichiestaOAO>
<Richiesta>
    <Testata>
        <CodiceOlo>MTW</CodiceOlo>
        <CodiceOrdineOlo>FCC00000002</CodiceOrdineOlo>
        <CodiceProgetto>GC24748302</CodiceProgetto>
        <Note>Ordine prioritario PNRR</Note>
    </Testata>
    <Ordine>
        <Attivazione>
            <AccessoGPON>
                <DataDAI>20/02/2026</DataDAI>
                <DatiCliente>
                    <CognomeRagSoc>ROSSI S.R.L.</CognomeRagSoc>
                    <CF>IT03015168123456</CF>
                </DatiCliente>
                <Sede>
                    <ParticellaToponomastica>VIA</ParticellaToponomastica>
                    <Via>XXIV MAGGIO</Via>
                    <Civico>50</Civico>
                    <Localita>MILANO</Localita>
                    <CodiceIstatIndirizzo>15_146_03111_50</CodiceIstatIndirizzo>
                    <CodiceEGON>380100031111543</CodiceEGON>
                    <CodiceComune>03015146</CodiceComune>
                </Sede>
                <Referente>
                    <Progressivo>1</Progressivo>
                    <QualificaRef>REP</QualificaRef>
                    <CognomeReferente>ROSSI</CognomeReferente>
                    <NomeReferente>FABIO</NomeReferente>
                    <TelefonoFissoReferente>0248124567</TelefonoFissoReferente>
                </Referente>
                <ReferenteOnField>
                    <CognomeReferente>DITTA INSTALLER</CognomeReferente>
                    <NomeReferente>POSA FIBRA</NomeReferente>
                    <TelefonoMobileReferente>3388123456</TelefonoMobileReferente>
                </ReferenteOnField>
                <!-- NUOVO ST_354378 -->
                <IdGPON>FWE0000000000093520</IdGPON>
                <TipologiaAccessoUBB>F</TipologiaAccessoUBB>
                <ReferentiOAOProgettiSpeciali>GC24748302;Support@fastweb.it</ReferentiOAOProgettiSpeciali>
            </AccessoGPON>
        </Attivazione>
    </Ordine>
</Richiesta>
</RichiestaOAO>
```

**Differenze tra Esempio 1 e 2:**
- Es1: TipologiaAccessoUBB = **"S"** (SemiGPON - rete secondaria)
- Es2: TipologiaAccessoUBB = **"F"** (FullGPON - rete primaria+secondaria)
- Es2: IdGPON diverso (nodo GPON diverso)
- Es2: CodiceProgetto valorizzato (progetto PNRR)

---

## 12. SCHEMA XSD TRACCIATO (Definizioni Elementi)

### 12.1 Struttura XSD TracciatoOLONGA v5.2

**Definizione radice:**
```xml
<xs:element name="TracciatoOLONGA">
    <xs:complexType>
        <xs:sequence maxOccurs="1000">
            <xs:element name="Richiesta">
                <xs:complexType>
                    <xs:sequence>
                        <xs:element name="Testata">
                            <!-- Definizioni Testata (vedi sopra) -->
                        </xs:element>
                        <xs:element name="TipoRichiesta">
                            <xs:complexType>
                                <xs:choice>
                                    <xs:element name="Attivazione">...</xs:element>
                                    <xs:element name="Cessazione">...</xs:element>
                                    <xs:element name="Variazione">...</xs:element>
                                </xs:choice>
                            </xs:complexType>
                        </xs:element>
                    </xs:sequence>
                </xs:complexType>
            </xs:element>
        </xs:sequence>
    </xs:complexType>
</xs:element>
```

**Definizione CodiceOlo:**
```xml
<xs:element name="CodiceOlo">
    <xs:simpleType>
        <xs:restriction base="xs:string">
            <xs:maxLength value="3"/>
            <xs:minLength value="3"/>
        </xs:restriction>
    </xs:simpleType>
</xs:element>
```

**Definizione CodiceOrdineOlo:**
```xml
<xs:element name="CodiceOrdineOlo" minOccurs="0">
    <xs:simpleType>
        <xs:restriction base="xs:string">
            <xs:maxLength value="30"/>
            <xs:minLength value="1"/>
        </xs:restriction>
    </xs:simpleType>
</xs:element>
```

**Definizione TestMode:**
```xml
<xs:element name="TestMode" minOccurs="0">
    <xs:simpleType>
        <xs:restriction base="xs:string">
            <xs:maxLength value="1"/>
            <xs:enumeration value="0"/>
        </xs:restriction>
    </xs:simpleType>
</xs:element>
```

**Definizione Choice Servizi Speciali:**
```xml
<xs:choice>
    <xs:element name="semiVULA" minOccurs="0">
        <xs:simpleType>
            <xs:restriction base="xs:string">
                <xs:enumeration value="SI"/>
            </xs:restriction>
        </xs:simpleType>
    </xs:element>
    <xs:element name="semiBitstreamNGA" minOccurs="0">...</xs:element>
    <xs:element name="semiEasyIPNGA" minOccurs="0">...</xs:element>
    <xs:element name="FullGPONPLUS" minOccurs="0">...</xs:element>
</xs:choice>
```

### 12.2 Nuovo Tracciato V8 JPUB (ST_354378)

**Evoluzione schema JPUB:**
```
Versione Attuale:  OrdiniCRMWS_Reg_2.23.xsd (2.23)
                   ↓
Versione Nuova:    OrdiniCRMWS_Reg_V8.xsd (8.0)

Campi Aggiunti:
├── TipologiaAccessoUBB (CHAR 1)
│   ├── Enumerazione: S | F
│   └── Obbligatorio per Semi/Full GPON
├── CaratterizzazioneGeografica (VARCHAR)
│   ├── Enumerazione: PNRR | AREA_NERA | AREA_GRIGIA
│   └── Obbligatorio per Semi/Full GPON
├── IDGPON (VARCHAR)
│   ├── Identificativo univoco albero GPON
│   └── Obbligatorio
└── CLLI_CRO (VARCHAR)
    ├── Codice CLLI Cabinet Ripartizione Ottica
    └── Obbligatorio
```

**Distribuzione XSD V8 nei moduli TIBCO:**
```
Moduli interessati:
├── tim.now.billingmanager
├── tim.now.timerbilling
├── tim.now.tool.billing
├── tim.now.bpm.localBOM
└── tim.now.proxytoexternal
```

---

## 13. FASE TRANSITORIA vs FASE A REGIME (ST_354378)

### 13.1 FASE TRANSITORIA (C1-2026) - DA OGGI FINO 30/03/2026

**Periodo:** Gennaio 2026 → 30 Marzo 2026

**Caratteristiche:**
- JPUB **NON** pronto a ricevere nuovo tracciato V8
- Ordini SemiGPON/FullGPON vengono acquisiti da NOW normalmente
- Dopo espletamento tecnico → **PARCHEGGIO** anzichè invio a JPUB
- Tabella `ORDINE_PARCHEGGIATO_JPUB` con status `IN_ATTESA`

**Flusso:**
```
OAO invia ordine
    ↓
NOW acquisisce + elabora
    ↓
Espletamento tecnico OK
    ↓
Ordine marcato con status "ESPLETAMENTO_TECNICO_OK"
    ↓
❌ NON invia a JPUB (V8 non pronto)
    ↓
Inserisce in ORDINE_PARCHEGGIATO_JPUB
    ├─ STATUS: "IN_ATTESA"
    ├─ IDGPON: da ordine
    ├─ TIPOLOGIA_ACCESSO_UBB: S|F da ordine
    └─ TIMESTAMP_PARCHEGGIO: ora attuale
```

**Gestione Errori Fase Transitoria:**
- Se IdGPON mancante → Ordine scartato (non parcheggiato)
- Se TipologiaAccessoUBB non coerente → Ordine scartato
- Se validazioni falliscono → Task utente per recovery manuale

### 13.2 FASE A REGIME (C3-2026) - DAL 31 MARZO 2026 IN POI

**Periodo:** 31 Marzo 2026 → ...

**Prerequisiti:**
- JPUB versione 8 disponibile e stabile
- ANAGRAFICA_GPON sincronizzata con NNI
- Tutti i test A/B completati

**Caratteristiche:**
- Ordini SemiGPON/FullGPON **inviati direttamente** a JPUB senza parcheggio
- Tracciato JPUB V8 operativo
- Bonifica ordini parcheggiati in fase transitoria

**Flusso Bonifica Ordini Parcheggiati:**
```
Batch process schedulato (es. giornaliero)
    ↓
Seleziona ordini ORDINE_PARCHEGGIATO_JPUB status "IN_ATTESA"
    ↓
Per ogni ordine:
    1. Legge IDGPON
    2. Query ANAGRAFICA_GPON
    3. Recupera:
       ├─ CaratterizzazioneGeografica
       ├─ CLLI_CRO
       └─ [dati aggiornati infrastruttura]
    ↓
    4. Aggiorna payload JPUB con campi V8
    5. Valida coerenza dati
    6. Invia a JPUB via WS
    ↓
    7. Attende risposta JPUB
    ↓
    ├─ ✅ OK → Status "INVIATO" + "FATTURATO"
    └─ ❌ KO → Status "ERRORE", Task utente
```

**Flusso Nuovi Ordini Fase A Regime:**
```
OAO invia ordine
    ↓
NOW acquisisce + elabora
    ↓
Espletamento tecnico OK
    ↓
Preleva da ANAGRAFICA_GPON:
├─ CaratterizzazioneGeografica
├─ CLLI_CRO
└─ [dati geolocalizzazione]
    ↓
Prepara payload JPUB V8 con TUTTI i campi
    ↓
✅ Invia direttamente a JPUB (NO parcheggio)
    ↓
├─ JPUB OK → Fatturazione OK
└─ JPUB KO → Task utente, Retry
```

---

## SINTESI ESECUTIVA

### Punti Chiave
1. **Modalità invio:** FILE XML (primaria) + WS (opzionale)
2. **Naming file:** `<OLO>_<GGMMAAAA>_<HH>_NGA.xml`
3. **Formato dati:** TracciatoOLONGA XSD v5.2 (NOW-OLO interface)
4. **Nuovi campi ST_354378:**
   - IdGPON (obbligatorio)
   - TipologiaAccessoUBB: "S" (SemiGPON) | "F" (FullGPON)
   - CaratterizzazioneGeografica (verso JPUB v8): PNRR / AREA_NERA / AREA_GRIGIA
   - CLLI_CRO (verso JPUB v8)
5. **Validazioni:** 3 livelli (sintassi XSD → logica DB → business rules)
6. **Notifiche risposta:** Email (5 min) + File XML compresso SFTP (orario/giornaliero/straordinario)
7. **Flusso:** Ricezione → Acquisizione → Validazione → Orchestrazione BPM → [PARCHEGGIO transitorio] → Invio JPUB → Fatturazione
8. **Fase Transitoria:** Parcheggio fino 30/03/2026 (JPUB V8 non pronto)
9. **Fase A Regime:** Invio diretto a JPUB da 31/03/2026 (JPUB V8 operativo)

---

**Data Documento:** 26 Febbraio 2026  
**Versione:** 1.0  
**Scope:** Analisi completa flusso ST_354378 SemiGPON/FullGPON OAO→NOW→JPUB

___BEGIN___COMMAND_DONE_MARKER___0
