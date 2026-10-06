---
name: ig-dm
description: >-
  Write Instagram DMs that get replies - the keyword delivery (and the
  comment-to-DM automation that sends it through PostZen), the first message
  to someone who engaged, the collab pitch, and the two follow-ups. Use when
  the user says "DM this person", "what do I send them", "set up the keyword
  DM", "outreach message", "how do I follow up", "pitch this brand", or is
  reaching out to someone specific.
---

# ig-dm

Instagram DMs are the only place on the platform where money actually changes
hands, and they are also where most accounts burn the goodwill their content
earned. The difference is entirely about who moved first.

## The three kinds of DM, and only three are worth writing

**1. The reply to a hand raised.** They commented the keyword, answered the
poll, replied to a story, or saved and asked. They moved first. This is 90% of
the DMs worth sending and it converts because it is not outreach.

**2. The warm approach.** Someone whose posts the user has genuinely been
commenting on for weeks. There is a shared thread of conversation already.

**3. The collab or brand pitch.** A specific proposal to a specific account,
with a reason it is them.

Everything else is cold DMing strangers, which is what everybody else does, and
it is why reply rates sit where they do. If the user is asking for a cold
sequence, say plainly that it is the lowest-yield thing they could do with the
same hour, and offer the alternative: comment on those ten accounts for two
weeks first. Then write it if they still want it.

## Before writing, get the specifics

Ask in one batched question:

1. **Who** - handle, what they do, and what they posted or did that started
   this.
2. **The trigger** - the actual reason to message today. A keyword they
   commented, a story they replied to, a post they published. Not "they fit the
   ICP".
3. **What the user wants** - a conversation, a sale, a collab, a referral. Be
   honest internally, even if the message does not lead with it.

If there is no trigger, there is no message. Say so.

## The keyword delivery

The most common DM in this pack and the easiest to ruin. They commented one
word. They are expecting the thing. So:

```
{their name}, here it is: {the thing, or the link}.

{one line on how to use it}

{one question they can answer in four words}
```

Send the thing **first**, in message one, with no gate. A keyword post that
delivers a "before I send it, can I ask what you do?" is a bait and switch and
it is remembered. The question at the end is what starts the conversation, and
it is optional for them.

Automated keyword replies are a supported feature for professional accounts,
through Instagram's own tools or an approved partner. Using that is fine.
Sending unsolicited bulk DMs is not, and it is the fastest route to a
restricted account.

### Making it an automation through PostZen

If the PostZen MCP tools are in this session, the keyword delivery does not
have to be sent by hand. `createCommentAutomation` watches the comments on a
post (or the whole account) and sends the DM for each matching comment, using
Meta's one private reply per comment. Write the message as above, then build
the call. The parts:

**Where it listens.**
- `accountId` from `listAccounts({ platform: "instagram" })`. Fixed after
  creation.
- `name`, 1 to 120 characters, for the user's own list.
- `trigger`: `"comment"` (default) or `"story_reply"`.
- Scope: `platformPostId` (the Instagram media id) for one post, or
  `postId` (a PostZen post id; it resolves when the post publishes, so you
  can set this up before `/ig-publish` sends the reel), or neither for the
  whole account. One active per-post automation per post; a second returns
  `duplicatePostAutomation`. A per-post automation wins on its own post;
  otherwise account-wide ones are tried oldest first.

**What it matches.**
- `keywords`: up to 50, each 1 to 100 characters. Case- and
  accent-insensitive. An empty list matches every comment, which is almost
  never what the user wants on a public post.
- `matchMode`: `"contains"` (default), `"exact"` or `"word"`. For a one-word
  keyword like CONTRACT use `"word"`, so "contractor" does not fire.
- `typoTolerance: true` works with `"word"` only, and forgives one edit on
  keywords of 4 to 7 characters, two on 8 or more. It cuts both ways: two
  edits turn "contract" back into "contractor", so leave it off when the
  keyword has a longer everyday form.
- `excludeKeywords`: up to 50, same mode, vetoes a match.

