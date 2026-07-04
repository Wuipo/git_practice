# HANDOVER — Progetto Copy-Trading Analytics (Solana + Polymarket)

> **Come usare questo file**: incollalo (o committalo come `CLAUDE.md` / `HANDOVER.md`) all'inizio di una nuova sessione Claude Code. Contiene TUTTO il contesto, le decisioni, gli ID delle query Dune, i bug trovati/risolti e i prossimi passi. Serve a continuare il lavoro come se la conversazione non si fosse mai interrotta.

---

## 0. TL;DR / Stato attuale

Sto costruendo su **Dune Analytics** (accesso via MCP) due sistemi di analisi per **copy-trading**:

1. **Solana memecoin (pump.fun)** — strategia "MIRROR": copio il *first buy* di un target e replico lo *schedule di vendita* (le sue frazioni %) sulla mia posizione. Deliverable = ranking di wallet da copiare.
2. **Polymarket (prediction markets, Polygon)** — strategia "copy-once": un bot copia UNA volta il primo buy del target, a size fissa, poco dopo. Deliverable = ranking di wallet da copiare (voglio poterne copiare ~50 per diversificare).

**Obiettivo di profittabilità**: sempre e solo **"quanto guadagnerei IO copiando"**, mai quanto guadagna il trader. Metriche in ROI% per operazione dal MIO prezzo d'ingresso, al netto di fee/slippage.

**Query finali live su Dune (team_id 34063):**
| Scopo | query_id | Nome |
|---|---|---|
| Solana MIRROR (corretta) | **7631308** | SPW20260524 - MIRROR v2 |
| Solana MIRROR ranking (fee-adjusted) | **7631396** | SPW20260524 - MIRROR v2 RANKING |
| Solana originale (base, NON toccare) | 7568154 | SPW20260524 |
| **Polymarket copy-once (FINALE v5)** | **7633123** | Polymarket - Copy-once ranking v5 |

**Verdetto finale Solana vs Polymarket**: dopo modelli realistici, **Polymarket vince ~2×**. Solana mirror ≈ +30%/mese su bankroll; Polymarket portfolio 50-wallet ≈ **+107%/mese Wilson-pessimistico** / +175% mean su $1000.

---

## 1. Ambiente & vincoli operativi (LEGGERE PRIMA DI ESEGUIRE QUERY)

- **Dune via MCP**: la connessione può cadere e va **riautorizzata** (da `/mcp` in sessione interattiva o dalle impostazioni connector claude.ai). Senza auth non posso eseguire/leggere query.
- **Data "corrente" dell'ambiente**: **2026-07-04**. I dati Dune arrivano fino a maggio 2026. Le finestre di analisi usano **Aprile+Maggio 2026** (`block_month IN (DATE '2026-04-01', DATE '2026-05-01')`).
- **Piano free, i crediti contano**: query costano ~20–50 crediti. Test su dataset piccoli prima di scalare.
- **Timeout**: `getExecutionResults` va in timeout a 60s → fare polling ripetuto. Finestre di **3 mesi vanno in timeout** sulle tabelle Polymarket grosse: tenersi a **2 mesi**.
- **Limiti tool Dune MCP**:
  - `updateDuneQuery` accetta max **10.000 caratteri** di SQL. Se la query è più lunga → NON si può aggiornare in place; usare `createDuneQuery` (limite 500k) che però crea un NUOVO id.
  - Risultati grossi (>~25k token) vengono salvati su file dal tool: leggerli con `jq`/`python`, non a occhio.
- **MCP tools chiave**: `searchTables`, `getTableSize`, `getDuneQuery`, `createDuneQuery`, `updateDuneQuery`, `executeQueryById`, `getExecutionResults`. (Il prefisso server MCP cambia tra sessioni.)

---

## 2. PARTE A — Solana MIRROR

