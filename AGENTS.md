# Agent notes

LrGeniusTagAI is an Adobe Lightroom Classic plugin (Lua) that sends a temporary JPEG of selected photos to a vision model (Gemini, ChatGPT, Ollama, or LM Studio) and writes title, caption, alt text, and keywords back to the catalog. See [README.md](README.md).

## Docs

- [Lua 5.1 reference manual](https://www.lua.org/manual/5.1/manual.html) — Lightroom’s Lua version
- [Programming in Lua](https://www.lua.org/pil/) (2nd ed. matches 5.1)
- [SDK API summary](docs/LrC_15.3_202604090947-8f3672ed.release_SDK/API%20Reference/SUMMARY.md) — scan this first for Lightroom SDK functions; each entry links to the full HTML docs
- [Lightroom Classic SDK](https://developer.adobe.com/lightroom-classic)

Stay on Lua 5.1: no `goto`, bitwise ops, `//`, or later standard libraries. Lightroom does not support Lua 5.1 `module()`.

## Layout

Plugin code lives in `LrGeniusTagAI.lrdevplugin/`.

| File | Role |
|------|------|
| `Info.lua` | Plugin manifest (SDK 11+, menus, metadata) |
| `Init.lua` | SDK `import`s onto `_G`, `require`s modules, **pref defaults** (`if prefs.x == nil`) |
| `AnalyzeImageTask.lua` | Library/export entry: export JPEG, call AI, save metadata |
| `AnalyzeImageProvider.lua` | Preflight, validation, and photo-context dialogs |
| `PluginInfo.lua` / `PluginInfoDialogSections.lua` | Plug-in Manager settings UI (`startDialog` / `endDialog` load/save prefs) |
| `AiModelAPI.lua` | Dispatches to provider APIs; builds the base task/system prompt |
| `GeminiAPI.lua`, `ChatGptAPI.lua`, `OllamaAPI.lua`, `LmStudioAPI.lua` | Provider HTTP + `analyzeImage` |
| `Defaults.lua` | Models, URLs, pricing, default **prompts** — not submit-metadata prefs |
| `TranslatedStrings_*.txt` | Localization. `de` and `fr` load for those UI languages; `en` is not loaded |

Do not edit vendored `JSON.lua` or `inspect.lua`.

## User-facing prefs

User options live as plugin prefs, not in `Defaults.lua`. A new checkbox typically needs **all** of:

1. Nil-guard default in `Init.lua`. Existing installs pick up that default on next load (`nil` means “never set”, not “new users only”).
2. Load + widget + save on the right UI surface. Missing a surface means the setting silently desyncs.
   - Per-run options (generate*, submit*, validation, stacks, model, …): **both** `AnalyzeImageProvider.showPreflightDialog` and `PluginInfoDialogSections` (`startDialog` / UI / `endDialog`).
   - Install/config only (API keys, export JPEG size/quality, keyword hierarchy, logging): Plug-in Manager only. Do **not** add these to preflight.
3. The same `LOC` key in both dialogs that show it, plus the same key in all three `TranslatedStrings_*.txt` files. Keywords currently use two different keys; do not copy that.
4. Behavior in `AnalyzeImageTask` and/or each `*API.lua` `analyzeImage`. Options that do not change the prompt (e.g. stacks) belong in `AnalyzeImageTask` only, not in the four provider files.
5. README bullets if the option is user-facing.

`PluginInfoDialogSections.startDialog` / `endDialog` are not grouped by feature — later prefs (e.g. folder names) sit far below GPS/keywords. Search the pref name rather than inserting next to the first `submit*` you see.

Do **not** extract a shared “append metadata to task” helper. Each provider copies the GPS/keywords/folder/collection blocks. Prompt fragments for that metadata are hardcoded English even when unused `*Addon` LOC strings exist.

`LrView` rows do not wrap. A fourth checkbox on the submit-metadata row will overflow in de/fr; put extras on a continuation row with `f:spacer { width = share 'labelWidth' }` (see `showPreflightDialog`).

Cheap catalog reads (GPS, keyword tags) are always copied into the `metadata` table and gated only when appending to the task. Gate **expensive** SDK walks (collections, tree scans) on the pref in `AnalyzeImageTask` so unchecked options do not pay the cost per photo.

Collection membership: `photo:getContainedCollections()` is standard library collections only (no smart collections, no published collections) and must run inside `LrTasks` (the analyze path already does). Build set paths with `collection:getParent()` / `getName()`.

Debug: `log:trace` is the reliable check. Gemini/ChatGPT dump the request body via `Util.dumpTable`; Ollama often does not. Reload the plug-in in Lightroom after Lua edits — there are no automated tests.

## Catalog reads and writes

- IPTC/keywords (`setRawMetadata`, `createKeyword`, `addKeyword`) use `withWriteAccessDo`. Plugin fields (`setPropertyForPlugin`: `aiLastRun`, `aiModel`, `photoContext`) use `withPrivateWriteAccessDo` — a catalog lock around a **per-photo** write.
- Folder stacks are `getRawMetadata` keys (`isInStackInFolder`, `stackInFolderMembers`, `topOfStackInFolderContainingPhoto`), not `LrPhoto` methods. Collection stacks are not in the SDK. Must run inside `LrTasks`.
- `catalog:getTargetPhotos()` is the visible filmstrip selection; a collapsed stack typically yields only the top photo.

## Localization (`LOC`)

`LOC "$$$/lrc-ai-assistant/Path/Key=English default"` looks up the key only in the `TranslatedStrings` file for the current Lightroom UI language.

- English is the text after `=` in the Lua call. `TranslatedStrings_en.txt` is not read. Change English copy there, in the `LOC` call.
- `TranslatedStrings_de.txt` and `TranslatedStrings_fr.txt` apply only when the UI language is German or French.
- Any other language uses that same inline default. It does not fall back to `TranslatedStrings_en.txt`.

Still add the same key to all three `TranslatedStrings_*.txt` files.

## Style

Match existing files; do not reformat unrelated code.

- 4-space indent. Mix of `'` and `"` is fine.
- Modules are PascalCase tables (`Util`, `GeminiAPI`). Load with `require "ModuleName"`.
- Instance APIs: `Module.__index = Module` and `function Module:new()` via `setmetatable`.
- Helpers as `function Module.foo()`; instance methods as `function Module:bar()`.
- Globals from `Init.lua`: `prefs`, `log`, `JSON`, plus `Lr*` SDK namespaces.
- User-facing strings: `LOC` with the English default in the call. See [Localization (`LOC`)](#localization-loc).
- Catalog mutations only inside `catalog:withWriteAccessDo` / `withPrivateWriteAccessDo`.
- Background work: `LrTasks.startAsyncTask` + `LrFunctionContext.callWithContext`. HTTP: `LrHttp`. Errors: `ErrorHandler.handleError`. Log: `log:trace` / `log:error`.
- New AI providers: implement `:new()` and `:analyzeImage(filePath, metadata)` like the existing `*API.lua` files, then register in `AiModelAPI` and `Defaults`.
