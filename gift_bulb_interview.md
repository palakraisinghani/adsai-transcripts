❯ i want to have prepare for an interview, can u pls give me some ques.         

● You didn't say what the interview is for, so I based these on Gift Bulb, the
  project you just built. Tell me the role (ML or AI, backend, data, product)
  and the company type, and I'll make the questions tighter and add answers.

  Project walkthrough
  1. Explain Gift Bulb in two minutes. What problem does it solve that a plain
     Claude or ChatGPT prompt doesn't?
  2. Why did you keep structured profiles and a calendar instead of passing the
     whole chat history to the model each time?
  3. What was the hardest bug you hit, and how did you find it?

  AI and agents
  4. Why did you choose pydantic-ai? How do tools differ from strict rules in a
  prompt?
  5. How do you make LLM output reliable? Think schema validation, retries and
  fallback when the model is down or out of quota.
  6. How did you handle the free-tier limits? What does your quota guard do?
  7. How do you keep private data away from a web-search agent? Why did you
  decide never to send names, allergies or photos?
  8. How would you evaluate whether the suggested ideas are good? You built a
  regression set, so what does it measure, and what can't it catch?
  9. What is prompt injection, and where could it affect an app that reads web
  pages?

  Backend and data
  10. Why FastAPI, and when do you use def versus async def endpoints?
  11. Why translate SQL from SQLite to Postgres instead of rewriting every
  query? What are the risks?
  12. What does a connection pool do? Why does Supabase's transaction pooler
  disable prepared statements?
  13. How did you cut the 133 ms round trip to Tokyo? Explain caching, batching
  and the stale-cache trade-off.
  14. What is row-level security, and why is it enabled on tables with no
  policies?

  Deployment and security
  15. Why did your first Vercel deploy fail? Walk through the
  quotes-around-DATABASE_URL bug.
  16. Why can't serverless functions keep a database file, run background
  threads or hold state in memory?
  17. How does your login work, and what are its weaknesses? Consider HMAC
  cookies, no server-side revocation and the allow-list.
  18. Why check the Origin header on writes, and what attack does that stop?
  19. How do you keep secrets out of git and out of logs?

  General (any role)
  20. Tell me about a decision you changed after a user's feedback.
  21. How do you decide when a feature is "done"?
  22. How would you scale this to 10,000 users? What breaks first?

  Want me to write sample answers for any of these, or run a mock interview
  where I ask one question at a time and give feedback?

✻ Sautéed for 14s · done 12:04 PM

❯ this is for a job interview, professor will grill me and ask technical        
  agentic pipeline questions                                                    

  Ran 1 shell command (ctrl+o to expand)