### 2.1 Contesto strategia
- Query base **7568154 (SPW20260524)**: modella "compro 0.3 SOL una volta al primo buy del target, vendo il 100% al primo sell". Produce colonne `MY_F_*` (profitto % e SOL con prezzo d'ingresso/uscita **pessimistico** = prossima trade dopo quella del target).
- Variante MIRROR voluta: invece di vendere tutto al primo sell, **vendo la stessa % che vende il target ad ogni sua sell** (come TradeWiz "sell same %"). Il buy resta invariato (una sola volta).

### 2.2 Modello matematico MIRROR (confermato dall'utente con esempio)
Per ogni `(SWAPPER, SYMBOL_MINT)`:
- Entro una volta: `Q` token investendo 0.3 SOL a `MY_F_BUY_PRICE`.
- Per ogni sell `i` del target (ordine per slot): frazione liquidata `f_i = sell_tok_i / saldo_target_PRIMA_della_sell_i`. **Il denominatore include i re-buy del target** (confermato: es. compra 1000, vende 500 → 50%; poi ricompra fino a 3000, vende 1000 → 33.3%).
- Io vendo `f_i` del MIO saldo corrente a `MIRROR_SELL_PRICE_i`. Saldo: `bal *= (1 - f_i)`. Ricavo: `proc += bal * f_i * price_i`.
- Residuo non venduto = valutato **0** (pessimistico).
- `MIRROR_PROCEEDS_PER_TOKEN = Σ_i [Π_{j<i}(1-f_j)] * f_i * price_i`
- `MIRROR_SOL_PROFIT_PERC = MIRROR_PROCEEDS_PER_TOKEN / MY_F_BUY_PRICE * 100 - 100`
- `MIRROR_SOL_PROFIT   = F_BUY_SOL_AMT * (MIRROR_PROCEEDS_PER_TOKEN / MY_F_BUY_PRICE - 1)`
- Implementato in SQL con `reduce(array_agg(ROW(f, price) ORDER BY slot), ROW(1.0, 0.0), (s,x)->..., s->s.proc)`.

### 2.3 Decisioni prese
1. Mirror **sequenziale fedele** (ricostruisce il saldo del target riga per riga, non VWAP).
2. Prezzo di vendita mirror = proxy "prossima trade reale entro +150 slot" (stesso pessimismo di MY_F).
3. Finestra mirror **+150 slot** (leva costo/pessimismo, confermata).
4. **Filtro finale su MIRROR** (`AVG_MIRROR_SOL_PROFIT_PERC_NOMAX >= 10`), non su MY_F, perché mi interessa solo il mirror.

### 2.4 BUG trovato e risolto
Nella CTE `TRADES_AGG_2`, `MIRROR_SOL_PROFIT` aveva fallback **`0`** per operazioni dove il target non vende mai (bag morto) → incoerente: `MIRROR_SOL_PROFIT_PERC` diceva `-100%` ma in SOL contava 0, gonfiando `SUM_MIRROR_SOL_PROFIT`.
**Fix**: fallback → `-W.F_BUY_SOL_AMT`. Applicato nella copia corretta 7631308.
(Non potevo aggiornare in place per il limite 10k char → creato nuovo id 7631308.)

### 2.5 Calibrazione fee (Solana)
Dato empirico utente: su TradeWiz "+10% lordo = pari netto" → **fee drag reale ≈ 10 pp** sul rendimento %. (Il modello iniziale 2.3pp — gmgn 1%×2 + Jito — era troppo ottimista; usare 10pp.)

### 2.6 Risultati Solana
- 75 trader superano il filtro MIRROR.
- Top GREEN pick: **`DPrfLfreJzVMuwM5Uw7Kbr56eGMfRx1J95sY5V9HUdhZ`** — 26 markets/2mo, ROI net ~22.6%, WR 61%, MED +9%, EV ≈ +29%/mese su bankroll.
- Query ranking (Wilson lower bound WR + composite score): **7631396**.

---

## 3. PARTE B — Polymarket (il grosso del lavoro)

### 3.1 Tabelle Dune usate (Polygon)
- **`polymarket_polygon.market_trades`** (spell, ~142 GB) — tabella principale. `maker`=trader retail, `taker`=contract exchange. `maker_side`/`taker_side` = BUY/SELL rispetto all'outcome token. `amount`=USD. `shares`=token. `price`. `is_taker_side` (bool). `condition_id` (varbinary). `asset_id` (uint256, il token_id). Partizione `block_month`.
- **`polymarket_polygon.market_prices_hourly`** — prezzo orario per token. **USATA COME GROUND TRUTH di risoluzione**: dopo settlement il prezzo snap a ~$1 (winner) o ~$0 (loser).
- `polymarket_polygon.market_details` — metadata. **ATTENZIONE**: `outcome` testuale NON matcha `token_outcome` per mercati con label custom (Up/Down, Over/Under, Odd/Even, squadre). `condition_id` qui è varchar `'0x...'`. `enable_order_book` è varchar 'True'/'False' e va a 'False' dopo la risoluzione.
- `dune.dune.dataset_polymarket_markets` — ha `token_1_winner`/`token_2_winner` ma è **STALE (fermo a Gennaio 2026)**: NON copre Apr-May → inutilizzabile. token_id lì ha prefisso `'str:'`.

### 3.2 Gotchas di join risolti
- `condition_id`: varbinary in trades, varchar in details → normalizzare con `'0x' || lower(to_hex(condition_id))`.
- Contract exchange da **escludere come maker** (non sono retail): `0x4bfb41d5b3570defd03c39a9a4d8de6bd8b8982e`, `0xc5d5b407255d499dc8da9aab85c4d2e1cf95e80a`, `0xe111180000d2663c0091e4f400237545b87b996b` (v2 CTF), `0xe2222d279d744050d28e00520010520000310f59` (v2 NegRisk).

### 3.3 Modello copy-once (quello che conta per ME)
- Il bot copia UNA volta il primo BUY significativo del target (`amount >= $20`, per saltare test-buy), poco dopo, a **size fissa**.
- Il MIO prezzo d'ingresso: `my_entry = guru_first_buy_price + max($0.01, 1.5% * price)` (slippage realistico; il floor +1¢ riflette il tick di Polymarket ed è CRUCIALE: senza, i "lottery bot" che comprano a $0.01-0.02 sembravano avere edge falso).
- Payout binario: `$1` se il token vince, `$0` se perde (da last hourly price: `>=0.95 → 1`, `<=0.05 → 0`, else escluso/ambiguo).
- **MIO ROI per market** = `(payout - my_entry) / my_entry * 100`. Aggregato in media/mediana/percentili = mio EV per bet. **NON dipende dal profitto del guru.**
- Fee: **-2pp** (1% bot + 1% taker Polymarket V2). Slippage già dentro `my_entry`.
- Assunzione: **hold-to-resolution** (compro e tengo fino al settlement). Se in futuro volessi copiare anche le USCITE del guru → serve query diversa.

### 3.4 Evoluzione query (bug per bug)
- **v1 (7631677)**: primo tentativo. **BUG CRITICO**: payout via match testuale `token_outcome = outcome` falliva sul **79% dei mercati** non-neg-risk (label custom). → risultati compromessi.
- **v2**: provato `dataset_polymarket_markets` (stale) e ordinamento token_id (non funziona). Scartati.
- **v3 (7633123)**: **FIX payout** usando last hourly price snap a 0/1 (`market_prices_hourly`). Copre TUTTI i tipi di mercato. Verificato: ~1.0M winner-token + 986k loser-token (bilanciato 1:1), 226k ambiguous esclusi.
- **v4**: aggiunto slippage realistico, filtro first-buy≥$20, score Wilson-weighted, colonne EV $/mese.
- **v5 (FINALE, sempre 7633123)**: bug fix finali (vedi sotto). **8/8 sanity check passati.**

### 3.5 BUG risolti in v5 (rispetto v4)
1. `arbitrary(price ORDER BY hour DESC)` (sintassi non standard Trino, rischio valore random) → **`max_by(price, hour)`**.
2. `MARKET_LIQUIDITY` contava il volume **2×** (ogni match Polymarket emette 2 leg) → aggiunto **`WHERE is_taker_side = true`**. La soglia $50k ora è volume reale (prima era ~$25k mascherato). Questo ha correttamente **eliminato wallet come `0xfe4c06ca...`** che sembravano buoni solo per la liquidità gonfiata.
3. Wilson EV dipendeva da `avg_entry` (approssimazione grezza) → formula pulita: **`ev_roi_pessim = (WR_low95/WR) * (avg_roi_net + 100) - 100`**.

### 3.6 Sanity check v5 (tutti PASS)
`avg_roi in [-102,5000]` · `roi_wilson <= avg_roi` · `n_markets >= 15` · `WR_low95 <= WR` · `ev_wilson <= ev_mean` · `avg_entry in [0.01,0.99]` · `bet_size in [5,100]` · `GREEN ⇒ MED >= 0` (43/43).

### 3.7 Filtri anti-scam / qualità applicati (v5)
- Market volume reale (`is_taker_side`) **>= $50.000** (liquidità copy-tradable).
- `total_guru_volume >= $1000` (anti-noise: escludi hobbyisti da $14/bet).
- `trades_per_market <= 50` (anti market-maker/scalper).
- `win_rate <= 90%` (anti arbitrageur "free money").
- `price BETWEEN 0.01 AND 0.99` (escludi mercati già decisi).
- Escludi contract exchange come maker.
- **Tier**: GREEN (n>=30, WR_low95>=45, MED>=0, AVG>=15%, t/m<=20, vol>=$5k) · YELLOW (n>=20, WR_low95>=35, AVG>=10%, vol>=$1k) · ORANGE (resto).

### 3.8 Risultati v5
- **150 wallet** in output, **43 GREEN**, 150/150 con EV_Wilson>0, 104/150 con MED>=0. Costo esecuzione ~19 crediti.
- **Top pick robusti (EV Wilson pessimistico alto):**
  - `0x0006af12cd4dacc450836a0e1ec6ce47365d8c63` — #1 GREEN, EV_Wilson ~$1049/mo, WR_low 60.7%, MED +49%, ~100 bet/mo.
  - `0xeb6789ca6b1425ff908a69a2a5469c38532cd696` — YELLOW, EV_Wilson ~$2065/mo, WR_low 64.7%, MED +85%.
  - `0x78af83a1c6bc5b56c2a8efe6a365b18789e2c974` — YELLOW, EV_Wilson ~$2010/mo, MED +131%.
  - `0x98f5d88798331947dc2a0f9087308a17c41f754f` — GREEN #13.
  - `0x9bbd88140ccba06100da00476257d9cffce56e72` — GREEN #21, MED +107%.

### 3.9 Portfolio diversificato 50-wallet (richiesta dell'utente)
- Fattibile: 85 candidati GREEN+YELLOW con EV_Wilson>0 e t/m<=30. Presi i top 50 per EV_Wilson.
- Bankroll $1000 equipartito ($20/wallet), bet size dinamico = `$20 / bet_al_mese`, cap min $1 / max $50.
- **EV portfolio: ~$1748/mese (mean, +175%) / ~$1071/mese (Wilson pessimistico, +107%)**. ~1400 bet/mese (~47/giorno).
- Diversificare è statisticamente MEGLIO (variance ↓ ~17×, Sharpe ~4× vs 3-wallet).
- **Vincoli operativi reali**: min bet Polymarket $1 → con bankroll $1000 alcuni wallet ad alta frequenza non sono copiabili; consigliato **bankroll $2000-3000**. Gas Polygon trascurabile. Serve infra bot per monitorare 50 wallet (websocket + rate limit Gamma API).

---

## 4. Fee/slippage — ricerca web (Luglio 2026)
- **gmgn / bot Solana**: ~1% per trade (buy E sell), + 0.1-0.3% priority/Jito. Round-trip nominale 3.2-6.6% con slippage. MA dato empirico utente TradeWiz → drag reale **~10pp** (usare quello per Solana).
- **Polymarket V2**: taker ~1%, fee per-trade = `C·r·p·(1-p)`. Modello usato: **-2pp** copy (1% bot + 1% taker); slippage a parte nel prezzo d'ingresso.

---

## 5. Caveat onesti (validi per entrambi)
- **Campione = solo 2 mesi (Apr-May 2026)**: l'edge potrebbe non persistere → ri-eseguire mensilmente e ricalibrare.
- **Survivor bias**: sono i top di quel periodo.
- **Wash trading / sybil**: non ho detection di wallet diversi controllati dallo stesso operatore (mitigazione futura: raggruppare per `tx_signer`/first-funder).
- **Slippage variabile**: in mercati thin il vero slip può essere 5-10%, non 1.5%.
- **Coverage `market_prices_hourly`** ~99%, non 100% → possibile leggera sotto-conta di `n_markets` per alcuni wallet.
- Top wallet Polymarket con WR alta potrebbero avere **edge informativo** (insider/sondaggi) difficile da replicare in tempo.

---

## 6. Prossimi passi possibili (da decidere con l'utente)
1. Ri-eseguire v5 (7633123) su dati più recenti quando disponibili; ricalibrare tier.
2. Aggiungere **detection sybil/wash** (raggruppo wallet per first-funder / tx_signer).
3. Modellare slippage **variabile per market depth** invece di flat 1.5%.
4. Estendere il modello per copiare anche le **uscite** del guru (non solo hold-to-resolution).
5. Collegare i wallet ai profili/username Polymarket (via Gamma API `data-api.polymarket.com`).
6. Decidere size/bankroll reale e generare la lista operativa finale dei 50 wallet + bet size per il bot.
7. (Solana) eventualmente rendere la finestra mirror configurabile e testare +50/+150/+1000 slot sul trade-off costo/pessimismo.

---

## 7. Snippet SQL chiave da riusare

**Ground truth risoluzione Polymarket (payout binario):**
```sql
WITH RESOLVED_TOKENS AS (
    SELECT token_id,
        CASE WHEN last_price >= 0.95 THEN 1.0
             WHEN last_price <= 0.05 THEN 0.0 END AS payout_per_share
    FROM (
        SELECT token_id, max_by(price, hour) AS last_price
        FROM polymarket_polygon.market_prices_hourly
        WHERE block_month IN (DATE '2026-04-01', DATE '2026-05-01')
        GROUP BY token_id
    )
    WHERE last_price >= 0.95 OR last_price <= 0.05
)
```

**Slippage realistico d'ingresso:**
```sql
first_price + GREATEST(0.01, 0.015 * first_price) AS my_entry
```

**Wilson lower bound 95% sul win rate (WR in %):**
```sql
100.0 * GREATEST(0.0,
  ((wr/100.0) + (1.96*1.96)/(2.0*n)
   - 1.96 * sqrt(((wr/100.0)*(1.0-wr/100.0))/n + (1.96*1.96)/(4.0*n*n)))
  / (1.0 + (1.96*1.96)/n)) AS win_rate_lower95
```

**Volume liquidità one-sided corretto (no 2×):**
```sql
SUM(amount) FILTER (WHERE is_taker_side = true)  -- oppure WHERE is_taker_side = true in una CTE dedicata
```

---

## 8. Nota importante per la nuova sessione
Questa è ripartenza da un **repo/account git NUOVO**: il connettore **Dune MCP va (ri)autorizzato** prima di poter eseguire query. Le query esistenti (7633123, 7631308, 7631396, 7631308) vivono su Dune sotto il team 34063 dell'account originale — se il nuovo account Dune è diverso, potrebbe servire **ricrearle** (il SQL completo si recupera con `getDuneQuery` finché ho accesso all'account vecchio, altrimenti va rigenerato dai modelli descritti sopra). I CSV dei risultati (portfolio 50, dataset 150 wallet) sono già stati consegnati all'utente come file scaricabili.
