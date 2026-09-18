a user sets up an account
with the information on the account a request can be made on the openFDA API
the API used for now are the openfda

when users log in they already make one request to the APIs
the users are shown recall information on food based on their location

location data is set by the user
the user manually puts in the location that is relevent

the user can also make changes to the filter
- location
-if the recall is open or not

when the user makes the account 3 things are needed
- username or email (to be decided)
- password (hash the password store the hash do not store the plain text)
- initial location (this will be used to make the first request)

on log in the user will see critical recalls first and if they are still active

an alert or warning would be shown if the location the user is has a notification

requesting from the api with be done through httpx
requests are made with the location as the main request and limit of the current year

limitations:
httpx is not able to access the USDA api so the data is not used
- no meat poultry or egg products (access was denied)

normalizing the data from the API
-the openFDA has its own documention on what requests can be done
-the affected areas are in the distribution_pattern field
    - openFDA uses the abbriviations of the states
-data from the API needs to be normalized but still show what is important

the request of the data from the API
- the request data is not saved  (see OPEN DECISION 1)
- the request data should be cached in the frontend for refreshed (do not worry for now)


endpoint lists   [SUPERSEDED 2026-09-17 - user endpoints dropped in phase 1]
user
- POST /register: makes a user
- POST /login: login the user with username and password
- GET /me: get the last changed user informaiton location and severity
- DELETE /me: delete the user account (delete user account casscade)


service
- PATCH /me {state?, severity?}: changes the location or Class used to search the API
- GET /recalls: lists all of the recalls that affect the currect locaiton
    the list recall with be automatically sent when logged in of changes location
- GET /health: tests if the app works

endpoints that need to be with authenticated
- GET /me
- PATCH /me
- DELETE /me
- GET /recalls


================================================================
everything below added 2026-08-26
================================================================


NORMALIZED RECALL MODEL   [AMENDED 2026-09-17 - see GROUPING below:
                           source_id is now event_id, raw is a LIST of
                           rows, and products/product_count are added]
-----------------------
the canonical shape every recall is converted into before it reaches
the rest of the app. one mapper per source, all mess stays in the mapper.

  source          str      "openfda" (kept so FSIS can be added later)
  source_id       str      openFDA recall_number, eg "H-0814-2026"
  title           str      truncated product_description
  description     str      product_description
  reason          str      reason_for_recall
  firm            str      recalling_firm
  firm_state      str      openFDA state - DISPLAY ONLY, never filter on it
  severity        enum     CLASS_I | CLASS_II | CLASS_III
  status          enum     ACTIVE | CLOSED
  states          list     two letter codes parsed from distribution_pattern
  is_nationwide   bool     parsed from distribution_pattern
  recall_date     date     recall_initiation_date, YYYYMMDD -> needs validator
  report_date     date     report_date, YYYYMMDD -> needs validator
  url             str|None openFDA has no url field, always None for now
  raw             dict     original json, so re-normalizing needs no refetch

unique key is (source, source_id)


FIELD MAPPING - OPENFDA
-----------------------
severity:
  "Class I"   -> CLASS_I    (most serious)
  "Class II"  -> CLASS_II
  "Class III" -> CLASS_III

status:
  "Ongoing"    -> ACTIVE
  "Pending"    -> ACTIVE
  "Completed"  -> CLOSED
  "Terminated" -> CLOSED

dates:
  openFDA sends YYYYMMDD strings, not ISO. eg "20160808"
  parse with a pydantic validator


LOCATION FILTERING - THE THREE RULES
------------------------------------
these are the most important correctness rules in the project.

1. filter on distribution_pattern, NEVER on state
   state is the recalling firm's own address, not where the food went.
   verified: distribution_pattern CA = 7016 records, state CA = 4011.
   different fields, different meaning.

2. a state match MUST also include nationwide recalls
   verified for 2026: CA specific = 202, nationwide = 214, together = 416.
   filtering on "CA" alone hides more than half of what affects a CA user.
   query is: is_nationwide == True OR "CA" in states

3. when distribution_pattern cannot be parsed, set is_nationwide = True
   over-including is safe, hiding a real recall is not.

parsing distribution_pattern:
  free text. examples seen: "Nationwide", "FL, MI, MS, and OH.",
  "Distributed to retail stores in the Midwest"
  match uppercase only with word boundaries: \b[A-Z]{2}\b
  reason: OR IN ME OK HI DE PA are english words. prose uses lowercase
  "or"/"in" so case sensitivity removes most false positives for free.
  also check for "nationwide" / "nationally" / "all 50 states"


OPENFDA INTEGRATION NOTES
-------------------------
base url: https://api.fda.gov/food/enforcement.json

- no api key needed at this volume
- limit max is 1000 per request. 1001 returns 400
- paginate with skip
- returns HTTP 404 NOT_FOUND when a search has zero matches,
  NOT 200 with an empty list. must be handled:
      if r.status_code == 404: return []
      r.raise_for_status()
- no rate limit headers in the response, you cannot see remaining quota
- unauthenticated limit is roughly 1000 requests/day per IP
- every response contains meta.disclaimer - surface it in the UI
- pass the query via httpx params={} , do not build the url by hand

query shape that works:
  search=(distribution_pattern:"CA" OR distribution_pattern:"nationwide")
         AND report_date:[20260101 TO 20261231]
  limit=1000

volume: 861 records for all of 2026 so far. 29317 all time.


AUTH   [SUPERSEDED 2026-09-17 - no accounts in phase 1, see PROJECT DIRECTION]
----
- password hashing: argon2 or bcrypt via pwdlib. never store plaintext
- POST /login returns {"access_token": "...", "token_type": "bearer"}
- token presented as: Authorization: Bearer <token>
- stateless JWT, no server side session
- no /logout endpoint - client drops the token (deliberate)
- no /refresh endpoint in v1 (deliberate)
- token expiry: 24h  (see OPEN DECISION 2)
- protected routes grouped under a router with the auth dependency
  applied to the whole router, so a new route is protected by default
