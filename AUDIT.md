# Step 1 — Hardcoding audit (what's specific to your agency)

Goal of this file: list every value in `agency-tracker.jsx` that is specific to *your*
chatting agency, so the config layer (Step 2+) has a complete checklist. Line numbers are
approximate and will drift as we edit — search by the quoted text.

Two important distinctions used throughout:
- **Display string** = text a user reads. These get swapped for config-driven labels.
- **Internal identifier** = a variable/object key in code or saved data (e.g. `chatterCut`,
  `clientId`). These should usually KEEP their names even after relabeling — renaming them
  means a data migration. The plan: config maps a *label* onto an existing internal key.

---

## 1. Business identity / branding
| What | Where | Current value | Becomes |
|---|---|---|---|
| App/brand name | lines ~496, 579, 1239, 1263 | "Fanlink Chatting" / "fanlink" | `config.business.name` |
| Tagline | lines ~1240, 1264 | "Chatting Agency" | `config.business.tagline` |
| Logo image | `LOGO` const (~39); used ~491, 578, 1257 | `/logo.svg` | `config.business.logo` (URL or data-URI) |
| Logo letter in loader/header tile | ~1237 | hardcoded "f" | derive from `config.business.name[0]` |
| Accent / brand color | `THEME` object (~57) | mint `#5EEAD4` | `config.branding.accent` (single source) |
| Section emoji icons | ~1319 📌, ~1439 💬 | 💬 is chatting-specific | neutral or `config`-driven icon |

## 2. Locale / currency / tax (mostly India + USD mix)
| What | Where | Current value | Becomes |
|---|---|---|---|
| Currency formatter | `fmt` (~11) | `Intl.NumberFormat("en-US", USD)` | driven by `config.locale.currency` + `locale` |
| Amount-in-words | `toWords` (~41) | short-scale words, hardcoded "Dollars"/"Cents" | locale/currency-aware, or optional toggle |
| Currency symbol in Smart-Paste UI | ~2003 | literal `$` prefix | `config.locale.currencySymbol` |
| Smart-Paste parser currency regex | ~879–880 | accepts `$ £ € ¥ rs usd eur gbp inr` | fine to keep; ensure display uses config currency |

## 3. Invoice (heavily India / your-business specific)
| What | Where | Current value | Becomes |
|---|---|---|---|
| Sender address block | ~580–584 | Ghansoli / Navi Mumbai / Vashi 400703 / Maharashtra MH / India | `config.business.address[]` (free lines) |
| Tax/supply line | ~588 | "Place of supply: Maharashtra" | `config.invoice.taxLine` (optional, locale-based) |
| Invoice number format | ~556 | `"INV/25-26/" + id` (fiscal year "25-26" hardcoded!) | `config.invoice.numberFormat` w/ tokens |
| Invoice title | ~594 | "Customer Invoices" | `config.invoice.title` |
| Line-item label | ~617 | "Agency Fees - {client}" | `config.invoice.lineItemLabel` |
| Notes | ~650 | "Please make the payment within 7 days." | `config.invoice.notes` |
| Signatory | ~654 | "Authorized Signatory" | `config.invoice.signatory` |
| Static labels | ~597,601,629,637 | Invoice Date / Due Date / Untaxed / Amount Due | keep generic; tax row only if tax configured |

NOTE: invoice number "25-26" is a frozen fiscal year — it will be wrong next year even for
you. The number format needs date tokens (e.g. `INV/{FY}/{SEQ}`).

## 4. Terminology (the vocabulary to generalize)
Replace these **display strings** only. The internal keys stay.

| Display string | Where (examples) | Becomes (config term) |
|---|---|---|
| "Chatter" / "Chatters" | tab data, headers, modals, labels (~447, 996, 1604, 1632, 1807…) | `config.terms.staff` (singular/plural) |
| "Client" / "Clients" | tabs, headings, modals (~447, 1317, 1604, 1794…) | `config.terms.client` |
| "Sale" / "Sales" / "Add Sales" | tab name, "Total Sales", "record sales" (~447, 1308, 1403…) | `config.terms.revenue` (e.g. Sale/Project/Retainer) |
| "Agency Cut" | labels (~996, 1802, 1844) | `config.terms.agencyShareLabel` |
| "Chatter Pay" | labels (~996, 1052, 1096, 1807, 1849) | `config.terms.staffShareLabel` |
| "Clients & Chatters" (heading) | ~1604 | composed from terms |
| "By Client" / "Analytics" | ~1283, 1317 | composed from terms |
| Tab names array | `TabBar` (~447) | composed from terms |

Do NOT rename (internal — would need migration): `chatterId`, `clientId`, `agencyCut`,
`chatterCut`, `chatterStats`, `chatterNameFn`, `salesClientId`, etc. (~261 "chatter"
mentions are mostly these.)

## 5. Commission / payout model (the core of the product)
| What | Where | Current value | Becomes |
|---|---|---|---|
| Default agency rate | `AGENCY_CUT` (~37) | `0.075` (7.5%) | `config.commission.defaults` |
| Default staff rate | `CHATTER_CUT` (~38) | `0.125` (12.5%) | `config.commission.defaults` |
| Split math | record creation ~860, edit ~767, invoice ~558 | `amount * (agencyCut + chatterCut)` — % only | pluggable engine: %, flat fee, tiered, … (Step 8) |
| Stored per-client rates | client objects `agencyCut`/`chatterCut` | percentages only | engine config per client/staff |

The whole app assumes a percentage-of-revenue split. That assumption is baked into record
creation, editing, dashboards, invoices, and share cards. Generalizing it (Step 8) is the
biggest single change and should be designed around the first target agency type.

## 6. Storage / app keys (rename carefully — touches saved data)
| What | Where | Current value | Notes |
|---|---|---|---|
| localStorage key | `STORAGE_KEY` (~3) | `"fanlink-tracker-v4"` | rename → write a migration that reads old key |
| Backup envelope | exportBackup (~1006), filename (~1009) | `_app: "fanlink-tracker"`, `fanlink-backup-*.json` | generic name; keep reading old `_app` on import |

## 7. Data model (today's shape — for reference when designing config)
- `data = { clients[], chatters[], records[] }` (+ we'll add `config`)
- client: `{ id, name, agencyCut, chatterCut }`
- chatter (staff): `{ id, name, clientId }`
- record: `{ id, chatterId, amount, date, agencyCut, chatterCut, invoiceNo? }`

When we add config, fold it into this object and into the JSON backup so a full setup stays
portable (and so it's ready to lift into a backend later).

---

## Suggested config schema (preview — built in Step 2)
```
config = {
  business:   { name, tagline, logo, address: [..lines..], country },
  locale:     { currency: "USD", currencySymbol: "$", taxLabel, taxRate, amountInWords },
  terms:      { client:{one,many}, staff:{one,many}, revenue:{one,many},
                agencyShareLabel, staffShareLabel },
  branding:   { accent },
  invoice:    { title, numberFormat, lineItemLabel, notes, signatory, taxLine, dueDays },
  commission: { model: "percent", defaults: { agencyShare, staffShare } },  // engine grows in Step 8
}
```

## Risk flags
- Fiscal year `"25-26"` frozen in the invoice number — bug even for you next year.
- `toWords` says "Dollars" while real intent may vary — make currency-aware or optional.
- Renaming `STORAGE_KEY` / data field keys = migration; prefer keeping keys, mapping labels.
- The percentage-only split assumption is everywhere — plan Step 8 before promising
  "works for any agency."