**What it sends.**
- `dmMessage`: the delivery, written as above. **At most 1,000 UTF-8
  bytes**, not characters: emoji and accents cost several bytes each. With
  buttons the cap is 640 characters.
- `dmMessageVariations`: up to 5 alternates, same limits, picked at random
  with the base message. Same warmth, different words, so forty people do
  not post identical screenshots.
- `buttons`: up to 3 of `{ type: "url", title (1 to 20 chars), url }`. A link
  as a button beats a link in the text. Cannot be combined with `template`.
- `template`: an image card instead of buttons. `{ type: "generic",
  imageAspectRatio: "horizontal" | "square", elements: [{ title (≤80),
  subtitle? (≤80), imageUrl, buttons? (≤3) }] }`, 1 to 10 elements; several
  elements render as a swipeable carousel. **`imageUrl` must be a stable
  public HTTPS URL on the user's own hosting.** Do not use a
  `createMediaPresign` URL here: an upload that is never attached to a post is
  deleted after about 24 hours, and the card goes blank. If Meta rejects a
  card or buttons, PostZen falls back to plain text with the titles and URLs
  and the log shows `buttonsDropped: true`.
- `dmDelaySeconds`: 0 to 86400, default 0. A short delay reads less like a
  bot. Thirty to ninety seconds is plenty.

**The public reply.**
- `commentReply`: 1 to 1,000 characters, posted under their comment **after**
  the DM succeeds. "Sent, check your DMs" is the whole job. Ignored for
  `story_reply`.
- `commentReplyVariations`: up to 5, rotated independently of the DM. One
  base reply plus a few variations is enough; PostZen does not require a
  minimum.
- `commentReplyDelaySeconds`: the reply never lands before the DM, whatever
  you set.

**Who gets it.**
- `audience`: `{ followerStatus: "any" | "follower" | "non_follower",
  minFollowerCount, whenUnknown: "send" | "skip" | "verify" }`. Any audience
  rule forces an **opening DM**, because Instagram only reveals the follow
  relationship after the person has messaged the account, and tapping the
  opening DM's button counts as that message.
- `openingDm`: `{ message (≤640), buttonLabel (≤20) }`. Send `{}` for the
  defaults: a short "tap below and I'll send you the link" message with a
  "Send me the link" button. Story replies skip it.
- `followGate`: `{ message (≤640), buttonLabel (≤20), notFollowingMessage
  (≤1000) }`. "Follow to get it" gates. They convert fewer people and they
  are the bait-and-switch this section warns about, so offer it only if the
  user asks.
- Each person gets at most one DM per automation. A second comment logs
  `skipped` with `already_sent_to_contact`. Blocked and unsubscribed contacts
  (`updateContact` with `isBlocked: true` or `isSubscribed: false`) are
  skipped too. The account's own comments never trigger.

**The rules underneath, which are Meta's:** one private reply per comment,
ever, and within 7 days of the comment. PostZen caps private replies at 750
an hour per Instagram account and holds the overflow rather than letting Meta
reject it. The account needs both the comment and messaging permissions;
accounts connected since 2026-09-12 have them. An older one that returns
`platformCapabilityMissing` needs a reconnect. At most 100 automations per
user.

**The gate.** Show the user the keyword, the match mode, the full DM text
with its byte count, the buttons or card, the public reply, the scope (which
post, or account-wide) and the account handle. Get an explicit yes. Then
create it. Then `getCommentAutomation({ automationId })` to read it back, and
`listCommentAutomationLogs({ automationId })` later to see `sent`, `failed`,
`skipped` and `gated` with their reasons. `updateCommentAutomation({
automationId, isActive: false })` pauses it; `deleteCommentAutomation`
removes it and its logs. Both are the user's call.

An example, for the CONTRACT reel:

```json
{
  "accountId": "<_id from listAccounts>",
  "name": "CONTRACT clause - Oct 7 reel",
  "postId": "<PostZen post id from /ig-publish>",
  "keywords": ["contract"],
  "matchMode": "word",
  "dmMessage": "Here it is: https://example.com/clause\n\nDrop it in section 4, under payment terms.\n\nWhat kind of work do you mostly contract for?",
  "dmDelaySeconds": 45,
  "commentReply": "Sent, check your DMs.",
  "commentReplyVariations": ["Just sent it over.", "In your inbox now."]
}
```