- PATCH /me and DELETE /me resolve the user FROM THE TOKEN.
  never take a user id from the path - that lets anyone edit anyone.
- POST /login and POST /register are public but need rate limiting


DATABASE   [SUPERSEDED 2026-09-17 - no db in phase 1, in-memory cache instead]
--------
users table:
  id             int pk
  email          str unique
  password_hash  str
  state          str  two letter code, default filter
  severity       enum default filter, nullable = all
  created_at     datetime

sqlite for development, postgres later.
alembic for migrations. commit alembic/versions/, it is source code.


API CONTRACT   [SUPERSEDED 2026-09-17 - see PHASE 1 API CONTRACT below]
------------
POST   /register   {email, password, state} -> {id, email, state}
POST   /login      {email, password}        -> {access_token, token_type}
GET    /me                                  -> {id, email, state, severity}
PATCH  /me         {state?, severity?}      -> updated user
DELETE /me                                  -> 204
GET    /health                              -> {"status": "ok"}
GET    /recalls?state=&status=&severity=&limit=&offset=  -> [Recall]
GET    /recalls/{source_id}                 -> Recall

notes:
- state/status/severity query params OVERRIDE the saved default for that
  request only. they do not change the saved user setting.
  PATCH /me is the only thing that persists a new default.
  reason: "let me check texas" should not overwrite the user's home state.
- state must be a validated enum, not a free string. fastapi then
  rejects bad values with 422 and renders a dropdown in /docs
- default state is the user's saved state when no param is given


SORT ORDER
----------
GET /recalls returns:
  1. severity ascending  (CLASS_I first)
  2. then recall_date descending (newest first)
active recalls before closed ones


THE ALERT
---------
"an alert or warning would be shown" = an in app banner, driven by a
field on the recall list response, not a separate endpoint.
adding GET /alerts would mean a second round trip for data already fetched.
suggested: has_critical: bool  (any returned recall is CLASS_I and ACTIVE)


DISCLOSURE - REQUIRED
---------------------
this is a food safety app, users may act on what it shows.

- the UI must state that meat, poultry and egg products are NOT covered.
  measured: openFDA has 0 ground beef and 0 deli meat recalls 2024-2026,
  because those are USDA jurisdiction. a user assuming full coverage is
  a real harm, not just a missing feature.
- surface openFDA's meta.disclaimer
- show the last sync / last fetched time so users can judge freshness


NON GOALS FOR V1   [REVISED 2026-09-17 - push notifications moved to phase 2]
----------------
- FSIS / USDA meat poultry egg data (blocked, see limitations)
- password reset (needs email sending)
- email or push notifications
- ZIP or city level filtering, state level only
- saving or bookmarking recalls
- ingredient or allergen search


OPEN DECISIONS   [1-4 RESOLVED 2026-09-17, see bottom. 5 still open]
--------------
1. STORAGE - the big one, still unresolved.
   spec currently says "the request data is not saved".

   option A (as written): call openFDA live on every request
     - simple, no sync job
     - every login waits on an external API
     - openFDA down or slow = your app down or slow
     - shares a ~1000/day IP quota across ALL users
     - no history, no recall detail after it ages out of the query

   option B (recommended): daily sync into the database
     - the whole 2026 dataset is 861 records and fits in ONE request
       (limit=1000), so a daily sync uses ~0.1% of the quota
     - user queries served from your own db - fast, always available
     - state parsing happens once at ingest, not per request
     - openFDA outage costs freshness, not availability
     - needs: a recalls table, a sync job, a last_synced_at timestamp

   option B is why the model has a raw field - when the
   distribution_pattern parser improves you re-normalize locally
   instead of refetching everything.

2. token expiry - 24h is a guess, confirm it

3. username or email for login - still marked "to be decided" above.
   this shapes the users table, decide before the first migration.

4. DELETE /me says "cascade" - cascade what? if recall data is not
   stored per user there are no child rows. resolve this, it tells you
   whether more tables are needed.

5. which date drives the "current year" filter - report_date or
   recall_initiation_date? they differ, sometimes by months.
   sample record: initiated 20160808, reported 20161102.


================================================================
everything below added 2026-09-17
================================================================


SCOPE NOTICE - OPENFDA ONLY, DATA IS INCOMPLETE
-----------------------------------------------
for now this project uses ONE source: the openFDA food enforcement api.
the recall data it shows is NOT a complete picture of us food recalls.

what is missing: everything usda/fsis regulates - meat, poultry and
processed egg products. an fsis recall NEVER appears in openfda.
the two agencies run separate recall systems.

why fsis is not used (re-verified 2026-09-16):
  the fsis recall api at
  https://www.fsis.usda.gov/fsis/api/recall/v/1
  is free and needs no api key, but the whole fsis site sits behind
  akamai bot protection. every non-browser client gets HTTP 403:
  httpx, curl, and curl with a chrome user-agent all fail. the docs
  page and the docs pdf are 403 too. this is not an httpx problem.
  workarounds exist (tls fingerprint impersonation, headless browser,
  paid scrapers) - all rejected: they break without warning, and an
  alert system that silently stops is worse than one that admits a gap.
  even recallbench.com, which mirrors the fsis api nightly, has served
  stale fsis data since 2026-07-23 for what looks like the same reason.
  next step is to ask fsis for access, not to bypass the block.

