# Automating the Social Content Hub

Goal: you edit the Excel files in Office Online as you already do, and the live site
updates itself — no Claude, no manual commits.

## How it works

```
You edit Excel (Office Online)
        │
        ▼
Power Automate  ──copies the .xlsx files──►  GitHub repo  (build/sources/)
        │
        ▼
GitHub Action   ──runs build/build_data.py──►  data.js committed
        │
        ▼
GitHub Pages redeploys → site is live-updated
```

Two moving parts: **Power Automate** (gets your files out of the corporate tenant)
and the **GitHub Action** (turns them into the site). The Action is already in the
repo (`.github/workflows/build.yml`); you only build the Power Automate flow once.

> Why Power Automate and not a share link? Your SharePoint links require a BoyleSports
> login, so an outside server can't download them. Power Automate runs **as you, inside
> the tenant**, so it already has access — no IT app registration, no public sharing.

---

## One-time setup

### 1. GitHub repo
Make sure this project is a GitHub repo with **Pages** enabled (Settings → Pages →
deploy from branch → `main` / root). Confirm `.github/workflows/build.yml` is present.

### 2. GitHub token (once)

The Power Automate **GitHub connector has no file actions** — you write files by
calling GitHub's REST API from the **HTTP** action. That needs a token:

GitHub → **Settings → Developer settings → Fine-grained personal access tokens →
Generate new token**:
- **Repository access**: only your Pages repo.
- **Repository permissions → Contents: Read and write**.
- Generate and copy the token (starts `github_pat_…`). Keep it safe.

> The HTTP action is a **premium** Power Automate connector. If your plan doesn't
> include it, use Make.com instead (its GitHub connector has a native "Update a file"
> module — ask and I'll write that version).

### 3. Seed the files once

Before automating, commit an empty placeholder for each target path so the API has
something to update (the repo already has `build/sources/` seeded, so you can skip
this if the files are present).

### 4. Power Automate flow

In [make.powerautomate.com](https://make.powerautomate.com), **+ Create → Scheduled
cloud flow** (every 30 min). *(Or "Automated → When a file is modified" to fire on edit.)*
For **each** source file, add these three actions:

1. **OneDrive/SharePoint → Get file content** → pick the file (e.g. the promotions `.xlsx`).

2. **HTTP → get current sha**
   - Method: `GET`
   - URI: `https://api.github.com/repos/OWNER/REPO/contents/build/sources/Promotions.xlsx?ref=main`
   - Headers:
     - `Authorization: Bearer YOUR_TOKEN`
     - `Accept: application/vnd.github+json`
     - `User-Agent: power-automate`

3. **HTTP → write the file**
   - Method: `PUT`
   - URI: `https://api.github.com/repos/OWNER/REPO/contents/build/sources/Promotions.xlsx`
   - Headers: same three as above.
   - Body:
     ```json
     {
       "message": "Update Promotions.xlsx via Power Automate",
       "content": "@{base64(body('Get_file_content'))}",
       "sha": "@{body('HTTP')?['sha']}",
       "branch": "main"
     }
     ```
   `Get_file_content` = the SharePoint step; `HTTP` = the GET step. On the very first
   run (before seeding) drop the `sha` line — GitHub creates the file. After that the
   `sha` line lets it overwrite.

Repeat steps 1–3 for each file, changing only the path at the end of both URIs. Target
paths (they match `build/config.json`):

   | Source file | Path in both URIs (`…/contents/<PATH>`) |
   |---|---|
   | Promotions database | `build/sources/Promotions.xlsx` |
   | Gaming plan | `build/sources/Social Media Organic Plan.xlsx` |
   | SBK plan | `build/sources/SBK Plan.xlsx` |
   | Brandwatch CSV (Mon–Thu) | `build/sources/Brandwatch_Part1.csv` |
   | Brandwatch CSV (Fri–Sun) | `build/sources/Brandwatch_Part2.csv` |

**Save**, then **Test → Manually** once to seed and confirm the commits land.

> **Tip:** put the GET+PUT pair for all files in one flow so a single run updates
> everything. If a file 404s on the GET (doesn't exist yet), open the PUT step's
> **⋯ → Configure run after → “has failed”** so it still runs the first time.

That's it. From now on: edit Excel → within ~30 min the flow copies it → the Action
rebuilds `data.js` → the site updates.

### 5. Adjust `build/config.json` each week
The only thing that changes weekly is the week label/date at the bottom of
`config.json` (`week_label`, `week_commencing`, `week_theme`). Edit those and the
build targets the right week. *(This can later be derived automatically from the
sheet's own date column so you don't touch it at all.)*

---

## What's automated vs. what you control

- **Automated (deterministic):** promo captions from your database (verb rotation,
  Bet & Get amounts, market/promo links), Super Boost = design-box only, EPO Monday-only,
  gaming cards, schedule.
- **You control (the QC point):** the promotions database itself. Anything you want
  changed on the site, you change in the sheet.
- **Preserved, not regenerated:** the Calendar (`campaign`) and quick `links` —
  edited by hand or via a later SBK pass.

## Running it yourself (to test)

```
pip install openpyxl
python3 build/build_data.py --check   # prints data.js to screen, writes nothing
python3 build/build_data.py           # writes data.js
```

## Known limitations (being tightened)

- **Gaming column mapping** is pinned to the current sheet layout. If the gaming
  workbook's columns move, the parser in `build_data.py` (`build_gaming`) needs the
  new column numbers. Keeping the gaming sheet's layout stable keeps this reliable.
- **SBK plan** isn't wired in yet — the link needs a BoyleSports login I don't have.
  Once its structure is confirmed, it plugs into the same build.
- **Images** still live in `images/` (matched by fixture name) or as Air links in the
  sheet; they aren't pulled automatically.
