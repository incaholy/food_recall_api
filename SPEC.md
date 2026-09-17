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


NORMALIZED RECALL MODEL
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
5. report_date vs recall_initiation_date - STILL OPEN. it decides what
   the query window means. related: the window should be TRAILING
   (last 12-24 months), not the calendar year. on jan 2 a calendar year
   filter leaves the cache nearly empty while december recalls are
   still very much active.


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