THE GAP IS BY REGULATOR, NOT BY INGREDIENT
  do not tell users "no meat products". that is wrong and confusing,
  because fda does regulate many meat-containing foods and those DO
  show up in our data.

  measured in openfda 2026-09-17:
    "deli meat"        0 records
    "ground beef"      3 records - all plant based (impossible foods)
    "chicken"          528 records - broths, wraps, closed sandwiches
    firm "Tyson"       7 records
    all food records   29406

  the split: fsis owns raw meat, poultry, processed egg, deli meat and
  open-faced sandwiches. fda owns closed-face sandwiches, broths and
  soups with low meat content, and all plant based analogues.

  correct wording for the ui disclosure:
    "does not include usda-regulated meat, poultry or egg product
     recalls (ground beef, deli meat, raw poultry)"

  also note one real event can split across both agencies - a recalled
  ingredient can produce an fda recall for a sauce and an fsis recall
  for a meat product using it. we would only ever see the fda half.

revisit this when: fsis grants server side access, or the project
accepts a clearly labelled best effort second source with a staleness
check that falls back to the disclosure when the data goes stale.


PROJECT DIRECTION - WRAPPER FIRST, NOTIFICATIONS LATER
------------------------------------------------------
decided 2026-09-16. this replaces the account based design written
above. the sections marked SUPERSEDED stay in the file as history, they
are not the plan.

what this project is:
  a readable wrapper over the openFDA food enforcement api.
  it builds the search, it cleans up the response, it is honest about
  what it does not cover. that is the whole product.

why the accounts went away:
  register/login/me was 5 of 8 endpoints and close to none of the value.
  the value is the three location rules, the distribution_pattern
  parser, the normalizing, and the disclosure. none of that needs a user
  row. a saved default location works fine in localStorage on the client
  until there is a reason for it to live on the server.


PHASE 1 - THE WRAPPER  (now)
----------------------------
no database. no accounts. no auth. no alembic. no migrations.
requires: fastapi, httpx, pydantic, uvicorn. nothing else yet.

going in - query construction
  caller sends simple params, we build the openfda search string.
  the caller never types a field name and cannot get the three
  location rules wrong, because they are not the caller's job.

    ?state=CA&severity=CLASS_I
      ->
    search=(distribution_pattern:"CA" OR distribution_pattern:"nationwide")
           AND classification:"Class I"
           AND report_date:[<window>]
    limit=1000

coming out - the cleanup
  openfda returns ~25 fields per record, most of it noise (firm address
  lines, center_classified_date, more_code_info, event_id, an empty
  openfda:{}). we return the normalized model defined above and nothing
  else. the transforms that earn their place:
    "20160808"                -> a real date
    "Class I"                 -> CLASS_I
    "Ongoing" / "Pending"     -> ACTIVE
    "FL, MI, MS, and OH."     -> ["FL","MI","MS","OH"], is_nationwide false
    unparseable pattern       -> is_nationwide true  (rule 3)
    404                       -> []

  the response envelope carries, every time:
    disclaimer     openfda meta.disclaimer
    last_fetched   cache timestamp, so staleness is visible
    total          count
    has_critical   any returned recall is CLASS_I and ACTIVE


PHASE 1 API CONTRACT
--------------------
GET /health                 -> {"status": "ok", cache age + status}
GET /recalls?state=&severity=&status=&limit=&offset=  -> envelope + [Recall]
GET /recalls/{source_id}    -> Recall

state is now a REQUIRED param with no user default to fall back on.
still a validated enum, so a bad value is a 422 and /docs renders a
dropdown. sort order and the three location rules are unchanged.


CACHING - NO DATABASE
---------------------
the whole current window fits in ONE request (861 records for 2026,
limit is 1000), so we hold one shared snapshot and filter it in python.

  cache = { records: list[Recall], fetched_at, disclaimer, status }

  - fetch on startup in the fastapi lifespan, then a background task
    refreshes every few hours. requests never wait on openfda.
  - build the new list fully, THEN rebind. never mutate in place, a
    single rebind is atomic and readers need no lock.
  - a failed refresh KEEPS the old data, logs, leaves fetched_at alone
    so the staleness shows. never replace good data with an empty list -
    a food safety app showing zero recalls is worse than a stale one.
  - if the very first fetch fails the app still starts and /recalls
    returns 503. do not crash loop because someone else's api is down.
  - do NOT cache per query results. filtering 861 objects is microseconds
    and a key of (state,severity,status,limit,offset) buys nothing.

  quota: ~4-8 openfda calls a day total, no matter how many users.

  single worker only. --workers N gives N independent caches, N times
  the quota burn, and a different last_fetched depending on who answers.
  same reason this does not suit serverless / scale to zero.

  the frontend cache mentioned in the original spec is a separate layer -
  etag / cache-control on OUR responses. fetched_at is the natural etag.


PHASE 2 - PHONE APP + NOTIFICATIONS  (later, not now)
------------------------------------------------------
the goal: a phone alert when a MAJOR recall lands in the area the user
picked. major = CLASS_I and ACTIVE. not every new recall - alert too
often and people turn notifications off, and then the feature is worth
nothing.

this is the point where a database becomes REQUIRED, for one reason:
you cannot tell that a recall is new without remembering what you
already saw, and that memory has to survive a restart.

  recalls    the normalized rows + first_seen_at.
             new = a source_id we have never stored. that is the
             entire alert trigger.
  devices    push token + the state it watches. NOT accounts -
             a phone can register a token and a state with no email
             and no password, so we store no personal data and there
             is no breach surface.

  delivery: expo push, or fcm/apns direct. sms was considered and
  dropped - a2p 10dlc registration, per message cost, and tcpa consent
  for a feature push does for free.

  wording matters: openfda publishes a recall after fda classifies it,
  which can be days or weeks after the firm announced it. the app must
  say "newly reported", never imply breaking news.

  keep the phase 1 -> phase 2 swap cheap: the location rule must be a
  FUNCTION OVER NORMALIZED RECALL OBJECTS
      is_nationwide or state in states
  not a sql string and not a lucene string. then the only thing that
  changes is where the list comes from. this is also why the model
  keeps raw - when the parser improves you re-normalize locally.

  switch to a database when any ONE of these is true, not before:
    1. you want notifications          (automatic yes)
    2. you need more than one worker or instance
    3. you want history, trends, or permanent recall links
    4. you deploy somewhere that scales to zero
    5. you want real accounts instead of client side localStorage


