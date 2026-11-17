---
keywords: migration, version, manifest, upgrade, odoo, version folder, duplicate, backfill, folder
always: false
---

# PR lessons: migrations

- R1 ×1 Place migration scripts in a folder named for the shipping manifest version, not the installed one. (#368)
- R26 ×1 Delete superseded migration folders when copying a script to a newer version; never ship two copies of one backfill. (#368)
