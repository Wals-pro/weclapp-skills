---
name: weclapp-csv-import
description: "Use for building weclapp CSV import files — Massenimport / Datenübernahme of articles, customers, suppliers, contacts, prices and 30+ other data types via the Import/Export-Wizard. Covers template column specs, file format rules, import order, and pre-upload validation. Not for single-record changes through the API (master-data skill) and not for auditing existing records (data-quality skill)."
version: 0.2.5
---

# Build weclapp CSV import files

weclapp's Import/Export-Wizard is the bulk data channel: articles, customers, suppliers, contacts, prices — up to 49 999 rows per file. Its contract is exact: column names, order, formats, and mandatory fields come from official per-type templates. `get_schema` with `detail="import_guide"` serves those template specs so nobody has to download templates from the wizard by hand. The assistant builds the file; **the user uploads it** in weclapp under Globale Einstellungen → Import/Export-Wizard — there is no upload API.

## Workflow

1. **Pick the import type.** `get_schema(detail="import_guide")` without an entity lists all ~35 types with entity mapping and the global file rules. Respect dependencies between imports (also surfaced in the catalog): master data before documents that reference it — e.g. Warengruppen before Artikeln, Artikel before Artikelpreisen and Bezugsquellen, Kunden/Lieferanten before Rechnungen.
2. **Fetch the spec.** `get_schema(entity=<type_key>, detail="import_guide")` returns the exact column list (order matters), per-column required/format/allowed values on enriched types, wizard options, encodings, and sample rows from the official demo file.
3. **Map and clean the source data.** Every target column gets a value or stays empty — never delete unused columns, never rename headers, never reorder. Values referencing existing records (Artikelnummern, Kundennummern, Einheiten, Steuersätze, Währungen, Ländernamen) must exist in weclapp before the import.
4. **Build the file by the rules.** Semicolon separator, header row exactly as specified, German decimal comma, dates as `dd.MM.yyyy`, booleans `ja`/`nein`, one file ≤ 49 999 rows. Save as UTF-8 and pick the matching Kodierung in the wizard (ISO-8859-1 / UTF-8 / windows-1252).
5. **Validate before handing over.** Re-check against the spec: header row identical to the column list, mandatory columns filled in every row, enum values verbatim, date/decimal formats, no stray separators inside unquoted text.
6. **Hand over for upload — small test first.** Deliver the file and instruct the user: upload a 5–10 row test slice first, review the wizard's Importmeldungen (downloadable per run), then run the full file. For update-style imports, recommend exporting the current data first as a backup.

## Boundaries

- The import itself is a user action in the weclapp UI; never claim the assistant imported data.
- Updating existing records via import can overwrite fields — flag that risk and prefer the test slice whenever identifiers already exist in the system.
- A few types (Zusatzfelder, Übersetzungen) use tenant-generated templates: the wizard builds the demo file from the tenant's own configuration, so the spec's sample columns are structural examples, not the tenant's exact file.

## Scope

Bulk file-based imports through the Import/Export-Wizard. Creating or updating single records through the API belongs to the master-data and record-updates skills; auditing existing data quality belongs to the data-quality skill.
