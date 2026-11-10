# Data Sources

Findings from [`notebooks/01_explore_api.ipynb`](../notebooks/01_explore_api.ipynb),
run on **2026-11-10 09:22–09:27 UTC** (16:22–16:27 Bangkok). All numbers below come
from that run unless marked *(documented)* or *(community-reported)*.

## TL;DR — what to use

| Task | Use | Why |
|---|---|---|
| 1. Top ~100 games | **Steam Charts** `GetGamesByConcurrentPlayers` | Official Steam host, no key, returns exactly 100 games ranked by live players. SteamSpy failed 3/3 attempts. |
| 2. Current players | **Same Charts call** (`concurrent_in_game`) | One request per hour instead of 100. Values match `GetNumberOfCurrentPlayers` within ~2 %, and it covers games that endpoint 404s on (Deadlock). |
| 2b. Players for a specific game | `GetNumberOfCurrentPlayers` | Only official documented endpoint. Use it as a fallback, or to keep tracking games after they drop out of the top 100. |
| 3. Game metadata | **Store** `appdetails` (one appid per call) | Only source with name + genres + `is_free` + USD price + release date. 100/100 top games succeeded at 1.5 s spacing. Fetch once per new game. |
| 3b. Daily price snapshot (optional) | `appdetails?filters=price_overview` (multi-appid) | The only mode where several appids fit in one call. |
| Tags (optional) | SteamSpy `appdetails` | User tags are finer-grained than Store genres, but the data is stale and the service is unreliable. |

## 1. Steam Charts — `GetGamesByConcurrentPlayers`

```
GET https://api.steampowered.com/ISteamChartsService/GetGamesByConcurrentPlayers/v1/
```

| | |
|---|---|
| API key | **Not required** (200 OK without a key) |
| Parameters | none needed |
| Returns | 100 games: `rank`, `appid`, `concurrent_in_game`, `peak_in_game` + `last_update` (unix seconds) |
| Latency | 0.4–0.5 s |
| Rate-limit headers | none (`cache-control: no-cache`, `x-eresult: 1`) |

Example response (truncated):

```json
{"response": {
  "last_update": 1794302297,
  "ranks": [
    {"rank": 1, "appid": 730, "concurrent_in_game": 567044, "peak_in_game": 1080352},
    {"rank": 2, "appid": 570, "concurrent_in_game": 431772, "peak_in_game": 707682}
  ]}}
```

Observations
- `last_update` was **4.2 minutes old** at request time and did not change across calls 5 s apart, so the list is refreshed every few minutes. **Store `last_update` as the observation time**, not the request time.
- **No game names.** Names have to come from `appdetails`.
- `peak_in_game` is the game's peak over a window Steam does not specify (probably the last 24 h). Treat it as informational.
- Entries are not all `type=game`: the top 100 included a `mod` (tModLoader) and a `demo`.
- This interface is **not listed** in the keyless `ISteamWebAPIUtil/GetSupportedAPIList`, and I found no public documentation page for it. It powers store.steampowered.com/charts, so it could change without notice. This is the main risk of this source, mitigated by the fallback in §2.

## 2. Current players — `GetNumberOfCurrentPlayers`

```
GET https://api.steampowered.com/ISteamUserStats/GetNumberOfCurrentPlayers/v1/?appid=730
```

| | |
|---|---|
| API key | **Not required**. It is listed in the keyless `GetSupportedAPIList` (`appid` is the only parameter). |
| Success | `200 {"response":{"player_count":561518,"result":1}}` |
| Unknown appid | `404 {"response":{"result":42}}` |
| Latency | ~0.05 s. 20 calls with 1 s sleep took 21.0 s, so 100 games ≈ **1.8 min** |
| Cache | `cache-control: public, max-age=12` |

Observations
- **Deadlock (1422450)**, rank 8 in Charts with ~90k players, returns **404** here. A game can be in the top 100 and still fail this endpoint.
- A **DLC appid** (2678630) returns `200` with `player_count: 1`, not an error. Do not assume a 200 means the appid is a real game.
- Compared with Charts for the top 5 games, the difference was **0 % to −2.2 %**, and the Charts value was the same or higher in every case. The two endpoints were sampled a few minutes apart, so this is timing, not disagreement.

## 3. Store — `appdetails`

```
GET https://store.steampowered.com/api/appdetails?appids=730&cc=us&l=english
```

| | |
|---|---|
| API key | **Not required** |
| Official docs | None. This is an undocumented Store endpoint. |
| Latency | ~0.3–0.5 s |
| Cache | `cache-control: public, max-age=3600` |
| Multiple appids | **Only with `filters=price_overview`**. Without filters, `appids=730,1086940,271590` returns `400 null`. |

Fields used (34 top-level keys exist in `data`):

| Field | Example | Notes |
|---|---|---|
| `name` | `"Baldur's Gate 3"` | Can contain trademark symbols. Read and store as UTF-8. |
| `type` | `"game"` | Also `mod`, `demo`, `dlc`, … |
| `is_free` | `true` | 38 of the top 100 |
| `genres` | `[{"id":"1","description":"Action"}, …]` | Can be **empty** (Eternal Return) |
| `price_overview` | `{"currency":"USD","initial":5999,"final":5999,"discount_percent":0}` | Prices in **cents**. Missing for free games **and for some paid games** (see below) |
| `release_date` | `{"coming_soon":false,"date":"Aug 3, 2023"}` | `date` is free text, e.g. `"To be announced"` |

