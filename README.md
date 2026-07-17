# CXDX — Prescription vs Conversion

A single-file web app that tracks prescriptions against monthly raw billing
dumps to show what converted and what leaked, by date and by month.

**Files in this folder**
| File | Purpose |
|---|---|
| `index.html` | The entire app — UI, logic, and Firebase calls in one file |
| `firestore.rules` | Security rules to deploy alongside your Firebase project |
| `README.md` | This guide |

Everything runs client-side against **Firebase** (Authentication + Firestore).
There is no separate backend to host — you deploy `index.html` as a static
file (Firebase Hosting and/or GitHub) and it talks to Firebase directly.

---

## 1. Create a Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project** → name it (e.g. `cxdx-prod`) → finish the wizard.
2. In the left sidebar under **Build**, open **Authentication** → **Get started** → enable the **Email/Password** sign-in method.
3. Under **Build**, open **Firestore Database** → **Create database** → start in **Production mode** → pick a region close to your users.

## 2. Connect `index.html` to your project

1. In Firebase Console → ⚙️ **Project settings** → **General** → scroll to *Your apps* → click the **Web** icon (`</>`) → register an app (nickname doesn't matter, skip Hosting setup here).
2. Copy the `firebaseConfig` object it gives you.
3. Open `index.html`, find this block near the top of the `<script type="module">` section, and paste your values in:

   ```js
   const firebaseConfig = {
     apiKey: "YOUR_API_KEY",
     authDomain: "YOUR_PROJECT.firebaseapp.com",
     projectId: "YOUR_PROJECT_ID",
     storageBucket: "YOUR_PROJECT.appspot.com",
     messagingSenderId: "YOUR_SENDER_ID",
     appId: "YOUR_APP_ID"
   };
   ```

   Until this is filled in, opening the file shows a "Firebase setup needed" screen instead of the login page — that's expected. A Firebase **web API key is not a secret** (it identifies your project, it doesn't authorize access on its own), so it's normal for it to live in client-side code like this; real access control comes from the security rules in the next step.

## 3. Deploy the security rules

These rules make sure Admins can only ever read/write their own company's
data, and that only a Super Admin can create companies or admin accounts.

**Easiest — paste into the console:**
Firestore Database → **Rules** tab → replace the contents with everything
inside `firestore.rules` → **Publish**.

**Or with the Firebase CLI**, from a folder containing both files:
```bash
npm install -g firebase-tools
firebase login
firebase init firestore   # point it at this firestore.rules file
firebase deploy --only firestore:rules
```

## 4. Create your first Super Admin

No one can sign in until one user document exists with `role: "superadmin"`,
so this one-time step is manual:

1. Firebase Console → **Authentication** → **Add user** → enter an email and a temporary password → **Add user**. Copy the **User UID** it's assigned.
2. Firebase Console → **Firestore Database** → **Start collection** → Collection ID: `users` → **Document ID**: paste the UID you copied → add these fields:

   | Field | Type | Value |
   |---|---|---|
   | `name` | string | Your name |
   | `email` | string | the same email you signed up with |
   | `role` | string | `superadmin` |
   | `companyId` | null | *(leave as null)* |

3. Save. You can now open `index.html` and sign in with that email/password as the Super Admin.

From inside the app, the Super Admin creates each **Company**, and can add
**Admin** users to it directly from the Companies screen — no more manual
Firestore editing needed after this.

## 5. Deploy `index.html` — Firebase Hosting + GitHub

**Option A — Firebase Hosting via CLI**
```bash
firebase init hosting     # public directory = the folder holding index.html
firebase deploy --only hosting
```

**Option B — GitHub + auto-deploy**
1. Push this folder to a GitHub repo.
2. `firebase init hosting:github` walks you through connecting the repo and creates a GitHub Actions workflow that runs `firebase deploy` automatically on every push to your main branch.

**Option C — GitHub Pages**
Since it's a single static HTML file with no build step, you can also just
enable GitHub Pages on the repo and point it at `index.html` — Firebase is
only used for Auth/Firestore, not for hosting, in this option.

---

## How it works

**Roles**
- **Super Admin** — creates and manages company accounts, adds/removes Admin users per company, can open any company's workspace directly.
- **Admin** — everything inside their own company: prescription entry, master data, billing dump uploads, reports.

