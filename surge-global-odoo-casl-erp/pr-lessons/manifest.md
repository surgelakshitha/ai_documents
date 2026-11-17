---
keywords: manifest, assets, static, missing file, bundle, comment, load order, data files, depends, references, module
always: false
---

# PR lessons: manifest

- R15 ×2 Declare only static assets that exist in the same change; missing files break the asset bundle. (#386, #414)
- R14 ×1 Keep manifest ordering comments consistent with the actual load order of the listed files. (#386)
- R16 ×1 List every dependency and data file the module's code or XML references in the manifest. (#414)