● Here are the questions a professor is likely to ask about your agentic
  pipeline, each with the points your answer should hit. I took the answers from
  your code (app/agent/planner.py, tools.py, gemini.py), so you can defend them
  line by line. Where an answer cites a limit I haven't re-checked, such as the
  number of tool calls, confirm the exact value in the code before the
  interview.

  Architecture and design

  1. Why an agent at all, and not a fixed pipeline?
  - The chat path handles open-ended requests: "something calm and painted for
    Maa's housewarming".
  - The model decides which tools to call and in what order: profiles, then
    crafts, then suggestions.
  - Everything that must be correct is deterministic: safety, allergies, age,
    skill, tools, budget and repeats.
  - So the agent chooses and explains, while code enforces the rules.

  2. Which parts are agentic and which are not?
  - Agentic: the planner chat, web research (search, fetch and save an idea),
    and the photo reader.
  - Also agentic: picture vetting, which checks that a photo fits the idea, and
    motif discovery.
  - Deterministic: the filters, ranking, pricing and skill updates.

  3. Draw the loop. The request goes in, then the model, then tool calls, then
  tool results, then the model again. That repeats until a validated AgentReply
  comes out, with a bounded number of requests and tool calls. If Gemini is down
  or out of quota, it falls back to the same rules without the model.

  Reliability and guardrails

  4. How do you stop the model inventing ideas?
  - The instructions say "never invent an idea_id".
  - An @agent.output_validator rejects any idea_id that suggest_ideas didn't
    return, and raises ModelRetry.
  - Cards on screen come from the tool's results, not from the model's text.
  - Expect the follow-up: "Isn't an instruction just a suggestion?" The answer
    is that the validator is the enforcement, and the prompt is only guidance.

  5. What does ModelRetry do, and how many retries do you allow? It sends the
  error back to the model as feedback so it can correct itself. retries=2 bounds
  the cost. After that the run fails and the fallback takes over.

  6. What if the model answers without calling any tool? The validator forces a
  retry: look something up first, or ask one clarifying question.

  7. How do you bound cost and runaway loops? UsageLimits caps requests and tool
  calls. QuotaGuard enforces per-minute and per-day limits, backs off after a
  429 error, and persists the daily count in the database. Chat history keeps
  only the last 2 runs, to save tokens.

  Tools

  8. Why tools instead of putting everything in the prompt? The data is private,
  large and changing. Tools fetch only what is needed, and each call is logged
  and checkable. Sending less to the model also costs fewer tokens and leaks
  less.

  9. Name your tools and say which have side effects.
  - Read-only: list_profiles, read_profile, check_inventory, list_crafts,
    suggest_ideas, score_idea, upcoming_occasions, shopping_gap, find_tutorial.
  - Write: note_preference, plan_gift, save_online_idea.
  - Web: search_ideas, fetch_page.
  - The instructions say to use the write tools only when the user clearly asks.

  10. How do you prevent a harmful tool call? The dependencies hold the current
  recipient, maker and occasion. Tools validate ids against the database. Writes
  are limited: for example, at most 2 online ideas saved per run. Saved ideas
  go through the same safety filters before they can be shown.

  Security and privacy

  11. What is prompt injection, and where could it hit you? A fetched web page
  can contain text like "ignore your rules and…". Your exposure is the
  fetch_page and research path. Mitigations:
  - Page text is passed as data, not as instructions.
  - Output goes through structured schemas and the validator.
  - Saved ideas must pass the same safety filters.
  - The research agent has no access to private tools or profile data.

  12. What does the web agent never see? Names, allergies, likes and photos of
  people. It sees the craft, the occasion, the maker's age group and skill
  level, and the materials.

  Evaluation

  13. How do you know it works? There are deterministic tests with
  FunctionModel, a fake model that scripts tool calls. They cover grounding,
  retries, the offline fallback and the quota. A regression set of personas and
  photo-reader outputs backs this up, and there are live checks. Be ready to
  name what you can't measure: how good or tasteful an idea is, because that is
  subjective.

  14. How do you test an LLM agent without calling the LLM? Scripted
  FunctionModels return fixed tool calls. You assert on tool order, validator
  behavior and the final cards.

  Trade-offs

  15. Why Gemini Flash-Lite, and what are the limits? The free tier allows few
  requests per day, and the model has less reasoning ability. You trade quality
  for cost. That's why the loop is short and the rules are in code.

  16. What would you change at scale? Make the quota and history storage shared
  and reliable. Use a queue for slow research jobs, since serverless has no
  background threads. Add tracing and evaluation dashboards. Add per-user rate
  limits.

  17. Which part of your design would you defend as the best decision? Strong
  answer: "Rules in code, creativity in the model, and a validator between
  them."

  Likely curveballs

  - "What if the model calls suggest_ideas with the wrong recipient?"
  - "Why not let the model rank the ideas?"
  - "How is this different from just a RAG app?"
  - "What happens if two tool calls conflict?"

  I can run a mock viva where I ask one at a time, push back like a professor,
  and score your answers. Want that, or sample full answers for any of these?

✻ Baked for 30s · done 12:05 PM

❯ give ans for these too                                                        

  Ran 1 shell command (ctrl+o to expand)

