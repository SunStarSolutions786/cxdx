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

**Prescriptions** — entered with Date, UHID, Patient Name, Consultant, Test
Given, Test Amount, Consultation Follow Up, Test Follow Up, Remarks, and
Entered By. The **UHID field auto-strips spaces and forces capital letters**
as you type, since matching depends on it being exact. One prescription
visit can include several tests — the entry form lets you add multiple test
rows in one go.

**Masters** — Doctors, Tests, and Agents are each manageable with add/edit/
delete, and each has a **Download Template → fill in Excel → Import**
workflow for bulk loading.

**Billing dumps** — uploaded one calendar month at a time. Uploading again
for a month that already has data replaces it (after a confirmation), just
like the January/February flow you'd expect. Because raw billing exports
differ company to company, each upload shows a **column-mapping step**
where you tell CXDX which of *your* file's columns are UHID, Date, Amount,
and (optionally) Test Name / Patient Name. That mapping is remembered as the
default for next time, per company.

**Matching engine** — for every prescription row, CXDX looks for a billing
row with the *same UHID* dated on the prescription date **or within a
configurable window after it** (default: same day or the next day — set
this in **Settings**). Each billing row can only be consumed by one
prescription, so amounts are never double-counted. A match marks the row
**Completed** and records the actual billed amount as its *Conversion
Value*; no match leaves it as **Leakage**, valued at the originally
prescribed Test Amount. Matching re-runs automatically after every
prescription entry, import, or billing upload, and can also be triggered
manually.

**Reports** — a **Monthly Summary** (Total Prescriptions, Total Tests Given,
Total Converted, Prescription/Conversion/Leakage Value, Conversion % and
Leakage %, with an all-time Total row) and a **Date-wise Detail** view of
every row and its status. **Export Excel** downloads both as a two-sheet
workbook.

**Bulk prescription import** — for historical data, same
download-template-then-import pattern as the masters, available from the
Prescriptions screen.

---

## Data model (Firestore)

```
users/{uid}                          name, email, role, companyId
companies/{companyId}                name, active, matchingConfig
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

## Adding a new prescription field later

The core prescription fields (Date, UHID, Test Given, etc.) are wired
directly into the matching engine, so they stay as-is. For anything
*beyond* those, there's a config array built for exactly this — add one
entry and it shows up everywhere: the New/Edit form, the Prescriptions and
Reports tables, the Excel template + bulk import, and the Excel export.

Find `CUSTOM_PRESCRIPTION_FIELDS` near the top of the script and add an
entry, for example:

```js
const CUSTOM_PRESCRIPTION_FIELDS = [
  { key: 'referralSource', label: 'Referral Source', type: 'select',
    options: ['Walk-in', 'Doctor Referral', 'Online'], required: false },
];
```

Supported `type`s: `'text'`, `'number'`, `'date'`, `'yesno'`, and `'select'`
(needs an `options` array). Each field needs a unique `key` (used
internally) and a `label` (shown to users and used as the Excel column
header). Existing prescriptions just won't have a value for a field added
after they were created — nothing breaks, the cell shows a dash until it's
filled in.

If you outgrow this (e.g. you want a field to pull its own master list, the
way Consultant pulls from Doctors), that's a bigger change to the form
logic rather than a config addition — happy to help with that when you get
there.

## Customizing

- **Currency symbol** — change `CURRENCY_SYMBOL` near the top of the script (defaults to `₹`).
- **Matching window** — each company sets its own under Settings (default: 1 day).
- **Ambiguous dates in uploads** — a date like `03/04/2026` is treated as **DD/MM/YYYY** (3 April) — the Indian convention. If your billing exports use US-style `MM/DD/YYYY`, export dates in `YYYY-MM-DD` format instead to avoid ambiguity, or adjust the `parseFlexibleDate` function.

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

- Matching re-reads all prescriptions and all billing rows for the company
  each time it runs — comfortable for the low tens of thousands of rows a
  clinic/diagnostics business would typically generate, but a large
  multi-year dataset would benefit from moving this step to a Cloud
  Function later.
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
