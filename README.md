# NeuroDex

A thalamocortical dysrhythmia (TCD) phenotyping pilot, run as NeuroSky
neurofeedback training with a collection-game layer on top (Ions, chests,
a 51-specimen Dex).

The core idea: certain thalamic circuits may get stuck bursting in slow
theta instead of their normal fast rhythm, and that slow rhythm leaks into
whichever cortical area the circuit feeds — auditory cortex for tinnitus,
somatosensory cortex for pain, prefrontal/limbic circuits for depression.
The app reads that signature (a theta-vs-beta+gamma ratio, and separately
SEF95) off a NeuroSky headset, scores it against your own resting
baseline, and trains against it. See the in-app Research tab for the full
pilot design.

## What's in this repo

```
index.html                  the whole frontend — single file, hash-routed, no build step
api/contact.js               Vercel Node function backing the contact form
api/analyze-artifacts.py     Vercel Python function running an MNE artifact audit
api/requirements.txt         Python deps for the MNE function (kept in api/, not the project root — see note below)
vercel.json                  explicit builds/routes config — bypasses Vercel's framework auto-detection entirely
package.json                 present for local tooling only; not required by the builds/routes config below
```

> **Why `vercel.json` uses `builds`/`routes` instead of zero-config:** this project hit repeated
> Vercel auto-detection issues — the platform kept guessing "this whole project is a Python app"
> because of `requirements.txt`, and separately failed to match a `functions` glob against the
> `api` directory. Both are symptoms of relying on Vercel's automatic framework/function detection,
> which reads signals (root files, dashboard Framework Preset, etc.) that are easy to get out of
> sync with what's actually in the repo. The `builds` + `routes` format is the older, fully
> explicit config: it names every source file, which builder handles it (`@vercel/static` for
> `index.html`, `@vercel/python` for the MNE function, `@vercel/node` for the contact function),
> and exactly how each URL maps to a file. Presence of `builds` tells Vercel to skip framework
> detection altogether, so there's nothing left for it to misdetect.
>
> One tradeoff: Vercel's `functions` property (for setting custom memory/`maxDuration` per
> function) **cannot be combined with `builds`** — Vercel rejects the config outright if both are
> present. So this config runs on Vercel's defaults (currently 1024 MB / 10s on Hobby, more on
> Pro) rather than the 30s we'd have liked for larger MNE sessions. If a session's artifact audit
> ever times out, either keep the raw capture shorter (the app already caps it), or upgrade the
> plan, which raises the default duration without needing per-function config.
>
> `requirements.txt` still lives in `api/`, next to `analyze-artifacts.py` — `@vercel/python`
> looks for it there when installing dependencies for that function.

## Deploying to Vercel

1. Push this folder to a GitHub/GitLab/Bitbucket repo (or run `vercel` from
   inside it directly with the Vercel CLI — no build step is needed).
   **Double-check there's no extra wrapper folder**: `index.html`, `api/`,
   and `vercel.json` should sit at the repo root you're deploying, not one
   level inside another folder.
2. Import the repo in the Vercel dashboard, or run `vercel deploy`. Because
   `vercel.json` now declares `builds` explicitly, the dashboard's
   Framework Preset dropdown is not used for this deploy — you can leave
   it on whatever it's currently set to.
3. Vercel will pick up `api/requirements.txt` automatically and install it
   for `api/analyze-artifacts.py`. First deploy may take a bit longer than
   usual because of MNE's dependency size; that's expected.
4. Wire up `api/contact.js` to an actual email/notification service (see
   the TODO comment in that file) before relying on the contact form.

Locally, `vercel dev` will serve both functions on `localhost` alongside
the static file, so the "Analyze with MNE" button and the contact form
work the same as in production.

## The MNE artifact audit

The six artifact filters on the **Models** tab run live, in the browser,
in milliseconds — they're fast heuristics, not a diagnostic pipeline.
`api/analyze-artifacts.py` is a slower, independent second opinion: it
runs an exported session back through
[MNE-Python's](https://mne.tools) published artifact-detection routines
so you can see how well the live filters agree with a research-grade
pipeline. It does **not** compute the theta/gamma ratio or SEF95 — those
stay in the client, on the same pipeline that scores every session.

Because a NeuroSky headset is a single dry forehead channel, ICA-based
artifact removal (which needs multiple channels to unmix a component)
isn't usable here. Instead the endpoint uses:

- `mne.preprocessing.annotate_muscle_zscore` for EMG/muscle bursts
- `mne.preprocessing.annotate_amplitude` (flat-only) for a disconnected
  or dead channel
- a manual sliding-window peak-to-peak check for blinks and motion —
  MNE's `annotate_amplitude` fires on *consecutive-sample* jumps rather
  than peak-to-peak within a window, which turns out to miss ordinary
  blinks (they rise and fall over ~100–300ms, not between two adjacent
  samples), so a conventional windowed-PTP check is used instead, the
  same statistic tools like autoreject or a manual `mne.Epochs` reject
  dict use.

To use it:

1. In the app, go to **Progress → Settings** and turn on **Record raw for
   MNE audit**. This keeps the full-resolution waveform for your *next*
   session in memory only — it's not saved to your profile and adds
   nothing to Export save.
2. Run one session.
3. Back on **Progress**, use **Analyze with MNE** to POST it to
   `/api/analyze-artifacts`, or **Download raw** to save it and run
   the same analysis offline:

   ```
   pip install -r api/requirements.txt
   python api/analyze-artifacts.py neurodex-session-raw.json
   ```

The response reports a reject fraction, the flagged bad segments, and
time attributed to muscle vs. amplitude artifacts — compare it against
the live reject rate on the Models tab for the same session.

**Caveat noted in the endpoint's own output:** raw NeuroSky samples
aren't calibrated volts, so the amplitude thresholds use a rough 1e-6
scaling factor to land in a plausible EEG microvolt range. That's a
normalization, not a calibration — treat the amplitude-based numbers as
directional, not absolute.

## Not a medical device

Nothing here diagnoses or treats tinnitus, chronic pain, or depression.
Every result from this pilot is exploratory self-experimentation (n=1),
against a theory that is itself still debated in the literature.
