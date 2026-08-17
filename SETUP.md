# Nodalis waiting-list site — setup guide

This package has three files:

| File | What it is |
|---|---|
| `index.html` | The waiting-list website. Upload this to your GitHub repo / Pages. |
| `Code.gs` | The Google Apps Script backend — stores signups in a Google Sheet and emails you. |
| `SETUP.md` | This guide. |

The site is fully self-contained (one HTML file, no build step). The only thing you need to do before it's live is connect it to a Google Sheet, which takes about 10 minutes.

> **Note on secrets:** nothing in this package is a private API key — the Apps Script "Web App URL" you'll generate below is more like an unlisted form endpoint than a password. Still, treat it as sensitive: anyone with the URL can submit rows to your Sheet (they can't read existing rows through it, only add new ones), so don't post it anywhere public other than inside `index.html` itself. The one setting that *is* meant to be private is `APPROVE_SECRET` in `Code.gs` (Part 1, step 4) — it's what makes the one-click APPROVE link in your notification email safe to click from your inbox.

---

## Part 1 — Create the Google Sheet + backend

1. Go to [sheets.google.com](https://sheets.google.com) and create a new blank spreadsheet. Name it something like **"Nodalis Waitlist"**.
2. In the menu, click **Extensions → Apps Script**. A new tab opens with a script editor already attached to this Sheet.
3. Delete anything in the default `Code.gs` file, then paste in the entire contents of the `Code.gs` file from this package.
4. At the top of the pasted code, check two constants:
   ```js
   const NOTIFY_EMAIL = 'h24creationz@gmail.com';
   const APPROVE_SECRET = 'change-this-to-any-random-string-only-you-know';
   ```
   `NOTIFY_EMAIL` is already set to the address you asked for — change it if you ever want notifications to go somewhere else. **`APPROVE_SECRET` you should change** to any random string of your own (mash the keyboard — nobody else needs to know it or type it). It's what makes the one-click APPROVE link in your notification email tamper-resistant, so a stranger can't guess a link and approve/email other people's rows. You only set this once, before your first deploy.
5. Click the **Save** icon (or `Ctrl/Cmd + S`).
6. Optional but recommended: run it once by hand to confirm it works and to grant permissions early.
   - In the function dropdown at the top (next to "Debug"), select **`_manualTest`**.
   - Click **Run**. The first time, Google will ask you to authorize the script — click through **Review permissions → (your account) → Advanced → Go to (project name) → Allow**. This is Google's standard warning for scripts you wrote yourself; it's safe to accept.
   - Check your Sheet — a new tab called **Waitlist** should appear with a header row (`Timestamp | Email | Source | Status`) and one test row marked `Pending`. Check your inbox — you should get a test notification email with an **APPROVE & WELCOME THEM** button in it.
   - Note: this manual test run won't have a working APPROVE link yet (that needs the Web App to be deployed first — that's Part 2, right below). It's still useful for confirming the Sheet and basic email work.

---

## Part 2 — Deploy it as a Web App

This is the step that turns the script into a URL your website can call.

1. In the Apps Script editor, click **Deploy → New deployment**.
2. Click the gear icon next to "Select type" and choose **Web app**.
3. Fill in:
   - **Description:** `Nodalis waitlist v1` (anything you like)
   - **Execute as:** **Me** (your account)
   - **Who has access:** **Anyone** — this is important. If it's set to "Anyone with a Google account" or restricted, the public website won't be able to reach it.
4. Click **Deploy**, then **Authorize access** again if asked.
5. Copy the **Web app URL** shown — it looks like:
   ```
   https://script.google.com/macros/s/AKfycb.../exec
   ```
   Keep this tab open; you'll need this URL in the next step.

### Whenever you edit `Code.gs` later

Apps Script does **not** auto-publish edits. After changing the script:
`Deploy → Manage deployments → (pencil/edit icon on your deployment) → Version: New version → Deploy`.
The `/exec` URL stays the same — you don't need to update the website again.

---

## Part 3 — Wire the URL into the website

1. Open `index.html` in a text editor.
2. Find this line (near the top of the `<script>` block, just before `</body>`):
   ```js
   const WAITLIST_ENDPOINT = "PASTE_YOUR_GOOGLE_APPS_SCRIPT_WEB_APP_URL_HERE";
   ```
3. Replace the placeholder with the URL you copied in Part 2, keeping the quotes:
   ```js
   const WAITLIST_ENDPOINT = "https://script.google.com/macros/s/AKfycb.../exec";
   ```
4. Save the file.

### Sanity check before going live

Paste your deployment URL into a browser with `?ping=1` on the end, e.g.:
```
https://script.google.com/macros/s/AKfycb.../exec?ping=1
```
A working deployment replies with something like `{"status":"pong","time":"..."}`. If you instead see a Google sign-in page, "Script function not found", or a blank error page, go back to Part 2 and re-check the "Who has access: Anyone" setting and that you deployed (not just saved).

---

## Part 4 — Put it on GitHub Pages

1. Create a new repository on GitHub — e.g. `Nodalis-Waitlist` (or reuse your `Nodalis` repo in a `waitlist/` subfolder, whichever you prefer).
2. Upload `index.html` to the repo root (drag-and-drop on the GitHub web UI works, or `git add` / `commit` / `push` from your machine).
3. In the repo, go to **Settings → Pages**.
4. Under **Build and deployment → Source**, choose **Deploy from a branch**, pick your default branch (usually `main`) and folder `/ (root)`, then **Save**.
5. GitHub gives you a URL like `https://<your-username>.github.io/Nodalis-Waitlist/` — it takes a minute or two to go live the first time.

