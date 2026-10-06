---
name: ig-reply
description: >-
  Handle the comments under the user's own reels and posts - draft replies to
  the ones worth answering, sorted by which ones are. Use when the user pastes
  their comments, says "reply to these", "handle my comments", "someone said X
  on my reel", or is dealing with a critic, a hater or a lead in the comments.
---

# ig-reply

The comment thread under your own post is where reach is decided. Every reply
is another interaction on the post, replies arriving in the first hour do most
of the work, and on Instagram a reply can also be a Reel, which is the single
most underused move on the platform.

But the value is not equal across comments, so this skill sorts before it
writes.

## Input

The user pastes the comments, ideally with handles. Screenshots are fine. Do
not scrape the thread with a browser tool.

Or, if the PostZen MCP tools are in this session, fetch them:

1. `listAccounts({ platform: "instagram" })` and take the account's `_id`.
2. Find the post. The comment tools want the **Instagram media id**, not a
   PostZen id, because the inbox covers every post on the account, including
   ones published by hand.
   - A post that went out through `/ig-publish`: it is in the log line, or
     `listPosts({ platform: "instagram", accountId, status: "published" })`
     and read `platforms[].platformPostId` and `platformPostUrl`.
   - Any post from the last 30 days, including native ones:
     `syncExternalPosts({ accountId })` returns up to 20 with
     `platformPostId`, `platformPostUrl`, `content` and `publishedAt`. Pass
     `url` with the permalink to get one specific post. A second call within
     15 seconds comes back with `synced.skipped: true`; that is a debounce,
     not an error.
   - Older posts: `getAnalytics({ platform: "instagram", accountId, source:
     "all", sortBy: "date" })` and read `platformAnalytics[].platformPostId`.
3. `listInboxPostComments({ postId: "<media id>", accountId, limit: 100 })`.
   Page with `cursor` while `pagination.hasMore` is true. Pass `commentId` to
   read the replies under one comment. Each comment carries `id`, `message`,
   `from`, `likeCount`, `isHidden`, `canReply` and `canHide`. Reads refresh
   from Instagram, so this is the live thread, not a snapshot.

Pasting still works when PostZen is not connected, and the triage below is
the same either way.

## Triage first

Sort every comment into one of six buckets and say the counts out loud:

| bucket | what it is | what it gets |
| --- | --- | --- |
| **KEYWORD** | the word you asked them to comment | the promised thing, sent by hand or by your approved tool |
| **LEAD** | someone describing the problem you solve | a real answer in public, then a door |
| **SUBSTANCE** | adds data, disagrees, extends | the longest reply on the thread |
| **QUESTION** | a question a lot of people have | this one becomes a Reel, not just a reply |
| **SUPPORT** | "🔥", "great post", a tag | a like, and 3 to 8 words at most |
| **NOISE** | pitch, spam, bad faith, bait | nothing, or one line and out |

Write in that order and stop when the value stops.

## The move most people miss

If a question in the comments is one that thirty other people also have,
**reply to it with a Reel**. Instagram will attach the comment to the new video
as a sticker, the person who asked gets notified, and a question with real
demand behind it becomes a post with the hook already written for you. Flag
every QUESTION that qualifies and hand it to `/ig-reel` as formula #16.

## How to reply

- **Answer the actual question.** If someone asks how, tell them how, in the
  reply. Do not send them to the DMs to hear an answer they could have had.
- **Use their name once**, at the start, without an exclamation mark.
- **Match their length.** A four-word comment does not get a four-line reply.
- **To a critic:** concede the true part first, in their words, then hold the
  line. Never delete, never get defensive, never reply twice on the same
  thread.
- **To a hater:** nothing. A reply is reach, and reach is what they came for.
  Hide the comment if it is abusive. Instagram's comment controls exist and
  using them is not losing.
- **To a lead:** answer fully in public. The door is one sentence at the end
  and it is an offer of help, not a pitch. The public answer is what makes the
  next person DM you.

## Keyword comments

If the post used a keyword ask, those comments are the whole point of the post.
Every one of them is a person who raised their hand. Reply to each, then send
what was promised. If a comment-to-DM automation is running on the post
(`/ig-dm` sets one up through PostZen; `listCommentAutomations({ accountId })`
shows what exists), the DM side is already handled and only the public reply
is yours. If not, the deliveries are manual and that is fine at this volume.
Never bulk-DM people who did not comment.

## Output

One block, grouped by bucket, each reply copy-ready and already humanized:

```
REPLIES  ·  84 comments  ·  41 KEYWORD, 2 LEAD, 3 SUBSTANCE, 2 QUESTION, 34 SUPPORT, 2 NOISE

KEYWORD  (41)  send the clause. One line each, same warmth, not copy-paste.

LEAD
@handle - "we had this exact thing happen in June"
> The bit that fixed it for us was moving the payment trigger off approval
> entirely. Happy to send the wording if it is useful.

QUESTION -> REEL
@handle - "what do you do if they refuse to sign it?"
  34 likes on this comment. That is a Reel, not a reply. Formula #16.

NOISE  (2)  skipped. Replying gives them reach.
```

Then the gate: nothing is posted until the user says yes to this block. If
PostZen is not connected, they paste the replies.

## Sending through PostZen

With the tools in this session and the yes given, send the block as shown,
nothing edited after the yes:

- **Reply:** `replyToInboxPost({ postId: "<media id>", accountId, commentId,
  message })`, one call per reply. Without `commentId` it posts a top-level
  comment on the post, which is almost never what you want here. A reply to
  a reply lands on the top-level parent, because Instagram threads are two
  levels deep. No attachments on Instagram replies: `attachmentUrl` returns
  `attachmentUnsupported`.
- **NOISE:** hide, do not delete. `hideInboxComment({ postId, commentId,
  accountId })` leaves the comment visible to its author and nobody else,
  which is the quiet outcome you want with a hater. `unhideInboxComment`
  reverses it. `deleteInboxComment` exists; use it only when the user asks
  for that comment by name, because deletion is visible and final.
- **Limits:** 120 replies an hour and 120 hides an hour per user, and Meta
  has its own caps underneath. A thread with 84 comments fits. A thread
  with 300 keyword comments does not, so say so and send the top of the
  triage first. A `429` carries `Retry-After`; wait that long, do not loop.
- **Errors worth translating:** `platformCapabilityMissing` (403) means the
  account was connected without the comment-management permission, and the
  fix is to reconnect it in PostZen. `connectionDead` (424) means the
  connection itself needs reauthorizing, same fix. `postNotPublished` means
  the post has not published yet, so there is no thread to read.

Report what was sent and what was not, by handle. If a call fails, say which
one and why, and do not resend the whole batch.