OPEN DECISIONS - RESOLVED 2026-09-17
------------------------------------
1. STORAGE - resolved. in-memory snapshot cache for phase 1, database
   in phase 2. option B's argument was right, it just does not need
   postgres yet.
2. token expiry - moot, no tokens in phase 1.
3. username or email - moot, no users in phase 1.
4. DELETE /me cascade - moot, endpoint is gone. with nothing stored per
   user there was never anything to cascade.
5. report_date vs recall_initiation_date - RESOLVED 2026-09-17:
   report_date. it is when fda PUBLISHES, which is what makes "new"
   meaningful, and it is what the weekly batching lands on. everything
   downstream already assumes it - the cache window, sort=report_date:asc,
   the 30 day default, data_as_of.
   recall_initiation_date is kept on the model for display only; it can
   be months earlier and would make records appear out of order.
   the window is TRAILING (18 months), never a calendar year. on jan 2 a
   calendar filter leaves the cache nearly empty while december recalls
   are still very much active.


BUILD ORDER
-----------
1. openfda client + distribution_pattern parser, tested against the
   real examples in this file. highest risk code, no deps, pure
   functions, no network needed in the tests.
2. the normalized model + the mapper.
3. the cache: lifespan fetch, background refresh, atomic swap.
4. GET /recalls, GET /recalls/{source_id}, the envelope, /health status.
5. ship it. phase 2 only after a frontend or app exists.


WRAPPER INTERNALS - LAYOUT AND FLOW
-----------------------------------
decided 2026-09-17. where each piece lives and what it is allowed to know.

  app/
    models.py          Recall, Severity, Status, State enum
                       canonical. knows NOTHING about any source.
    openfda/
      __init__.py      fetch_recalls() -> list[Recall] + meta
                       the only thing outside this folder may import
      query.py         build the openfda search string   (pure)
      client.py        http, status codes, paging -> raw dicts
      mapper.py        one raw dict -> one Recall
      patterns.py      distribution_pattern -> (states, is_nationwide)
    filters.py         matches_state(), sort order. works on Recall,
                       any source, no openfda knowledge
    cache.py           the snapshot + refresh loop
    routes.py          /health, /recalls, /recalls/{source_id}

THE DEPENDENCY RULE
  app/openfda/ imports app/models.py.
  NOTHING outside app/openfda/ ever names an openfda field.
  grep for distribution_pattern or recall_number outside that folder -
  a hit means something leaked.

  litmus test: adding fsis later = create app/fsis/ with its own client
  and mapper, change nothing else. routes, filters, cache and models
  keep working because they only ever saw Recall. if adding a source
  forces an edit to routes.py, the boundary is wrong.

REQUEST FLOW
  user -> routes.py -> cache.py -> openfda.fetch_recalls()
                                     query -> client -> mapper
                                                          |
          filters.py (matches_state, sort) <-- Recall ----+
                                |
          routes.py wraps in the envelope -> user

  the user never triggers a live openfda call. routes read the snapshot.
  the only caller of fetch_recalls() is the cache refresh.

WHO RETURNS WHAT
  client.py   returns RAW dicts. deliberate - real responses get saved
              as test fixtures, and the mapper is then tested with no
              network at all.
  mapper.py   returns Recall.
  __init__.py does both, exposes Recall objects + meta. callers get a
              clean surface, both halves stay independently testable.

MAPPER vs PATTERNS - why they are separate files
  mapper.py   renames fields, maps the two enums, parses the dates,
              truncates the title, drops the ~12 noise fields
              (center_classified_date, more_code_info, event_id,
              the empty openfda:{}, firm address lines), keeps raw.
              boring on purpose. ~30 lines.
              owns the judgment calls: title length, a missing
              recalling_firm, what a malformed date does.

  patterns.py the free text parser. the three rules already written
              above under LOCATION FILTERING. ~15 lines and the highest
              risk code in the project.

  they differ in every way that matters:
    failure mode  - a mapper bug shows a wrong label.
                    a patterns bug HIDES a recall from someone who
                    lives in that state.
    test weight   - patterns gets dozens of cases including every odd
                    real string ever seen. mapper needs a handful.
    lifespan      - patterns keeps improving as new phrasings turn up.
                    that is exactly why Recall keeps raw: with a db in
                    phase 2 you replay the better parser over stored
                    records instead of refetching.
    portability   - fsis would need NO patterns file at all, its states
                    come back as a real list. it would still need its
                    own mapper.

PARSING IS SOURCE SPECIFIC, FILTERING IS NOT
  patterns.py is inside openfda/ because distribution_pattern is an
  openfda field.
  matches_state(recall, "CA") is outside in filters.py because it reads
  is_nationwide and states off the normalized model. it must stay a
  function over Recall objects - not a sql string, not a lucene string -
  so the phase 2 database swap changes only where the list comes from.


PAGING - THE WINDOW NO LONGER FITS IN ONE REQUEST
--------------------------------------------------
added 2026-09-17. this corrects the earlier "861 records fit in ONE
request" assumption, which is now false. measured today:

  calendar 2026          950 records   (was 861 in august, cap is 1000)
  trailing 12 months    1419 records
  trailing 18 months    2314 records

calendar-year will cross the 1000 cap on its own within months, so
paging is on the MAIN PATH, not a someday feature.

