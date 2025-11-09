# Real-Time Collaborative Drawing Canvas

A multi-user drawing application where multiple people can draw simultaneously on the same canvas with real-time synchronization.

## ▶️ Quick Start

```bash
npm install
npm start
```
Open `http://localhost:3000` in two or more browser windows (or devices) to test.

### Multiple Users
- Open the app in separate tabs or browsers. Each client is assigned a color and user id on connect.
- Optional: add `?room=demo` to the URL to test isolated rooms, e.g. `http://localhost:3000/?room=team-a`

## ✨ Features

- Brush & Eraser (size slider + color picker)
- Real-time streaming of strokes (point-by-point)
- Live cursors with user colors
- Global undo/redo (affects the latest operation regardless of author)
- Online users list with assigned colors
- Basic rooms support via `?room=<name>`
- Resilient to reconnects (server is the source of truth; clients resync history)

## 🧪 How to Test

1. Open two or more tabs to `http://localhost:3000`.
2. Draw on one tab; observe the other tabs update in real time.
3. Try the eraser, change brush size/color.
4. Use **Ctrl+Z** (Undo) or toolbar button; watch all clients update.
5. Use **Ctrl+Shift+Z** (Redo) or toolbar button.
6. Toggle high activity by drawing quickly; the app batches points efficiently.

## 🧰 Scripts

- `npm start` – runs the Express + Socket.IO server and serves the static client.

## 🧩 Known Limitations

- Canvas is re-rendered from the global history on undo/redo and on resync; for very large histories this can be slow. (See ARCHITECTURE.md for incremental strategies.)
- No persistence layer by default (in-memory only). Optional TODO: plug in Redis or a DB.
- Mobile touch support is basic but present; advanced gestures are not implemented.
- Security/auth is intentionally omitted for scope reasons.

## ⏱️ Time Spent

~6–8 hours for core features + docs.

## 🛠️ Stack

- Frontend: Vanilla HTML/CSS/JS + `<canvas>`
- Backend: Node.js, Express, Socket.IO
