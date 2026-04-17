# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run compile       # Compile TypeScript → /out
npm run watch         # Compile in watch mode
npm run lint          # ESLint on all .ts/.tsx files
```

Tests run inside a VS Code Extension Development Host — there is no standalone `npm test`. To run tests, press **F5** in VS Code, which launches the host and runs the Mocha suite via `src/test/runTest.ts`.

To publish: `npm run vscode:prepublish` (runs compile).

## Architecture

This is a VS Code extension that auto-converts quoted strings to template literals when the user types `${`.

**Two parallel implementations:**

- `src/extension.ts` — Desktop (Node.js). Full feature set including `autoRemoveTemplateString` (tracks previous document state to revert backticks when `${` is deleted).
- `src/web/extension.ts` — Web/browser (WebWorker). Same core logic but without `autoRemoveTemplateString` and optimized for browser constraints. Bundled via `webpack.config.js`.

Both entry points are declared in `package.json`: `main` for desktop, `browser` for web.

**Core event loop (both implementations):**

1. `onDidChangeTextDocument` fires on every keystroke.
2. Configuration is read fresh each time (enabled, quoteType, validLanguages, filesExcluded).
3. The language ID is checked against `validLanguages`; excluded file patterns are checked.
4. For each selection, the current line is scanned to find the enclosing quote characters.
5. A `WorkspaceEdit` replaces the opening and closing quotes with backticks.

**Language activation (package.json):**

Languages must appear in **both** `activationEvents` (so VS Code loads the extension) and the `validLanguages` default array (so the runtime check passes). Svelte is in `validLanguages` defaults but not in `activationEvents` — a known inconsistency.

## Tests

Tests live in `src/test/suite/extension.test.ts` (Mocha TDD style). The key helper is `withRandomFileEditor` in `testUtils.ts`, which creates a temp file of a given extension, opens it in the editor, runs the test callback, then cleans up.

Test pattern:
```typescript
withRandomFileEditor(initialContent, 'ts', async (editor, doc) => {
  await editor.insertSnippet(new vscode.SnippetString('${'), position);
  await delay(500); // wait for extension to process
  assert.equal(doc.getText(), expectedContent);
});
```

The compiled test file (`src/test/suite/extension.test.js`) is what Mocha actually runs — recompile after changes.
