# Circuit podcast summarizer

Watches a Slack channel in The Circuit's workspace. When someone posts a YouTube
link, a podcast episode, a tweet, or a news article, the bot pulls a transcript
(or the article text), asks Gemini for the moments a Circuit editor would flag,
and posts them as a threaded reply — broadcast to the channel so nobody has to
open the thread.

Port of [`ji-podcast-summarizer`](https://github.com/LevelThreeLabsO/ji-podcast-summarizer).
Same plumbing; different beat.

## What it considers on-beat

The rubric lives in `BEAT_RUBRIC` at the top of `poll.py`. It is not guesswork —
it is the relevance model from
[`circuit-newswire`](https://github.com/LevelThreeLabsO/circuit-newswire), which
derived it by counting 269 of The Circuit's own headlines and every outbound link
in 300 of their posts.

Four ways a moment earns a slot:

1. **A named commercial entity** — sovereign wealth and state holdings (PIF,
   Mubadala, ADIA, ADQ, QIA, IHC, Lunate), tech and AI (MGX, G42, Core42, Humain),
   energy (ADNOC, Aramco, QatarEnergy, Masdar, ACWA), ports and logistics (DP World,
   AD Ports), aviation (Emirates, Etihad, Qatar Airways, Riyadh Air), banks and
   exchanges (FAB, Emirates NBD, QNB, Tadawul, ADX), real estate and giga-projects
   (Emaar, Aldar, NEOM, Qiddiya, Diriyah, Red Sea Global), telecom and platforms
   (stc, e&, Ooredoo, Careem, Talabat, Noon), free zones (DIFC, DMCC, ADGM), and
   sport/events used as investment vehicles (LIV Golf, Savvy Games, FII, GITEX).
2. **A named principal** — the Gulf elite network is a beat in its own right, and
   these moments often carry no business vocabulary at all. MBS, MBZ, Sheikh
   Tahnoun, Al-Rumayyan, Khaldoon Al Mubarak, Alabbar, Amin Nasser, Sultan Al Jaber,
   Bin Sulayem, Sheikh Tamim — plus the offices (energy minister, central bank
   governor, sovereign fund chief).
3. **Conflict economics** — regional conflict priced as trade. Hormuz, the Red Sea,
   Bab el-Mandeb, Suez; chokepoints and rerouting; war-risk premiums; closed
   airspace; force majeure.
4. **Gulf business substance** — stakes, IPOs, funding rounds, valuations, JVs and
   MoUs, tenders, earnings, sukuk, ratings, AUM — attached to the region.

**The load-bearing constraint: geography is never enough on its own.** A Middle
East place name plus a war-and-diplomacy moment is not a Circuit story. It needs
capital, a commercial entity, a principal acting commercially, or conflict
economics. This is the same rule that keeps `circuit-newswire` at 94.8% recall on
Circuit's real headlines and 8.3% on a tabloid negative set — and it is what stops
this channel filling up with regional politics.

Verified in both directions before first deploy:

| Input | Result |
| --- | --- |
| Hakeem Jeffries on CNN (US politics, no Gulf business) | `0 moments` — correctly skipped |
| CNBC interview with Mubadala CEO Khaldoon Al Mubarak | 3 moments: $330bn AUM, MGX/G42 stakes in OpenAI and Anthropic, China allocation |
| The Circuit's own Kushner/Humain story | 1 moment, correctly surfaced |

Note the bot returns **up to** five moments, not exactly five. An off-beat item
costs the channel more than a missing one, so it is told to prefer four strong
moments over five where the fifth is filler.

## How it runs

Single-shot: one GitHub Actions tick = one run. Cadence is the `*/5` schedule cron
plus a self-chain step that re-dispatches at the end of every run (using the
built-in `GITHUB_TOKEN`, no PAT), so real-world latency is a minute or two. A
`concurrency` group serializes runs so the chain can't stack up.

State (`watcher_state.json`) is committed back to the repo each run, so nothing is
re-processed. Duplicate protection is belt-and-braces: `processed_ts` plus a
`thread_has_bot_reply` check before posting.

Failures are silent by design — if the bot can't produce a summary it says nothing
rather than posting an error into the channel. Transient problems (network blips,
Gemini 5xx, rate limits) raise `TransientError` and retry on the next tick;
genuinely permanent ones (paywall, no captions, deleted tweet) are marked done.

## Setup

**Requires a Slack app in The Circuit's own workspace** (team `T039JT99YMD`) — the
JI bot token will not work here. Create it at api.slack.com using "Sign in to
another workspace"; the dev site has its own login, separate from the Slack client.
Scopes needed: `channels:history`, `chat:write`, `groups:history` if the channel is
private.

Repo secrets:

| Secret | Notes |
| --- | --- |
| `SLACK_BOT_TOKEN` | **New** — from the Circuit-workspace app |
| `SLACK_CHANNEL_ID` | **New** — the Circuit channel to watch |
| `GEMINI_API_KEY` | Same value as the JI bot |
| `GROQ_API_KEY` | Same value — Whisper fallback for videos with no captions |
| `YT_TRANSCRIPT_IO_TOKEN` | Same value — primary YouTube path, 25/day free |
| `CLIPMAKER_URL` | Same value — Mac fallback for YouTube and podcasts |
| `CLIPMAKER_AUTH_TOKEN` | Same value |

Then invite the bot to the channel and let the schedule pick it up.

## Local use

```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
export GEMINI_API_KEY=...

.venv/bin/python poll.py --url "<any URL>"   # test one URL end-to-end, no Slack post
.venv/bin/python poll.py --dry-run           # poll the channel, print, don't post
.venv/bin/python poll.py                     # real run
```

`--url` is the fastest way to sanity-check a beat-rubric edit: run it against a
story you know is on-beat and one you know isn't, and check both answers.

## Tuning the beat

Edit `BEAT_RUBRIC` in `poll.py`. All three prompts (video, tweet, article) compose
it, so one edit moves the whole bot. If you add a category, test the negative
direction too — the failure mode that matters is regional politics leaking in, and
you only see it by running something off-beat through `--url`.

`circuit-newswire`'s `scoring.yaml` is the reference for entity and principal
rosters; it is the list that gets stale first as new funds and executives appear.