hard limits, verified against the live api 2026-09-17:
  limit=1001   -> 400 "Limit cannot exceed 1000 results for search
                  requests. Use the skip or search_after param"
  skip=26000   -> 400 "Skip value must 25000 or less."
  sort=report_date:asc  -> supported
  skip=1000 on the 18mo window -> 1000 rows, meta.results.total 2314
  skip=2000                    -> the remaining 314

THE WINDOW
  config value WINDOW_MONTHS, default 18, trailing - never a calendar
  year. on jan 2 a calendar filter leaves the cache nearly empty while
  december recalls are still active. one setting, easy to tune.

THE PAGING LOOP
  page size is always 1000 (the max - fewer pages, fewer round trips).
  always send sort=report_date:asc. skip paging without a sort is
  undefined ordering, and rows can shift between pages.

  1. fetch page 1: limit=1000, skip=0, sort=report_date:asc
  2. read meta.results.total
  3. pages_needed = ceil(total / 1000)
  4. fetch the rest with skip=1000, 2000, ...
  5. assemble, de-duping on recall_number as a belt-and-braces guard
     against a record being added mid-page-run

  sequential, not parallel. three requests a few hundred ms apart is
  polite and simple; there is nothing to win by hammering them.

  MAX_PAGES = 10 (10k records). if total exceeds that, raise instead of
  looping. either the window got too wide or something is wrong - do not
  silently pull 25k records.

  skip caps at 25000. we are nowhere near it at 2314, but if the window
  ever grows past that, switch to search_after, which the 400 message
  above names as the supported alternative.

ALL OR NOTHING
  the refresh builds a complete new list and only then rebinds the
  snapshot. if ANY page fails the whole run is discarded, the old
  snapshot stays, fetched_at is untouched, and it retries next cycle.

  a half built snapshot is worse than a stale one: a user in a state
  whose recalls happened to live on page 3 would silently see nothing
  and have no way to know. same reasoning as rule 3 in the location
  filtering - never hide a real recall.

  fetch_recalls() therefore returns the COMPLETE window or raises. it
  never returns a partial result.

QUOTA
  3 pages x ~6 refreshes a day = ~18 calls/day against an unauthenticated
  limit of roughly 1000/day per ip. still ~2%. an api key is not needed
  but raises the ceiling to ~120k/day if it ever is.


THE DEFAULT RESULT WINDOW - 30 DAYS, NOT 7
-------------------------------------------
decided 2026-09-17. two different windows, do not confuse them:

  CACHE window    18 months   what we hold in memory (2314 rows, 3 pages)
  RESULT window   30 days     what GET /recalls returns by default

7 days was the first instinct and the data killed it. openfda does not
publish continuously, it publishes in WEEKLY BATCHES, every report_date
in the last 90 days is a wednesday:

    20260805  rows=  3  events=  3
    20260812  rows= 35  events= 14
    20260819  rows= 29  events=  8
    20260826  rows= 25  events= 10
    20260902  rows= 21  events= 12
    20260909  rows= 20  events= 10

measured 2026-09-17:
    last 7 days     0 rows   <- openfda returns 404, EMPTY APP
    last 14 days   20 rows
    last 30 days   95 rows   (~40 events)
    last 90 days  250 rows   (109 events)

today is the 17th and the newest batch is the 9th. because batches land
exactly 7 days apart, a 7 day lookback has ZERO margin - it goes empty
in the day or two before each batch and stays empty whenever a publish
slips. an empty recall app looks broken, or worse, looks like good news.

30 days always spans about 4 batches, so it is never empty, and ~40
events is a reasonable first screen.

  GET /recalls              -> last 30 days
  GET /recalls?days=7       -> caller can narrow
  GET /recalls?days=180     -> caller can widen, CAPPED at the cache
                               window. asking for more than we hold must
                               not silently return less than asked.

  the envelope states the window used, in days and as actual dates, so
  "nothing found" is never ambiguous about what was searched.
  also surface the newest report_date present - that is the real
  "data as of", and it is not the same as our fetched_at.

  note the batching means a 404 from openfda on a narrow window is
  NORMAL, not an error. rule stands: 404 -> [].


GROUPING - ONE RECALL IS ONE EVENT, NOT ONE PRODUCT
----------------------------------------------------
decided 2026-09-17. openfda's enforcement endpoint returns ONE ROW PER
PRODUCT. the earlier model treated each row as a Recall keyed on
recall_number, which turns one real recall into dozens of near identical
entries.

measured over the 18 month window:
    rows                        2314
    actual events (event_id)     841
    single row events            563   (67%)
    events with >=10 rows         44   but those hold 944 rows = 41%
    biggest                      121   rows, Albertsons Companies LLC

the damage is concentrated. most recalls are one row, a handful flood
everything. with a 30 day default (~95 rows) one big event can be most
of the screen. in phase 2 it would be 121 push notifications for ONE
recall.

the albertsons event, 121 rows, is a single real recall:
    one firm, one status (Terminated), one distribution (Nationwide.),
    reason differs only by capitalisation -
      "Contains statement does not declare pecan"
      "Contains statement does not declare Pecan"

GROUPING IS SAFE - every field we filter or sort on is already constant
inside an event. counted across all 841:
    recalling_firm          0 mixed     report_date              0 mixed
    status                  0 mixed     recall_initiation_date   0 mixed
    distribution_pattern    0 mixed     state / city / country   0 mixed
    classification         11 mixed     reason_for_recall       46 mixed
  event_id present on 100% of rows. recall_number unique across the
  window. only two fields need a rule.