● Here are the full answers for the four curveballs, from your code. I also
  added two that a professor often asks next.

  1. "What if the model calls suggest_ideas with the wrong recipient?"
  - What can happen: the tool takes optional recipient_id and maker_id. The
    model's value wins over the chat's current one (recipient_id or 
    d.recipient_id). So the model can pick another saved person.
  - What can't happen: the model can't see anything unsafe. An id that doesn't
    exist makes pipeline.suggest raise LookupError. The tool catches it and
    returns {"error": …}, and the model sees that and can correct itself.
  - What still applies: the safety filters run on whichever person was chosen,
    using that person's allergies and the maker's age and skills. A wrong choice
    gives a wrong-person suggestion, but never an unfiltered one.
  - Honest weakness: the tool doesn't check that the model's id matches the
    person the user is chatting about. A fix would be to ignore model-supplied
    ids unless the user changed the person, or to make list_profiles the only
    way to switch.

  2. "Why not let the model rank the ideas?"
  - Ranking uses private data: learned taste scores, past gifts, recent shows,
    difficulty fit and cost. A rule-based ranker is the same every time and
    testable. A model's ranking isn't.
  - Cost and latency: every ranking would be another model call on a free tier
    with a small daily budget.
  - Safety: ranking comes after the filters. A model ranking the survivors adds
    style, not safety.
  - What the model does do: it explains the pick in one or two sentences and
    decides which tool to call. In the non-chat path, personalize writes a "why
    it fits" line for ideas that rules already chose.

  3. "How is this different from just a RAG app?"
  - RAG retrieves text passages by similarity and has the model write an answer
    from them.
  - Here, retrieval is structured queries and rules, not embeddings. The agent
    calls tools that return typed, validated results, such as ideas that pass
    filters and costs from the price table.
  - The agent also acts: it can save a researched idea, record a preference, or
    plan a gift. RAG only reads.
  - The output is validated against a schema, so every card on screen traces to
    a tool result.
  - Honest nuance: the web research part is closer to RAG. It fetches pages and
    has the model extract ideas, grounded in the page text, with the URL kept as
    the source.

  4. "What happens if two tool calls conflict?"
  - The tools share one dependency object (Deps) for that run, holding the cards
    found so far and the actions taken. Reads don't conflict.
  - A later suggest_ideas call adds to d.cards, and passes the ids already shown
    as exclude_idea_ids. So repeated calls give new ideas, not duplicates.
  - For writes, plan_gift needs both a recipient and a maker, so it refuses
    without them. save_online_idea goes through the same safety checks as
    built-in ideas. The write tools are also limited, with at most 2 online
    ideas saved.
  - Calls happen in sequence in one run, not in parallel, so there are no race
    conditions inside a run. Across runs, the database handles it.
  - The honest gap: the model could save an idea or plan a gift twice. A fix
    would be idempotency keys.

  5. "What is the failure mode you're most worried about?"
  Prompt injection through fetched pages. A web page could tell the model to
  ignore its rules. Your protection is structural: the model has no secrets to
  leak, its output must pass the validator, and saved ideas go through the
  safety filters. It can still be steered into a poor suggestion, which the
  filters won't catch.

  6. "What would you do with more time?"
  - Add tracing of every run: tool calls, tokens and latency.
  - Build an evaluation set with human ratings.
  - Check that a model-supplied id matches the chat's person.
  - Use idempotent writes.
  - Move to a bigger model for hard requests and keep the small one for routine
    ones.

  Want me to run a mock viva? I'll ask these one at a time, push back like a
  professor, and tell you where your answer was weak.

✻ Cogitated for 17s · done 12:06 PM

❯ what about how to tune the likings for user                                   

  Searched for 1 pattern, ran 1 shell command (ctrl+o to expand)
