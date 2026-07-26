# Localization

The project-wide MSW `LocaleDataSet` lives at:

```text
RootDesk/MyDesk/Data/Localization/
├─ GameText.localedataset
└─ GameText.csv
```

`GameText.csv` is the translation source of truth. Its fixed leading columns are
`Key`, `Source`, and `Note`; locale columns follow them.

```csv
Key,Source,Note,zh-TW,en
```

## Editing

- Preferred: open `GameText` in the Maker DataSet editor and edit it as a table.
- External editor: save `GameText.csv` as UTF-8 CSV without renaming it.
- Keep `GameText.localedataset` and `GameText.csv` together with the same base
  name.
- After an external CSV edit: stop Play Test, refresh the Maker workspace, then
  start Play Test again.
- Maker table edits do not require a separate import/export cycle.

## Runtime contract

Localization lookup is client-only:

```lua
local text = _LocalizationService:GetText("ui_example")
local formatted = _LocalizationService:GetTextFormat("message_example", value)
```

Server code sends localization keys and arguments to the client instead of
resolving translated strings.

## Current migration scope

Only the LocaleDataSet infrastructure is installed. Existing UI `Text` values
and Config CSV localization-key columns are intentionally unchanged.