**Prescriptions** — a dedicated full-page form (not a popup) with Visit
Details (Date, UHID, Patient Name, Consultant, Entered By) first, then the
tests given, then Follow Up, then Remarks last. **Consultant, Entered By,
and Test Given are all type-to-search fields** — click in, type a few
letters, and pick from the filtered list (arrow keys + Enter work too)
rather than scrolling a long dropdown. The **UHID field auto-strips spaces
and forces capital letters** as you type, since matching depends on it
being exact. One prescription visit can include several tests — each row
is just Test + Amount, added with the button or by pressing **Enter** in a
test row for another instantly. A running "N tests · ₹total" summary
updates live as you add rows. Every test row defaults to **"No Test
Given"** with the amount locked at zero — pick a real test only if one was
actually prescribed. A visit saved this way still counts toward Total
Prescriptions on the dashboard and reports, but is left out of Test Given,
Conversion, and Leakage entirely (it shows as a neutral **No Test** badge,
filterable from the Prescriptions list) — since there was nothing to
convert. **Consultation Follow Up** and **Test Follow Up** sit side by side
in their own section near the end of the form — both are optional dates,
and Test Follow Up applies to the whole visit rather than each test
individually. The Prescriptions list also has a **From/To date filter**
alongside the search and status pills, handy for finding an older entry to
edit — all filters reset to blank each time you open the tab. The whole
filter area can be collapsed with the **Hide Filters** toggle above it, for
more table space.

**Masters** — Doctors, Tests, and Agents are each manageable with add/edit/
delete, and each has a **Download Template → fill in Excel → Import**
workflow for bulk loading.

**Billing dumps** — uploaded one calendar month at a time. Uploading again
for a month that already has data replaces it (after a confirmation), just
like the January/February flow you'd expect. Because raw billing exports
differ company to company, each upload shows a **column-mapping step**
where you tell CXDX which of *your* file's columns are UHID, Date, Amount,
Test Name, and (optionally) Patient Name — UHID, Date, Amount, and **Test
Name are all required** since matching relies on all four. That mapping is
remembered as the default for next time, per company.

**Bulk prescription import** — same idea as billing dumps: pick your file
and map its columns to Date, UHID, Patient Name (required) plus Consultant,
Test Given, Test Amount, both Follow Up dates, Remarks, and Entered By
(optional) — it doesn't need to match the downloadable template's exact
headers. The preview shows how many rows are ready versus skipped (missing
UHID or an unreadable date) *before* you commit, so a mismatch is obvious
immediately rather than discovered after the fact.

