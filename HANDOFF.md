# Handoff: DripStack Integration into last30days

## Status

**PR:** https://github.com/mvanhorn/last30days-skill/pull/791
**Branch:** `drip` on `zimoo354/last30days-skill` (fork of `mvanhorn/last30days-skill`)
**Ticket:** DRIP-62 on Linear (Drip team)
**Assignee:** Charlie Ruiz (zimoo354)
**State:** Awaiting review from @mvanhorn (repo owner)

## What was done

Added DripStack as a first-class source to the last30days skill. DripStack indexes paid financial newsletters (SemiAnalysis, Bloomberg, Doomberg, etc.) and exposes a free, public search API. The integration lets last30days discover what professional analysts are writing about — complementary to StockTwits (retail sentiment) and Polymarket (prediction odds).

## Files changed (7)

| File | Change |
|------|--------|
| `skills/last30days/scripts/lib/dripstack.py` | **New.** Source module. Calls `https://dripstack.xyz/api/v1/search?q={query}&limit={n}` (free, no auth). Returns engine-style item dicts. |
| `skills/last30days/scripts/lib/normalize.py` | Added `"dripstack": _normalize_dripstack` to the normalizers dict. Normalizer function placed right before `_normalize_reddit`. |
| `skills/last30days/scripts/lib/pipeline.py` | Imported `dripstack`. Added `"dripstack"` to `MOCK_AVAILABLE_SOURCES`. Registered in `available_sources()` (always available, like GitHub/HN). Added `if source == "dripstack":` block in `_retrieve_stream()`. Added mock data in `_mock_stream_results()`. |
| `skills/last30days/scripts/lib/planner.py` | Added `"dripstack": {"reference", "analysis", "link"}` to `SOURCE_CAPABILITIES`. Added `"dripstack"` to `SOURCE_PRIORITY` for `opinion` and `prediction` intents. |
| `skills/last30days/SKILL.md` | Added `dripstack`→DripStack to the display name mapping (line ~620). |
| `CONFIGURATION.md` | Added DripStack row to the source-by-source table. |
| `tests/test_diagnose_compat.py` | Added `"dripstack"` to `KNOWN_SOURCE_NAMES` set. |

## Architecture: how sources work in last30days

Each source is a Python module in `skills/last30days/scripts/lib/` that follows this pattern:

1. **Search function** — hits the platform's API, returns raw items
2. **Parse function** — normalizes into engine-style dicts (`id`, `title`, `url`, `author`, `date`, `engagement`, `relevance`, `snippet`, `metadata`)
3. **Normalizer** — registered in `normalize.py`, converts dicts to `schema.SourceItem`
4. **Pipeline registration** — imported in `pipeline.py`, added to `available_sources()`, dispatch in `_retrieve_stream()`
5. **Planner registration** — capabilities + intent priority in `planner.py`

The planner (an LLM) decides which sources to use based on topic intent. For `opinion` and `prediction` intents, DripStack runs alongside StockTwits.

## Key design decisions

1. **No API key required.** DripStack's search endpoint is free and public. Always available, like GitHub, HN, Reddit. No opt-in friction, no setup wizard changes needed.

2. **Purchase flow stays in DripStack.** If a user wants to read a full article, they use DripStack directly with their own x402/MPP wallet. last30days handles discovery, DripStack handles commerce.

3. **Publication as attribution.** Results carry publication names (e.g. "SemiAnalysis", "Doomberg") as the `author` field. This is high-credibility signal for synthesis — professional analyst attribution, not social handles.

4. **Relevance-gated by the API.** DripStack's own search does hybrid RRF (semantic + keyword) scoring. Off-topic searches return thin results naturally; the engine's relevance scoring and thin-retry handle the rest.

## Testing

All 2700+ existing tests pass. Run with:

```bash
cd ~/Documents/projects/last30days-skill
python3 -m pytest tests/ -x -q --tb=short
```

Quick smoke test of the source module:

```bash
cd ~/Documents/projects/last30days-skill
python3 -c "
import sys
sys.path.insert(0, 'skills/last30days/scripts')
from lib.dripstack import search_dripstack, parse_dripstack_response

raw = search_dripstack('AI capex risk', depth='default')
items = parse_dripstack_response(raw, query='AI capex risk')

print(f'{len(items)} results')
for it in items[:5]:
    print(f'  [{it[\"relevance\"]:.0%}] {it[\"title\"]} ({it[\"author\"]}, {it[\"date\"] or \"no date\"})')
"
```

## DripStack API reference

| Endpoint | Auth | Returns |
|----------|------|---------|
| `GET /api/v1/search?q={query}&limit={n}` | None (free) | Article metadata, relevance scores, snippets |
| `GET /api/v1/publications/search?q={query}` | None (free) | Publication lookup |
| `GET /api/v1/publications/{slug}` | None (free) | Publication metadata + recent post summaries |
| `GET /api/v1/publications/{slug}/{postSlug}` | x402/MPP only | Full article summary (paid) |
| `POST /api/v1/publications/{slug}/{postSlug}` | API key or x402 | Full article summary (credits) |
| `GET /api/v1/stock-picks` | x402/MPP or API key | Structured stock picks (paid) |

OpenAPI spec: `https://dripstack.xyz/openapi.json`

## What's next

- **Wait for @mvanhorn's review** on PR #791
- **Monitor for feedback** — he may want changes to how it's registered (capabilities, intent priority, etc.)
- **Future enhancement:** If there's demand, add `DRIPSTACK_API_KEY` support for fetching full article summaries (paid enrichment, opt-in, top-N only after reranking)
- **Future enhancement:** Stock picks integration (`/api/v1/stock-picks`) for financial topic runs with a key

## Context for the reviewer

Matt Van Horn (@mvanhorn) invited this integration via [this tweet](https://x.com/mvanhorn/status/2074986990555258963) — replying to Michael Blau (@blauyourmind, DripStack CTO) saying "tell me more / make a PR!"

The PR started as a README link but evolved into a full source integration because that's what "make a PR" actually means in an open-source context.

## Repo access

- Fork: `zimoo354/last30days-skill`
- Upstream: `mvanhorn/last30days-skill`
- The fork is behind upstream (1 commit ahead). If upstream has moved, rebase before merging.
- Linear ticket: DRIP-62 (Drip team, Charlie assigned)
- Archie token for ticket updates: in `.env` at `/Users/zimoo354/Documents/royal/dripstack/.env`
