# CLAUDE.md — Ajax U11 Holdstyr

Kontekst og regler for Claude Code når der arbejdes på dette projekt.

---

## Projektoverblik

Holdstyringsapp til Ajax U11 Piger, sæson 2025/26. Bygget som én enkelt HTML-fil uden backend-framework. Data synkroniseres via Cloudflare KV mellem trænere.

**Hosting:** GitHub Pages — `https://claustakman.github.io/ajax-holdstyr/`  
**Deploy:** Upload `index.html` via GitHub web-editor (Cmd+A → paste → Commit)

---

## Filer

| Fil | Beskrivelse |
|-----|-------------|
| `index.html` | Hele appen — HTML, CSS og JavaScript i én fil |
| `kv-worker.js` | Cloudflare Worker til data sync (KV database) |
| `holdsport-worker.js` | Cloudflare Worker til Holdsport API proxy |

---

## Arkitektur

```
Browser (index.html)
    │
    ├── Cloudflare KV Worker  (/db GET + POST)
    │       └── HOLDSTYR_KV namespace
    │
    └── Holdsport Worker  (/teams, /activities, /users)
            └── Holdsport API (Basic auth)
```

### JavaScript-moduler (bagt ind i index.html)

| Modul | Ansvar |
|-------|--------|
| `app_v6.js` | UI, kamplogik, spillere, statistik, opsætning |
| `drive_sync.js` | KV sync — load/save via Cloudflare Worker |
| `holdsport_sync.js` | Holdsport import — kampe og tilmeldte |

---

## Data model

Data gemmes som ét JSON-objekt i KV under nøglen `appdata`:

```json
{
  "players": [{ "nickname", "fullName", "birthYear", "active", "defaultNumber" }],
  "coaches": [{ "name", "role" }],
  "games": [{
    "id", "date", "time",
    "team1": { "opponent", "players": [{ "slot", "name", "factor" }] },
    "team2": { "opponent", "players": [] },
    "status": "planned|done",
    "result": { "b", "c" },
    "coachesB": [], "coachesC": [],
    "playerStatus": { "name": "played|cancelled|benched" },
    "playerStats": { "name": { "goals", "assists", "saves" } },
    "motmB", "motmC",
    "focus": ["","",""],
    "wentWell", "wentBad",
    "hsId", "hsTeamId"
  }],
  "settings": {
    "teamConfig": [{ "slot", "label", "hsName", "row" }],
    "hsPlayerMap": { "holdsportNavn": "appNickname" }
  }
}
```

**Lokale indstillinger** (gemmes kun i `localStorage`, aldrig i KV):
- `kvWorkerUrl` — KV Worker URL
- `hsWorkerUrl` — Holdsport Worker URL
- `appToken` — KV adgangskode
- `hsToken` — Holdsport adgangskode

---

## Kendte bugs og løsninger

### Tidspunkt på kampe sættes til browserens default
**Problem:** `<input type="time">` har ingen HTML-standard default-value, men browseren viser og sender sin egen default (fx 12:30) hvis brugeren ikke aktivt ændrer feltet. Det betyder at tidspunktet gemmes som 12:30 selvom brugeren ikke har sat det.

**Løsning:** Tidspunktsfeltet er et almindeligt **tekstfelt** (`type="text"`) med `placeholder="fx 16:30"`. Validering i `saveEditMeta` sikrer at kun gyldige `HH:MM`-formater gemmes:
```javascript
const timeRaw = (document.getElementById('em-time')?.value || '').trim();
const time = /^\d{1,2}:\d{2}$/.test(timeRaw) ? timeRaw.padStart(5, '0') : '';
g.time = time || null;
```

### KV sync race condition
**Problem:** `save()` skriver til localStorage øjeblikkeligt, men `saveToDrive()` har 1500ms delay. Hvis en anden træner trykker ↓ Hent inden de 1500ms, overskrives ændringen.

**Workaround:** Vent 2+ sekunder efter gem inden manuel KV-sync. Der er ingen automatisk polling — sync sker kun ved sideindlæsning og manuelt tryk på ↓ Hent.

### Holdsport tider i UTC
**Problem:** Holdsport API returnerer `starttime` i lokal ISO-format (fx `2026-04-03T16:30:00`). Koden bruger `raw.split('T')[1].slice(0,5)` til at udtrække tid — dette virker korrekt så længe Holdsport ikke sender UTC-normaliseret tid med `Z`-suffix.

---

## Cloudflare setup

### KV Worker (`kv-worker.js`)
- Secrets: `APP_TOKEN`
- KV binding: `HOLDSTYR_KV`
- Endpoints: `GET /db`, `POST /db`

### Holdsport Worker (`holdsport-worker.js`)
- Secrets: `HOLDSPORT_USER`, `HOLDSPORT_PASS`
- Endpoints: `GET /teams`, `GET /teams/:id/activities`, `GET /activities/:id/users`

---

## Vigtige designbeslutninger

- **Ingen backend-framework** — alt er i én HTML-fil for simpel deploy via GitHub Pages
- **KV som eneste datakilde** — localStorage er kun cache; KV er master
- **Ingen login** — adgangskontrol via delt `appToken` i localStorage
- **`isSeedData()`-check** — forhindrer at seed-data ved fejl overskrives til KV; tjekker om alle kampe har demo-IDs (`g_demo1`, `g_demo2`)
- **`fmtDate` bruger `T12:00:00`** — for at undgå UTC-offset problemer ved dato-visning i dansk tidszone; det sætter ikke `g.time`
- **Holdsport `00:00` ignoreres** — tidspunkter fra Holdsport der er `00:00` behandles som "ingen tid" (`gameTime = null`)

---

## Nøglefunktioner

- Kampoversigt med filter (status, hold, dato, modstander)
- Opstillingsstyring med dobbeltbookings-detektion
- Import af kampe og tilmeldte fra Holdsport
- Deltagelsesregistrering (spillet/afbud/oversidder)
- Statistik for spillere og trænere inkl. stævne-factor
- MoTM, fokuspunkter, mål/assists/redninger
- Delt sync via Cloudflare KV (flere trænere, ingen login)