**Matching engine** — for every prescription row, CXDX looks for a billing
row with the *same UHID*, dated on the prescription date **or within a
configurable window after it** (default: same day or the next day — set
this in **Settings**, or switch it to **Unlimited** to match any billing
date on or after the prescription date, with no cutoff). The test name has
to match too (case and spacing don't matter) — a same-day billing row for a
*different* test is never counted as a conversion. Each billing row can
only be consumed by one prescription, so amounts are never double-counted.
A match marks the row **Completed** and records the actual billed amount as
its *Conversion Value*; no match leaves it as **Leakage**, valued at the
originally prescribed Test Amount.

Saving a single prescription (new entry or edit) matches just that
visit — a handful of reads, not a scan of your whole company's data. Billing
dump uploads and bulk prescription imports **don't** auto-match, on
purpose: uploading several months back-to-back would otherwise re-scan
everything after every single upload, which is what burns through
Firebase's free-tier quota fastest. Upload or import everything first, then
click **Run Matching** once (on the Dashboard or the Billing Dumps page) to
process it all together. Deleting a billing dump or changing Matching Rules
still re-matches everything automatically, since those need a full,
guaranteed-correct pass.

**Reports** — a **Monthly Summary** (Total Prescriptions, Total Tests Given,
Total Converted, Prescription/Conversion/Leakage Value, Conversion % and
Leakage %, with an all-time Total row) and a **Date-wise Detail** view of
every row and its status. **Export Excel** downloads both as a two-sheet
workbook, with the Date column written as a real Excel date (not text) so
it sorts, filters, and works in date formulas natively.

---

## Data model (Firestore)

```
users/{uid}                          name, email, role, companyId
companies/{companyId}                name, active, matchingConfig, customFields
  doctors/{id}                       name, specialization
  tests/{id}                         name, amount
  agents/{id}                        name
  prescriptions/{id}                 date, uhid, patientName, consultant, testGiven,
                                      testAmount, consultationFollowUp, testFollowUp,
                                      remarks, enteredBy, matchStatus, matchedAmount,
                                      custom (your added fields live here)
  billingDumps/{YYYY-MM}             rowCount, uploadedAt
    rows/{id}                        uhid, date, amount, testName, patientName, used
```

## Adding an extra prescription field for one company

The core prescription fields (Date, UHID, Test Given, etc.) are wired
directly into the matching engine and are the same for every company. For
anything beyond those, each company can define its **own** extra fields —
one company having a "Referral Source" field doesn't add it anywhere else.

From that company's **Settings** page (Super Admin: open the company first),
**Custom Prescription Fields → Add Field**, set:

- **Label** — what shows on the form and as the Excel column header (an
  internal key is generated from this automatically and stays fixed even if
  you rename the label later, so existing data never gets orphaned)
- **Type** — Text, Number, Date, Yes/No, or Dropdown (Dropdown asks for a
  comma-separated list of options)
- **Required** — whether the prescription form should block saving without it

It shows up immediately on that company's New/Edit Prescription form, in
the Prescriptions and Reports tables, and in the Excel template, bulk
import, and export — for that company only. Deleting a field just hides
it going forward; prescriptions that already had a value for it keep that
data.

If you outgrow this (e.g. you want a field to pull its own master list, the
way Consultant pulls from Doctors), that's a bigger change to the form
logic rather than a Settings addition — happy to help with that when you get
there.

## Customizing

- **Currency symbol** — change `CURRENCY_SYMBOL` near the top of the script (defaults to `₹`).
- **Matching window** — each company sets its own under Settings (default: 1 day — same day or next day). Check **Unlimited** there instead to match any billing date on or after the prescription date, with no cutoff.
- **Ambiguous dates in uploads** — a date like `03/04/2026` is treated as **DD/MM/YYYY** (3 April) — the Indian convention. If your billing exports use US-style `MM/DD/YYYY`, export dates in `YYYY-MM-DD` format instead to avoid ambiguity, or adjust the `parseFlexibleDate` function. If a date column has a time merged into the same value (`15-01-2026 14:30:00`, `2026-01-15T14:30:00`, etc.), the time is simply dropped and the date is used as-is — this applies everywhere dates are read: billing dumps, prescription import, and follow-up dates.

## Using this in India

Nothing about Firebase, Firestore, or the app itself is region-locked — this
runs the same everywhere. Two things worth knowing if you're deploying for
an Indian company:

- **Data residency** — when you create the Firestore database (Step 1), you
  can pick `asia-south1` (Mumbai) as the region so patient/prescription data
  is stored within India, if that matters for your organization's policy.
- **Personal data handling** — UHID, patient names, and doctor names are
  personal data, so India's DPDP Act (Digital Personal Data Protection Act,
  2023) is likely relevant to how you operate this, not just where it's
  hosted. This app enforces per-company access control via `firestore.rules`,
  but things like consent notices, retention policy, and grievance handling
  are organizational decisions the software alone can't cover — worth a
  quick check with whoever handles compliance at your company. (Not legal
  advice — just flagging it since real patient data is involved.)

## Scale & limitations to know about

- A full **Run Matching** pass (from the Dashboard or Billing Dumps page)
  re-reads every prescription and every billing row for the company — fine
  for the low tens of thousands of rows a clinic/diagnostics business would
  typically generate, but a large multi-year dataset would benefit from
  moving this step to a Cloud Function later. Day-to-day single prescription
  saves use a cheap, targeted match instead (queries just that patient's
  billing rows), so routine entry doesn't carry this cost — only bulk
  uploads/imports and the manual full run do.
- Firebase's free (Spark) tier has daily read/write quotas. If you're
  loading many months of historical data at once, upload/import everything
  first and run matching once at the end rather than after each file — see
  "Matching engine" above. If you outgrow the free tier's quota
  permanently, upgrading to the pay-as-you-go Blaze plan removes the daily
  cap (Firestore is still inexpensive at this app's scale — Blaze is
  billed per operation with no fixed monthly cost until usage is
  substantial).
- Deleting a company from the Super Admin console removes its Firestore
  data, but **not** the linked Firebase Authentication accounts (the client
  SDK can't delete other users' sign-in accounts for security reasons) —
  remove those from Authentication in the console if needed.
- Excel/CSV parsing runs in the browser via SheetJS; very large files (tens
  of thousands of rows) may take a few seconds to process.

## Troubleshooting

- **"Firebase setup needed" screen** → `firebaseConfig` in `index.html` still has placeholder values (Step 2).
- **"Missing or insufficient permissions"** → security rules aren't deployed yet (Step 3), or the signed-in user has no matching `users/{uid}` document.
- **Can't sign in at all** → double check the account exists in Authentication *and* has a `users/{uid}` document with a `role` field (Step 4).
- **"Your project has exceeded no-cost limits" in Firebase Console** → the free Spark plan's daily quota is used up; the app will start failing reads/writes until it resets (quotas reset daily) or you upgrade to Blaze (pay-as-you-go, still cheap at this scale — see "Scale & limitations"). Loading historical data in smaller batches and running matching once at the end, per the note above, avoids hitting this again.