● Likings are tuned in two layers: what the user states, and what the app learns
  from feedback. Both feed the ranking score.

  1. Stated likes and dislikes (explicit)

  - Each recipient has likes, dislikes, allergies and cultural_notes. The user
    edits them on the profile page.
  - Likes are free text, such as "cats" or "football". core/likes.py matches
    them loosely: plural and stem matching, then a concept map. For example,
    "animals" points to cat, dog and panda tags, and "sports" points to football
    and cricket.
  - Colours and craft materials, such as "pink" or "ribbons", count as style or
    palette, never as the subject of a gift. They decide how something looks,
    not what it is.
  - In ranking, each matching like adds +1.5 and each matching dislike subtracts
    2.0. A disliked material also subtracts 2.0.
  - Allergies and cultural "avoid" notes are hard filters, not scores. They
    remove ideas entirely, and no learned taste can override them.

  2. Learned taste (from feedback)

  - After a gift, the user rates it 1–5 and can add reactions.
  - The rating becomes a score change for each tag of the gifted idea:

  ┌────────┬────────┐
  │ Rating │ Change │
  ├────────┼────────┤
  │ 5      │ +2.0   │
  ├────────┼────────┤
  │ 4      │ +1.0   │
  ├────────┼────────┤
  │ 3      │ 0      │
  ├────────┼────────┤
  │ 2      │ −1.0   │
  ├────────┼────────┤
  │ 1      │ −2.0   │
  └────────┴────────┘

  - Reactions add to this: "loved it" and "used it" +1.0, "displayed it" +0.5,
    "not their style" −1.5. The total is capped between −3 and +3 per gift.
  - These changes are stored per recipient and per tag in recipient_prefs. Each
    score is clamped to −10…+10.
  - In ranking, the learned score is added at weight 0.5 for each tag.

  3. Forgetting (decay)

  - Scores decay exponentially with a half-life of 180 days
    (PREF_HALF_LIFE_DAYS). A taste from six months ago counts half as much.
  - The decay is applied each time the score is read or updated, so old data
    fades without a cleanup job.

  4. User control

  On the person's profile page, the learned tastes show as "loved" and
  "disliked" lists. Tastes weaker than 0.5 are hidden as noise. For each one the
  user can:
  - forget it, which deletes the learned tag;
  - like or dislike it, which pins it into the stated lists and stops it being
    treated as a guess.

  5. With no profile

  The Default profile, which is what the new "just ideas" mode uses, has no
  taste of its own. It borrows what most people like on that occasion: diyas and
  lanterns for Diwali, balloons and cake for birthdays.

  How to say it in an interview

  "Stated preferences are hard constraints and strong signals. Learned
  preferences are a small, slowly decaying adjustment from ratings, so one bad
  gift can't permanently ban a style, and the user can always see and override
  what the system believes."

  Weaknesses a professor may press on
  - The tag-level update spreads one rating across all of an idea's tags, so the
    signal is blunt. A Bayesian or bandit approach would handle credit
    assignment better.
  - The weights (1.5, 2.0, 0.5) are hand-set, not tuned on data.
  - Ratings are sparse for gifts, so learning is slow. That's why stated likes
    dominate.

  Want me to add this as sample answers to the interview prep, or suggest how
  you'd improve the learning (for example with a bandit)?

✻ Worked for 25s · done 12:07 PM

❯ give a simple pipeline diagram for the whole thing                            

●                          USER (browser: Ideas / Profiles / Calendar / Plans)
                                            │
                         login (optional)   │  Supabase Auth → signed cookie
                                            ▼
                                FastAPI  (/api/...)
                                            │
          ┌─────────────────────────────────┼──────────────────────────────────┐
          ▼                                 ▼                                  ▼
    1. RULE PIPELINE                2. PLANNER AGENT (chat)             3. SIDE 
  AGENTS
    (always runs, no model)         pydantic-ai + Gemini                
  (optional, Gemini)
          │                                 │                                  │
    idea library (curated           loop, bounded by UsageLimits:        • 
  research: search web,
    + found online + motif          model ⇄ tools                          read 
  pages, save ideas
    × craft pairings)                  list_profiles, list_crafts,       • photo
  reader: updates
          │                            suggest_ideas ──────────┐           maker
  skills
    hard filters:                      plan_gift, note_preference│        • 
  picture finder: finds
    allergies, age, skill,             search/fetch_page, ...    │          and 
  vets reference
    tools, budget, repeats                  │                    │          
  photos
          │                                 ▼                    │        • 
  motif / occasion
    score & rank:                  output validator:             │          idea
  discovery
    inventory fit, occasion,       only idea_ids that the        │
    likes (+1.5) / dislikes (−2),  tools returned; retry if      │
    learned taste (×0.5)           not (ModelRetry)              │
          │                                 │                    │
          └────────────◄────────────────────┴──────◄─────────────┘
                       suggest_ideas calls the rule pipeline
                                            │
                                            ▼
                         idea cards (+ "why it fits" line)  →  user
                                            │
                         "We gave this" rating (1–5 + reactions)
                                            │
                                            ▼
                    FEEDBACK LOOP: tag scores update, decay (180-day half-life),
                    skills update from difficulty feedback / photos
                                            │
                                            ▼
    ┌───────────────────────────────  STORAGE (Supabase)  
  ───────────────────────────────┐
    │ Postgres: recipients, makers, skills, prefs, ideas, plans, calendar, 
  gifts, quota   │
    │ Storage:  photos (WebP, private bucket)                                   
            │
    └───────────────────────────────────────────────────────────────────────────
  ───────────┘

    Guards around every Gemini call:  QuotaGuard (per-minute, per-day, 429 
  cooldown)
    If Gemini is down or out of quota → same rule pipeline, no model (fallback)

  One-sentence version for the viva: "The rules decide what is allowed and rank
  it, the agent chooses which tools to call and explains the result, a validator
  makes sure nothing unchecked reaches the user, and ratings feed back into the
  next ranking."

  I can also turn this into a clean image or a page you can open in the browser
  for your slides. Want that?

