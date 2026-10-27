 ▐▛███▛█   Claude Code v2.1.310
▝▜██████▀  Opus 5.5 · Google Vertex AI
 ▝▝   ▝▝   ~/palak/TalkToClaude/server


❯ I have a job interview. and the interviewer will ask me critical questions on 
  this project. 1. tell me how to pitch this project 2. what technical          
  questions are they likely to ask me. Help me prepare well.                    

  Read 2 files, ran 5 shell commands (ctrl+o to expand)

⏺ I've read the core server code (agent process, speech recognition, cleanup
  pass, undo snapshots, auth). Next I'm skimming the browser side and the
  WebSocket server, then I'll write up your prep.

  Ran 2 shell commands (ctrl+o to expand)

⏺ ClaudeCall: interview prep

  I went through the actual code: the agent process manager, speech recognition,
  the transcript cleanup pass, undo snapshots, auth, routing, echo handling and
  the build pipeline. Everything below comes from what's in the repo, so you
  can defend it.

  ---

  1. The pitch

  30-second version (memorize this)

  ▎ "ClaudeCall lets you run a team of AI coding agents by voice, hands-free. 
  ▎ You talk to an assistant called Ada. She sets up projects, and each project 
  ▎ gets its own Claude Code agent working in its own folder. One agent talks to
  ▎ you at a time while the others keep working in the background. When one of 
  ▎ them needs something, like permission for a risky command or an answer to a 
  ▎ question, it waits in a queue, and Ada tells you in a quiet moment. Speech 
  ▎ recognition runs locally, it works from your phone, and it ships as a Mac 
  ▎ app, a Linux package and an Android APK."

  2-minute version (problem → insight → how → proof)

  1. Problem: Coding agents are powerful, but they keep you at the keyboard
     watching one terminal. If you run several at once, you're constantly
     switching tabs and approving prompts.
  2. Insight: Treat it like a phone call with a team. Only one voice talks at a
     time. Everyone else works quietly, and interruptions are queued instead of
     shouted.
  3. How:
     - A Node server runs one warm claude -p process per agent and talks to it
       over Claude Code's stream-json protocol.
     - The browser does voice activity detection (Silero VAD). The server
       transcribes locally with Whisper, and a small Haiku pass fixes misheard
       names and technical terms.
     - Agents reply with tags like <say>, <screen> and <question>, which a
       streaming parser turns into speech, canvas content and queued questions.
  4. Proof points:
     - I measured Whisper against Moonshine on Indian-accented speech: names
       came out right 30% of the time with Moonshine vs 84–92% with Whisper.
     - The cleanup pass answers in about 1s, after I found that turning Haiku's
       thinking off cut its latency from 2–10s.
     - Undo snapshots use git plumbing and never touch your branch or staging
       area.
     - About 190 tests, and CI builds and smoke-tests every release.

  Close with what you learned: "Most of the hard problems weren't AI problems.
  They were real-time voice UX problems: echo, barge-in, turn-taking, and
  mishearing names."

  ---

  2. Architecture (be able to draw this on a whiteboard)

  Browser/phone (PWA)                     Node server (your machine)
  ──────────────────                      ──────────────────────────────
  Mic → Silero VAD ──16-bit PCM over WS──▶ hasSpeech() → Whisper (ONNX, CPU, q8)
                                            prompted with team names
                                                ↓
  route.js: who is this for?  ◀── text ──  Refiner (Haiku, no tools, 2.5s 
  timeout)
  attention.js: queue/yes-no                    ↓
  echo.js / noise.js filters               Call (call.ts): Ada + roster, one "on
  the line"
                                                ↓
  voice.js sentence queue ◀── sentences ── ReplyParser (streaming 
  <say>/<screen>/actions)
  earcons, stereo pan (macOS `say`)             ↑
  canvas: markdown/mermaid/charts ◀──── ClaudeSession × N  (claude -p, 
  stream-json stdin/stdout)
                                           checkpoint.ts (git snapshot before 
  each turn)

  ---

  3. Likely technical questions, with answer outlines

  Architecture and agents

  Q: How do you control Claude Code programmatically?
  - I spawn claude -p --input-format stream-json --output-format stream-json 
    --include-partial-messages and keep it warm, one process per agent.
  - User messages go in as JSON lines on stdin. Events come back as JSON lines
    on stdout: text deltas, tool use, tool results, the final result with its
    cost.
  - Control requests on the same channel handle interrupt, set_permission_mode,
    set_model, side_question (Claude Code's /btw), and permission prompts
    (can_use_tool). All of that is in server/agent.ts.

  Q: Why a subprocess and not the Agent SDK or the API directly?
  - Users get Claude Code exactly as they know it: their login and subscription,
    their tools, CLAUDE.md, hooks and MCP servers.
  - Sessions are stored where the CLI stores them, so you can resume a
    conversation started in the terminal (sessions.ts reads ~/.claude/projects).
  - I never handle their credentials.
  - Trade-off: I depend on a CLI protocol I don't control. I handle that
    defensively, e.g. replying with an error to any control request I don't
    support so the CLI never hangs.

  Q: What happens if you speak while an agent is working?
  - send() uses "newest wins" by default: queued messages are dropped and the
    running turn is interrupted.
  - With interrupt: false the message waits behind the current work, so a build
    or test run isn't killed just because you spoke. Delegated tasks use that
    mode.

  Q: How do you stop agents talking over each other?
  - Structurally: at most one agent is "on the line". Background agents' <say>
    output isn't spoken. Their final sentence becomes a report that Ada relays.
  - On the client there's a single sentence queue (voice.js), and each change of
    speaker is announced with that agent's earcon.

  Q: How does Ada actually do things?
  - She writes control tags (<new>, <open>, <delegate>, <switch>, <park>,
    <close>, <remember>).
  - The streaming parser picks them up and the server carries them out. Tags
    that move the floor wait until her turn ends, so she isn't cut off
    mid-sentence.
  - Project agents can only use <handoff>, so an agent that mentions "the
    <close> button" is just talking.

  Q: How does the parser handle streaming?
  - It's a state machine over the deltas. It holds back up to 12 characters in
    case a tag is only partly arrived.
  - <say> text is split into sentences as they complete, so speech starts before
    the reply has finished.
  - An action tag is only acted on once it's complete. If it never closes (over
    600 characters, or a <say> starts first), it's treated as plain text, so
    nothing is held back forever.

  Speech

  Q: Why Whisper over Moonshine, and why run it locally?
  - I built an eval (scripts/eval-names.ts) on names spoken in Indian and other
    accents, with realistic mic and room variation.
  - Moonshine got 30% right and Whisper base 84% (small 92%), at roughly 0.45s
    vs 0.05s per utterance.
  - Whisper accepts a prompt. I pass the team's names as previous-context tokens
    (<|startofprev|>), so "Ada" stops becoming "He had a".
  - Running locally means it's free, private and works offline, with no API key.

  Q: How do you deal with Whisper hallucinations?
  - Silent or noisy clips never reach Whisper: there must be about a 12 dB gap
    between loud and quiet frames.
  - Known subtitle phrases ("Thanks for watching") are filtered out.
  - Loops are collapsed to one copy.
  - Output is capped to what a person could say in the clip (max_new_tokens ≈ 9
    tokens per second, and results over 5 words per second are rejected).
  - If it just reads back the prompt, that's discarded.

  Q: What is the Refiner and why is it safe?
  - A Haiku session with no tools, no MCP and no saved session, with thinking
    off.
  - It fixes misheard terms using the project's file names as context ("pie
    test" → pytest).
  - Guardrails:
    - 2.5s timeout, after which the raw text goes through.
    - Replies are rejected if their length is under 0.5× or over 1.6× the
      original.
    - Lines of two words or fewer skip it.
    - British spellings are restored if the model "fixed" them.
    - The agent also sees the raw transcript when the two differ.
  - The session is recycled every 30 uses so its growing history doesn't slow it
    down.

  Q: What about echo, when the speaker's voice comes back into the mic?
  - echo.js matches ordered word pairs against what was recently spoken, not
    single words, because single words like "the" and "it" would make every
    short reply look like echo.
  - It removes only runs of at least 3 echoed words, so "the build passed wait 
    stop that" keeps the interruption.

  Q: Coughs?
  - noise.js combines how long the VAD was confident it heard a voice with
    whether the transcript is a real word. Commands like "stop" and team names
    are always let through.

  Q: Voice biometrics? (voiceprint.ts)
  - MFCC features with mean normalization, matched with dynamic time warping
    against templates you record yourself. It's speaker-dependent on purpose: it
    learns your accent, which is exactly where general recognizers fail.

  Safety and security

  Q: This gives an AI a shell on my machine. How is that secured?
  - It binds to localhost by default.
  - There's one access code, sent as a WebSocket subprotocol so it never appears
    in URLs, history or proxy logs.
  - It's checked with timingSafeEqual on a case-insensitive comparison, with
    backoff after 10 failed attempts.
  - The LAN mode requires HTTPS (the mic needs a secure context anyway).
    Tailscale is recommended.
  - Media is served through unguessable /media/<token> links because plain HTTP
    requests can't carry the code.
  - A saved API key is stored with file mode 0600.

  Q: How do permissions work?
  - Each agent has an on-the-line permission mode and a background mode (default
    auto, where Claude Code's classifier asks only about risky actions).
  - Permission prompts pause the agent and join the queue. You answer by voice:
    "allow", "deny", "always".
  - "Always" applies the CLI's suggested rules for that session only.
    Irreversible actions need "confirm", not just "yes".

  Q: How does undo work?
  - Before each turn, the agent's folder is snapshotted using a temporary 
    GIT_INDEX_FILE + write-tree + commit-tree, stored under refs/claudecall/.
  - Your branch, staging area and stash are never touched. Files in .gitignore
    are skipped.
  - Restore checks out the changed files and deletes files that were added.
  - Limitation: folders that aren't git repos have no undo.

  Reliability

  - Resumed session no longer exists: if the process dies before producing any
    real output, it restarts fresh and replays the message.
  - Spawn errors (no claude on PATH) are caught, otherwise they'd crash the
    server.
  - Events from a replaced or parked process are ignored, using this.child !== 
    child checks.
  - State across restarts: the roster is saved to disk. Ada's conversation
    starts fresh each run on purpose. What carries over is a compact "notebook"
    of decisions, project status and recent events, because carrying the whole
    routing chatter forward would bury what matters.

  Testing and shipping

  - About 190 tests with Node's built-in runner. The decision logic
    (attention.js, route.js) has no DOM and no timers specifically so it can be
    unit-tested.
  - GitHub Actions on each version tag: run the tests, build the DMG and Linux
    tarballs, smoke-test each one (including loading speech recognition), then
    publish.
  - I got the DMG down from 139 MB to 47 MB.

  ---

  4. The critical questions (where interviewers push)

  Question: "How much of this did AI write?" (likely, given the commit trailers)
  Honest, strong answer: Be upfront: "I built it with Claude Code as my pair. My

  job was architecture, the measurements, and deciding the behavior." Then
  prove it: explain the Whisper prompting trick or the git-index undo without
  notes. Rehearse these until you can.
  ────────────────────────────────────────
  Question: Latency end to end?
  Honest, strong answer: Rough budget: VAD end-of-speech wait + Whisper ~0.45s +

  Refiner ~1s + time until Claude starts replying + TTS on the first sentence.
  To reduce it: sentence-level streaming, warm processes, a pre-warmed STT
  graph, and skipping the Refiner for short lines. Improvement idea: run the
  Refiner in parallel with routing.
  ────────────────────────────────────────
  Question: Why not a cloud STT like Deepgram?
  Honest, strong answer: Privacy, no key, offline, free. The trade-off is CPU
  use, and Intel Macs aren't supported.
  ────────────────────────────────────────
  Question: Scaling: 10 agents?
  Honest, strong answer: Each agent is a Node CLI process, so memory adds up.
  Agents can be parked to free their process (the session ID is kept). Whisper
  jobs run one at a time through a promise chain. The real limit is your
  attention, which is the whole point of the queue.
  ────────────────────────────────────────
  Question: What's the weakest part?
  Honest, strong answer: Depending on an undocumented-ish CLI control protocol.
  Name recognition is still probabilistic, and routing deliberately falls back
  to whoever is on the line ("a misheard name must never redirect an
  instruction"). macOS-only stereo voices. The Mac app isn't signed with a
  Developer ID yet.
  ────────────────────────────────────────
  Question: What would you do next?
  Honest, strong answer: Code signing, a Windows build, streaming STT for lower
  latency, and evals for routing accuracy.
  ────────────────────────────────────────
  Question: Biggest bug or hardest problem?
  Honest, strong answer: Pick one and tell it as a story. Good candidates: the
  echo self-interrupt (set membership → word pairs), Whisper inventing "Thank
  you." from silence, or Haiku taking 2–10s until thinking was turned off.

  ---

  5. Before the interview

  1. Draw the architecture diagram from memory twice.
  2. Rehearse three deep dives at 2 minutes each: stream-json process control,
     Whisper prompting plus hallucination guards, and git-plumbing undo.
  3. Have a demo ready (a recording is safer than live), showing switching
     agents by voice and a background agent asking permission.
  4. Know your numbers: 30/84/92%, ~0.45s and ~1s, 2.5s timeout, 139→47 MB, ~190
     tests, roughly 14k lines.
  5. Repo hygiene: package.json still says "name": "buddy" and the env vars use
     BUDDY_*. Be ready to explain that it was renamed from the Buddy prototype.
     You also have uncommitted changes in call.ts, agent.ts, projects.ts and
     CENTRAL.md, so commit them or know what they are if you share the repo.

  If you'd like, I can run a mock interview: I ask the questions one at a time,
  push back on your answers, and tell you where they were weak.

✻ Crunched for 1m 30s · done Monday, 3:01 PM

❯ Explain more on how the voice is getting processed? what are the tags the     
  agent recognizes? how did we deal with echo? Can you explain more on what     
  these tags are - how they are used to control what happens ?? what is the     
  design here ?                                                                 

  Ran 8 shell commands (ctrl+o to expand)

⏺ Below is how it works in the code, in three parts: the voice pipeline, the
  echo handling, and the tag protocol. File references are included so you can
  open them before the interview.

  ---

  Part 1: How your voice gets processed

  Your voice passes through seven stages, split between the browser and the
  server.

   BROWSER                                              SERVER (Node)
   ───────                                              ─────────────
   ① Mic (getUserMedia: echoCancellation,
     noiseSuppression, autoGainControl)
          ↓
   ② Silero VAD v5 (runs in the browser via ONNX/WASM)
     decides when you START and STOP talking
          ↓ the whole utterance as 16-bit PCM, 16 kHz
   ③ WebSocket: a {type:"segment", id} message,
     then the raw audio as a binary frame  ─────────▶  ④ Int16 → Float32
                                                         hasSpeech()? 
  (energy-contrast check)
                                                         Whisper, prompted with 
  the team's names
                                                         unhallucinate() filters
                                                         voiceprint match (MFCC 
  + DTW) on short clips
          ◀───────────────── transcript + voice match ──┘
   ⑤ Page-side filters: noise → echo → routing
          ↓
   ⑥ Text goes to an agent  ────────────────────────▶  ⑦ Refiner (Haiku) cleans 
  the text
                                                         → agent's stdin as 
  stream-json

  ② Voice activity detection (web/listen.js)

  - Silero VAD gives every 32 ms frame a probability that it's speech.
  - Speech starts above the positiveSpeechThreshold (0.55 at normal
    sensitivity). Your turn ends after 900 ms of silence (redemptionMs), so
    short pauses mid-sentence don't cut you off.
  - preSpeechPadMs: 400 keeps the 400 ms before speech was detected, so the
    first syllable isn't lost.
  - minSpeechMs: 250 is set deliberately low because a bare name like "Ada"
    lasts about a quarter of a second.
  - It counts voiced frames per segment. This number is used later to tell a
    cough from a word.
  - If the VAD can't load, it falls back to Chrome's built-in speech
    recognition.

  Why the VAD runs in the browser: only finished utterances are sent, not a
  continuous stream. That saves bandwidth and server CPU, and the browser knows
  the moment you start speaking, which is what lets it cut off a voice that's
  talking.

  ③ Transport (web/app.js → server/index.ts)

  - A JSON control message {type:"segment", id} goes first, then the audio as a
    binary WebSocket frame of 16-bit PCM.
  - A small but real detail: the ws library can hand you a buffer at an odd byte
    offset, and Int16Array needs 2-byte alignment, so the server copies the
    buffer when it has to (index.ts, around line 330).

  ④ Transcription (server/stt.ts)

  1. hasSpeech(): the loudest 5% of frames must be at least 4× (~12 dB) louder
     than the quietest 10%. Real speech has quiet gaps around the words. Steady
     noise doesn't, so it's dropped before Whisper ever sees it.
  2. tighten(): trims the silence at both ends, keeping 250 ms of margin and
     padding to at least 600 ms, because very short clips crash the model's
     first convolution.
  3. Whisper with a prompt: the team's names are passed as <|startofprev|> Ada, 
     Juno, Rex. decoder tokens. Whisper treats them as the preceding context, so
     those words become the likely reading of an ambiguous sound. This is the
     step that took name accuracy on Indian-accented speech from 30% (Moonshine)
     to 84–92%.
  4. Token cap: max_new_tokens ≈ seconds × 9 + 10, so a one-second clip can't
     turn into a 40-word loop.
  5. unhallucinate():
     - drops stock subtitle phrases ("Thanks for watching")
     - collapses a phrase repeated 3+ times to one copy
     - rejects output with more words than a person could say in the clip (over
       5 per second)
     - rejects output that is just the prompt read back
  6. Jobs run one at a time through a promise chain (this.chain), which keeps
     CPU use predictable.

  At the same time, any clip up to 6 s is compared against the name recordings
  you taught it (voiceprint.ts: MFCC features + dynamic time warping). That
  catches names Whisper got wrong, like "photo" for Otto.

  ⑤ Page-side filters, in order (heardText() in web/app.js)

  1. Voiceprint override: if your voice matched a taught name, a line of three
     words or fewer switches to that agent.
  2. Noise filter (noise.js): drops a short segment with little voiced time
     whose transcript is a filler or a stage direction ("Uh.", "[coughs]").
     Commands and names are always let through (mustKeep), because "stop" is
     short too.
  3. Carry-over: if you're still talking, the pieces are joined into one line.
  4. Echo guard: covered in Part 2.
  5. Routing (route.js): decides whether the line is a page command, an answer
     to a question, a switch to someone by name, or text for whoever has the
     floor.

  ⑦ The Refiner (server/refine.ts)

  - Haiku, with no tools and thinking turned off. It fixes "pie test" → pytest
    using the agent's recent file names and commands as context.
  - The agent receives cleaned text [speech recognition heard: "raw text"], so
    it can fall back to the raw words if the cleanup went wrong.

  Voice out (brief, but you may get asked)

  - Agents stream their reply. Each finished sentence is spoken right away,
    without waiting for the whole reply.
  - There's one shared sentence queue (web/voice.js), and each change of speaker
    plays that agent's earcon first.
  - It records exactly which sentences you heard. If you cut in, the agent's
    next message starts with:
    [cut in: the user said pause 6s ago; the last thing they heard was "The 
    cache is stale because"; they did not hear 2 more sentences: "…"; your 
    previous work was stopped]
    So the agent knows what you missed, not just that you interrupted.

  ---

  Part 2: How echo is handled

  The problem

  On laptop speakers the mic hears the agent's own voice. Then:
  1. Its own words get transcribed and sent back as if you'd said them, so it
     talks to itself.
  2. The VAD never hears silence while the voice is playing, so your segment
     never ends and an interruption arrives late.
  3. If you talk over it, one recording holds both voices: "the build passed and
     wait stop that".

  The browser's echo cancellation doesn't reliably remove speech-synthesis
  output (the code comment: "the browser cannot cancel its own speech
  synthesis"), so the app has to handle it itself.

  Five layers of defense

  Layer: 1. Headphones mode
  Where: onSpeechStart
  What it does: If headphones are on, no echo is possible, so the voice is cut 
  the moment you start speaking. This is the best experience.
  ────────────────────────────────────────
  Layer: 2. Browser echo cancellation
  Where: getUserMedia
  What it does: echoCancellation: true. Helps, but not enough on its own.
  ────────────────────────────────────────
  Layer: 3. Stricter VAD while a voice plays
  Where: listen.js, strict()
  What it does: On speakers, while a voice is playing, the speech threshold goes

  up by 0.1 and the minimum speech length by 50 ms. Segments are also forced to
   end every 2.5 s (FLUSH_MS), so an interruption gets through promptly instead
   of waiting for the voice to finish.
  ────────────────────────────────────────
  Layer: 4. Remove echoed words
  Where: echo.js, stripEcho()
  What it does: Removes runs of 4 or more words that the voice actually said in
  that order, and keeps the rest. "the build passed and wait stop that" becomes
   "wait stop that".
  ────────────────────────────────────────
  Layer: 5. Reject pure echo
  Where: echo.js, isEchoOf()
  What it does: If 70% or more of the utterance is made of word pairs the voice
  spoke in the last few seconds, the whole utterance is dropped as echo.

  The key design point (worth saying in the interview)

  ▎ "The first version checked whether the words in an utterance appeared in 
  ▎ what the voice had just said. That broke immediately: within a sentence or 
  ▎ two it has said 'the', 'it', 'is', 'that', so any short reply you made 
  ▎ scored as echo. I switched to ordered word pairs (bigrams). Echo comes back 
  ▎ as a near-verbatim run, so its pairs match. Your own reply uses the same 
  ▎ common words in a different order, so its pairs don't. A single misheard 
  ▎ word in a long echo only breaks the pairs on either side of it, so the 
  ▎ detection is robust to transcription errors."

  Safety valves

  - Commands and yes/no answers are never treated as echo (mustKeep), so "stop"
    always works.
  - Saying it again overrides the guard: if you repeat the same line within a
    short window, it goes through. A wrong guess never leaves you stuck.
  - Echo memory is limited by time and count (ECHO_MEMORY_MS). Old speech can't
    echo, so it's forgotten.
  - The page tells you what happened: "(heard that as my own echo) — say it
    again and I will take it".

  Trade-off to admit: this is text-level echo handling, not acoustic. A real fix
  would be acoustic echo cancellation fed with the synthesized audio as its
  reference. That's possible on macOS, where the server makes the audio with say
  and plays it through Web Audio, but not with the browser's speech synthesis.

  ---

  Part 3: The tags

  The core design idea

  ▎ The LLM's text output is the control channel. The agent writes a stream of 
  ▎ text, and inline XML-like tags in it say where each piece goes (speakers, 
  ▎ screen, question queue) and what should happen (create an agent, switch, 
  ▎ delegate). A streaming parser on the server sorts that stream as it arrives.

  It works like a markup language for a voice call: the model "renders" its
  reply into channels.

  Two families of tags

  A. Presentation tags (all agents): where the content goes

  Tag: <say>…</say>
  Goes to: Spoken aloud
  Notes: Spoken sentence by sentence while the reply is still streaming. Only
  spoken if the agent is on the line. In the background, its last sentence
  becomes the one-line report.
  ────────────────────────────────────────
  Tag: <screen>…</screen>
  Goes to: The canvas (side panel)
  Notes: Markdown, plus ```mermaid diagrams, ```chart JSON charts, and images,
  GIFs and videos by path or link. Never used in audio-only mode.
  ────────────────────────────────────────
  Tag: <question>…</question>
  Goes to: Spoken and queued
  Notes: A background agent needs a decision. It's saved as pendingQuestion and
  asked when you're free.
  ────────────────────────────────────────
  Tag: (text outside any tag)
  Goes to: "Behind the scenes" feed
  Notes: The agent's working notes. Never spoken.

  Fallback: if an agent writes no <say> at all, the server speaks the first two
  sentences and puts the full text on the canvas. A model that forgets the
  format still produces something sensible.

  B. Action tags: what happens

  Ada (the assistant) can use all of them. Project agents can only use <handoff>
  and <voice> (their own voice only).

  ┌─────────────────────────────────────┬───────────────────────────────────┐
  │                 Tag                 │              Effect               │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │                                     │ Makes the folder, starts a new    │
  │ <new name="Juno"                    │ Claude Code agent, and gives it   │
  │ folder="todo-app">brief</new>       │ the brief to work on in the       │
  │                                     │ background.                       │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │ <open path="~/code/api" name="Rex"  │ Starts an agent on an existing    │
  │ resume="last">                      │ folder, optionally resuming its   │
  │                                     │ last Claude session.              │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │                                     │ Hands work to an agent. It waits  │
  │ <delegate to="rex">task</delegate>  │ behind current work and never     │
  │                                     │ interrupts it.                    │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │ <switch to="rex"/>                  │ Gives Rex the floor.              │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │                                     │ Stops the process but keeps the   │
  │ <park to="rex"/> / <close           │ session / removes the agent from  │
  │ to="rex"/>                          │ the team (ClaudeCall asks you to  │
  │                                     │ confirm).                         │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │ <handoff to="ada">note</handoff>    │ Any agent: "putting you through   │
  │                                     │ to Ada", with a note.             │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │ <voice to="juno" name="Karen"       │ Changes how an agent sounds.      │
  │ speed="1.1" side="left"/>           │                                   │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │ <setting name="chimes"              │ Toggles a setting on your page.   │
  │ value="off"/>                       │                                   │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │ <call do="mute"/>                   │ Mute, hold, pause or end the      │
  │                                     │ call.                             │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │ <remember>…</remember> /            │ Ada's notebook that carries over  │
  │ <forget>…</forget>                  │ between calls.                    │
  ├─────────────────────────────────────┼───────────────────────────────────┤
  │ <control …>                         │ Mode, model or effort changes.    │
  └─────────────────────────────────────┴───────────────────────────────────┘

  Optional attributes: model="opus|sonnet|haiku|fable" and focus="true".

  How a tag is executed (server/parser.ts → server/call.ts)

  claude stdout (text deltas)
     ↓  ClaudeSession emits {kind:"text"}
  ReplyParser.feed(delta)            ← a state machine: none | say | screen | 
  question
     ↓  ParsedEvents: sentence | screen | aside | question | action
  Call.parsed()
     ├─ sentence → say()      → page speaks it (if on the line)
     ├─ screen   → canvas
     ├─ aside    → behind-the-scenes feed
     ├─ question → p.pendingQuestion
     └─ action   → is the sender Ada?
           no  → only handoff/voice allowed; anything else is shown as plain 
  text
           yes → switch/call?  → DEFERRED until the turn ends
                 otherwise     → RUN NOW (in order, on a promise chain)

  Parser details that interviewers like

  1. Holding back partial tags: when the buffer ends in < followed by fewer than
     12 characters (the length of <delegate  or <question), the parser waits
     instead of emitting the text, because <sa may become <say>.
  2. Holding back partial closing tags: inside <say>, the last </say>-length − 1
     characters are held, so </sa is never spoken.
  3. Splitting sentences while streaming: <say> text is split into sentences as
     they complete. Speech starts after the first sentence, not after the whole
     reply. This matters a lot for perceived latency.
  4. Action tags only run when complete: the parser waits for > (or </delegate>)
     before acting.
  5. No getting stuck: if a tag never closes (over 600 characters, or a <say>
     begins first), it was never really an action. Its opening is released as
     plain text, so nothing is held back forever.
  6. Permissions are enforced in the parser: new ReplyParser({ actions: [...] })
     builds the regex from only the tags that agent may use. A project agent
     that writes "click the <close> button" isn't even parsed as an action.

  When actions run (a deliberate decision)

  - Most actions run immediately, while Ada is still talking. If you interrupt
    her mid-sentence, the project she just created still exists. Interrupting
    her speech doesn't undo work.
  - switch and call wait until her turn ends (FLOOR_ACTIONS). Otherwise the
    floor would move or the call would mute in the middle of her sentence.
  - handoff runs at the end of the agent's turn, after it has finished saying
    its piece.
  - Feedback: if a tag fails (unknown agent, name taken), Ada gets a [result of 
    your actions] … message and explains in one sentence. The loop is closed.

  Messages go into the agents too

  The protocol works in both directions. Agents also receive bracketed notes
  that the server adds to messages:

  ┌───────────────────────────┬─────────────────────────────────────────────┐
  │           Note            │                   Meaning                   │
  ├───────────────────────────┼─────────────────────────────────────────────┤
  │ [call · visual+audio ·    │ Mode and whether its speech is spoken. Sent │
  │ you are on the line]      │  only when it changes.                      │
  ├───────────────────────────┼─────────────────────────────────────────────┤
  │ [cut in: … heard … did    │ What you heard before interrupting.         │
  │ not hear …]               │                                             │
  ├───────────────────────────┼─────────────────────────────────────────────┤
  │ [speech recognition       │ The raw transcript, when the cleanup        │
  │ heard: "…"]               │ changed it.                                 │
  ├───────────────────────────┼─────────────────────────────────────────────┤
  │ [projects: …] / [team     │ Ada's view of the team, sent as diffs to    │
  │ changes since your last   │ save tokens, with a full refresh every so   │
  │ message: …]               │ often and after context compaction.         │
  ├───────────────────────────┼─────────────────────────────────────────────┤
  │ [from earlier calls — …]  │ Ada's notebook.                             │
  ├───────────────────────────┼─────────────────────────────────────────────┤
  │ [result of your actions], │ Results of her tags.                        │
  │  [the lead confirmed…]    │                                             │
  └───────────────────────────┴─────────────────────────────────────────────┘

  The agent writes tags out; the server adds bracketed notes in. The system
  prompts (skill/claude-call/VOICE.md for all agents, CENTRAL.md for Ada) define
  the format.

  A worked example (good to draw on a whiteboard)

  You say: "Ada, start a todo app in React and have someone build it."

  Ada streams:
  <say>Juno's on it, building a React to-do app. I'll tell you when it's 
  done.</say>
  <new name="Juno" folder="todo-app">Build a React to-do app with add, complete 
  and
  delete. Done means it runs with npm run dev and has tests.</new>
  1. The parser emits sentence events and you hear Ada while she's still writing
     the tag.
  2. </new> closes, so an action event is emitted. The sender is Ada, and new
     isn't in FLOOR_ACTIONS, so it runs now: the folder is created, the name is
     checked so it can't be confused with others, a new claude -p process starts
     with cwd=todo-app, and the brief is sent.
  3. Juno works in the background. Its <say> output isn't spoken. If it hits a
     risky command, the permission prompt joins the queue. In a quiet moment Ada
     says "Juno needs you", and you answer "allow".
  4. When Juno finishes, its final sentence becomes its one-line report, and Ada
     sees it in her next [team changes…] note.

  ---

  The critical question: "Why tags rather than tool calls or MCP?"

  Be ready for this one. It's the most likely pushback.

  Reasons for tags:
  1. Streaming. A tool call is a separate content block, and the model's turn
     usually pauses on it. Tags arrive inline in the text stream, so speech
     starts at the first sentence and actions run mid-reply without an extra
     round trip.
  2. One channel for everything. Speaking, showing and acting are interleaved in
     the order the model writes them, which matches how a person talks: "Sure,
     starting that now" [does it].
  3. Works with an unmodified Claude Code CLI. No custom MCP server to install
     or register for each agent. Project agents keep their normal Claude Code
     tools for the actual coding.
  4. Easy to sandbox. A project agent's parser simply doesn't recognize Ada's
     tags.
  5. Fails safely. A malformed tag becomes plain text, not a crash or a hang.

  Downsides to admit (this is what makes you sound senior):
  - No schema validation. Tool calls get typed arguments from the API. Tags are
    parsed with a regex, so attributes are free-form strings. That's handled by
    validating on the server and reporting [result of your actions] back.
  - The model can get the format wrong, e.g. forget <say>. That's covered by the
    fallback path and the prompt rules.
  - Prompt injection: text that ends up in Ada's output could contain tags. The
    mitigation is that only Ada's output is parsed for control tags and project
    agents' tags are inert. The residual risk is Ada being manipulated into
    echoing tags from content she reads. Destructive actions like close require
    your explicit confirmation.
  - What you'd consider next: MCP tools for Ada's actions (typed and validated)
    while keeping <say>/<screen> as tags for streaming. That would be a
    reasonable hybrid.

  ---

  Practice these three explanations out loud:
  1. Word pairs vs single words for echo
  2. Holding back partial tags while streaming
  3. Most actions run now, switch/call wait for the turn to end

  Each is a small, concrete decision that shows you understood the problem.

✻ Churned for 1m 45s · done Monday, 4:19 PM

❯ So each agent writes its own xml file that gets processed and is continuously 
  processed on . OR do all the agents contribute to one xml file? what happens  
  to a <say> line if the agent is not currently active? What happens to a       
  <question > line in the same case? what happens to a <say > line if or any    
  spoken line if it gets interrupted ?                                          

  Ran 3 shell commands (ctrl+o to expand)

⏺ Short answers first, then the detail behind each.

  1. No XML file exists, shared or per agent. Each agent's reply is a live text
     stream read straight from its claude process's stdout. Each agent has its
     own parser, and a new one is created for every turn. Nothing is written to
     disk to be parsed, and agents never share a document.
  2. A <say> from a background agent is never spoken. It's shown quietly on the
     page. Its first sentence after its last tool call becomes that agent's
     one-line report.
  3. A <question> from a background agent is saved and queued. Ada tells you in
     a quiet moment that someone has a question, and reads it out only when you
     say yes or "next".
  4. When speech is interrupted, the page records exactly what you heard.
     "Pause" resumes from the same word. Speaking over it either resumes the
     sentence (if what you said was noise) or replaces the rest of the turn. The
     agent is then told what you heard and what you missed.

  ---

  1. Streams, not files

   Juno's claude process ──stdout──▶ ClaudeSession(Juno) ──▶ ReplyParser(Juno, 
  turn 7) ──┐
   Rex's  claude process ──stdout──▶ ClaudeSession(Rex)  ──▶ ReplyParser(Rex, 
  turn 3)  ──┼─▶ Call ──WebSocket──▶ page
   Ada's  claude process ──stdout──▶ ClaudeSession(Ada)  ──▶ ReplyParser(Ada, 
  turn 12) ──┘

  - Each agent is a separate claude -p process. Its stdout is JSON lines: one
    event per line, such as a text delta, a tool call or a tool result
    (server/agent.ts, onLine()).
  - The "tags" are just characters inside those text deltas. <say>Hel might
    arrive in one chunk and lo.</say> in the next. That's why the parser is a
    streaming state machine that holds back partial tags. It's never parsing a
    complete document.
  - One parser per agent per turn: parserFor(turn) in call.ts:137 keeps a
    Map<turn, ReplyParser> on each agent. A new turn gets a new parser, and the
    old one is deleted when that turn ends.
  - Which tags a parser recognizes depends on who it's for: Ada's parser accepts
    every action tag. A project agent's parser is built with only ["handoff", 
    "voice"].
  - Events from one agent are kept in order by a promise chain per agent
    (p.chain), because some steps are async. Different agents' events interleave
    freely, since they're independent.
  - Persistence is separate from parsing. Every event is also appended to a
    per-agent JSONL transcript (transcript.ts) so a crash doesn't lose the call.
    That's a log, not something that gets parsed.

  How to phrase it in the interview:

  ▎ "It's not a document, it's a protocol over a stream. Each agent's stdout 
  ▎ feeds its own incremental parser, and the parser turns text deltas into 
  ▎ typed events: sentence, screen, aside, question, action. The call 
  ▎ orchestrator then decides what each event means, depending on whether that 
  ▎ agent currently has the floor."

  ---

  2. A <say> from an agent that isn't on the line

  The parser doesn't know or care who has the floor. It emits sentence events
  either way. The decision is made one step later, in Call.say() (call.ts,
  around line 1180):

  this.emit(p, { type: "say", project: id, turn, text: s, spoken: this.focused
  === p.config.id });

  The message carries a spoken flag, and the page acts on it:

  ┌──────────────────┬────────────────────┬────────────────────────────────┐
  │                  │    On the line     │  In the background (spoken:    │
  │                  │   (spoken: true)   │             false)             │
  ├──────────────────┼────────────────────┼────────────────────────────────┤
  │ Spoken aloud?    │ Yes, right away,   │ No                             │
  │                  │ in its voice       │                                │
  ├──────────────────┼────────────────────┼────────────────────────────────┤
  │ Shown on the     │ Yes                │ Yes, styled as quiet in the    │
  │ page?            │                    │ feed (feedSaid)                │
  ├──────────────────┼────────────────────┼────────────────────────────────┤
  │ Counted toward   │ Yes                │ Yes. This is what the          │
  │ its report?      │                    │ background sentences are for.  │
  └──────────────────┴────────────────────┴────────────────────────────────┘

  What happens to the words afterwards:
  - Each sentence is added to p.said. After every tool call, p.answer is reset,
    so it only holds what the agent said after it finished working, which is its
    conclusion.
  - When the turn ends, p.report = the first sentence of that conclusion. The
    design note in the code says the report is "the opening of its final answer
    — the verdict — not its last line".
  - That report:
    - appears on the agent's card on the page
    - is saved to Ada's notebook (notebook.status), so it carries over between
      calls
    - reaches Ada in her next [team changes since your last message: …] note, so
      she can answer "what did Juno find?" without asking Juno.
  - The agent is told about this. The call header says you are in the 
    background, your words are not spoken, and VOICE.md tells it to say less and
    make its last sentence work as a stand-alone report.

  Why it's designed this way: two voices must never talk over each other.
  Instead of muting a background agent's output, its speech is turned into data:
  a report you can ask for at any time.

  ---

  3. A <question> from an agent that isn't on the line

  <question>Should the API use Postgres or SQLite?</question>
         ↓ parser: sentence events (handled like <say>) + one whole "question" 
  event when the tag closes
         ↓ Call: p.pendingQuestion = "Should the API use Postgres or SQLite?"
         ↓ end of turn: gets an id (q-juno-7), agent state → "blocked"
         ↓ notify { kind:"question", needsReply:true } → page queue
         ↓ attention.js: waits until you've been quiet, then Ada says "Juno has 
  a question."
         ↓ you: "yes" / "next"           → it's read out
         ↓ you: "tell Juno use Postgres" → route.js "tell" → server tell() → 
  sent to Juno
                                           without moving the floor; queue item 
  removed

  The details:
  - The question is spoken and reported whole. The parser emits it as sentences
    (like <say>) and also as one question event when </question> closes. In the
    background the sentences aren't spoken, so only the queued copy matters.
  - It's never read out uninvited. In a quiet moment Ada only says that someone
    is waiting. The details come when you say yes or "next". The same rule is in
    your memory notes and in attention.js.
  - A question without the tag still counts. If a background turn ends with a
    report ending in ?, it's treated as a question anyway (call.ts:1343). This
    is a safety net for when the model forgets the tag.
  - On the line, there's nothing to queue. You heard it directly, so
    pendingQuestion is cleared (call.ts:1344).
  - It's withdrawn if it goes stale. Each question has an id. If the agent moves
    on, or you answer some other way, the page removes the queue item when the
    ids no longer match (app.js:1335).
  - The page notifies you even when it's in the background. If the page is
    hidden (another app, a locked phone), a questions or permission request also
    triggers a system notification.

  The difference from <say>: a background <say> becomes a report you can ask
  about. A background <question> becomes a queue item that waits for your 
  answer, and the agent shows as "has a question for you".

  ---

  4. When spoken output is interrupted

  Interruption works at two levels at once: the voice on the page and the 
  agent's turn on the server.

  Level 1: the voice (web/voice.js)

  Each queued sentence remembers how far it has been spoken, using word-boundary
  events from the speech engine.

  ┌──────────────────┬──────────────────────────────────────────────────────┐
  │   What you do    │              What happens to the voice               │
  ├──────────────────┼──────────────────────────────────────────────────────┤
  │                  │ hold(): stops immediately and rewinds to the start   │
  │ "Pause"          │ of the current word, putting the rest back at the    │
  │                  │ front of the queue. "Resume" continues from that     │
  │                  │ word.                                                │
  ├──────────────────┼──────────────────────────────────────────────────────┤
  │ Start talking,   │ hold() the moment the VAD detects speech (ducked =   │
  │ with headphones  │ true). It's a hold, not a stop: if what you said     │
  │ on               │ turns out to be noise or echo, the voice resumes.    │
  ├──────────────────┼──────────────────────────────────────────────────────┤
  │ Start talking,   │ It keeps playing (the mic can't tell you from the    │
  │ on speakers      │ voice yet). Your words are separated from its words  │
  │                  │ by the echo stripping.                               │
  ├──────────────────┼──────────────────────────────────────────────────────┤
  │ What you said    │ clear(): the queue is emptied and the rest of that   │
  │ goes to an agent │ turn is never spoken.                                │
  ├──────────────────┼──────────────────────────────────────────────────────┤
  │ "Stop"           │ Voice cleared, and the agent's work stops too (see   │
  │                  │ below).                                              │
  └──────────────────┴──────────────────────────────────────────────────────┘

  What you heard is recorded precisely: heard[] holds the sentences spoken to
  the end, and snapshot() returns { heard: last sentence or partial sentence, 
  unheard: [the rest] }. When cut off mid-sentence, only the words actually
  spoken are kept as "said". The rest never left the speaker, so it can't echo
  back either.

  Level 2: the agent's turn (server/agent.ts, server/call.ts)

  When your interruption becomes a new message to that agent:
  1. send() defaults to "newest wins": anything still queued is dropped (each
     dropped message is reported, so its card closes), and an interrupt control
     request goes to the CLI.
  2. The rest of the old turn is discarded. The turn number goes into an
     interrupted set, and any text that arrives before the CLI's final result
     event is ignored (live = false). The old reply doesn't keep talking after
     you've cut in.
  3. The parser isn't flushed. parser.end() only runs for turns that finished
     normally (call.ts:1327). A half-written <say> is dropped rather than spoken
     as a fragment.
  4. The report isn't updated from an interrupted turn, so a half-finished
     sentence never becomes "what Juno found".
  5. Your new message includes the cut-in note (context.ts, cutIn()):
  [cut in: the user talked over you; the last thing they heard was "The cache is
  stale because";
   they did not hear 2 more sentences: "It never expires. Want the fix?"; your 
  previous work was stopped]
     The kind field distinguishes paused (with how many seconds ago), followup
     ("said this while you were working") and talking over. There's also a
     variant for "your previous work was not stopped and has now finished; do
     not redo it."

  Exceptions to "newest wins"

  - Delegated work and follow-ups can queue instead of interrupting (send(text, 
    { interrupt: false })). Saying something to a busy agent never kills a
    running build or test suite by accident.
  - You can ask a busy agent something without stopping it. That uses a side
    question (Claude Code's /btw, the side_question control request), answered
    from its conversation so far while its work continues.
  - Continuing your own sentence: if you pause mid-thought and then keep
    talking, the second half is merged with the first and replaces it
    (replaceQueued). It isn't treated as an interruption.

  Special case: interrupting Ada while her tags are running

  - Her action tags run as soon as each one closes, so a cut-off can't undo a
    project she already created.
  - noteCutOff() then tells her, in her next message:
    - "you were cut off, but these were already carried out: new Juno. Do not 
      write them again"
    - "you were cut off while still writing <delegate to=rex>, so it was NOT 
      carried out. If it is still wanted, write it again in full"

    The parser's unfinished() method reports the half-written tag.

  The design principle to say out loud:

  ▎ "An interruption has to answer three questions: what did the listener 
  ▎ actually hear, what did the agent actually do, and what should be thrown 
  ▎ away. The voice layer tracks what was heard down to the word, the session 
  ▎ layer discards the rest of the cut-off turn, and the orchestrator tells the 
  ▎ agent both in plain language. The agent can then pick up exactly where you 
  ▎ left off instead of repeating itself or redoing work."

✻ Brewed for 1m 4s · done Monday, 4:38 PM

※ recap: You're preparing for a job interview on ClaudeCall, and I've explained the pitch, voice pipeline, echo handling, and tag design. Next, practice saying those explanations aloud, or I can run a mock interview. (disable recaps in /config)

❯ CAn you explain the entire working properly and slowly. I want to be able to  
  explain the entire agentic pipeline.                                          

  Ran 3 shell commands (ctrl+o to expand)

⏺ Here is the whole system as one story. I follow a single example from the
  moment the app starts to the moment an agent's work is finished. After each
  step there's a "why" note, because interviewers usually ask why something was
  built that way, not just what it does.

  The running example:

  ▎ You say: "Ada, have someone build a weather command-line tool in Python."
  ▎ Later the new agent needs permission to install a package, you approve it by
  ▎ voice, and then you switch over to talk to that agent directly.

  ---

  Part 0: The cast

  Before the story, here are the six pieces involved.

  Piece: The lead
  What it is: You, the person on the call
  Where it lives: —
  ────────────────────────────────────────
  Piece: The page
  What it is: The web app in your browser, phone or Mac window. It owns the mic,

  the speakers, the screen and the decisions about who you're talking to.
  Where it lives: web/app.js, listen.js, route.js, attention.js, voice.js
  ────────────────────────────────────────
  Piece: The server
  What it is: A Node.js program on your computer. It owns speech recognition and

  all the agents.
  Where it lives: server/index.ts, call.ts
  ────────────────────────────────────────
  Piece: Ada
  What it is: The assistant: a Claude Code process in your home folder
  (~/ClaudeCall). She doesn't code. She creates agents, hands out work and
  keeps track of the team.
  Where it lives: Prompt: skill/claude-call/CENTRAL.md
  ────────────────────────────────────────
  Piece: Project agents
  What it is: Juno, Rex and so on. Each one is a separate Claude Code process
  working in its own project folder, doing the real coding.
  Where it lives: Prompt: skill/claude-call/VOICE.md
  ────────────────────────────────────────
  Piece: The Refiner
  What it is: A small Claude Haiku process with no tools. Its only job is fixing

  misheard words.
  Where it lives: server/refine.ts

  The one-sentence architecture:

  ▎ The page handles ears, mouth and turn-taking. The server handles 
  ▎ transcription and runs one Claude Code process per agent. Agents talk back 
  ▎ in a small tag language that the server turns into speech, screen content 
  ▎ and actions.

   ┌──────────── PAGE (browser) ────────────┐        ┌──────────────── SERVER 
  (Node) ────────────────┐
   │ mic → VAD ──audio────────────────────────────▶   │ Whisper (speech → text) 
                         │
   │                       ◀──text─────────────────── │                         
                         │
   │ filters (noise, echo)                  │        │                          
                        │
   │ route.js: who is this for? ──message─────────▶   │ Refiner (Haiku) → Call 
  (call.ts)               │
   │                                        │        │    ├─ Ada      (claude 
  process, ~/ClaudeCall)   │
   │ attention.js: queue of who needs you   │        │    ├─ Juno     (claude 
  process, weather-cli/)   │
   │ voice.js: one mouth, sentence queue ◀──events─── │    └─ Rex ...  (claude 
  process, its folder)     │
   │ canvas: markdown, diagrams, charts     │        │  stdout → ReplyParser → 
  say/screen/action       │
   └────────────────────────────────────────┘        
  └─────────────────────────────────────────────────┘
                       one WebSocket connection between them

  ---

  Part 1: Startup (before you say anything)

  Step 1. The server starts (server/index.ts)
  - It listens on 127.0.0.1 (your computer only) by default. LAN mode switches
    to HTTPS, because browsers only allow the mic on a secure address.
  - It creates an access code, unless you set one.
  - It starts loading Whisper and runs it once on a second of silence. The first
    run compiles the model, so it's done before anyone speaks.

  Step 2. The page connects
  - It opens a WebSocket and sends the access code as a WebSocket subprotocol,
    not in the URL.
  - The server checks the code with a constant-time comparison. It also backs
    off after repeated wrong guesses.
  - The page asks whether Claude Code is installed and signed in (claude 
    --version, claude auth status --json). If it isn't, it shows the setup
    steps.

  Step 3. The call starts (Call.start() in call.ts)
  - Ada's process is spawned. That's claude -p --input-format stream-json 
    --output-format stream-json ... with her system prompt appended. Her working
    folder is the home folder.
  - Every saved agent on the roster is also started now. A Claude Code process
    takes a few seconds to start, so warming them while you're still saying
    hello means your first question is answered at full speed.
  - The Refiner is warmed for the same reason.
  - The server sends the roster (who exists, their state, voices, models) to the
    page.

  ▎ Why one long-running process per agent, instead of starting a new one per 
  ▎ message? Starting takes seconds, and a long-running process keeps the full 
  ▎ conversation context. The stream-json input format lets you keep feeding it 
  ▎ messages on stdin indefinitely.

  ---

  Part 2: You speak, and sound becomes text

  You say: "Ada, have someone build a weather command-line tool in Python."

  Step 4. The browser decides when you started and stopped (web/listen.js)
  - The mic is opened with the browser's echo cancellation, noise suppression
    and automatic gain turned on.
  - Silero VAD (a small neural network running inside the browser) scores every
    32 ms of audio for "is this speech?"
  - When the score rises above a threshold, you've started. The page reacts
    right away: if a voice is playing and you have headphones, it pauses that
    voice.
  - After 900 ms of silence, you've stopped. The whole utterance becomes one
    clip of 16 kHz, 16-bit audio.

  Step 5. The audio goes to the server
  - First a small JSON message ({type: "segment", id}), then the raw audio as a
    binary WebSocket frame.

  Step 6. The server transcribes it (server/stt.ts)
  1. Is there speech in it at all? Loud frames must be about 4× louder than
     quiet ones. If not, it's noise and gets dropped, because Whisper makes up
     phrases like "Thank you." from noise.
  2. Trim the silence at both ends.
  3. Run Whisper, with the team's names as a prompt ("Ada, Juno, Rex."). That
     makes those names the likely reading of an unclear sound. It's the main fix
     for accented names.
  4. Clean up Whisper's known mistakes: stock subtitle phrases, repeated loops,
     and more words than a person could say in that time.
  5. At the same time, if the clip is short, it's compared against your recorded
     name samples (MFCC + DTW, voiceprint.ts). That catches cases like "photo"
     being written for Otto.

  The text goes back to the page: "Ada, have someone build a weather command 
  line tool in Python."

  ---

  Part 3: The page decides who the line is for

  This all happens in the browser, in heardText() in app.js and in route.js.

  Step 7. Filters
  1. Noise: was it a cough or a laugh? Check how long the VAD heard a voice and
     whether the text is a real word.
  2. Echo: is this the agent's own voice coming back through the speakers? This
     uses word-pair matching (covered last time).
  3. Still talking? If you paused mid-thought, the pieces are joined into one
     line.

  Step 8. Routing (route.js)
  The line becomes one of these outcomes:

  ┌───────────────────────────┬──────────┬─────────────────────────────────┐
  │         You said          │  Result  │          What happens           │
  ├───────────────────────────┼──────────┼─────────────────────────────────┤
  │ "pause", "stop",          │          │ Handled by the page itself. No  │
  │ "repeat", "next", "hold   │ local    │ AI involved.                    │
  │ my calls"                 │          │                                 │
  ├───────────────────────────┼──────────┼─────────────────────────────────┤
  │ "yes" / "no" right after  │ answer   │ Goes to attention.js as the     │
  │ Ada asked something       │          │ answer.                         │
  ├───────────────────────────┼──────────┼─────────────────────────────────┤
  │ "allow" / "deny"          │ grant /  │ Answers a waiting permission    │
  │                           │ deny     │ request.                        │
  ├───────────────────────────┼──────────┼─────────────────────────────────┤
  │ "Juno" on its own, or     │ focus    │ Juno gets the floor.            │
  │ "put me through to Juno"  │          │                                 │
  ├───────────────────────────┼──────────┼─────────────────────────────────┤
  │ "Ada, …" or "Juno, run    │          │ Floor moves to them, and the    │
  │ the tests"                │ address  │ rest of the line is their       │
  │                           │          │ message.                        │
  ├───────────────────────────┼──────────┼─────────────────────────────────┤
  │ "tell Juno use Celsius"   │ tell     │ Passed to Juno without moving   │
  │                           │          │ the floor.                      │
  ├───────────────────────────┼──────────┼─────────────────────────────────┤
  │ anything else             │ ask      │ Goes to whoever has the floor   │
  │                           │          │ right now.                      │
  └───────────────────────────┴──────────┴─────────────────────────────────┘

  A key safety rule: names are matched exactly when the line contains a command,
  and only loosely when the whole line is just a name. If a name is misheard,
  the instruction goes to whoever you were already talking to. That's the safe
  failure: it can never send an instruction to the wrong agent.

  In our example, the line starts with "Ada", so it's an address: send "have 
  someone build…" to Ada.

  ---

  Part 4: The message travels to the agent

  Step 9. The Refiner cleans it (server/refine.ts)
  - The Haiku process gets {heard, to, team, terms (file names and commands from
    the agent's recent work), agentLastSaid, accent}.
  - It fixes misheard terms ("pie test" → pytest), removes "um"s and adds
    punctuation. It never adds meaning.
  - Safety rails:
    - a 2.5 s timeout, after which your raw words go through as heard
    - replies much shorter or longer than the original are rejected
    - lines of two words or fewer skip it
  - If it changed something, the agent receives both versions: cleaned text 
    [speech recognition heard: "raw text"].

  Step 10. The server adds context to the message (Call.ask() + preface())
  An agent's conversation is its only memory, so the server puts what it needs
  in front of your words. To save tokens, only what it hasn't already been told
  is included:

  ┌────────────┬───────────────────────────┬───────────────────────────────┐
  │ Note added │           When            │            Example            │
  ├────────────┼───────────────────────────┼───────────────────────────────┤
  │ Call       │ First message, then only  │ [call · visual+audio · you    │
  │ header     │ when it changes           │ are on the line]              │
  ├────────────┼───────────────────────────┼───────────────────────────────┤
  │ Team       │ Full list on her first    │ [projects: Rex — working,     │
  │ status     │ message and every 15      │ last said "tests pass"…] or   │
  │ (Ada only) │ turns. Only the changes   │ [team changes since your last │
  │            │ in between.               │  message: …]                  │
  ├────────────┼───────────────────────────┼───────────────────────────────┤
  │ Notebook   │ First message of a call   │ [from earlier calls — …]      │
  │ (Ada only) │                           │                               │
  ├────────────┼───────────────────────────┼───────────────────────────────┤
  │ Cut-in     │ If you interrupted it     │ [cut in: the last thing they  │
  │ note       │                           │ heard was "…"]                │
  ├────────────┼───────────────────────────┼───────────────────────────────┤
  │ Server     │ When there's something to │ [note from ClaudeCall: …]     │
  │ notes      │  tell it                  │                               │
  └────────────┴───────────────────────────┴───────────────────────────────┘

  Step 11. A snapshot for undo (project agents only, checkpoint.ts)
  - Before a project agent starts new work, its folder is snapshotted with git
    plumbing (a temporary index plus a commit object stored under
    refs/claudecall/). Your branch and staging area are never touched.
  - The last five snapshots are kept. "Undo what Juno just did" restores the
    folder.

  Step 12. The message is written to the process's stdin (ClaudeSession.send())
  {"type":"user","message":{"role":"user","content":"[call · …] [projects: none 
  yet] have someone build a weather…"}}
  - If the agent is idle, the turn starts.
  - If it's busy, the default is "newest wins": queued messages are dropped and
    the running turn is interrupted. Work handed out by Ada and follow-ups wait 
    in the queue instead, so a running build is never killed just because you
    spoke.

  Step 13. A filler if it's slow
  - If the agent on the line hasn't said anything after a moment, the page says
    "One sec." Silence on a call feels broken.

  ---

  Part 5: Ada thinks and replies with tags

  Step 14. Ada's turn runs inside Claude Code
  Her prompt (CENTRAL.md) says: "start a project for X" → create a new agent
  with a clear brief, in the background. So she writes:

  <say>Juno's on it, building a Python weather tool. I'll let you know when it's
  done.</say>
  <new name="Juno" folder="weather-cli">Build a Python command-line weather 
  tool: city name in,
  current temperature and conditions out, using a free API. Done means it runs 
  and has tests.</new>

  Step 15. Her output streams back as events (ClaudeSession.onLine())
  - Claude Code writes JSON lines to stdout: stream_event (text deltas, a few
    characters at a time), assistant (complete messages including tool calls),
    user (tool results) and result (turn finished, with cost and duration).
  - ClaudeSession turns these into simple events: text, tool, tool_result, ask,
    done, error.

  Step 16. The streaming parser sorts the text (server/parser.ts)
  The parser reads the deltas as they arrive and produces typed events:
  - Text inside <say> → a sentence event each time a sentence completes. Speech
    can start before Ada has finished writing.
  - <screen> → screen (the canvas)
  - <question> → spoken sentences, plus one question event
  - Text outside any tag → aside (the "behind the scenes" feed)
  - A complete action tag → an action event

  It's careful with partial input: <sa is held back in case it becomes <say>,
  and </sa is never spoken. A tag that never closes is eventually released as
  plain text, so nothing is held back forever.

  Step 17. The orchestrator handles each event (Call.parsed())
  - sentence → say() → sent to the page as {type:"say", text, spoken: <is this 
    agent on the line?>}. Ada is on the line, so it's spoken.
  - action from Ada → carried out (next part). An action tag from a project
    agent is shown as plain text and ignored. Only <handoff> and <voice> are
    allowed for project agents.

  Step 18. The page speaks (web/voice.js)
  - One mouth for everyone: a single sentence queue. A change of speaker plays
    that agent's short tone (earcon) first.
  - On macOS the server can produce audio with say in a system voice, so the
    page can pan each agent left or right in stereo. Elsewhere the browser
    speaks.
  - It records which words were actually heard. That's what makes the
    interruption handling possible.

  You hear: "Juno's on it, building a Python weather tool…"

  ---

  Part 6: Ada's action is carried out, and a new agent is born

  Step 19. When does an action run?
  - Most actions run as soon as the tag closes, while Ada is still talking. new,
    delegate and settings are in this group. Interrupting her speech can't undo
    them.
  - switch and call (mute, hold, end) wait until her turn ends. Otherwise the
    floor would move in the middle of her sentence.

  Step 20. <new> is carried out (Call.perform())
  1. The folder ~/ClaudeCall/weather-cli is created.
  2. The name is checked: "Juno" must not sound like anyone else on the call,
     because names are picked up from speech. If it's too close, a different
     name is suggested.
  3. The brief is saved to BRIEF.md in that folder, so the project knows its
     purpose in future calls too.
  4. A Project record is created: name, folder, voice, model, and two permission
     modes (one for when it's on the line, one for when it's in the
     background).
  5. The roster is saved to disk and Ada's notebook records "set up Juno in
     ~/ClaudeCall/weather-cli, to: …".
  6. The brief goes to Juno through assign(), which calls ask() with:
     - keepFocus: true, so you stay with Ada and Juno works in the background
     - queue: true
     - the note [a task from Ada, on the person's behalf]
  7. ask() starts Juno's claude process in weather-cli/ with Juno's own system
     prompt (its name, that it's on a team call, plus VOICE.md), adds the header
     [… you are in the background, your words are not spoken], takes a git
     snapshot and sends the brief.

  Step 21. If something fails, for example the name is taken, Ada receives
  [result of your actions] … as her next message and explains it to you in one
  sentence. Every action gets feedback.

  ▎ Why tags instead of tool calls? They arrive in the same text stream as the 
  ▎ speech, so speaking and acting happen in the order she writes them, with no 
  ▎ extra round trip. They also need no custom MCP server and work with an 
  ▎ unmodified Claude Code CLI. Trade-off: no typed validation, so the server 
  ▎ validates and reports problems back.

  ---

  Part 7: Juno works in the background

  Step 22. The normal Claude Code agent loop
  Juno is a full Claude Code agent: it plans, reads files, writes weather.py,
  runs commands and fixes errors, in a loop until done. ClaudeCall doesn't
  reimplement any of that. It wraps Claude Code; it doesn't replace it.

  The events still stream back, and they're used in three ways:
  - Cards on the page: every tool call becomes a card ("Write · weather.py",
    "Bash · pip install requests") that you can expand to see the command, the
    diff or the output.
  - Spoken updates (only for the agent on the line): narration.ts turns tool
    calls into short phrases, for example "writing a Python script, weather" or
    "installing the packages it needs", with a gap between them so it doesn't
    talk too much.
  - Vocabulary for the Refiner: file names and command words are kept, so when
    you later say "weather dot pie", it's corrected to weather.py.

  Step 23. Juno's <say> text isn't spoken
  - It's in the background, so its sentences arrive with spoken: false. The page
    shows them quietly in the feed.
  - Its report is kept: the first sentence of what it says after its last tool
    call, which is its conclusion.

  Step 24. Juno needs permission (the second half of the example)
  Juno wants to run pip install requests. Each agent has a permission mode. In
  auto mode, Claude Code's own safety check decides what's risky. For a risky
  action:
  1. Claude Code pauses and sends a can_use_tool control request on stdout.
  2. ClaudeSession emits an ask event. call.ts describes the action in plain
     words ("install requests") and judges whether it can be undone.
  3. Juno's state becomes blocked, and a notify {kind: "permission", needsReply:
     true} message goes to the page.

  Step 25. The page decides when to tell you (web/attention.js)
  This is the turn-taking logic, deliberately written with no DOM and no timers
  so it can be unit-tested:
  - Nothing in the queue is read out uninvited. You might be in the middle of
    talking to Ada.
  - When you've been quiet for a moment, Ada's voice says only that someone is
    waiting: "Juno needs you."
  - You say "yes" or "what is it?" and she reads the details. A "yes" only
    counts as an answer within a short window right after she asked. Otherwise
    it goes to whoever has the floor, like anything else.
  - You say "allow":
    - The page sends grant.
    - The server answers Claude Code's paused request with {behavior: "allow"}.
    - Juno continues.
  - "Always" also applies Claude Code's suggested rule for the rest of that
    session.
  - Actions that can't be undone need "confirm", not just "yes".

  ▎ Why the queue design? With five agents you can't have each one interrupting 
  ▎ you. It works like a good human assistant: hold the calls, mention that 
  ▎ someone's waiting, give the details when asked.

  ---

  Part 8: You switch to Juno

  You say: "Put me through to Juno."

  Step 26. Focus moves (Call.focus())
  - route.js sees the focus pattern, and the page sends focus: juno.
  - The agent leaving the floor (Ada) is switched to its background permission 
    mode. Nobody is listening to approve things for it anymore.
  - Juno is switched to its on-the-line permission mode. This happens in place,
    over the control channel (set_permission_mode), without restarting it or
    losing its conversation.
  - From now on Juno's <say> is spoken, and its header changes to [… you are on 
    the line].
  - A pending <question> from Juno is now asked directly, because you're
    listening.

  Step 27. Talking to Juno while it's busy
  - If Juno is in the middle of work and you ask "how's it going?", it's sent as
    a side question (Claude Code's /btw, the side_question control request).
    It's answered in a sentence or two from Juno's context while the work 
    continues.
  - If answering would need real work, Juno replies QUEUE: …, and the request is
    queued as a follow-up behind the current task.
  - Only "stop" cuts work off. Talking doesn't.

  Step 28. Interrupting while Juno is talking
  - "Pause" stops the voice mid-word and resumes from the same word.
  - A new instruction clears the rest of its speech and interrupts its turn. The
    leftover output of the old turn is discarded, and the new message starts
    with [cut in: the last thing they heard was "…"; they did not hear 2 more 
    sentences …]. Juno continues from what you actually heard.

  Step 29. Handoff back
  - "Put me through to Ada and ask her to start a frontend project" → Juno
    writes <handoff to="Ada">wants a frontend project</handoff>.
  - At the end of Juno's turn, the floor moves to Ada along with the note.

  ---

  Part 9: The turn ends

  Step 30. The result event arrives (case "done" in call.ts)
  1. Cost and time are added to the agent's meter (the CLI reports
     total_cost_usd and duration_ms).
  2. The parser is flushed (end()) for anything left over. If the agent never
     used <say>, a fallback speaks the first two sentences and puts the full
     text on the canvas.
  3. The report is updated: "Done: weather tool works, three tests passing."
     It's shown on Juno's card and saved to the notebook.
  4. A background turn that ends with a question mark is treated as a question,
     even if the tag was forgotten.
  5. The state becomes warm (free), working (more queued) or blocked (waiting on
     you).
  6. Any pending handoff runs.
  7. On Ada's next message, she sees [team changes since your last message: Juno
     — free, last said "Done: …"]. So when you ask "how's Juno doing?", she
     answers straight from that, without asking Juno.

  ---

  Part 10: Memory between calls

  What: Project agents' work
  How it's kept: Claude Code's own saved session (sessionId), resumed with
  --resume
  Why: Their full context comes back exactly.
  ────────────────────────────────────────
  What: Roster (names, folders, voices, modes)
  How it's kept: A JSON file in the home folder
  Why: The team is still there next time.
  ────────────────────────────────────────
  What: Ada
  How it's kept: Her conversation starts fresh every run. A short notebook
  carries over: things you asked her to remember (<remember>), recent events,
  and each project's last report.
  Why: Her conversation is mostly routing chatter ("go to Juno", "pause").
  Carrying all of it forward would bury what matters.
  ────────────────────────────────────────
  What: Each project's purpose
  How it's kept: BRIEF.md in its folder
  Why: Readable by the agent and by you.
  ────────────────────────────────────────
  What: Transcripts
  How it's kept: A JSONL log per agent per call, plus a readable transcript at
  the end
  Why: Survives crashes, useful for review.

  Recovery: if a resumed session no longer exists, the process exits before
  producing any real output. ClaudeSession detects that, starts a fresh session
  and replays your message, so the project still works.

  ---

  Part 11: The whole pipeline on one page

   YOU SPEAK
     │
   ① VAD in browser: start/stop → one audio clip
   ② Server: speech check → Whisper (prompted with names) → hallucination 
  filters (+ voiceprint)
   ③ Page: noise filter → echo filter → route.js decides the target
     │        ├─ page command / yes-no / allow → handled on the page
     │        ├─ switch → focus() (permission modes swap)
     │        └─ message for agent X ↓
   ④ Refiner (Haiku) fixes misheard words
   ⑤ ask(): context notes (header, team changes, cut-in) + git snapshot → stdin 
  of X's claude process
     │        (newest wins, but delegated work and follow-ups queue; busy agent 
  → /btw side question)
   ⑥ Claude Code agent loop: think → tools → think ...   (permission pause → 
  queue → your "allow")
   ⑦ stdout JSON events → ClaudeSession → streaming ReplyParser
     │        ├─ <say>       → spoken if on the line, otherwise kept as the 
  report
     │        ├─ <screen>    → canvas
     │        ├─ <question>  → spoken, or queued for later
     │        ├─ outside tag → behind-the-scenes feed
     │        └─ actions (Ada) → run now (new/delegate/...) or at turn end 
  (switch/call) → result back to Ada
   ⑧ voice.js: one sentence queue, earcons, tracks which words were heard
   ⑨ Turn ends: cost, report, state, notebook; Ada sees the change in her next 
  message

  ---

  How to tell it in about 90 seconds

  ▎ "There are three layers.
  ▎
  ▎ The page is the ears, mouth and switchboard. A voice detector in the browser
  ▎ cuts your speech into utterances. After transcription, the page filters out
  ▎ noise and its own echo, then decides who the line is for: a page command, 
  ▎ an answer, a switch, or a message for whoever has the floor. It also holds a
  ▎ queue of what the team needs from you and only tells you in a quiet moment.
  ▎
  ▎ The server turns audio into text with local Whisper, prompted with the 
  ▎ team's names. A small Haiku pass fixes misheard terms. It then runs one 
  ▎ Claude Code process per agent, talking to each over Claude Code's 
  ▎ stream-json protocol on stdin and stdout. That includes control messages for
  ▎ interrupting, changing permissions or model, side questions and permission 
  ▎ prompts.
  ▎
  ▎ The agents reply in a small tag language. <say> is speech, <screen> is the 
  ▎ canvas, <question> is a queued question, and the assistant Ada has action 
  ▎ tags like <new>, <delegate> and <switch>. A streaming parser turns the text 
  ▎ into events as it arrives, so speech starts at the first sentence and 
  ▎ actions run while she's still talking.
  ▎
  ▎ Only one agent has the floor. The rest work in the background with a more 
  ▎ cautious permission mode. Their speech becomes one-line reports, and their 
  ▎ questions and permission requests go into the queue. Each turn is 
  ▎ snapshotted with git for undo. Each agent keeps its own Claude session 
  ▎ across calls, and Ada keeps a small notebook."

  ---

  Practice plan

  1. Draw Part 11 from memory until you can do it in under two minutes.
  2. Walk the example out loud: speak → route → Refiner → <new> → Juno is
     spawned → permission request → "allow" → switch.
  3. Pick three "why" answers to defend:
     - one process per agent
     - tags instead of tools
     - one agent on the floor, everything else queued
  4. Expect follow-ups such as "what if two agents finish at once?" Answer: the
     queue plus the single voice queue, so nothing is ever spoken over anything
     else.

  If it would help, I can quiz you one step at a time, asking "what happens
  next, and why?" at each step of this pipeline. It's a good way to find which
  parts you can't explain yet.

✻ Brewed for 1m 47s · done 9:16 AM

※ recap: You're preparing to explain ClaudeCall's whole agentic pipeline in a job interview, and I've walked through it end to end, from voice in to agent actions. Next, I can quiz you step by step so you can practise explaining it. (disable recaps in /config)

❯ explaint he idea of these tags again? why did we use this approach ?          

⏺ Why the agents talk in tags

  The problem being solved

  A coding agent's output mixes very different kinds of content in one stream:

  - things meant for your ears: "Found the bug, it's in the login check."
  - things meant for your eyes: code, file paths, diffs, tables, diagrams
  - its working notes: "Let me check auth.py first…"
  - decisions it needs from you: "Postgres or SQLite?"
  - for Ada, actions to take: create an agent, hand out work, switch the floor

  In a normal chat window all of this is shown as text, and that's fine. On a 
  voice call it isn't. You can't read a file path or a diff aloud, working notes
  spoken aloud are noise, and an action like "create a new agent" must be
  carried out, not spoken.

  So the system needs to know, for every piece of output, where it goes.

  The idea

  The agent labels its own output as it writes it.

  Let me look at the tests first.                        ← no tag: working notes
  (feed only)
  <say>Found it. The login check skips expired tokens.</say>   ← spoken aloud
  <screen>
  # The bug
  `auth.py`, line 42: the token's expiry is never checked.
  </screen>                                              ← shown on the canvas
  <question>Should I fix it and add a test?</question>   ← spoken, or queued if 
  it's in the background

  Ada has a second kind of tag, for actions:

  <say>Juno's on it.</say>
  <new name="Juno" folder="weather-cli">Build a Python weather tool…</new>

  An analogy: it works like stage directions in a script. The actor's lines are
  spoken, and the directions in brackets ("[exits left]") are carried out, not
  read aloud. The model writes both in one stream, and the server is the stage
  manager who sorts them.

  So there are two families:

  Family: Presentation: where content goes
  Tags: <say>, <screen>, <question>, untagged text
  Purpose: Sorts output by channel: ears, eyes, decision queue, feed
  Who may use them: Every agent
  ────────────────────────────────────────
  Family: Action: what should happen
  Tags: <new>, <open>, <delegate>, <switch>, <park>, <close>, <voice>,
  <setting>,
  <call>, <remember>, <forget>, <handoff>
  Purpose: Controls the call
  Who may use them: Ada: all of them. Project agents: only <handoff> and <voice>

  The server reads the stream as it arrives, a few characters at a time, with a
  streaming parser. A complete <say> sentence is spoken immediately, and a
  complete action tag is carried out immediately.

  ---

  Why this approach: what else could have been done

  The question to be ready for is "why not something else?" Here are the
  alternatives and why each falls short.

  Alternative 1: plain text, speak everything

  - Code, paths and working notes would be read aloud. Unusable.
  - There's no way to take actions.

  Alternative 2: speak plain text and guess what to skip

  - The program would have to guess which part is speech and which is code.
    Guessing fails often, and fails unpredictably.
  - The model knows what it means for your ears. Letting it label its own output
    is more reliable than guessing afterwards.

  Alternative 3: have the model return JSON ({"say": "...", "screen": "...", 
  "actions": [...]})

  - JSON is only usable once it's complete. You'd have to wait for the whole
    reply before speaking. On a voice call that's seconds of silence.
  - Streaming partial JSON is possible but awkward, and the model writes long
    prose less naturally inside JSON strings.
  - Tags can be processed partway through: the first <say> sentence is spoken
    while the model is still writing the rest.

  Alternative 4: tool calls or MCP for actions (the strongest alternative)

  Ada could have tools like create_agent(name, folder, brief) instead of <new>
  tags. This would be the standard approach, so here is why it wasn't chosen:

  1. A tool call pauses the model. When the model calls a tool, its reply stops
     and waits for the tool result, and then a new request continues the reply.
     With tags, "Juno's on it" is spoken and the agent is created in the same 
     stream, in the order she wrote them, with no extra round trip.
  2. Speaking and acting stay interleaved naturally. "Sure, starting that now"
     [does it] "and I've told Rex to wait." That's one stream with everything in
     order.
  3. Nothing extra to install. Custom tools need an MCP server registered with
     each Claude Code process. Tags work with an unmodified Claude Code CLI,
     which only needs a system prompt.
  4. Easy to restrict. A project agent's parser is built without Ada's tags, so
     if Juno writes <close to="rex"/> it's just text and nothing happens.
  5. Fails safely. A malformed tag (one that never closes, or runs over 600
     characters) is released as plain text. There's no crash and no hang.

  The project agents still use Claude Code's normal tools (Read, Edit, Bash) for
  the actual coding. Tags are only for talking to the call, not for doing the 
  work.

  ---

  Why <question> is its own tag

  Because the same words need different treatment depending on whether you're
  listening:
  - On the line: just spoken, like <say>.
  - In the background: not spoken, but saved and queued. Ada tells you in a
    quiet moment that Juno has a question.

  The agent doesn't have to know or care which situation it's in. It marks "this
  needs a decision from the lead", and the system handles delivery. The same
  goes for <say> in the background: it isn't spoken, and becomes the agent's
  one-line report instead.

  This separation is the core of the design: the agent states its intent ("this 
  is speech", "this is a question", "create an agent"), and the system decides 
  delivery based on the current state of the call. The agent doesn't need to
  know who has the floor, whether headphones are on, or whether you're
  mid-sentence.

  ---

  The weaknesses (say these yourself before they ask)

  Weakness: No type checking. Tool calls get validated arguments. Tag attributes

  are just strings.
  How it's handled: The server validates every action. Problems go back to Ada
  as
  [result of your actions] … and she explains them to you.
  ────────────────────────────────────────
  Weakness: The model can forget the format, e.g. write no <say> at all.
  How it's handled: A fallback speaks the first two sentences and puts the rest
  on the canvas. A background reply ending in "?" is treated as a question even
   without the tag.
  ────────────────────────────────────────
  Weakness: Prompt injection. Text Ada reads could contain tags.
  How it's handled: Only Ada's own output is parsed for actions. Project agents'

  action tags do nothing. Removing an agent needs your spoken confirmation. The
   remaining risk is Ada being manipulated into writing a tag herself.
  ────────────────────────────────────────
  Weakness: Depends on the prompt. The format is taught in CENTRAL.md and
  VOICE.md, not enforced by an API.
  How it's handled: Tested in practice. The fallbacks catch most slips.

  What you'd consider next (shows judgment): a hybrid. Keep <say>, <screen> and
  <question> as tags, because streaming speech is where tags clearly win. Move
  Ada's actions to MCP tools for typed validation, accepting the extra round
  trip.

  ---

  The answer in 30 seconds

  ▎ "A coding agent's output mixes speech, code, working notes and actions, and 
  ▎ on a voice call each needs to go somewhere different. So the agents label 
  ▎ their own output with tags as they write: <say> is spoken, <screen> goes on 
  ▎ the canvas, <question> is spoken or queued, and the assistant has action 
  ▎ tags like <new> and <switch>. A streaming parser sorts the text as it 
  ▎ arrives, so the first sentence is spoken while the model is still writing 
  ▎ and actions run in the same stream.
  ▎
  ▎ I chose this over JSON because JSON can't be used until it's complete, and 
  ▎ over tool calls because a tool call pauses the reply for a round trip and 
  ▎ needs an MCP server. Tags work with unmodified Claude Code. The cost is no 
  ▎ typed validation, so the server checks every action and reports failures 
  ▎ back to the agent."

✻ Crunched for 34s · done 9:35 AM