THE RULES
  source_id      = event_id.  recall_number moves into the product list
  severity       = the WORST class in the event. class I wins.
                   albertsons is {Class II, Class III} -> CLASS_II.
                   never let a class I hide inside a group - this feeds
                   both the severity filter and the phase 2 alert.
  status/dates/  = take from any row, they are provably identical
  distribution
  reason         = distinct reasons, compared case-insensitively so
                   pecan/Pecan collapse. keep the list when they really
                   differ (46 events).
  products       = [{recall_number, description}]  + product_count,
                   so the ui says "121 products", not 121 rows
  raw            = the LIST of source rows. keeps the re-normalize
                   promise: a better parser replays over stored rows.

  GET /recalls/{id} accepts EITHER an event_id OR a recall_number.
  recall_number is what fda shows publicly and what a user would paste.

  fallbacks: a row with no event_id becomes its own event (0 cases in
  the window today). one row in the window has NO recall_number - do not
  assume it exists.

  ?flat=true returns the ungrouped source rows. escape hatch for
  debugging and for anyone who wants what openfda actually sent.

WHERE IT LIVES
  in the mapper, inside app/openfda/. event_id is an openfda concept,
  fsis has no equivalent. the interface changes shape:
      map_records(rows) -> list[Recall]
  batch in, batch out - grouping has to see all the rows at once. this
  is the only structural change; everything downstream still sees Recall
  objects and needs no edit.

COUNTS NOW MEAN EVENTS
  total: 40 means 40 recalls, which may cover 200 products. the envelope
  reports BOTH (total events + total products) so nothing looks missing.
  limit/offset paginate EVENTS, which is what makes limit meaningful -
  20 flat rows could be a fifth of one recall.


THE ENVELOPE, LIMITS, ERRORS AND CORS
--------------------------------------
decided 2026-09-17. the small contract details that a frontend cannot
be written without.

HAS_CRITICAL - IT DESCRIBES THE AREA, NOT THE PAGE
  the old wording "any returned recall is CLASS_I and ACTIVE" was
  ambiguous twice over: page 2 could flip it to false, and a user
  filtering severity=CLASS_III would see NO warning while an active
  class I sits in their state. that is the exact case the banner exists
  for.

  has_critical  computed over STATE + WINDOW only. ignores the severity
                filter, the status filter, and pagination.
                it is a safety signal about where you live, not a
                description of what is currently on screen.
  critical_count  how many.
  has_critical_in_results  the literal reading, for the current view.
                cheap to add and it removes the ambiguity entirely.

THE ENVELOPE
  results                 list[Recall]   (events)
  total                   event count matching the filters
  total_products          product rows behind those events
  limit / offset / has_more
  window_days_requested / window_days_effective / clamped
  window_start / window_end          actual dates, so "nothing found"
                                     is never ambiguous
  data_as_of              newest report_date in the cache - the REAL
                          freshness, not the same as fetched_at
  fetched_at              when WE last refreshed
  disclaimer              openfda meta.disclaimer, every response
  has_critical / critical_count / has_critical_in_results

LIMIT AND OFFSET
  default limit 50, max 200, counted in EVENTS. a 30 day window is ~40
  events so the default shows nearly everything without paging.
  offset past the end -> empty list, not an error.
  with ?flat=true limits count ROWS instead, max 500.
  rejected: cursor pagination (overkill for an in memory list) and no
  cap at all (a public endpoint footgun - limit=100000 would serialise
  the whole snapshot).

ERROR SHAPE - ONE FORMAT EVERYWHERE
  {"error": {"code": "...", "message": "...", "detail": {...}}}

  UNKNOWN_RECALL     404  id not in the window. message states the
                          window, since the id may be real but old.
  CACHE_EMPTY        503  first fetch never succeeded. include
                          retry_after. do NOT crash loop.
  VALIDATION_ERROR   422  normalised from fastapi's own shape through an
                          exception handler, so clients parse ONE format.
  never leak an upstream openfda error body to the client. log it.

  days over the cap CLAMPS, it does not error - but it is not silent:
  window_days_requested, window_days_effective and clamped say so.
  considered and rejected: 422 on an over cap days. stricter, but
  annoying for a caller who just wants everything.

CORS
  CORSMiddleware, origins from config, GET only, no credentials.
  default "*" - defensible here: the data is public, there is no auth
  and no cookies.
  the config knob exists from day one anyway, because an explicit origin
  list becomes REQUIRED the moment anything credentialed is added.
  phase 1 assumes the client stores the saved state in localStorage, so
  without cors nothing in a browser can call this api at all.


TIME, RATE LIMITING, AND TESTS AGAINST A LIVE API
--------------------------------------------------
decided 2026-09-17.

TIME IS UTC, ALWAYS
  fetched_at and every other timestamp we generate are UTC, timezone
  aware, serialised with an explicit offset. never a naive local time -
  the server's timezone is an accident of where it happens to run.

  openfda's dates are a different thing: report_date and
  recall_initiation_date are naive CALENDAR DATES (YYYYMMDD, no time, no
  zone). keep them as dates. do not invent a midnight and do not shift
  them into a timezone - a recall was reported on a day, not at an
  instant.

  the window is computed from utc "today". near midnight a user in
  another timezone may see a window edge that differs from their local
  date by a day. that is acceptable and it is the same for everyone.

RATE LIMITING - REQUIRED, BUT AFTER THE ENDPOINTS EXIST
  NOT built yet, deliberately. build the endpoints first, add limiting
  once their real shape is known. this is a TODO, not a non goal.

  why it is still needed even though everything is served from memory:
  the endpoints are public and unauthenticated, there is nothing to stop
  one client looping /recalls, and a 200 event response is not free to
  serialise. the cost is cpu and bandwidth, not openfda quota - our
  upstream calls are already fixed at ~18/day no matter what callers do.

  when it is added:
    - per ip, on the read endpoints
    - generous. this is a public safety data api, being stingy with it
      would be the wrong failure. something like 60/min/ip.
    - 429 with retry_after, in the standard error envelope
    - /health stays unlimited so uptime checks never trip it
    - behind a proxy, honour x-forwarded-for or the limit is applied to
      the proxy and locks out everyone at once