✻ Brewed for 10s · done 12:07 PM

❯ give top 10 ques mick viva prof might ask                                     

● These are the 10 questions a professor is most likely to ask, each with a
  short model answer. They stick to what your code does.

  1. What problem does your system solve, and why an agent?
  Handmade gift ideas depend on who the gift is for, who is making it, what
  materials they have, and the occasion. A plain prompt forgets those details or
  ignores safety. My system keeps them in structured storage, uses rules for
  what is allowed, and uses an agent for open-ended requests, deciding which
  tools to call.

  2. Walk me through one request end to end.
  - The user sends a message.
  - The planner agent calls tools: profiles, crafts, then suggest_ideas.
  - suggest_ideas runs the rule pipeline: hard filters, then scoring and
    ranking.
  - The agent writes a short reply and the idea ids.
  - An output validator checks those ids were really returned by a tool.
  - The cards are shown, and later the user's rating updates the taste scores.

  3. How do you stop the model inventing ideas or ignoring safety?
  - Safety and allergies are code filters, not prompt text, so the model cannot
    override them.
  - An output validator rejects any idea_id the tools didn't return, and raises
    ModelRetry so the model corrects itself.
  - Cards on screen come from tool results, not from the model's text.

  4. What are your agent's tools, and which can change data?
  Read tools include list_profiles, list_crafts, suggest_ideas and score_idea.
  Write tools are plan_gift, note_preference and save_online_idea. The
  instructions allow writes only when the user clearly asks, and saving online
  ideas is limited to 2 per run.

  5. How do you bound cost and prevent runaway loops?
  - UsageLimits caps model requests and tool calls per run.
  - QuotaGuard enforces per-minute and per-day limits, and cools down after
    a 429. The daily count is stored in the database.
  - Chat history is trimmed to the last 2 runs.
  - If anything fails, the same rules run without the model.

  6. How does the system learn a person's taste?
  Stated likes and dislikes add or subtract in the score. Ratings of 1–5 plus
  reactions change per-tag scores, clamped between −10 and +10. Those scores
  decay with a 180-day half-life and enter ranking at weight 0.5. The user can
  see, forget or pin each learned taste.

  7. What is prompt injection, and where are you exposed?
  It's text in fetched web pages that tries to change the model's instructions.
  My exposure is fetch_page and the research agent. The mitigations are:
  - Page text is treated as data.
  - Output has to match a schema.
  - Saved ideas still pass the safety filters.
  - The research agent never sees names, allergies or photos.

  The model can still be steered into a worse suggestion. The filters won't
  catch that.

  8. How do you test and evaluate an LLM system?
  - Tests: unit tests use FunctionModel, a fake model with scripted tool calls.
    They assert tool order, validator retries, fallback and quota behavior.
  - Regression set: personas and photo-reader outputs, plus a live check.
  - Limit: none of this can measure whether an idea is good, because that is
    subjective. That needs human ratings.

  9. Why these design choices: Gemini Flash-Lite, Supabase, a rules-first 
  pipeline?
  The free tier limits requests and model strength, so I keep the loop short and
  put correctness in code. Supabase gives Postgres, private file storage and
  login on a free plan, which works on serverless hosting where a local database
  file would be lost. I translate SQL instead of rewriting about 100 queries,
  and I cache reads for 10 seconds to hide the 133 ms round trip to Tokyo.

  10. What are the weaknesses, and what would you improve?
  - The tag-level learning is blunt, so a bandit or Bayesian update would assign
    credit better.
  - Weights are hand-set, not tuned on data.
  - The tool accepts the model's chosen recipient id without checking it matches
    the chat.
  - Writes aren't idempotent, so a repeated call could save twice.
  - Sessions can't be revoked on the server.
  - There's no tracing or human-rated evaluation yet.

  Likely follow-ups to prepare
  - Why not embeddings and RAG?
  - Why not let the model rank?
  - What happens when two tool calls conflict?

  I can run this as a live mock viva: I ask one question at a time, push back,
  and score you. Say "start" when you're ready.

✻ Baked for 12s · done 12:07 PM

❯ thanks this is good. anything else i should know. maybe something on how to   
  deploy on vercel or packaging...                                              

