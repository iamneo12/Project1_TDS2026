# Project Report: Telegram Data-Analyst Bot

**Course:** Tools in Data Science (TDS)
**Project:** Project 1 — Data Analyst Telegram Bot

---

## 1. Objective

The goal of this project was to build a chatbot that lives inside Telegram and
can answer data-analysis questions on its own. A person (or, during grading,
an automated grading account) sends the bot a question in plain English —
sometimes a single message, sometimes a short back-and-forth conversation —
and the bot has to figure out the answer by itself and reply with a single,
well-formed JSON object containing the answer and a link to a log of what it
did to get there.

The bot needed to handle several different kinds of questions: simple
calculations, statistics on data given directly in the message, and questions
that require looking up real, current information from the internet (since
the model behind the bot has a training cutoff and cannot be trusted to know
today's facts and figures from memory).

---

## 2. Overall Approach

The bot is built around a large language model (GPT-4o) that is given access
to tools it can use to look things up or run calculations, rather than just
answering from memory. The core idea is simple: the model reads the question,
decides whether it needs help (a calculation, a live fact), asks to use a
tool if so, reads the result, and repeats this until it's ready to give a
final answer.

Three parts of the system needed careful design:

1. **How to let the model run code safely**, without risking the whole bot
   crashing if the model writes bad code.
2. **How to decide when the bot has spent "enough" time looking things up**
   and should just commit to an answer, since Telegram/the grader expects a
   reply within a reasonable time.
3. **How to reliably read the model's final answer back out** as clean JSON,
   even when the model tries to add extra formatting around it.

---

## 3. System Design

### 3.1 Talking to Telegram

The bot doesn't wait for Telegram to notify it of new messages. Instead, it
repeatedly asks Telegram "anything new for me?" in a loop. Every incoming
message is handled independently, so if one question takes a while to answer
(for example, because it needs to search the internet), it doesn't hold up
replies to anyone else.

### 3.2 Letting the model run code safely

When the model needs to calculate something or fetch data from a website, it
writes a small piece of Python code and asks the bot to run it. Rather than
running that code directly inside the bot itself, the bot saves it to a
temporary file and runs it as its own separate program with a time limit. This
way, if the model's code has a bug — like an infinite loop, or a crash — only
that one attempt fails. It can never bring down the whole bot.

### 3.3 Deciding when to stop and answer

This turned out to be one of the trickier design questions. If the bot lets
the model keep trying forever, it risks never giving an answer in time. If it
cuts things off too early, the model won't get a fair chance to actually find
the answer. The final design combines three ideas together:

- **A hard limit on the number of attempts**, so the bot can never get stuck
  looping forever no matter what else happens.
- **Splitting the total time budget in two**: the first portion of time is
  for looking things up, and the last portion is set aside purely for the
  model to commit to a final answer, tools switched off.
- **A "time bank"**: if a lookup finishes faster than expected, that saved
  time gets added back, giving the model a little more room to try again
  later. If a lookup runs slow, time gets taken away instead. This way, a
  string of quick, successful lookups earns the model more chances, while a
  slow one tightens the window naturally.

If the model ever stops asking for tools and just gives a plain answer, that
is treated as a sign it's confident enough, and the bot accepts that answer
immediately rather than forcing it to keep going.

### 3.4 Reading the model's final answer

The model is instructed to reply with exactly one JSON object and nothing
else. In practice, models don't always follow formatting instructions
perfectly, so the bot scans the model's reply for the first complete,
correctly-nested `{ ... }` block and parses that, ignoring any stray text or
formatting around it.

### 3.5 Keeping a record of everything

Every step the bot takes — every question it receives, every tool it uses,
every reply the model gives, and any errors — is written down to a log file
as it happens. This log is made public through a link included in every
reply, so anyone reviewing the bot's answers can see exactly how it arrived
at them.

---

## 4. Trials, Problems Found, and Fixes

Building this bot wasn't a single clean pass — several real problems came up
during testing, and each one led to a meaningful change in the design.

**Trial 1 — Missing deployment setting.**
The very first deployment attempt crashed immediately on startup with a
"BASE_URL not found" error. The bot needs to know its own public web address
to include in its replies, and this setting hadn't been filled in yet on the
hosting platform. Once the correct address was added, the bot started
successfully.

**Trial 2 — Wrong date and time answered from memory.**
An early test asked the bot "what is the time in India," and it confidently
replied with a date from 2023 — even though the actual date was in 2026. This
revealed that the model was answering from its own memory instead of
actually checking, which is a serious problem for any question involving
current dates, times, or other facts that change over time. The fix was to
explicitly instruct the model that it has no built-in awareness of the
current date or time, and that it must always calculate or fetch this rather
than guess.

**Trial 3 — A tool call that silently did nothing.**
When asked to look up Japan's current population, the bot's tool call
technically ran without crashing, but produced no output at all — because
the code the model wrote calculated an answer but never printed it. Instead
of noticing this and trying again, the model simply made up a population
figure from memory and presented it as if it were real, current data. This
was a serious accuracy problem, since a plausible-looking but fabricated
number is harder to catch than an obvious error. The fix required two
changes: instructing the model to always print out any value it wants to
see, and telling it explicitly that empty results mean the attempt failed
and it should try again rather than fall back to guessing.

**Trial 4 — A duplicated field in the final answer.**
After fixing the above, a new small bug appeared: the model's own attempt at
including a link to the log ended up nested incorrectly inside the answer,
producing two "log_url" fields in the final reply instead of one. Since the
project rules are strict about the exact shape of the reply, this needed to
be fixed by stripping out any stray field the model added before finalizing
the answer.

**Trial 5 — Struggling to find live data at all.**
Once the bot correctly refused to guess, a new problem showed up: it made
seven different attempts to find Japan's population — trying several
statistics websites, government data sources, and search approaches — and
every single one failed, because most of those websites build their content
using technology (JavaScript) that a simple, behind-the-scenes web request
cannot see. The bot was doing the right thing by not giving up and
fabricating an answer, but it still couldn't get a real result within a
reasonable number of attempts.

The solution was to give the model two additional, more reliable ways to
look things up: one specifically for getting clean information from
Wikipedia, and one for reading data directly from services that provide it
in a simple, structured format. Both of these avoid the JavaScript problem
entirely. The system was also changed so that if two lookup attempts in a
row come back with nothing useful, the bot itself — not just the model —
notices this pattern and pushes the model to try a genuinely different
approach next.

**Result after these changes:** the exact same population question that
previously took seven failed attempts and produced no usable answer was
answered correctly in a single attempt, taking under half a second, using
real information pulled directly from a live Wikipedia article.

---

## 5. Final Testing

After all the fixes above, the bot was tested again across a range of
question types:

- Simple, direct calculations (arithmetic, statistics on given numbers)
- Multi-turn conversations, where an earlier message supplies data and a
  later message asks the actual question
- Questions requiring genuinely current information from the internet
- Questions specifying an exact, sometimes nested, shape for the answer

In each case, the bot's final reply was confirmed to be a single, correctly
formatted JSON object, matching the requested shape, with a working link to
its log of what happened during that conversation.

---

## 6. Reflections

The most valuable lesson from this project was that a language model on its
own is not naturally reliable for tasks involving current facts or figures —
it will confidently produce a plausible-sounding wrong answer unless it is
specifically guided to notice its own failures and try again rather than
guess. Building a bot that behaves well under these conditions required more
than just connecting a model to a chat interface; it required carefully
thinking through what happens when a lookup fails, how much time to allow for
retrying, and how to make sure the model's own mistakes (like a duplicated
field, or a silent failure) don't quietly make it into the final answer.

The final version of the bot reflects several rounds of real testing against
real failures, rather than a design that was assumed to work correctly from
the start.
