---
name: ig-plan
description: >-
  Build the week on Instagram - what to post, which format, when, and who to
  engage with. Use when the user says "plan my week", "what should I post",
  "content calendar", "I have nothing to post about", or wants a posting
  schedule and an engagement list.
---

# ig-plan

The control room. Everything else in this pack executes; this decides what gets
executed. Run it once a week, on the same day.

## Input

If `~/.claude/instagram/voice.md`, `swipe.md` and `log.md` exist, read them.
The swipe file is the user's own evidence from `/ig-viral` about which formulas
are landing in their niche right now, and it outranks anything in this file.
The log stops the plan repeating a theme from the last fortnight.

If they do not exist, ask for four things and write them down:

1. What the user sells, and to whom.
2. The three or four themes they want to be known for.
3. What actually happened this week: a client call, a number, a mistake, a
   thing they built, an argument they had. This is where posts come from.
4. Ten accounts worth being visible to.

## What to post

Four to five posts a week, and at least three of them Reels. Reels are the only
format on Instagram that reliably reaches people who do not follow the account.
Carousels go deep with the people who already do. Stories are daily and are
planned separately.

Mix across the week, never two of the same type back to back:

| type | share | job |
| --- | --- | --- |
| **Proof** | 1 per week | something that happened, with a number. Reel. |
| **Teach** | 1 to 2 per week | one thing the viewer can do today. Reel or carousel. |
| **Opinion** | 1 per week | a position that could lose you followers. Reel. |
| **Story** | 1 per fortnight | a scene with a cost. Reel. |
| **Offer** | 1 per fortnight | what you sell, said plainly, no apology. Carousel or stories. |

For each slot give: the theme, the specific angle from what actually happened
this week, the format, and the hook formula number from `ig-reel/hooks.json`.
Not a topic, an angle. "AI" is not a plan. "The proposal we lost because the
draft had an em dash in it" is a Reel.

## When to post

Post when the user's audience is awake and not at work. For most consumer
audiences that is early evening local time; for a business audience, early
morning.

But say this plainly: **the hour matters far less than the first two seconds.**
Instagram will keep showing a Reel for days if it performs, and will bury a
well-timed one that does not. If the user is optimising posting times before
their hooks work, they are polishing the wrong thing, and you should say so.

Anchor times to the audience's timezone, not the user's, if those differ.

### With PostZen connected

If the PostZen MCP tools are in this session, the "when" column stops being a
guess about the audience and becomes the account's own history:

- `getBestTimeToPost({ platform: "instagram", accountId })` buckets the
  account's posts from the last 366 days by publish weekday and hour and
  ranks the buckets by average engagement (likes + comments + shares +
  saves). Hours come back in **UTC**; convert them before they go in the
  plan. Show `post_count` next to each slot: there is no minimum, so a slot
  with one lucky post can rank first, and a slot built on one post is an
  anecdote. Reach and follower activity are not in this number.
- `getAnalytics({ platform: "instagram", accountId, source: "all", sortBy:
  "engagement" })` for which themes and formats carried the last 90 days,
  which outranks the share table above. `/ig-audit` does this properly; use
  its conclusions if it has run.
- `listPosts({ platform: "instagram", accountId, status: "scheduled",
  sortBy: "scheduledFor" })`, and again with `status: "queued"`, so the plan
  shows what is already booked and does not double up a day. Only posts
  created in PostZen appear here.

**Queue slots.** If the user wants a standing schedule rather than a time per
post, PostZen's queue does that: posts go in with `queuedFromProfile` and
take the next free slot.

- `listProfiles` for the `profileId`, then `listQueueSlots({ profileId, all:
  true })` to see what exists, including the next five free instants.
- `createQueueSlot({ profileId, timezone: "America/Toronto", slots: [{
  dayOfWeek: 2, time: "19:30" }, { dayOfWeek: 4, time: "19:00" }, { dayOfWeek:
  0, time: "18:00" }], name: "Instagram" })` creates a whole queue. `dayOfWeek`
  is 0 for Sunday. The first queue on a profile becomes its default. Show the
  slots with the timezone and get a yes before creating.
- `updateQueueSlot({ profileId, timezone, slots, queueId })` **replaces every
  slot** in that queue; it is not a nudge to one slot. Send the full list
  each time. Posts already placed keep their times unless
  `reshuffleExisting: true`.
- `deleteQueueSlot` **without a `queueId` deletes every queue on the
  profile.** Never call it without one, and confirm with the user first.
- `previewQueue({ profileId })` returns the upcoming free instants, not the
  posts in them. For posts, use `listPosts` as above.

Writing a time into the plan does not schedule anything. `/ig-publish` does
that, one post at a time, after each one is written and approved.

## The engagement round, which is not optional

20 minutes a day, before posting, not after. Build a list of 10:

- **5 reach** - accounts with an audience the user wants, where a good comment
  gets seen. Comment early, before the thread is 200 deep.
- **3 peers** - same size, same field. This is the group that reciprocates.
- **2 buyers** - people who could actually buy. Comment for weeks before any
  DM, and never pitch in a comment.

Hand the list to `/ig-comment`.

## Output

```
WEEK OF SEP 15

MON  engage only  (20 min, list below)
TUE  7:30pm  REEL      PROOF    #5  Time Collapse   - 5hr proposal to 20 min
WED  stories only + engage
THU  7:00pm  CAROUSEL  TEACH    Job B caption       - the 4-slide clause breakdown
FRI  7:30pm  REEL      OPINION  #2  Negative Command - stop doing discovery calls
SAT  -
SUN  6:00pm  REEL      STORY    #21 Mid-Sentence    - the refund email

STORIES  every day, 3 to 5 frames, question box on Thursday.

ENGAGE  (5 reach / 3 peers / 2 buyers)
  ...

Say "write Tuesday" and I will draft it.
```

Write the plan to `~/.claude/instagram/plan.md` so the other skills can read
it. Nothing is scheduled or posted by this skill. This is a plan; the user
runs it, and `/ig-publish` schedules each post once it exists.