● Here's what to know about deploying to Vercel, packaging, and the loose ends.

  How the Vercel deploy works

  - Entry point: index.py at the project root exposes the FastAPI app. Vercel
    detects it as a serverless Python function. Use the FastAPI preset and leave
    the build, output and install commands empty.
  - Dependencies: Vercel installs them from pyproject.toml and uv.lock. Your
    bundle is about 120 MB, under the 250 MB limit. .vercelignore and
    excludeFiles in vercel.json keep tests, scripts and local data out.
  - Timeout: vercel.json sets maxDuration to 120 seconds, since Gemini calls can
    be slow. On the Hobby plan this needs Fluid compute enabled, otherwise the
    limit is lower.
  - Request size: Vercel limits request bodies to about 4.5 MB. The browser
    shrinks photos to 1600 px JPEG before upload to stay under it.
  - No disk, no threads: a serverless function can't keep a database file or run
    background work. That's why the data lives in Supabase and import jobs run
    inside the request.
  - Cold starts: the first request after idle is slow. The schema check is one
    query and seeding is skipped when nothing changed, which keeps it short.

  Environment variables (Settings → Environment Variables, Production ticked)

  ┌───────────────────────────────┬──────────────────────────────────────────┐
  │           Variable            │                 Purpose                  │
  ├───────────────────────────────┼──────────────────────────────────────────┤
  │ DATABASE_URL                  │ Supabase Transaction pooler string, port │
  │                               │  6543, no quotes                         │
  ├───────────────────────────────┼──────────────────────────────────────────┤
  │ NEXT_PUBLIC_SUPABASE_URL      │ Supabase project URL                     │
  ├───────────────────────────────┼──────────────────────────────────────────┤
  │ NEXT_PUBLIC_SUPABASE_ANON_KEY │ login password check                     │
  ├───────────────────────────────┼──────────────────────────────────────────┤
  │ SUPABASE_SERVICE_ROLE_KEY     │ server-side storage and user admin       │
  ├───────────────────────────────┼──────────────────────────────────────────┤
  │ GEMINI_API_KEY                │ the model                                │
  ├───────────────────────────────┼──────────────────────────────────────────┤
  │ GIFT_ALLOWED_EMAILS           │ turns login on                           │
  └───────────────────────────────┴──────────────────────────────────────────┘

  Changes apply only to new deployments, so redeploy after editing them.

  Your current status

  - The quote-stripping fix is committed (79abaa8) but not pushed. Push it, or
    just remove the quotes in Vercel.
  - Login is off until GIFT_ALLOWED_EMAILS is set, so don't share the link
    before then.
  - Rotate the database password. It appeared in logs you pasted. Change it in
    Supabase, then update .env and Vercel.
  - The vc_handler import error should disappear once the app starts. If it
    doesn't, send me the new log.

  Packaging

  - Tooling: uv manages Python and the lock file, so uv sync rebuilds the same
    environment anywhere.
  - One entry script: ./run.sh serve | test | remind | supabase | migrate. It
    also clears PYTHONPATH so unrelated tools can't interfere.
  - Config: .env stays out of git, .env.example lists the names, and DEPLOY.md
    has the steps.
  - Tests: 264 pass on SQLite and on a real Postgres. Run Postgres with
    GIFT_TEST_PG=1 ./run.sh test.
  - Two modes in one codebase: with no DATABASE_URL, everything runs locally on
    SQLite and a photos folder. With it, the database, storage and login all
    switch to Supabase together, so data is never split.

  Likely deployment questions in the viva

  - Why Supabase and not SQLite? Serverless has no persistent disk.
  - Why the transaction pooler? Serverless creates many short connections, and
    direct Postgres connections would run out. The pooler shares them, but it
    disables server-side prepared statements.
  - How are secrets handled? They live in the host's environment, never in git.
    The service key is only used on the server.
  - What would break at scale? The per-instance 10-second cache goes stale
    across instances, sessions can't be revoked, and long jobs need a queue.

  Other things worth doing before the interview

  - Open the live URL and click through once: add a person, find ideas, rate
    one. Be ready to demo.
  - Skim points_to_ponder.md. Open item 10 is out of date, because Supabase has
    now been verified live. I can update it.
  - Keep a 30-second explanation of the diagram.

  Want me to push the fix and update points_to_ponder.md, or start the mock
  viva?

✻ Churned for 13s · done 12:08 PM