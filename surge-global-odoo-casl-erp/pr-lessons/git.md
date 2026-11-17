---
keywords: line-endings, crlf, diff, gitattributes, rewrite, merge, conflict, stale branch, revert
always: false
---

# PR lessons: git

- R2 ×1 Preserve original line endings so diffs show only real changes, not whole-file rewrites. (#368)
- R3 ×1 Never commit generated __pycache__ directories or .pyc files. (#368)
- R17 ×1 After merging, check shared files for stale-branch reverts of unrelated entries or documentation. (#414)
