# nav benchmark project page

Static project page for the navigation world of a six-world full-duplex voice benchmark. Plain
HTML and CSS, no build step, no JavaScript, no external requests.

Live locally at `http://localhost:8910/`.

## Deploy to GitHub Pages

```bash
git init
git add .
git commit -m "nav benchmark project page"
git branch -M main
git remote add origin git@github.com:<org>/<repo>.git
git push -u origin main
```

Then in the repository settings, **Pages -> Build and deployment -> Source: Deploy from a branch**,
branch `main`, folder `/ (root)`. The page appears at `https://<org>.github.io/<repo>/`.

For a `<org>.github.io` user or org site, name the repository exactly `<org>.github.io` and it
serves from the root domain instead.

`.nojekyll` is present so Pages serves the files as they are rather than running Jekyll over them.
Do not delete it.

## Layout

```
index.html                 the page, single file
.nojekyll                  disable Jekyll processing
assets/css/style.css       all styling, light and dark
assets/img/*.png           23 figures, most in _light and _dark pairs
assets/audio/*.mp3         3 full conversation recordings plus 1 clip
data/nav_per_episode.csv   630 rows, one per scored conversation
data/nav_per_scenario_counts.csv   90 rows, reliability curve input
data/appendix.md           the full appendix, 4,435 words
data/excerpts.md           verbatim transcripts with tick indices
```

Total about 23 MB, dominated by audio. Well inside the GitHub Pages soft limit of 1 GB per site,
though note Pages has a **100 MB per file** hard limit; nothing here approaches it.

## Editing

**Numbers.** Do not hand-edit them. Everything on the page comes from
`data/nav_per_episode.csv`, and the generators that produced it live in the analysis repo at
`scratchpad/ms/`. Regenerate there, then re-copy.

**Theme.** The page respects the visitor's OS setting via `prefers-color-scheme` and also honours a
`data-theme` attribute on `<html>` if you later add a toggle. Figures ship in light and dark
variants and are swapped by CSS, so a figure never sits on the wrong ground.

**Palette.** accent violet `#6B2FBF` and magenta `#C026A8`, held to accent duty: rules, kickers,
links and the single data colour. The body is grayscale, in the academic project-page idiom. All
colours are custom properties at the top of `style.css`; change them there and the whole page
follows.

**Audio.** Re-encode from the source WAVs with
`ffmpeg -i in.wav -ac 1 -b:a 48k out.mp3`. The clip is
`ffmpeg -ss 160 -t 70 -i in.wav -ac 1 -b:a 64k clip.mp3`.

## Content notes

Three claims on the page are deliberately hedged, and the hedges should survive editing:

- **Exploration** is described as moderate (Spearman rho -0.66) and explicitly *not* as a general
  inverse, because the correlation turns positive within a single model across its conditions.
- **Heading accuracy** leads with the finding that most of the gap is blank fields rather than wrong
  answers, which is a weaker and more accurate claim than the raw 0.22 to 0.93 spread.
- **Every zero is a lower bound**, because 95 percent of non-arrivals hit the simulated clock rather
  than going to the wrong place.

The "Scope and honesty" section also records a bug on our side: `directions()` mis-resolves the
long-horizon destination by two blocks in all three of those scenarios, so part of that condition's
failure is the harness rather than the model. Leave it in.

## Accessibility

Every image carries descriptive alt text. Links have a visible focus state.
`prefers-reduced-motion` disables smooth scrolling. Wide tables scroll inside their own container so
the page body never scrolls sideways.