TESTS NEVER HIT THE LIVE API
  no test makes a real openfda request. ever.
  reasons: ci would burn the shared ~1000/day ip quota, the suite would
  fail whenever fda is down or slow, and results would change weekly as
  new recalls land.

  instead: saved fixtures in tests/fixtures/ captured from real
  responses - a normal page, a 404 body, a 400 body, a multi page run,
  and the 121 row albertsons event for the grouping tests.
  httpx MockTransport serves them.

  one separate, opt-in script may hit the real api to REFRESH those
  fixtures. it is run by hand, never in ci.


PARSER - MEASURED, AND WHY FULL STATE NAMES ARE REQUIRED
---------------------------------------------------------
decided 2026-09-17. the three rules under LOCATION FILTERING still hold.
this is what running them over all 841 real events actually produced.

  code matching only          names + codes
  --------------------        ---------------
  nationwide keyword    80    nationwide keyword    80
  parsed to states     597    parsed to states     716
  rule 3 fallback      164    rule 3 fallback       45
                    (19.5%)                      (5.4%)

ONE IN FIVE RECALLS WAS BEING SHOWN TO EVERY STATE, and not because the
data was vague - because people write the names out:

    'Texas'
    'Florida and Georgia'
    'Distributed in Idaho, Oregon, and Washington'
    'Texas, Colorado, New Mexico, Kentucky, Oklahoma, Missouri,
     Arkansas, California'

rule 3 keeps that SAFE - nobody misses a recall - but it destroys
precision, and in phase 2 a texas only recall would push a notification
to all 50 states. alert fatigue is how a safety feature dies.

THE PARSER, IN ORDER
  1. nationwide keywords first. if any hit, done, is_nationwide = True.
     nationwide, nationally, all 50 states, all fifty states,
     united states, u.s., usa, across the us
     the last four were added because they were already landing on rule
     3 by luck. make it a rule, not an accident.
  2. DC special case BEFORE Washington.
     measured bug: 'Sold in Washington, DC' -> ['DC','WA'].
     match washington d.c. / washington dc first, consume it.
  3. full state names, LONGEST FIRST, consuming each match.
     order matters and is verified:
       'Distributed in West Virginia only' -> ['WV']   not VA
       'WEST VIRGINIA AND VIRGINIA'        -> ['VA','WV']
       'Shipped to Kansas and Arkansas'    -> ['AR','KS']  kansas does
                                              not match inside arkansas
       'Distributed in Mexico'             -> nationwide, not NM
     names match case-insensitively. they are words, the case trick does
     not apply to them.
  4. two letter codes on what is LEFT, still CASE SENSITIVE.
     this is the original trick and it stays: OR IN ME OK HI DE PA are
     english words, prose writes them lowercase.
  5. nothing found -> is_nationwide = True.  (rule 3, unchanged)

THE STATE ENUM MUST INCLUDE TERRITORIES
  measured in the window: DC 32, PR 8, GU 2, VI 2.
  leave them out and they parse fine and then fail validation.
  include AS and MP too, they are the same kind of thing.

KNOWN LIMITATION - UPPERCASE PROSE
  'Product distributed IN OR ME only' -> ['IN','ME','OR'].
  the case trick cannot survive prose written in caps. in the real
  window all 128 mostly-uppercase patterns were plain code lists, so
  this is rare, and it fails in the SAFE direction - a false match
  ADDS a state, which over-includes. leave it. write the test case so
  the behaviour is documented rather than discovered later.

THE 45 REMAINING FALLBACKS ARE GENUINELY UNPARSEABLE
    'Distribution for all products is limited to one direct account
     consignee (Tokyo Central)'
    'Product was sold to two(2) distributors.'
    'Will be provided when available.'
  there is no state in that text. nationwide is the right answer.

TEST CASES - ALL OF THE ABOVE ARE THE FIXTURE
  every string quoted in this section goes in the parser test suite,
  including the traps and the known limitation. they are real values
  from the live api, not invented ones.


PHASE 2 REVIEW - WHAT THE ALERT PATH ACTUALLY NEEDS
----------------------------------------------------
written 2026-09-17. still not being built now. this is the design so
that phase 1 does not paint us into a corner.

VOLUME - MEASURED, AND IT VALIDATES "CLASS I ONLY"
  over the 18 month window:
    841 events, 317 of them Class I  = 17.6 class I per month nationally
    61 of those 317 are nationwide (19%) - everyone gets those
    worst month: 28 class I events

  alerts a single user would receive:
    CA  7.4/month     NY  8.4/month     WY  4.0/month
  roughly 1-2 a week. noticeable, not spam. class I + class II would be
  about triple that, which is how you get notifications switched off.
  THRESHOLD STAYS: CLASS_I and ACTIVE.

1. THE BACKFILL STORM
   first sync stores 841 events, every one of them looks new, everyone
   gets hundreds of notifications.
   do BOTH of these, they fail differently:
     a. a seed run writes everything with alertable = false. alerting
        starts only for records first seen AFTER the seed finished.
     b. a freshness guard: never alert on anything whose report_date is
        older than ~14 days. this also covers fda republishing or
        backfilling an old record months later.

2. UPSERT, NEVER INSERT-ONLY
   openfda EDITS records in place - Ongoing -> Terminated, and
   classifications get revised. "skip if source_id exists" means status
   never updates and the app shows recalls as active forever.
   upsert on (source, source_id). preserve first_seen_at, update
   last_seen_at, keep a content hash to tell a real change from a no-op.
   considered: append every version as a new row. more faithful, better
   audit trail, but more storage and every query needs latest-version
   logic. not worth it yet.

