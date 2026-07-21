# Data Transfer Estimator

Web app per stimare tempi di trasferimento dati, capacità trasmissibile e API request generate da Veeam verso OCI Object Storage (SKU B91627).

🔗 **Live app:** [data-transfer-estimator.vercel.app](https://data-transfer-estimator.vercel.app/)

---

## Funzionalità

### ⏱ Stima il Tempo
Calcola il tempo necessario per trasferire un dato volume di dati in base alla banda upload disponibile e all'overhead di protocollo (TCP/TLS/metadata).

**Input:**
- Dati da trasferire (MB / GB / TB)
- Banda upload (kbps / Mbps / Gbps)
- Overhead di protocollo (slider 0–40%)

### 📦 Stima la Capacità
Calcola quanti dati puoi trasferire in una finestra temporale definita.

**Input:**
- Lasso di tempo (minuti / ore / giorni)
- Banda upload (kbps / Mbps / Gbps)
- Overhead di protocollo (slider 0–40%)

### 📡 API Requests — Veeam + OCI B91627
Stima le API request generate da Veeam Backup & Replication verso OCI Object Storage, confrontandole con le unità contrattualizzate dello SKU B91627 (1 unità = 10.000 request/mese).

Il tab offre **due stime complementari**:

#### 🎯 Stima realistica — Profilo backup (consigliata)
È il modo corretto per rispondere alla domanda «quante request genererà Veeam su OCI»: le request dipendono dai **dati effettivamente scritti** su object storage, non dalla banda. Si basa su full iniziale + change rate giornaliero.

**Input:**
- Dimensione full backup (sorgente protetta, GB/TB)
- Change rate giornaliero (% del full che cambia ogni giorno → incrementale)
- Full attivi/sintetici al mese (0 = forever-incremental, consigliato su object storage)

**Output:**
- **Request/mese a regime** (min–max) → il valore da usare per dimensionare le unità B91627
- **Full iniziale una tantum** → picco nel mese di seeding
- **Primo mese** (full + incrementali)
- Confronto OCI B91627 a regime e margine
- Unità B91627/mese e costo stimato
- Dati scritti su OCI al giorno

#### 📶 Stima da banda — tetto di throughput (verifica fattibilità)
Assume il link saturo per le ore attive: indica il **massimo** di dati/request che l'uplink può reggere. Serve a verificare che la banda sostenga il full iniziale e gli incrementali, **non** è la spesa reale.

**Input:**
- Banda upload nominale e utilizzo effettivo (%)
- Overhead protocollo WAN
- Ore attive/giorno e giorni attivi/mese
- Dataset sorgente totale (opzionale, per stimare i tempi di completamento)

**Parametri comuni (Storage Optimization + Contratto):**
- Dimensione blocco Veeam (256 KB / 512 KB / 1024 KB)
- Dedup ratio **local** (source-side) — riduce i blocchi trasmessi sulla WAN
- Dedup ratio **WAN target** (OCI-side) — riduce solo lo spazio occupato su OCI, non le request
- Range request/TB (min/max, default 700K–900K) — include già PUT/GET/LIST/DELETE
- Unità B91627 contrattualizzate e costo per unità

---

## Architettura

- **Single-page HTML** — nessun backend, nessun framework, nessun dato inviato a server esterni
- Tutti i calcoli avvengono **localmente nel browser**
- Design system Nexus (light/dark mode, token CSS, fluid type scale)
- Deploy su **Vercel** (static hosting)

---

## Logica di Calcolo — Veeam/OCI

**Fattore comune — Request per TB (dipende dal blocco):**
```
Request/TB_adj = Request/TB_base × (1024 KB / Dimensione blocco)
```
> Blocchi più piccoli ⇒ più oggetti/TB ⇒ più request. Il range Request/TB include già tutti i tipi di chiamata (PUT/GET/LIST/DELETE).

**Stima realistica (da profilo backup) — request effettivamente fatturate:**
```
Incrementale/giorno   = Full × Change rate%
Dati scritti su OCI/mese (a regime) = Incrementale/giorno × Giorni attivi + Full×Full_periodici
Request/mese a regime = Dati scritti/mese (TB) × Request/TB_adj
Request full iniziale = Full (TB) × Request/TB_adj   (una tantum, nel mese di seeding)
```

**Stima da banda (tetto di throughput) — massimo che il link può reggere:**
```
Banda netta = Banda nominale × Utilizzo% × (1 − Overhead%)
Dati WAN/giorno = Banda netta × Ore attive × 3600
Request/mese = Dati WAN mese (TB) × Request/TB_adj
```

> ⚠️ Per **dimensionare le unità B91627** usa la **stima realistica**: la stima da banda è solo una verifica di fattibilità dell'uplink e sovrastima se il link non è saturo.

> La dedup **local (source-side)** riduce i blocchi trasmessi sulla WAN e quindi le request.  
> La dedup **WAN target (OCI-side)** riduce solo lo spazio di storage su OCI, non influenza le request.

---

## Sviluppo locale

Essendo un file HTML statico, basta aprirlo direttamente nel browser:

```bash
git clone https://github.com/enricobrunazzo/data-transfer-estimator.git
cd data-transfer-estimator
open index.html
```