Example (`appid=1086940`, trimmed to the fields we keep):

```json
{"1086940": {"success": true, "data": {
  "type": "game", "name": "Baldur's Gate 3", "is_free": false,
  "genres": [{"description": "Adventure"}, {"description": "RPG"}, {"description": "Strategy"}],
  "price_overview": {"currency": "USD", "initial": 5999, "final": 5999, "discount_percent": 0},
  "release_date": {"coming_soon": false, "date": "Aug 3, 2023"}}}}
```

Invalid appid → `200 {"1": {"success": false}}`

Batched price call (`filters=price_overview`). Free or unpriced games come back as an **empty list**, not an object:

```json
{"730": {"success": true, "data": []},
 "1086940": {"success": true, "data": {"price_overview": {"currency": "USD", "final": 5999, "...": "..."}}}}
```

Coverage on the live top 100 (one call each, 1.5 s apart, 2.9 min total):
- **100/100 HTTP 200, 100/100 `success: true`, zero 429s**
- `type`: 98 `game`, 1 `mod`, 1 `demo`
- **Paid but no `price_overview`: 3**. Deadlock (unreleased, "To be announced"), Call of Duty, and GTA V Legacy. Some games are sold only through packages or editions, so `price_usd` must be nullable even when `is_free=false`.
- 1 game with no genres, 1 `coming_soon`
- `is_free=true` but no "Free To Play" genre: **6 games**. Use the `is_free` flag, not the genre, to split F2P from paid.
- Genre frequency: Action 62, Adventure 42, RPG 42, Indie 33, Free To Play 32, Simulation 31, Strategy 24, Massively Multiplayer 23, Casual 20, Early Access 9. Most games have several genres.

## 4. SteamSpy (third-party)

```
GET https://steamspy.com/api.php?request=top100in2weeks
GET https://steamspy.com/api.php?request=appdetails&appid=730
```

| | |
|---|---|
| API key | Not required |
| Rate limit *(documented on steamspy.com/api.php)* | 1 request/second, and 1 request/60 s for the `all` request |
| `top100in2weeks` | **Failed 3/3 attempts** (10 s and 20 s back-off): `200 text/html "Connection failed: Too many connections"` |
| `appdetails` | Worked. Returns `tags`, `genre` (comma-separated string), `owners` (a range), `ccu` |

Observations
- **Errors come back as HTTP 200 with an HTML body.** A client that only checks status codes would treat them as success. Always validate that the body is JSON.
- The data is stale. For CS2 it still reported the old name *Counter-Strike: Global Offensive* and `ccu: 1,013,936`, while Steam showed ~561k at the same moment. That value is a daily or historical figure, not a live one.
- `owners` is a wide range (`"100,000,000 .. 200,000,000"`), not a count.
- **Use only as an optional enrichment** (tags). Do not depend on it in the hourly pipeline.

## Rate limits and terms

| Source | Limit | Basis |
|---|---|---|
| Steam Web API (`api.steampowered.com`) | 100,000 calls/day | *(documented)* Steam Web API Terms of Use |
| Store `appdetails` | ~200 calls / 5 min per IP | *(community-reported)*. Not reached here: 100 calls at 1.5 s spacing gave no 429. |
| SteamSpy | 1 req/s (`all`: 1 per 60 s) | *(documented)* |

None of the endpoints returned rate-limit headers (`X-RateLimit-*`, `Retry-After`), so the client has to pace itself: a fixed sleep between calls, plus retry with exponential backoff on 429 and 5xx.

Planned hourly load: **1** Charts call + `appdetails` only for games not seen before (0–5 per hour after the first run, ~100 on the first run). That is far below every limit above.

Terms of use (summary, not legal advice):
- [Steam Web API Terms of Use](https://steamcommunity.com/dev/apiterms): attribute Valve/Steam as the data source, and don't present the project as endorsed by Valve. This project is non-commercial and links back to Steam.
- The Store endpoint is undocumented and comes with no guarantee. Keep the call volume low and cache results (metadata is fetched once per game).
- Game names, images and other store content belong to their publishers. The dashboard shows names and numbers only, with no store assets.
- SteamSpy data is provided "as is" by a third party. Credit it if it is used.

## Implications for the pipeline (Step 2)

1. Get **top games and player counts from one Charts call**, and use `last_update` as `collected_at_utc`.
2. Keep `get_current_players(appid)` as a fallback, and for tracking games after they leave the top 100. It must return `None` on 404 / `result != 1`.
3. `get_game_details(appid)`: one appid per call, `cc=us`, `l=english`, about 1.5 s apart. Fields: `name`, `type`, `is_free`, `genres` (list, may be empty), `price_usd = final / 100` (nullable even for paid games), `release_date` (parse `"%b %d, %Y"`, else `None`), `coming_soon`.
4. Optional daily price history: one batched `filters=price_overview` call per ~100 appids. Treat `data: []` as "no price".
5. Retry only transient errors (timeouts, 429, 5xx). Never retry 404 or `success: false`.
