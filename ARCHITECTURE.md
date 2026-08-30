# AiWorld Core Runtime v0.1 — unificazione dei moduli attorno ad AiAgents

Stato: **PROPOSED**. Nessun modulo qui dentro è ratificato: la ratifica è del gate umano.

## 1. Cosa contiene questo pacchetto

Un runtime agentico unificato, vanilla JS, zero dipendenze, browser + Node, in cui
ogni modulo ha **una sola responsabilità** e non ne conosce nessun altro: le dipendenze
sono iniettate in `index.js`.

| Modulo | File | Responsabilità unica | Non fa |
|---|---|---|---|
| AiLanguage | `ailanguage.js` | significato: AL-1 Task, AL-2 Handoff, AL-3 Evidence, AL-4 Failure, AL-5 Capability | non trasporta, non esegue |
| AiBridge | `aibridge.js` | trasporto: envelope `AiBridge/1`, handshake, capability discovery | non esegue task, non chiama provider |
| AiAgents | `aiagents.js` | esecuzione: route → plan → execute → verify → retry → human gate | non trasporta, non ricorda, non verifica |
| AiFold | `aifold.js` | stato: AgentRun append-only, checkpoint, `resume`, genealogia | non è un knowledge graph |
| AiEval | `aieval.js` | verifica deterministica, conteggi generati | nessun LLM-giudice |
| AiExecutors | `executors.js` | registry esecutori con capability verificate | non sceglie da solo |
| Constitutional | `constitutional.js` | guardrail P1–P6 pre/post-flight | non è Constitutional AI di Anthropic |

Il rischio che l'analisi precedente segnalava — «mettere logica di loop, memoria e
trasporto dentro AiAgents» — è evitato per costruzione: `aiagents.js` non contiene
una riga di trasporto, persistenza o validazione di merito.

## 2. Il loop

```
RECEIVE ─▶ [Constitutional pre-flight] ─▶ ROUTE ─▶ PLAN ─▶ EXECUTE ─▶ VERIFY
                    │ blocked                                            │
                    ▼                                                    │ fail
              HUMAN GATE ◀──────────── MAX_ATTEMPTS ◀──── RETRY (escludi esecutore)
                    │ approve/reject/retry
                    ▼
             AiFold: DONE / FAILED / AWAITING_HUMAN  (sempre persistito)
```

Invarianti applicati nel codice, non nel prompt:

1. **Nessun GREEN senza evidenza machine-readable.** `AiEval.run()` genera scenari,
   asserzioni, passate e fallite; nessun totale è scrivibile a mano.
2. **Nessuna azione irreversibile senza Human Gate.** `git_push`, `deploy`, `delete`,
   `send_email`, `publish`, `payment` sono bloccate da P4 se `human_gate !== true`.
3. **Nessuna auto-correzione autonoma.** Il retry cambia esecutore, mai il contratto
   del task; oltre `max_attempts` il run si ferma in `AWAITING_HUMAN` e resta riprendibile.
4. **Fallback provider reale.** Il caso 429 di Gemini è coperto: l'esecutore che fallisce
   viene escluso dal routing del tentativo successivo (cascata vera, non `if/else-if`).

## 3. Cosa risolve dei problemi aperti

| Problema | Dove è risolto |
|---|---|
| P0-A ConsentManager ↔ DeviceVault v1.2 | `p0/consent-manager.js` v1.1 — `get`/`put`, `await` sulla persistenza, JSON corrotto tollerato, `telemetry` separato da `ai` |
| P0-B AiBridge chiama metodi inesistenti | `aibridge.js` v2.1 — il bridge non chiama più un provider: instrada un envelope verso handler registrati. Un test verifica l'assenza di `AiAgents.cerca/chiedi` |
| P0-C discrepanza 11 vs 17 vs 18 | `p0/report-v2.mjs` — conteggio rigenerato dalla tabella, blocco idempotente, `--check` per la CI. Sul report d'esempio produce **7 scenari / 17 asserzioni** |
| AiEvents: manca il fallback dopo 429 | esclusione dell'esecutore fallito + `retryOrGate` |
| Caso Arena: capacità dichiarate ≠ reali | `executors.js` — `verify()` rifiuta una promozione senza evidenza; `match()` ordina per capability verificate |
| Caso Cologno: vincoli persi nella sintesi | AL-1 porta `constraints` fino a `AiEval` (C-14 data, C-15 geografia) |
| Agente che dichiara "20/20" a caso | il report è generato dal runner, non dall'esecutore |

## 4. Verifica eseguita

`node test/run-tests.js` → **38/38 asserzioni, GREEN**. `node --check` passa su tutti i file.
Coperti: contratti AL, envelope, handshake, append-only, checkpoint, conteggi generati,
routing per capability, i sei principi, run verde, nessun esecutore, fallback 429,
gate su verifica fallita, `resume(approve)`, `resume(reject)`, blocco P4, handoff, cancel,
ingresso task dal bridge, ConsentManager.

Il numero 38 esce dal runner. Se aggiungi un test, il numero cambia da solo.

## 5. Cosa NON è ratificato qui

Non ho potuto verificare né calcolare SHA su:

- `AiBridge v2` reale (il file del KB) — quello nel pacchetto è una **riscrittura**, non il tuo file corretto
- `AiGateway v1.1.1` (`cloudflare-worker-proxy-v2`)
- `DeviceVault v1.2` (`device-vault-impl(5).js`)
- `AiJob-0.28.0.html`
- il JS reale di `AiEvents 2.9.25`

Nessuno di questi era disponibile in sessione. Le ratifiche restano **bloccate** finché
i file non vengono caricati: `MODULE_REGISTRY.json` contiene solo gli SHA dei file di
questo pacchetto, dichiarati come tali.

## 6. Sequenza dopo questo pacchetto

```
P0  canonical reali + SHA          ← bloccato: servono i file
P1  AiAgents Runtime               ← FATTO qui (v0.1, da ratificare)
P2  AiBridge transport             ← FATTO qui (v2.1, da riconciliare col file reale)
P3  AiLanguage contracts           ← FATTO qui (AL-1..AL-5)
P4  AiFold run state               ← FATTO qui (MVP)
P5  Executor/browser adapter       ← registry pronto, mancano gli adapter concreti
P6  AiEval benchmark               ← base pronta, mancano i validatori di dominio
P7  AiEvo Observer                 ← solo dopo che esistono run reali in AiFold
```

Il collo di bottiglia non è più il design: sono i file mancanti.

## 7. Primo innesto consigliato su AiEvents

Strangler-fig, senza toccare la baseline:

```js
const w = AiWorld.boot({ persist: true });

w.agents.registerExecutor({
  id: 'EXT-GEMINI-01', type: 'api_provider',
  capabilities_claimed: ['search_events', 'chat'],
  cost_hint: 0,
  run: (task) => AiAccess.ask(task.intent, task.context)   // facade esistente
});

w.agents.registerExecutor({
  id: 'EXT-DASHSCOPE-01', type: 'api_provider',
  capabilities_claimed: ['search_events', 'chat'],
  cost_hint: 3,
  run: (task) => callQwenAI(task.intent, task.context)     // adapter legacy
});

const res = await w.agents.run({
  intent: 'search_events',
  constraints: { capabilities: ['search_events'], date: '2026-08-23', geography: ['Idroscalo'] },
  success_condition: ['C-14', 'C-15', 'C-04b']
});
```

`callQwenAI` e `AiAccess` restano intatti: diventano il corpo di due esecutori.
Nessun chiamante esistente viene modificato.
