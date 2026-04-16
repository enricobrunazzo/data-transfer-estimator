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

**Input:**
- Banda upload nominale e utilizzo effettivo (%)
- Overhead protocollo WAN
- Ore attive/giorno e giorni attivi/mese
- Dataset sorgente totale (opzionale, per stimare i tempi di completamento)
- Dimensione blocco Veeam (256 KB / 512 KB / 1024 KB)
- Dedup ratio **local** (source-side) — riduce i blocchi trasmessi sulla WAN
- Dedup ratio **WAN target** (OCI-side) — riduce solo lo spazio occupato su OCI, non le request
- Range request/TB WAN (min/max, default 700K–900K)
- Unità B91627 contrattualizzate e costo per unità

**Output:**
- Dati WAN trasferiti per giorno / settimana / mese
- Dati sorgente coperti al mese (post local dedup)
- Request stimate per giorno / settimana / mese (range min–max)
- Confronto con il limite contrattuale OCI e margine
- Unità B91627 necessarie e costo stimato mensile
- Dettaglio parametri di calcolo
- Stima completamento dataset totale (se inserito)

---

## Architettura

- **Single-page HTML** — nessun backend, nessun framework, nessun dato inviato a server esterni
- Tutti i calcoli avvengono **localmente nel browser**
- Design system Nexus (light/dark mode, token CSS, fluid type scale)
- Deploy su **Vercel** (static hosting)

---

## Logica di Calcolo — Veeam/OCI

```
Banda netta = Banda nominale × Utilizzo% × (1 − Overhead%)
Dati WAN/giorno = Banda netta × Ore attive × 3600
Request/mese = (Dati WAN mese in TB) × Request/TB_adj
Request/TB_adj = Request/TB_base × (1024 KB / Dimensione blocco)
```

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
