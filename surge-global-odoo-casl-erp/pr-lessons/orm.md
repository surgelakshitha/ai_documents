---
keywords: sequence, batch, create, vals_list, collision, copy, duplicate, state, guard, one2many, domain, field
always: false
---

# PR lessons: orm

- R4 ×1 Seed batch-generated sequences from explicit values in the same vals_list, not only existing database rows. (#368)
- R6 ×1 Set copy=False on guarded state fields so duplicates restart at the initial status instead of failing. (#368)
- R7 ×1 Filter displayed One2many rows in the field definition domain; a view domain only affects record selection. (#386)
- R9 ×1 Keep a stat button's count and the domain of the action it opens counting the same records. (#386)
- R10 ×1 Restrict aggregate counts to the rows the metric describes and reuse that filtered count for derived ratios. (#386)
- R11 ×1 Give duplicated compute logic the same edge-case handling and @api.depends coverage as the original. (#386)
- R13 ×1 Return a _search subquery in search-field methods instead of loading records into a literal id list. (#386)
- R20 ×1 Make search-then-create under a unique constraint race-safe with a savepoint and re-search on violation. (#414)
- R24 ×1 Enforce state-transition checks inside the model method so every caller path applies them. (#414)
- R25 ×1 In create guards, overwrite guarded fields in vals_list; key checks alone are bypassed by default_ context. (#368)
