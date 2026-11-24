# xcl-notes

A slow, resumable architecture analysis of [xcl-dev/xcl](https://github.com/xcl-dev/xcl), pinned to commit `c4d37be` (2026-11-24).

A scheduled task runs nightly at about 03:00 Oslo time. Each run reads the state in this repo, analyses one or two chunks of the code (about 4–10k lines), writes a note per chunk, commits and stops. When all 31 chunks are done, one more run writes `ARCHITECTURE.md` from the notes.

| File | What |
|---|---|
| `map.md` | Structure of the xcl repo and the ideas to keep in mind |
| `PROGRESS.md` | The chunk checklist and run log |
| `INDEX.md` | Short summary of each finished chunk |
| `notes/` | One note per chunk |
| `RUNBOOK.md` | Exact steps every run follows |
| `ARCHITECTURE.md` | The final synthesis (once the chunks are done) |

To check progress, look at `PROGRESS.md`. To pause the analysis, pause the scheduled task; nothing is lost, because the next run picks up from the first unticked chunk.
