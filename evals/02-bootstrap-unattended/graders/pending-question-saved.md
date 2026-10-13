---
type: regex
target: {source: file, path: .workflow/state.json}
match: contains
---
"pending_question":\s*\{
