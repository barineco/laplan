# VS Code Extension and WASM

`extension/` is the VS Code extension for `.lex`. Diagnostics, semantic tokens, and inlay hints come from WASM, while the Petri net UI mounts the Vue 3 build output from `@laplan/ui-render` inside the webview.

## Build

```bash
cd extension
npm run build:wasm
npm run compile
npm run package
```

- `build:wasm`: refreshes the node/web targets from `@laplan/core`
- `compile`: runs `tsc -p .` and then uses `copy-webview.mjs` to copy `packages/ui-render/dist/` into `out/webview/`
- `package`: emits the `.vsix`

## Webview Layout

- `src/graph/petriNetPanel.ts`: reads `out/webview/index.html`, rewrites asset URIs, and injects CSP nonce values before creating the `WebviewPanel`
- `src/graph/webviewHtml.ts`: injects nonce attributes into Vite output `script`, `stylesheet`, and inline `style` tags
- `packages/ui-render/src/webviewApp.vue`: receives host messages and mounts `<PetriNetEditor>`
- `packages/ui-render/src/webviewBridge.ts`: converts between graph data and `meta "view.graph.*"` payloads

`packages/ui-render/vite.config.ts` sets `build.modulePreload = false`, which keeps the CSP rewrite limited to `script`, `stylesheet`, and inline `style`.

## Host and Webview Messages

Host to webview:

- `updateGraph`: sends the latest `PetriNetGraph` plus base sources
- `updateMeta`: sends decoded `view/graph/*.lex` meta view files
- `externalChange`: notifies the UI that the workspace changed outside the webview

Webview to host:

- `openFile`: opens the matching `.lex`
- `applyLexEdit`: applies rule stub append/remove operations through `WorkspaceEdit`
- `applyMetaEdit`: creates or replaces `view/graph/<nsid>.lex` and saves it

Saving `.lex` reruns diagnostics. Saving `view/graph/*.lex` reloads meta views.

## Package Artifact

`npm run package` uses `vsce package --allow-missing-repository --no-dependencies`. This works in the monorepo with workspace links, and the measured artifact size is `248.08 KB` for `laplan-lex-0.2.0.vsix`.

## Manual Regression

1. `cd extension && npm run package`
2. `code --install-extension laplan-lex-0.2.0.vsix`
3. Open a `.lex` file and confirm syntax highlighting, diagnostics, semantic tokens, and inlay hints
4. Run `Laplan: Show Petri Net` and confirm the Vue webview appears
5. Move a node and confirm `view/graph/<nsid>.lex` updates
6. Add and remove wiring or missing-fact stubs and confirm text edits reach the base `.lex`
