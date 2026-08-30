# MD-07 riconciliato — AiEvents passa a ConsentManager v1.1 su DeviceVault

## Cosa c'era

| | prima (2.9.32) | dopo (2.9.33) |
|---|---|---|
| classe | oggetto in-file v1.0 | stessa classe v1.1 del kit P0-A |
| storage | `localStorage['aitrip_consents']` **in chiaro** | `aitrip_vaultstore_v1`, AES-GCM device-bound |
| chiave | nessuna | DeviceVault, non estraibile, in IndexedDB |
| tipi | ai, telemetry, marketing, **analytics** (mai usato) | ai, telemetry, marketing + `decided` |
| decisione presa | presenza della chiave localStorage | campo `decided` nel record cifrato |

## Come, e i due vincoli che hanno guidato l'adattamento

**1. Le chiamate esistenti sono sincrone.** `hasConsent()` viene invocata da
onclick, da render e dalla telemetria: 11 siti, nessuno dei quali puo' attendere.
La v1.1 del kit ha gia' la forma giusta — cache in memoria sincrona, persistenza
asincrona — quindi non ho cambiato la semantica: ho reso `init()` awaited al boot
(`await ConsentManager.init()`) e lasciato `hasConsent`/`isDecided` sincroni sulla
cache. `setConsent` aggiorna la cache **prima** di restituire la promise, cosi'
il render immediatamente successivo nell'onclick vede gia' il valore nuovo.

**2. DeviceVault espone `encrypt/decrypt`, non `get/put`.**
Aggiunto `DeviceVaultStore`, un adattatore che soddisfa il contratto v1.2
richiesto dal costruttore (`get`/`put`) e custodisce le buste in un unico
contenitore localStorage. Il costruttore della v1.1 rifiuta uno storage che non
espone `get/put`: quel vincolo e' rimasto attivo, non l'ho allentato.

## Migrazione, e perche' non e' silenziosa

Al primo `init()` senza record nel vault, se esiste `aitrip_consents` in chiaro
i consensi vengono importati, riscritti cifrati e **la chiave in chiaro viene
rimossa**. `analytics` viene scartato: non era usato da nessuna parte.
`getConsentInfo()` espone `migrated: true` per quella sessione.

Se DeviceVault non e' disponibile (niente `crypto.subtle` o IndexedDB, contesto
non sicuro), lo store degrada a `{plain: ...}` **dichiarandolo**:
`getConsentInfo().storageMode === 'plain'`. Un degrado di privacy silenzioso
sarebbe peggio del problema che stiamo chiudendo.

## Verifica

```
node p6/test-consent.js AiEvents-2.9.33-consent.html
```

| caso | cosa prova |
|---|---|
| C1 | dopo una scelta, `aitrip_consents` non esiste e la busta nel vault non contiene testo leggibile (`{v:1,iv,ct}`) |
| C2 | il consenso sopravvive al reload, riletto dal vault |
| C3 | migrazione dal v1.0: consensi trasferiti, chiave in chiaro rimossa |
| C4 | la migrazione non si ripete e non sovrascrive scelte successive |
| C5 | `revokeAll` azzera tutto ma resta `decided` |
| C6 | i chiamanti sincroni non cambiano: telemetria ancora gated |
| C7 | un tipo sconosciuto (`analytics`) viene rifiutato |
| C8 | il banner non riappare dopo una decisione |
| C9 | app e runtime AiWorld integri, zero errori di pagina |

**9/9 GREEN**, sigillo `267d3b0b`, evidenza `p6/evidence-md07-consent.json`.

Regressione sul file nuovo: P0 CONFORME, swipe 9/9, integrazione 9/9,
runtime 10/10, multimodulo 7/7, P6 browser 6/6.

**File:** `AiEvents-2.9.33-consent.html` — 458247 byte
**SHA-256:** `a9eb870069ba6f2e268cfcfb0b8b92ed20eb57617b5dc908fa11b7b797658dc6`

## Restano aperti (gate umano)

1. **`aiConfigured()`** conta ancora `aitrip2_agent_keys_enc` come prova di AI
   configurata: blob non piu' apribile dopo 2.9.32. Non toccato.
2. **`SecureStorage`** non ha piu' chiamanti di produzione (solo l'autotest
   interno). Candidato alla rimozione, ma e' codice crittografico: lo tolgo
   solo su tua parola.
