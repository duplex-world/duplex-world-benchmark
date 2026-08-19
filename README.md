# DuplexWorld

Project page for a benchmark that puts speech-to-speech voice agents through six worlds of
an ordinary day and scores each conversation three separate ways.

Live at <https://duplex-world.github.io/duplex-world-benchmark/>.

## What is here

A static site. No build step, no server, no external requests: every stylesheet, script,
figure, recording and payload is served from this repository.

```
index.html          the opening, and the film
motivation.html     why the benchmark exists, and how it compares to prior work
overview.html       the corpus at a glance
setup.html          what is tested
system.html         how one conversation is actually run
metrics.html        the three pillars and the twelve metrics inside them
samples.html        recorded calls, playable
leaderboard.html    every published figure, filterable

flight.mp4          the scroll-scrubbed film, 104 s
flight-small.mp4    a 960x540 encode, served to narrow or metered clients
audio/              the recordings, fetched only when a reader presses play
*.json              the walk and call payloads behind the players
```

## Running it locally

The film is scrubbed by writing `currentTime`, and that only works if the host answers HTTP
Range requests. Python's own one-line static server does not answer them, and against it the
video reports itself unseekable and sits frozen on the first frame with nothing in the
console. Use the included server instead:

```bash
python3 serve.py 8080      # then open http://127.0.0.1:8080/
```

Every real static host answers Range, so this only matters when looking at the folder
locally.

## Notes

`.nojekyll` is present so Pages serves these files as they are rather than running Jekyll
over them. Do not delete it.

Numbers on the page are not hand-written. Each one is read from `data.js`, so a figure
exists in exactly one place across all eight pages.
