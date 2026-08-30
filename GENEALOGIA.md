# AiEvents — genealogia completa 2.9.25 → 2.9.33

Nessun gap. I file 2.9.26–2.9.31 non erano "persi": erano nel mio workspace e
NON nel KB di Qwen perche' il pacchetto precedente escludeva i tre originali e
Roberto non ha ricaricato i derivati. Sotto la catena completa, verificabile.

| rev | origine | byte | SHA-256 | cosa cambia |
|---|---|---|---|---|
| 2.9.25 | caricata da Roberto | 334898 | `bb290dc1a8826e70580484a89618e24e42c85da5484e2477774e26ada275408b` | base. B1 **assente** (`window.ConsentManager` presente, riga 3825) |
| 2.9.26 | caricata da Roberto | 336473 | `67cc5ce4d31c93b245e3c449170bb04cd698efb05aecb1dc4780c558cdf2a409` | md5 del blocco provider identico a .25 |
| 2.9.27 | caricata da Roberto | 347807 | `2933cd54ce5dc041e3876edb958beb124a01aeeb5324bfa2e3d2d333e91f4e3a` | ultima originale. Nessun fallback provider (F1), nessun contratto risposta (F2) |
| 2.9.28 | derivata da .27 | 353955 | `109644e5100b9240d85302e42b272bcfeeebc68490ca2c82a3f2a3fbcfe2605a` | **AiAccess integrato**: cascata proxy→gemini→dashscope, contratto v1.0, errori classificati, cache con system prompt |
| 2.9.29 | derivata da .28 | 355464 | `d68cd865a860749bbd9ccd55f5dd4b0eb939cb54cdd1a9983b251a3c982554be` | **bug swipe chiuso**: `pointermove` rispetta `blocked`, multi-touch, toast diagnostico rimosso |
| 2.9.30 | derivata da .29 | 458726 | `ec4887e5b727f79f062dc81eb4550f7f96814f23ff6787d60c9de8fd8c26e72d` | **runtime AiWorld incorporato** (AiLanguage/Bridge/Fold/Eval/Executors/Agents), Claude registrato UNVERIFIED |
| 2.9.31 | derivata da .30 | 458487 | `7531faf6adfd1551ee21f5bebaf9c587e1634372ed640fdb1fe04786f398fbfc` | rimosse le **prime** dichiarazioni duplicate di `openKeyModal`/`confirmKeyModal` |
| 2.9.32 | derivata da .31 | 454405 | `2a8f0419d08e7feb44d98902ccf61027091077df6608d23b623bfe2dd65977c4` | rimosso il **modale key orfano** (markup + le due funzioni superstiti + `hasBlob`) |
| 2.9.33 | derivata da .32 | 458247 | `a9eb870069ba6f2e268cfcfb0b8b92ed20eb57617b5dc908fa11b7b797658dc6` | **MD-07 riconciliato**: ConsentManager v1.1 su DeviceVault (AES-GCM device-bound), migrazione dal formato in chiaro, chiave `aitrip_consents` rimossa |

Verifica: `sha256sum AiEvents*.html` nella cartella `p1/real/aievents/`.

## Nota sull'estrazione testuale
Uno SHA calcolato su un'estrazione UI di un HTML **non puo'** coincidere con lo
SHA del file. Se nel KB c'e' solo il testo estratto, lo SHA non e' verificabile
li': va confrontato sul file binario originale, non sulla sua trascrizione.
