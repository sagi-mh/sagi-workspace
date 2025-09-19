## Spell check in Cursor/VS Code (on-demand)

### What we set up
- Disabled automatic spell check for this workspace.
- You run spell checks only when you choose (via commands or optional user keybindings).

### Workspace settings
```json
{
	"cSpell.enabled": false,
	"editor.tabCompletion": "on",
	"editor.snippetSuggestions": "top"
}
```
File: `.vscode/settings.json`

### Run spell check on demand (no shortcuts)
- Cmd+Shift+P → "CSpell: Check Document"
- Cmd+Shift+P → "CSpell: Check Workspace"
- Optional: Cmd+Shift+P → "CSpell: Toggle Enable Spell Checker" (turn on, review, then toggle off)
- Review results: Cmd+Shift+M opens Problems; Cmd+. on a word for quick fixes or Add to dictionary

### Optional user keybindings (cannot be workspace-scoped)
Add to your user keybindings and scope to Markdown only:
```json
[
	{ "key": "cmd+alt+s", "command": "cSpell.toggleEnableSpellChecker", "when": "editorLangId == 'markdown'" },
	{ "key": "cmd+alt+c", "command": "cSpell.checkDocument", "when": "editorLangId == 'markdown'" }
]
```
Open with: Cmd+Shift+P → "Preferences: Open Keyboard Shortcuts (JSON)".

### Troubleshooting
- Install/enable "Code Spell Checker" (Street Side Software) → Cmd+Shift+X
- Reload window: Cmd+Shift+P → "Developer: Reload Window"
- Confirm language mode is "Markdown" (status bar)
- If commands don’t show, type "CSpell:" in the Command Palette

### Related: date snippets for journaling
- File: `.vscode/markdown.json.code-snippets`
- Added `"scope": "markdown"` so snippets trigger in Markdown
- If Tab doesn’t expand snippets in Markdown, Emmet may be capturing Tab. You can disable it just for Markdown:
```json
{
	"[markdown]": { "emmet.triggerExpansionOnTab": false }
}
```
