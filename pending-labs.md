# Pending labs

Labs that were read, analysed or started and then set aside. **Remove an entry from this list when its file lands in `labs/`.**

Anything not on this list either has a file in `labs/` or has never been looked at.

## The list

| Lab id | Name | State |
|---|---|---|
| GSP1210 | Multimodality with Gemini | analysed, never started |

An entry carries what is already known about the lab: the split between shell and console, the values that have to be read off the started page, the order that matters, and anything that failed before. The point of an entry is that a future attempt starts ahead rather than level.

## What is already known about each

### GSP1210

**A guided lab, not a challenge lab**, despite arriving alongside them. No `TODO` or `INSERT` blanks anywhere; every task says *"run through the X section of the notebook"*. Entirely JupyterLab, zero Cloud Shell.

- **Nine checkpoints**, one per notebook section: multi image understanding, video description, audio understanding, reason across a codebase, video and audio, all modalities at once, recommendations from images, ER diagrams, image comparison.
- **Twenty five minutes stated, expect thirty to forty**, because multimodal inference over video and audio is slow and there are nine sections of it.
- **The constraint is rate limiting, not difficulty.** Do **not** Run All. Run one section, read its output, click its checkpoint, move on, which is how the lab is structured anyway. A 429 means wait a minute and rerun; rerunning immediately makes it worse.
- **The checkpoints almost certainly verify the api call happened**, as on GSP515 and GSP517, so a cell that errored and got skipped fails its checkpoint even though the notebook looks finished.
- Check the Getting Started cell's location variable is not `global` before running nine sections against it, per the note in `gotchas.md`.
- **Lab text bug:** the "Reason across a codebase" section is described as *"Gemini can directly process audio for long-context understanding"*, copied from the Audio understanding section above it. Read the heading, not the description.

This entry was lost once. It was removed from the queue on 2026-08-18 while adding other labs, without either a file in `labs/` or a line under Not pending, and was recovered from the session transcript on 2026-08-20. **An entry comes off this list for exactly two reasons: its file lands in `labs/`, or it moves to Not pending.**

## Not pending

- **GSP364**, Managed Service for Prometheus: Challenge Lab, already has a file at `labs/gsp364-managed-prometheus-challenge.md` and will not be repeated. Check the README index before starting anything that feels familiar; it caught that one before a lab attempt was spent.