## Replying to someone who already messaged

If the person has already written to the account (a story reply, a question,
an answer to the keyword DM), the thread exists, and with PostZen connected
the reply can go out from here instead of from the phone:

1. Find the thread. `listInboxConversations({ platform: "instagram",
   accountId })` lists them newest first with `participantUsername`,
   `lastMessage`, `lastMessageAt` and `unreadCount`. Or
   `searchInboxConversations({ query: "contract", accountId })` to find one
   by what was said; it searches PostZen's synced copy, matching whole words.
2. Read it. `listInboxConversationMessages({ conversationId, accountId })`.
   Only the newest 20 or so messages come back with full text; older ones
   may be bare. Reading does not mark the thread read.
3. Write the reply the same way as any DM here: their words, something given,
   one small ask. Run it through `/ig-human`.
4. The gate: show the handle, the last thing they said, and the reply. Get
   the yes.
5. `sendInboxMessage({ conversationId, accountId, message })`. Set an
   `Idempotency-Key` so a retry cannot double-send. For an image, send
   `attachmentUrl` plus `attachmentType: "image"` in a **separate** call;
   text and attachment in one call is a 400, and the URL must be public
   because Meta fetches it. `markInboxConversationRead` is optional and local
   to PostZen.

The hard limit is Meta's: a send only works **within 24 hours of the person's
last message**. Outside that it fails with `PLATFORM_LIMITATION`, and the
honest move is to say so rather than hunt for a way around it. A `502
providerOutcomeUnknown` means the send may or may not have landed; read the
thread again before sending anything. Sends are capped at 120 an hour per
user.

**What PostZen cannot do:** open a new thread. `sendInboxMessage` takes a
`conversationId` and no recipient, so a person who has never messaged the
account cannot be messaged from here. That is why the next two sections stay
copy-ready.

## The first message to someone warm

- **Two to four sentences.** A screen of text is a delete.
- **Reference the specific thing.** The comment, the post, the reply. In their
  words.
- **Give before asking.** A number, a template, a name, an answer.
- **One ask, small.** "Worth a quick call?" beats "let me walk you through the
  platform".
- **No link and no calendar in message one.** It reads as a funnel because it
  is one.
- **No voice note to a stranger.** It is a great tool and it is for people who
  already know the user's voice.

## The collab pitch

Four lines, in this order: what you have watched them do, the specific idea,
what they get, what you need from them. A collab post lands on both grids and
reaches both audiences, which is the strongest single growth mechanic on
Instagram that does not involve paying anybody. Pitch it as that and be
concrete about who does what.

## Follow-ups

Two. That is the number.

- **+4 days** - add something new. Never "just bumping this". If there is
  nothing new, there is no follow-up.
- **+10 days** - the close-the-loop message. Say you will stop, and mean it.
  This one gets a surprising share of the total replies, because it removes the
  pressure.

Then stop. A third converts nobody and costs the relationship.

Follow-ups are sent by hand even with PostZen connected. At +4 and +10 days
the 24-hour window has closed unless the person wrote back in between, and
if they wrote back, it is a conversation, not a follow-up.

## Never

- Never automate outreach DMs, and never use a tool that sends on a schedule to
  people who did not interact. It violates Instagram's Terms of Use and
  restricts the account.
- Never fabricate having watched something, a mutual, or a shared anything.
- Never open with "Hey! Quick question" and then not ask a question.
- Never send the pitch in the same message as the compliment.

## Output

The message, the character count, and the two follow-ups with the day each
goes out, all run through `/ig-human`.

Then who sends what:

- **Keyword delivery:** a PostZen comment automation if connected and
  approved, otherwise by hand.
- **Reply in an existing thread, inside 24 hours:** `sendInboxMessage` after
  the yes, or by hand.
- **The warm first message, the collab pitch, the follow-ups:** by hand,
  always. No API opens a new thread, and that is the right constraint.
