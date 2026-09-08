# Board — cross-project task board

A single-file PWA (installable web app) with a shared task board. Runs local-only out of
the box; add a Firebase config to sync across your iPhone, your computer, and Claude.

Files:
- `index.html` — the whole app (UI + logic)
- `manifest.webmanifest` — makes it installable to the home screen
- `sw.js` — offline service worker
- `icon.png` — home-screen icon

## Data model (Firestore collection `tasks`)

Each task is one document, id = random string. Fields:

| field       | values                                   |
|-------------|------------------------------------------|
| `text`      | the task                                 |
| `project`   | key from PROJECTS (finance, tesla, …)    |
| `status`    | `todo` · `doing` · `waiting` · `done`    |
| `priority`  | `high` · `med` · `low`                   |
| `due`       | freeform string ("Jul 3") or ""          |
| `wake`      | ISO date ("2026-08-01") or ""            |
| `note`      | short note ("plan opens") or ""          |
| `source`    | `claude` or `loren`                      |
| `order`     | number (creation order)                  |
| `updatedAt` | epoch ms                                 |

Single-doc-per-task on purpose: Claude can update one field on one task via a REST PATCH
without touching (or clobbering) anything you're editing on your phone.

## Turn on sync — Firebase setup (~2 min, one time)

1. Go to https://console.firebase.google.com → **Add project** (name it e.g. `loren-board`).
   Skip Google Analytics. Create.
2. In the project, click the **web icon `</>`** ("Add app to get started") → give it a
   nickname → **Register app**. Firebase shows a `firebaseConfig = { … }` block. **Copy it.**
3. Left sidebar → **Build → Firestore Database → Create database** → start in **production
   mode** → pick a region → Enable.
4. Firestore → **Rules** tab → paste the rules below → **Publish**.
5. Paste the config from step 2 into `index.html` (replace `const FIREBASE_CONFIG = null;`).

### Firestore rules (starter)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /tasks/{id} {
      allow read, write: if true;
    }
  }
}
```

Note: `if true` = anyone who knows the project URL can read/write the `tasks` collection.
Fine for a personal list with no account numbers or secrets in it. Harden later with
Firebase Auth if wanted (see "Security" below).

## Deploy (GitHub Pages)

1. New GitHub repo (can be public), push the contents of this `app/` folder to it.
2. Repo **Settings → Pages → Source: Deploy from branch → main → /(root)** → Save.
3. Wait ~1 min → live at `https://<user>.github.io/<repo>/`.
4. iPhone: open that URL in **Safari → Share → Add to Home Screen**.

## How Claude updates the board

Claude reads/writes the same Firestore `tasks` collection via the REST API from a Claude
Code session. Project id and API key come from the config.

Read all tasks:
```
curl "https://firestore.googleapis.com/v1/projects/<PROJECT_ID>/databases/(default)/documents/tasks?key=<API_KEY>"
```

Move a task to In progress (PATCH one field):
```
curl -X PATCH \
  "https://firestore.googleapis.com/v1/projects/<PROJECT_ID>/databases/(default)/documents/tasks/<TASK_ID>?updateMask.fieldPaths=status&key=<API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"fields":{"status":{"stringValue":"doing"}}}'
```

Changes appear on the phone instantly (Firestore realtime listener).

### Snooze / wake dates

Set `wake` to an ISO date (`YYYY-MM-DD`) on a `waiting` task and the app moves it to
`todo` on/after that day (checked on load, on focus, and every 5 min). Tap the **☾** on any
card to set it; setting a future wake on an active task auto-parks it in `waiting`. Via CLI:
`board.sh set <id> wake 2026-08-01`.

## Security (optional hardening, later)

The starter rules are open. To lock the board to just you + Claude:
- Enable **Anonymous** or **Email/Password** auth in Firebase.
- Change rules to `allow read, write: if request.auth != null;`.
- App signs in on load; Claude authenticates via the Identity Toolkit REST endpoint to get
  a token, then sends it as a Bearer header on the Firestore calls.
Deferred for v1 — the data here is low-sensitivity task titles.

## Facts (`/facts/`)

A second tiny PWA in this repo: a chat-style log of interesting facts, live at
`https://lboykoff.github.io/board/facts/`. Same Firebase project and sign-in as Board.

- Type a line or paste a link, tap **Log**. The raw note saves at once.
- With an Anthropic API key saved in ⚙ (stored only in that browser), the page calls Claude
  straight from the browser, cleans the note into one to three sentences, tags it, and adds a
  caveat when the fact is contested. Without a key, the fact stays `raw` and Claude tidies it
  later from a Claude Code session (`~/.claude/facts.sh raw` / `done`).
- 🎲 shows a random fact. Search and tag chips filter the list.
- Share-in: open `/facts/?q=<text>` to prefill, or `/facts/?q=<text>&go=1` to log at once.
  An iOS Shortcut can pass a shared YouTube link this way.

Firestore collection `facts`, one document per fact:

| field | values |
|---|---|
| `raw` | what was typed, untouched |
| `fact` | the cleaned sentences ("" until cleaned) |
| `tag` | science · space · history · people · tech · body · money · nature · words · culture · me · other |
| `note` | caveat, or "" |
| `url`, `title` | source link and its title (from noembed), or "" |
| `status` | `raw` · `done` |
| `source` | `loren` · `claude` |
| `createdAt`, `updatedAt` | epoch ms |

Rules: add a `match /facts/{id}` block with the same auth check as `tasks`.
The root `sw.js` never caches `/facts/` (same bypass as `/cull/`).
