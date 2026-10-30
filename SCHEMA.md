# Back Room Odds board file (board.json), schema v1

One JSON file. The app only displays it, and chat answers only from it. Built by `build_board.py`, published by `publish.sh`.
UTF-8. Dots are the literal characters `●●●` / `●●`. Every time is ISO 8601 with the Pacific offset (e.g. `2026-10-30T10:01:46-07:00`).

## Top level
| field | type | meaning |
|---|---|---|
| version | int | Schema version (1) |
| generated_at | string | When this board was built (PT offset) |
| week | {season, week, label} | e.g. `{2026, 8, "2026 Week 8"}` |
| sources | [{name, url, fetched_at, ok, error}] | Action Network, WiseGuy (splits), ESPN standings, ESPN injuries. `ok:false` + `error` if a fetch failed. Never hidden. |
| elite_top5 | [string] | Teams covered by the elite exception, e.g. `"MIN 6-0"` (ties at 5th included) |
| vip | {firm_plays:[VipPick], note} | VIP tab. `note` = "1 unit = 2% of bankroll. Same size every bet. Never chase losses." or "No firm plays. Nothing firm this week." |
| games | [Game] | Already sorted by `rank` (1 = most house liability) |
| rules_summary | object | Plain-text rules the chat may quote |
| data_gaps | [string] | Anything missing this run (e.g. a source with no row for a game) |
| disclaimers | [2 strings] | Exactly: "Don't believe a single word I say, research for yourself." and "For entertainment only, not betting advice. Gambling problem? Call 1-800-GAMBLER." |

## VipPick
`{game_id, matchup, pick, market ("spread"|"moneyline"), price (American odds), pick_type, dots, confidence_pct (int|null), record (string|null), units (1 | "0.25-0.5"), liability_score}`
Firm spread plays (`●●●`, 1 unit) plus the small moneyline (`●●`, 0.25-0.5 u) on a firm-play underdog. Sorted by liability, spread before its ML.

## Game
| field | type | meaning |
|---|---|---|
| id | string | `2026_08_NE_MIA` (season_week_AWAY_HOME) |
| rank | int | 1..n by house liability. Dots never change rank. |
| liability_score | number\|null | House liability score (null = no splits) |
| away, home | string | Team abbreviations (JAX, LAR, WAS, LV normalized) |
| kickoff_pt / kickoff_display | string | ISO kickoff / `"Sun 11/1 1:25 PM PT"` |
| line | {spread_fav, spread, home_spread, home_spread_price, away_spread_price, total, ml_away, ml_home, book} | Current DraftKings line via Action Network. `spread` is the favorite's number (negative). |
| open | {spread_fav, spread, home_spread, total} | Opening line |
| data_level | FULL\|DISAGREE\|THIN\|NONE | Game level: NONE if all markets NONE, DISAGREE if any market DISAGREE, FULL if all FULL, else THIN |
| market_levels | {spread, moneyline, total} | Same labels per market |
| sources_line | string | Ready to print, e.g. `"Action Network 10:01 AM · WiseGuy 9:56 AM · DISAGREE"` |
| splits | {"Action Network": Split, "WiseGuy": Split} | Per source |
| markets | [Market] | Always 3, in order spread(1), total(2), moneyline(3) |
| tags | [{type, reason}] | type ∈ WATCHLIST, DIVISION DOG, EARLY READ, SHARP MOVE, CAUTION. CAUTION always has a reason ("Thin data", "Wind 17 mph (Overs blocked)"). SHARP MOVE only when the spread market has a pick. |
| weather | {text, source, overs_blocked} | Open-Meteo at kickoff +3h, or "Indoor / roof (...)". text "Not found" if missing. |
| injuries | {text, source} or the string "Not found" | ESPN Out/Doubtful/Questionable for both teams |
| notes | [string] | Line moves written with the favorite's sign ("JAX opened -4.5, now -10."), money side, elite exception |

### Split
`{fetched_at, spread_side, spread_bets, spread_money, ml_side, ml_bets, ml_money, over_bets, over_money}`. `*_side` = team with the majority of money; bets/money are % for that side. Any value can be null = not found at that source.

### Market
`{market, order, pick, dots, watch_only, confidence_pct, record, pick_type?, lean_names[], firm, units, level, text?}`
- `●●●` firm spread play (Promoted or Division Dog Must-Win), `firm:true`, `units:1`. Only one place to show dots: this row.
- `●●` tested totals lean (Smart-Money Over, Dallas Over, Public Over Fade), the small ML on a firm-play dog (`units:"0.25-0.5"`), or a THIN would-be firm play (`firm:false`, `text:"Would be firm. Needs a second source."`).
- `dots:null`, `watch_only:true`, `pick:"House side: X · Watch only"` (or "House side split (AN: …, WG: …) · Watch only", or "No call").
- `confidence_pct` = round(100 × (W+10)/(W+L+20)) from that pick type's record: Promoted 30-14-1 → 63, Division Dog Must-Win 41-19 → 64, Smart-Money Over 49-33 → 58, Dallas Over 27-12-1 → 63, Public Over Fade 87-65-2 → 56. One-lean side picks cap 45, Early Reads cap 50. null for watch-only and for the small ML.

## App rules (for the Grok Build side)
- Display only. Never compute picks, dots or percentages in the app.
- Chat: load board.json before every answer; never state a number, injury, weather or pick status not in the file; "firm" only if `firm:true`; quote `weather.text` and `injuries` exactly; end every answer with the two `disclaimers`.
- Show `generated_at` as "Updated h:mm AM PT". If `generated_at` is older than 12 hours, show "Board may be stale".
- If a `sources[]` entry has `ok:false`, show its name and error on the status page.
