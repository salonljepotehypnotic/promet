# Hypnotic Salon — Promet App

## Project Overview

Single-file React app (`index.html`) for tracking revenue, expenses, inventory and staff bonuses for **Salon ljepote Hypnotic** (two locations: Savska and Dobojska).

Hosted on GitHub Pages: `https://salonljepotehypnotic.github.io/promet/`  
Repo: `salonljepotehypnotic/promet`, branch `main`, file `index.html`

## Tech Stack

- **React \+ Babel** (standalone, loaded from CDN — no build step, everything in one HTML file)  
- **Firebase Realtime Database** for persistence  
  URL: `https://salon-ljepote-hypnotic-63b86-default-rtdb.firebaseio.com/`  
  Path: `/hypnotic`  
- **localStorage** as a fast-loading cache (synced from Firebase on first load)  
- **SheetJS (XLSX)** for Excel export (loaded from CDN)

## File Structure

Everything is in one file: `index.html` (\~11.5 MB, \~5300 lines of JSX)

## React Components

| Component | Purpose |
| :---- | :---- |
| `App` | Root — login state, tab routing, Firebase real-time sync |
| `LoginScreen` | PIN-based login (per salon) |
| `NoviUnos` | New revenue entry form |
| `Pregled` | Entry list with filters (date, salon, payment type), edit & delete |
| `Statistike` | Revenue stats & charts per salon/staff/month |
| `Provjera` | Staff bonus verification — pivot table with per-staff totals |
| `Troskovi` | Expense management — add, edit, delete, copy fixed costs |
| `Klijentice` | Client list |
| `Skladiste` | Inventory management \+ service consumption norms |
| `Alati` | Equipment/tools tracker |
| `Postavke` | Settings (staff, services, salons, payment types, PINs) |

## Key Data Model (Firebase)

/hypnotic/

  entries\[\]       — revenue entries {id, date, salon, staff, service, price, quantity, discount, total, payment, clientName, note}

  expenses\[\]      — cost entries {id, date, salonId, category, subcategory, amount, note, dobavljac, type:'fiksni'|'varijabilni', createdAt}

  salons\[\]        — \[{id:'savska', name:'Savska'}, {id:'dobojska', name:'Dobojska'}\]

  staff\[\]         — staff members per salon

  services\[\]      — services with price per salon

  payments\[\]      — payment type list

  clients\[\]       — client records

  inventory\[\]     — stock items

  stock{}         — current stock quantities per salon

  consumptions{}  — service consumption norms (keys are service names, URL-encoded for Firebase)

  config{}        — app config

  prometPayments\[\] — payment types shown in revenue view

  tools\[\]         — equipment list

  toolAssignments\[\] — equipment assignments

  alatInventar\[\]  — tool inventory

  alatRadovi\[\]    — tool work log

## Critical Business Logic

### Payment types & double-counting avoidance

- `PAKET-GOTOVINA +` \= package **sold** → counts as salon revenue (not staff promet)  
- `IZ PAKETA` \= package **delivered** → counts as staff promet (NOT salon revenue — avoid double count)  
- `POKLON BON-odrađeno` \= gift voucher delivered → same as IZ PAKETA

### Statistike exclusions

const EXCLUDE\_FROM\_STATS \= new Set(\['IZ PAKETA', 'POKLON BON-odrađeno'\]);

Statistike excludes these two payment types from total revenue to avoid double-counting.

### Provjera grandTotals (must match Statistike)

const PROVJERA\_EXCLUDE \= new Set(\['IZ PAKETA', 'POKLON BON-odrađeno'\]);

Staff card totals in Provjera use the same exclusion set as Statistike.

### Bonus payments (fixed set for staff bonus calculation)

const bonusPayments \= new Set(\['AKONTACIJA','GOTOVINA','KARTICA','OSTALO-NE','IZ PAKETA','POKLON BON-odrađeno'\]);

### Payment normalization

`normPayment(str)` maps raw payment strings to canonical names (handles typos, case differences).

### Firebase key encoding

Firebase keys cannot contain `/`, `.`, `#`, `$`, `[`, `]`.  
Service names used as `consumptions` keys are encoded:

const encFbKey \= k \=\> String(k).replace(/\[.\#$\\/\\\[\\\]%\]/g, c \=\> '%' \+ c.charCodeAt(0).toString(16).padStart(2,'0').toUpperCase());

const decFbKey \= k \=\> String(k).replace(/%(\[0-9A-F\]{2})/gi, (m,h) \=\> String.fromCharCode(parseInt(h,16)));

Applied in `saveData()` (encode) and `mergeWithFirebase()` (decode).

## Security — CRITICAL

⚠️ **NEVER display passwords in the login UI** — not as hints, defaults, or descriptions.  
Passwords are: Savska=1234, Dobojska=5678, Admin=9999 — treat as confidential, never put in UI.

## Login / Auth

- PIN-based, hashed with a simple hash stored in localStorage key `hypnotic_pins`  
- Session stored in `sessionStorage` with expiry  
- Admin (`salonId === 'all'`) sees all salons

## Data Flow

1. App loads → `loadData()` reads from localStorage (fast initial render)  
2. Firebase `ref.on('value')` fires → `mergeWithFirebase(snap.val())` merges and calls `setData()`  
3. On save → `saveData(data)` writes to both localStorage and Firebase  
4. `onRefresh(newData)` \= `setData(newData)` updates local React state immediately

## Known patterns

- `uid()` — generates unique IDs for new records  
- `today()` — returns `YYYY-MM-DD` string  
- `fmt(n)` — formats numbers as Croatian locale currency string  
- `fmtDate(d)` — formats `YYYY-MM-DD` to `DD.MM.YYYY`  
- `normSt(name)` — normalizes staff names to UPPERCASE for grouping  
- `toFbObj(arr)` — converts array to Firebase object keyed by `.id`  
- `fbObjToArr(obj)` — converts Firebase object back to array

## Recent changes (for context)

- Fixed double-counting in Statistike (exclude IZ PAKETA \+ POKLON BON-odrađeno)  
- Fixed Provjera per-salon UKUPNO cards to match Statistike totals  
- Added searchable service dropdown in Pregled (edit mode)  
- Added "+ Trošak" buttons in Provjera for adding staff bonuses directly to expenses  
- Fixed expense date bug (addExpense from Provjera now uses end-of-selected-period date)  
- Added edit (pencil) button for expenses in Troškovi pregled  
- Added autocomplete (datalist) for Dobavljač, Opis, Napomena in Troškovi  
- Added payment type filter in Pregled  
- Added "copy fixed costs from any month" picker in Troškovi (instead of always previous month)  
- Fixed Firebase key encoding for service names with slashes in consumptions  
- Fixed normativ (service consumption norms) save for service names containing `/`

