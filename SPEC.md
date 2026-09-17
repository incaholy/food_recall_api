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


endpoint lists
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


AUTH
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


DATABASE
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


API CONTRACT
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


NON GOALS FOR V1
----------------
- FSIS / USDA meat poultry egg data (blocked, see limitations)
- password reset (needs email sending)
- email or push notifications
- ZIP or city level filtering, state level only
- saving or bookmarking recalls
- ingredient or allergen search


OPEN DECISIONS
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