---

## Part 5 — Approving signups (the one-click email button)

Every time someone joins the waitlist, you get an email at `NOTIFY_EMAIL` that looks like a small dev-log card with a big **"→ APPROVE & WELCOME THEM"** button.

- **You don't have to click it right away.** The person is already saved in the Sheet with `Status = Pending` the moment they submit — approving is a separate, deliberate step for *you*, not something that blocks their signup.
- **Clicking it does two things:** it flips that row's `Status` to `Approved` in the Sheet, and it sends the visitor a formatted "You're in! Welcome to the Nodalis community" email — the excitement email, styled to match the site, telling them they took a real step and are part of the community now.
- **It's safe to click more than once.** If a row is already `Approved`, clicking the link again just shows "already approved" and does not send a second welcome email.
- **No login needed.** The link itself is the credential — it's signed with your `APPROVE_SECRET` from Part 1, so only links this script actually generated will work. That's why changing `APPROVE_SECRET` to your own random value (not the placeholder) matters before your first real deploy.
- **You can also approve by hand.** Just edit the `Status` cell in the Sheet directly to `Approved` — the site doesn't read that column, so nothing breaks either way. The only thing the APPROVE link gives you that hand-editing doesn't is triggering the welcome email automatically.

---

## How the waitlist actually works (for reference)

When someone submits the form, the site tries three things in order so a signup is never silently lost:

1. **JSONP call** to your Apps Script `/exec` URL — this is the "live" path. It gets a real answer back: `ok` (with a running total, used for the "your spot in line" popup), `duplicate` (already on the list, with the date they first joined), or `error`.
2. If that doesn't answer in 9 seconds, a **fallback POST** is sent to the same URL. The browser can't read the response for this one (a `no-cors` request), but the request still reaches your script, still gets written to the Sheet, and still triggers your notification email — it's a safety net, not a second chance to fail.
3. If `WAITLIST_ENDPOINT` is still the placeholder (not configured yet), the button instead opens the visitor's email client with a **pre-written `mailto:`** addressed to your notify email, so submissions still reach you even before you finish setup.

On a confirmed `ok`, the site pops up a celebratory "You're in the community now" modal with a little animation and their spot-in-line number — this is the "excitement" moment on the site itself, separate from (and in addition to) the welcome email they get once you click APPROVE.

Every new row written to the **Waitlist** tab also triggers the owner notification email above (to `NOTIFY_EMAIL`) with the APPROVE button — no need to keep the Sheet open to know someone joined.

A duplicate check (case-insensitive, against the Email column) means the same address can't be added twice; the visitor is told they're already on the list along with the date they first signed up. The site also remembers submitted addresses in the visitor's own browser (`localStorage`) so repeat visits show "you're already on the list" instantly, without even calling the script.

---

## Troubleshooting

- **"That didn't go through" error on the site, with a "Technical detail" box.** Click to expand it — it shows the exact reason (e.g. the endpoint returned a sign-in page). This almost always means the deployment's access isn't set to "Anyone", or you edited the code but deployed it as the same version instead of a new one.
- **No owner notification email arriving.** Check the Sheet first — if the row *is* there but no email came, check Gmail's Sent/quota: free Gmail accounts via `MailApp` have a daily sending quota (100/day for free accounts, more for Workspace) — plenty for a waitlist, but worth knowing if you're testing heavily. Also check spam.
- **Clicked APPROVE but the visitor never got a welcome email.** Check the Sheet — if `Status` shows `Approved`, the approval itself worked; the welcome email may have been blocked by the same Gmail quota, or bounced (bad/typo'd address). You can re-trigger it by hand: temporarily set the cell back to anything other than `Approved`, then click the APPROVE link again.
- **APPROVE link says "Invalid link" or "Link not recognised".** Usually means `APPROVE_SECRET` was changed *after* that particular notification email was sent (old links are signed with the old secret and stop working) — approve that row by hand in the Sheet instead. Keep `APPROVE_SECRET` stable once you're live.
- **Row appears twice for the same email.** Shouldn't happen — `handleSignup_` in `Code.gs` uses a script lock plus a case-insensitive scan before appending. If you see it, check whether two different deployments (old + new) are both live and both being called.
- **Want to export the list?** It's just a Google Sheet — File → Download → CSV, or Data → connect it to Google Data Studio/Looker Studio if you want a live signup chart.
- **Want a copy of the raw signups without opening Sheets?** `_manualTest` is only for testing; instead use **File → Download** on the Sheet, or add your own `doGet` branch to export as needed.

---

## Customizing later

- **Copy/branding:** all the marketing text lives directly in `index.html` — search for the section you want (`id="access"` for the waitlist panel, `id="faq"` for FAQ, etc.) and edit in place.
- **The welcome/excitement email:** its wording and styling live in the `sendExcitementEmail_` function in `Code.gs` — edit the `htmlBody` string there.
- **The owner notification email:** lives in `sendOwnerNotification_` in `Code.gs`, right next to it.
- **Default light/dark:** the site opens in light mode following the visitor's system preference; the moon/sun icon in the nav (and the LIGHT/DARK buttons in the "Personalize" section, which now control the same real site-wide theme) let them switch, remembered via `localStorage`.
- **Theme lab colours:** the three preview styles (Nodalis / Notion / Nothing) are defined in the `STYLES` array inside the `<script>` block — edit the hex values there if the app's palette changes.
- **Loading screen text:** the three boot messages ("INITIALIZING VAULT…", etc.) are in the `boot()` function near the top of the `<script>` block.
