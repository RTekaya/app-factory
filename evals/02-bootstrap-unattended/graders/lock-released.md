---
type: regex
target: {source: file, path: .workflow/state.json}
match: contains
---
"lock":\s*null
