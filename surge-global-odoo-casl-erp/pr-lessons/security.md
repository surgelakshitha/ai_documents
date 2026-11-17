---
keywords: groups, implied, security, comment, access, csv, export, formula, injection, spreadsheet, upload, attachment
always: false
---

# PR lessons: security

- R8 ×1 Make group records match their documented behavior, e.g. define implied_ids when comments claim an implication. (#386)
- R12 ×1 Prefix CSV cells beginning with =, +, - or @ so spreadsheets don't execute them as formulas. (#386)
- R19 ×1 Cap upload size and count, verify file type by magic bytes, and generate attachment names server-side. (#414)
- R22 ×1 Stamp audit fields server-side in create; never accept caller-supplied attribution through field defaults. (#414)
- R23 ×1 Grant model ACLs matching intended visibility; hidden menus and tabs don't block RPC or content access. (#414)
- R27 ×1 Keep docstrings and header comments accurate about actual permissions and behavior; update them whenever the implementation changes. (#414)
