# tui/

Terminal UI frontend. Visitors interact with a terminal-style shell in the browser.

## Stack (TBD)

Options:
- **xterm.js** — full terminal emulator, authentic feel
- **Custom React shell** — lighter weight, more design control

## Integration

- Sends commands (`resume`, `projects`, `ask <q>`) to `app/api/`
- Receives streamed responses (character-by-character for `ask`)
- Auto-types welcome message on load
- See [docs/tui-commands.md](../docs/tui-commands.md) for the full command set and design.
