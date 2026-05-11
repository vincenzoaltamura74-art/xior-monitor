# Xior Groningen Studio Monitor

Watches the public Xior pages for three long-stay residences in Groningen
and emails you when something changes — usually meaning a new studio
became available. Runs free on GitHub Actions.

## What it monitors

Three properties on `xiorstudenthousing.eu`:

- **Eendrachtskade**
- **Oosterhamrikkade**
- **Zernike Tower** (long-stay)

These pages are not Cloudflare-protected, so the script fetches them
directly with plain HTTP — no scraping API, no JavaScript rendering,
no monthly budget to manage.

## How it works

1. A GitHub Actions workflow runs **once per hour**.
2. Inside that single hourly job, the script loops for ~55 minutes,
   re-checking the three pages **every 5 minutes**.
3. For each property it extracts (a) any specific room numbers shown
   on the page (pattern `# X-XXX`, like `# 2-075`), and (b) a hash of
   the page text.
4. It compares against `state.json` (saved from the previous check).
5. If a new room appears or the page content changes, it emails you
   (and optionally other recipients) with the property name, a direct
   link to the page, and a reminder to book on `xior-booking.com`.

The hourly-run-with-internal-loop pattern is a workaround for a
documented GitHub Actions limitation: scheduled workflows configured to
run every 5 minutes get throttled to roughly hourly anyway, so we
schedule hourly and do the polling ourselves.

## Setup (≈ 25 minutes the first time)

### 1. Create a Gmail app password

The script sends notifications through your Gmail account.

1. Turn on 2-Step Verification on your Google account: <https://myaccount.google.com/security>
2. Generate an app password at <https://myaccount.google.com/apppasswords> — name it "Xior monitor".
3. Copy the 16-character password somewhere temporary (you'll paste it as a GitHub secret later).

### 2. Create a public GitHub repo

1. Go to <https://github.com/new>
2. Name it anything (e.g. `xior-monitor`).
3. Choose **Public** — public repos get unlimited free GitHub Actions minutes.
4. Don't tick "Add README/.gitignore/license" — the project already ships with them.
5. Create the repository.

### 3. Upload the project files

Drag-and-drop the four root-level files into the GitHub upload page:

- `monitor.py`
- `requirements.txt`
- `state.json`
- `.gitignore`

The fifth file — `check.yml` — lives in a subfolder. Easiest way:

1. On the repo home, click **Add file → Create new file**.
2. Type `.github/workflows/check.yml` as the filename — typing the slashes auto-creates the folders.
3. Paste the contents of `check.yml` and commit.

### 4. Add three repository secrets

Repo → **Settings → Secrets and variables → Actions → New repository secret**

| Name        | Value                                                                 |
|-------------|-----------------------------------------------------------------------|
| `SMTP_USER` | your Gmail address                                                    |
| `SMTP_PASS` | the 16-character app password from step 1                             |
| `TO_EMAIL`  | the address(es) that should receive alerts — multiple addresses can be supplied comma-separated (e.g. `me@gmail.com,mom@gmail.com`) |

### 5. Enable Actions and trigger the first run

1. Open the **Actions** tab. If GitHub asks to enable workflows, accept.
2. Click **Xior Groningen Monitor** in the left sidebar.
3. Click **Run workflow → Run workflow**.
4. The run will stay "in progress" for ~55 minutes — that's correct, it's
   doing the internal polling loop. Expand "Run monitor" to watch the
   log in real time.

### 6. Verify state.json was written

After a couple of minutes, open `state.json` on the repo. It should
contain something like:

```json
{
  "Eendrachtskade":            { "hash": "...", "rooms": [] },
  "Oosterhamrikkade":          { "hash": "...", "rooms": [] },
  "Zernike Tower (long-stay)": { "hash": "...", "rooms": [] }
}
```

If yes, you're done. The first run only records the baseline and won't
email you. Subsequent hourly runs will email you whenever a page changes.

## Other email providers

The script uses standard SMTP. For non-Gmail accounts, set two extra
secrets `SMTP_HOST` and `SMTP_PORT`:

| Provider | SMTP_HOST              | SMTP_PORT |
|----------|------------------------|-----------|
| Gmail    | smtp.gmail.com         | 587       |
| Outlook  | smtp-mail.outlook.com  | 587       |
| iCloud   | smtp.mail.me.com       | 587       |
| Yahoo    | smtp.mail.yahoo.com    | 587       |

## When you find a place

Disable the workflow so it stops running:

1. Repo → **Actions → Xior Groningen Monitor → ⋯ (top right) → Disable workflow**
2. (Optional) Revoke the Gmail app password at <https://myaccount.google.com/apppasswords>

## Troubleshooting

- **Workflow shows red X**: click the run, expand the failing step, read
  the error. Most common cause is a typo in one of the secrets.
- **No emails despite seeing a studio appear**: check Spam / Promotions
  folders in Gmail first. Also, the script only emails on *changes* —
  if it already recorded the studio in `state.json` on a previous run,
  it won't email again about the same one.
- **`state.json` stays as `{}` after the first run**: the workflow
  doesn't have write permission. Check `permissions: contents: write`
  in `check.yml`.
- **GitHub schedules can be late**: scheduled workflows are "best effort"
  and can be delayed during peak hours. This is a known GitHub Actions
  limitation.

## Important notes

- Don't trust the script blindly — check the booking site manually too,
  especially Monday mornings and the start of each month, when new
  contracts are often released.
- When an email arrives, **move fast**: studios get locked for 20 minutes
  as soon as anyone clicks "Let's book" on `xior-booking.com`.
- Some emails will be false positives (Xior changes a phone number or
  office hour on the page → email). That's intentional; better to over-
  notify than to miss a real release.
- The workflow auto-disables after 60 days of repo inactivity. GitHub
  sends an email warning when that's about to happen; just click the
  re-enable button.

## License

Use this however you want. No warranty.