3. RECLASSIFICATION IS A TRIGGER
   "new source_id" misses a class II upgraded to class I - a recall that
   just became serious, which is exactly when someone wants to know.
     trigger = new event that is CLASS_I
             OR existing event upgraded TO CLASS_I
   store the previous severity to detect it.
   DECIDE EXPLICITLY: an event that merely GAINS PRODUCTS is NOT a new
   alert. with grouping, albertsons style events accumulate rows over
   time and would otherwise re-fire.

4. AT LEAST ONCE DELIVERY NEEDS A DEDUPE KEY
   a crash mid send, or any retry, double notifies.
   sent_alerts table, UNIQUE (device_id, event_id, trigger_type).
   insert BEFORE sending. the duplicate insert failing is what makes
   retries safe.

5. QUIET HOURS NEED A TIMEZONE WE DO NOT HAVE
   we store a STATE, and state != timezone. some states span two.
   capture the device's utc offset at registration - the phone knows it.
   never derive it from the state.
   alternative, and the current preference: SKIP quiet hours in v1. a
   class I food recall at 2am is arguably worth waking up for, and it is
   less code. add it if people complain.

6. BATCHING
   a wednesday batch can land several class I at once (worst month: 28
   nationally).
   one sync run produces AT MOST ONE notification per device. two or
   more recalls become "3 new Class I recalls in TX" opening a list.
   considered: one notification per recall with a daily cap. better
   detail per alert, worse on bad days.

7. DEVICE LIFECYCLE
   push providers report dead tokens (unregistered). delete those rows
   on that response or the device table fills with uninstalled phones.
   cheap, and easy to forget until it is a problem.

8. SYNC CADENCE
   weekly wednesday batches mean a 6 HOUR cycle is plenty - alerts land
   within hours of publication and hourly polling buys nothing.
   the phase 1 refresh loop becomes the sync job with almost no change.

9. SILENT FAILURE GETS WORSE HERE
   in phase 1 a stalled refresh shows stale data. in phase 2 a stalled
   sync means NO ALERTS FIRE AND NOBODY NOTICES - the app looks calm,
   and calm reads as good news.
   needs an operator facing heartbeat, not just /health. same open item
   as cache monitoring, higher stakes.

10. RETENTION
   do NOT delete rows when they age out of the 18 month window. history
   is half the reason the database exists, and first_seen_at has to stay
   stable forever.


STALENESS MONITORING AND DEPLOYMENT REQUIREMENTS
-------------------------------------------------
decided 2026-09-17.

WHAT THE THRESHOLDS ACTUALLY MEASURE
  openfda publishes WEEKLY. so a snapshot being 24h old is not a data
  problem at all - it means OUR REFRESH HAS BEEN FAILING. the thresholds
  measure fetch health, not data freshness. do not confuse the two:
    fetched_at   when we last succeeded      -> fetch health
    data_as_of   newest report_date we hold  -> actual data freshness

  refresh interval   6h
    ok       age <  12h   (one missed cycle is normal, retries happen)
    stale    age >= 12h   two cycles gone. something is wrong.
    critical age >= 24h   four cycles gone. it is not coming back alone.
    empty                 the first fetch never succeeded, nothing to serve

/HEALTH
  returns, always:
    status              ok | stale | critical | empty
    fetched_at          utc
    age_seconds
    data_as_of          newest report_date in the snapshot
    event_count / product_count
    consecutive_failures
    last_failure_at     timestamp only. never the upstream error body.

  http status: 200 for ok/stale/critical, 503 ONLY for empty (we cannot
  serve at all). a stale cache still serves useful data and should not
  be reported as down.
  uptime checks that need more should match on the status FIELD, not the
  http code.
  /health is exempt from rate limiting so checks never trip it.

THE FAILURE THAT MATTERS IS ABSENCE
  a stalled refresh makes no noise. nothing errors, no request fails,
  the app just quietly serves older and older data. in phase 2 it is
  worse - no alerts fire and the app looks calm, and calm reads as good
  news.

  polling /health cannot catch this on its own, because something has to
  be doing the asking. use a DEAD MAN'S SWITCH:
    the refresh job pings a check-in url on every SUCCESS.
    if the pings stop for longer than the grace period, THAT service
    emails you.
    healthchecks.io has a free tier and is exactly this.
  it alerts on silence, which is the actual failure mode here.

  logging, in support of the above:
    one structured line per refresh - pages, records, events, duration
    failures at ERROR with the consecutive count, so a run of them is
    obvious in the log rather than buried
  no prometheus / metrics stack yet. overkill at this size.

DEPLOYMENT REQUIREMENTS - HOST CHOSEN LATER
  the host is not decided. these are the constraints it must satisfy,
  and they are not negotiable:

    - a LONG LIVED process. the background refresh loop must keep
      running with no request traffic.
    - NEVER SLEEPS / no scale to zero. an idle timeout kills the loop,
      and every cold start refetches.
    - EXACTLY ONE WORKER / one instance. N workers = N independent
      caches, N times the quota burn, and a different last_fetched
      depending on which one answers.
    - >= 256MB ram. the snapshot itself is only a few MB (2314 rows with
      raw), the rest is python.
    - outbound https to api.fda.gov.
    - restart policy that brings the process back up. on restart the
      cache is empty and /health returns 503 until the first fetch
      lands - that is expected, do not crash loop.

  this RULES OUT lambda, vercel functions, cloud run scaled to zero, and
  render's free tier (spins down after 15 min idle).
  it is satisfied by fly.io with auto_stop_machines = false, render's
  paid tier, railway, or any small vps with systemd.

  when the host IS chosen, record it here with the setting that keeps it
  awake - that setting is the whole ball game.
