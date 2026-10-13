# CivicSync

Turn a smartphone photo of a local hazard into a routed municipal complaint with a public deadline.
Gemma 4 (via the Gemini API) classifies the photo, rates severity, picks the department and drafts the
grievance letter. Every ticket gets an SLA clock: critical 48 hours, medium 7 days, low 15 days.

## Run it

```bash
npm install
cp .env.example .env.local   # add your GEMINI_API_KEY
npm run dev                  # http://localhost:3000
```

Without `GEMINI_API_KEY` the app runs in **demo mode**: the classification is guessed from the text note
and every screen labels it as demo mode. Get a key at https://aistudio.google.com/apikey.

Model: `gemma-4-26b-a4b-it` by default. Set `GEMMA_MODEL=gemma-4-31b-it` for the larger model.

## Pages

| Route | What it does |
| --- | --- |
| `/` | Landing page with the live top-priority ticket |
| `/report` | Photo + map pin + note, Gemma 4 analysis, editable letter, duplicate check |
| `/feed` | Open hazards ranked by upvotes, severity and age |
| `/map` | All hazards on a map: red overdue, amber within SLA, green resolved |
| `/hazards/[id]` | Ticket: live SLA clock, timeline, letter, upvote, mark resolved |
| `/leaderboard` | Wards ranked by fixes and speed, plus the Hall of Inaction |
| `/ranks` | Reporters ranked by points: 3 per report, 10 more when it gets fixed |
| `/my` | Your complaints: everything reported from this browser, with your rank |

## Where things live

- `src/lib/gemma.ts`: the Gemma 4 prompt, JSON parsing and demo fallback
- `src/lib/data.ts`: in-memory store and seed data (resets on restart; swap for SQLite later)
- `src/lib/sla.ts`: SLA states, "Overdue by" labels and the priority score
- `src/app/api/*`: analyze, create, upvote and resolve endpoints
- `src/app/globals.css`: colour, type and radius tokens

## Languages

English plus all 22 languages of the Eighth Schedule (Hindi first, then Assamese to Urdu).
The picker is in the sidebar; the choice is saved in a cookie. Urdu, Sindhi and Kashmiri switch the
layout to right-to-left. Each script has its own Noto font, downloaded only when that script is shown.

- `messages/en.json` and `messages/hi.json` are written by hand.
- Every other language is translated by Gemma 4. When someone picks a language that has no file yet,
  the banner starts a background job (`POST /api/translate`, needs `GEMINI_API_KEY`), shows live
  progress, and refreshes the page when `messages/<code>.json` is written.
- Speed: Gemma 4 "thinks" before answering by default, which can add a minute per call. Translation
  (and report analysis) switch thinking off with `thinkingLevel: MINIMAL`. Three chunks run in
  parallel, and the menu, home and feed strings go first, so the page switches language within
  seconds while the rest finishes. On a rate limit, all workers pause for the delay the API asks for.
  Progress is saved to `messages/.partial/`; running it again continues from there.
- Optional settings in `.env.local`: `TRANSLATE_MODEL` (a different model just for translation),
  `TRANSLATE_CONCURRENCY` (parallel requests, default 3; lower it if you see many 429s),
  `GEMMA_THINKING=high` (turn thinking back on for photo analysis and fix checks: slower, sometimes
  more careful).
- Best for a demo or launch: run `npm run translate` once and commit `messages/*.json`, so nobody ever
  waits for a translation. To prepare all of them before a demo:

  ```bash
  npm run translate            # every language without a file yet
  npm run translate -- ta bn   # just these
  ```

- Machine translations, especially for Bodo, Dogri, Kashmiri, Konkani, Maithili, Manipuri (Meetei
  Mayek), Santali (Ol Chiki) and Sindhi, need review by native speakers before a real launch.
- Report titles and grievance letters stay in English, the working language of most municipal
  offices. Residents can type their note in any Indian language; Gemma 4 understands it.

## Accounts, themes, photos

- No sign-up. The first report sets an anonymous `civicsync_reporter` cookie, which powers
  "Your complaints" and reporter ranks. Clearing cookies starts a fresh identity.
- Light and dark themes. The toggle is in the sidebar (and top bar on mobile); the choice is saved,
  and the first visit follows the device setting.
- Up to 4 photos per report. Gemma 4 receives all of them in one request.

## Accountability features

- **Fix proof.** A ticket closes only when someone uploads "after" photos and Gemma 4 compares them with
  the original report. A confident "fixed" resolves it; "not fixed" is rejected and stays on the public
  timeline; "unclear" asks for better photos. Live GPS at the spot is recorded when shared.
- **Still broken?** A resident within 200 m (live GPS) can reopen a resolved ticket with a fresh photo.
  The original deadline still applies, so it usually comes back overdue.
- **I see it too.** Neighbours within 200 m confirm a report with live GPS, once each. Confirmations count
  more than remote upvotes in the priority score. You can't confirm your own report.
- **Escalation ladder.** As the delay grows: reminder to the department (deadline), Municipal Commissioner
  (+7 days), ward councillor (+14 days), then an RTI application (30 days after reporting). Letters are
  polished by Gemma 4 from the ticket's facts; the RTI application is a fixed template under Sections 6(1)
  and 7(1) of the RTI Act, 2005, with the fee left blank because it varies by state. Nothing is sent
  automatically: copy it or open it in your email.
- **Privacy blur.** During analysis Gemma 4 also returns boxes for faces and number plates. They are
  pixelated in the browser before filing, and the reporter can add or remove boxes. Only blurred copies
  are stored. Gemma's boxes can miss things, which is why the manual tool is always there.

## Location trust

Reports are pinned from the reporter's **live GPS**, not a map tap:

- "Lock my live location" takes a fresh high-accuracy fix (`maximumAge: 0`).
- Fixes rougher than ±100 m, older than 10 minutes, or outside India are rejected.
- The pin can be nudged at most 50 m from the fix (the dashed circle).
- The server re-checks all of this in `src/lib/location.ts`; it never trusts the browser's word.
- Reporters who aren't at the spot can place the pin by hand. Those tickets are marked
  "Location not verified" and rank lower in the feed.

Browsers only share location on secure pages. `localhost` counts, but a phone opening your laptop's
IP does not. To test on a phone, run `npm run dev -- --experimental-https` or use a tunnel such as
`cloudflared tunnel --url http://localhost:3000`.

GPS can still be faked with a mock-location app. This raises the effort a lot, but it is not proof.

## All-India coverage

- `src/lib/india-outline.ts` is a simplified outline of India (Natural Earth 1:50m) used to reject pins
  outside the country, with a 12 km allowance at coasts and borders.
- The area (locality, city, state) is filled in automatically from the pin by `/api/geocode`, which uses
  OpenStreetMap Nominatim. Users can edit it. If geocoding fails, they type it in.
- A new area is created the first time someone reports from it, so the leaderboard grows on its own.
  Seed areas cover 10 cities; new areas show "Representative not on record".
- Nominatim allows light use only (about 1 request per second). For real traffic, self-host it or use a
  paid geocoder.

## Notes

- Seed reports and representative names are sample data.
- "Mark as resolved" is open to everyone for the demo. Put it behind staff login before real use.
- Built on Next.js 16 with Cache Components. Live data reads call `connection()` and sit inside `<Suspense>`.
