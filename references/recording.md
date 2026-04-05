# Recording Demos

Use **asciinema** to record terminal sessions and **agg** to convert to GIF.

## Record

```bash
asciinema rec -i 2 --cols=120 --rows=35 demo.cast
```

| Flag | Purpose |
|------|---------|
| `-i, --idle-time-limit=<sec>` | Compress pauses longer than N seconds |
| `--cols=<n>` | Terminal width |
| `--rows=<n>` | Terminal height |
| `-c, --command=<cmd>` | Run a specific command instead of shell |
| `--overwrite` | Overwrite existing file |

## Convert to GIF

```bash
agg demo.cast demo.gif --font-size 14 --theme monokai
```

| Flag | Purpose |
|------|---------|
| `--font-size=<n>` | Font size |
| `--theme=<name>` | Color theme (`monokai`, `solarized-dark`, `dracula`, etc.) |
| `--speed=<factor>` | Playback speed multiplier (e.g. `2` for 2x) |
| `--fps=<n>` | Frames per second |

## Tips

- Always set `--cols` and `--rows` for consistent sizing across recordings
- Use `-i 2` to keep demos snappy — dead time is boring
- `clear` before the interesting part
- Keep recordings short (~30–60s) — GIFs get huge fast
