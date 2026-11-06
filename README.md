# Web Animation Studio

A browser-based animation editor for 2D, 2.5D and 3D composition workflows.

## Features

- Scene mode switcher: 2D, 2.5D, 3D
- Layer management and inspector controls
- Timeline with frame navigation
- Play / pause playback and speed control
- Keyframe saving per layer
- Real-time canvas preview

## Getting started

```bash
npm install
npm run dev
```

Then open http://localhost:3000

## Project structure

- `src/App.tsx` — main studio editor UI
- `src/main.tsx` — app bootstrap
- `src/index.css` — base styling
- `index.html` — Vite HTML entry point

## Stack

- React
- TypeScript
- Vite

## Next roadmap

- Sprite import and timeline drag editing
- Export to GIF / MP4 / sprite sheet
- Multi-project state and local persistence
- True 3D camera and lighting system
