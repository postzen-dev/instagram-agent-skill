---
name: ig-publish
description: >-
  Publish, schedule or queue a finished Instagram post (feed, reel, carousel
  or story) through the PostZen MCP tools, verify it actually went out, and
  log it. Use when the user says "post it", "publish this", "schedule it for
  Tuesday", "put it in the queue", "connect my Instagram", or when another
  skill in this pack has a copy-ready block and the user wants it posted
  rather than pasted.
---

# ig-publish

The one skill in this pack that touches Instagram. Every other skill writes
and hands off here. This one checks the account, gets the media in, builds the
`createPost` call, gets the user's yes, sends it, and then checks that it
really published, because the success message does not prove that.

Without PostZen, this skill has nothing to call. Say so, print the copy-ready
block the content skill produced, and the user posts by hand. That path is
not a failure, it is how the pack worked before PostZen existed.

## Step 0: is PostZen connected?

Look for the PostZen MCP tools in this session: `listAccounts`, `createPost`,
`createMediaPresign`. If they are there, PostZen is connected. If they are
not, the user has three ways in:

1. **Plugin install.** The plugin ships `.mcp.json`, which registers the
   `postzen` server at `https://mcp.postzen.dev/mcp`. Run `/mcp`, pick
   `postzen`, then Authenticate.
2. **Manual skills copy.** Add the server first:
   `claude mcp add --transport http postzen https://mcp.postzen.dev/mcp`,
   then `/mcp`, `postzen`, Authenticate.
3. **An API key instead of OAuth.** If the user already has a PostZen key:
   `claude mcp add --transport http postzen https://mcp.postzen.dev/mcp --header "Authorization: Bearer pzn_live_..."`.
   Never ask for the key in chat and never paste it into a file you write.

Authenticate opens the browser at `app.postzen.dev/connect/mcp`. The user signs
in to PostZen there, nothing is typed into Claude. Two choices on that screen:

- **Read & Write or Read Only.** Publishing, replying and automations all
  need Read & Write. Read Only is for `/ig-audit`.
- **Profile access.** All profiles, or selected ones.

Approval mints an API key named `MCP: <client name>`, revocable at
`app.postzen.dev/api-keys`. The consent screen expires after 10 minutes, so
if the user wandered off, run `/mcp` again.

**Connecting the Instagram account.** PostZen needs an Instagram **Business
or Creator** account. Personal accounts cannot connect. No Facebook Page is
required.

1. `listProfiles` to get a `profileId` (there is usually one, marked
   `isDefault`).
2. `createConnectUrl({ platform: "instagram", profileId })`. It returns
   `authUrl` and `state`. The link is good for 10 minutes.
3. The user opens `authUrl` and completes Instagram Login. The redirect
   lands on PostZen's own callback, which finishes the connection.
   `completeConnect` is for integrations that catch the OAuth code on their
   own redirect, which this flow does not, so skip it and check Step 4.
4. `listAccounts({ platform: "instagram", profileId })` and check `status`
   is `connected`. If the user declined a permission on the way through, the
   account can come back as `needsReauth` or missing inbox, analytics or
   messaging ability. Run `createConnectUrl` again and ask them to accept
   everything.

**The free plan**, so the user is not surprised: $0, no card, 2 connected
accounts, 20 platform-posts per UTC calendar month, 60 API requests a minute.
Published, scheduled, queued and publishing posts count against the 20.
Drafts, failed and canceled ones do not. Going over returns a `402` with
`freePostLimitExceeded`. Paid plans remove the post cap.

Never ask for or handle the user's Instagram password or tokens. The browser
flow is the only path.

## Step 1: find the account

```
listAccounts({ platform: "instagram" })
```

Use `accounts[i]._id` as `accountId` everywhere. Show the `username` back to
the user so they can see which account is about to receive the post. If there
is more than one connected Instagram account, ask. Never guess an `accountId`
and never type one from memory.

Check `status`. Anything other than `connected` means reconnect first (Step
0, the connect flow).

## Step 2: get the media in

Instagram posts need media. There is no text-only Instagram post, and
`createPost` says so: "Instagram requires at least one image or video."

**Option A: a public URL.** Any direct `https` link to an image or video
file goes straight into `mediaItems[].url`. PostZen downloads and re-hosts
it. Limit 100 MB. Google Drive, Dropbox, OneDrive and iCloud share links
return a web page, not a file, so they fail. Use a direct file URL or go to
Option B.

**Option B: a local file.** Three calls:

```
createMediaPresign({ filename: "reel.mp4", contentType: "video/mp4", size: 48213990 })
```

returns `uploadUrl`, `publicUrl`, `key`, `type`. Then the bytes go up with a
plain HTTP PUT, which the MCP cannot do, so run it in the shell:

```bash
curl -X PUT --upload-file reel.mp4 -H "Content-Type: video/mp4" "<uploadUrl>"
```

Then `publicUrl` goes into `mediaItems[].url`. Do the PUT right away; the
upload URL is short-lived. If the PUT never ran, `createPost` fails with
"Media upload is not complete for <url>". `size` is the byte count of the
file, which you can read with `stat` or `wc -c`.

Accepted `contentType` values: `image/jpeg`, `image/jpg`, `image/png`,
`image/webp`, `image/gif`, `video/mp4`, `video/mpeg`, `video/quicktime`,
`video/avi`, `video/x-msvideo`, `video/webm`, `video/x-m4v`.

**Prefer JPEG for images.** PostZen hands Instagram the file as stored and
does no conversion for Instagram. Meta documents JPEG as the image format.
If the carousel renderer produced PNG, or the cover is WebP, convert to JPEG
locally first (ImageMagick, `sips` on macOS, Pillow) and upload the JPEG.
Do not tell the user PostZen will convert it, because it will not.

**Limits PostZen checks:** images over 8 MB and videos over 1 GB only produce
a warning, and Instagram may still reject them. Up to 10 `mediaItems` per
post. The caption (`content`) is capped at 2,200 characters, as is
`firstComment`.

**Limits PostZen does not check, so you do:** aspect ratio and video
duration. Build feed and carousel images at 1080x1350 (4:5) or 1:1, and
reels and stories at 1080x1920 (9:16). `/ig-reel` has the length guidance.

A presigned upload that never ends up in a post is deleted after about 24
hours. That is fine for posting. It matters for `/ig-dm`, which must not use
presign URLs for automation image cards.

## Step 3: build the call

One `createPost` per post. The caption goes in `content`. The Instagram
target goes in `platforms[]` with `platform: "instagram"`, the `accountId`
from Step 1, and `settings`.

`settings` keys are **camelCase only**. Snake case is silently ignored, and
a silently ignored `postType` means a reel gets posted as a feed image and
fails. The keys that exist:

| key | values | rules |
| --- | --- | --- |
| `postType` | `"feed"` (default), `"reel"`, `"carousel"`, `"story"` | decides everything below |
| `collaborators` | up to 3 usernames | feed, reel and carousel |
| `userTags` | `[{ username, x, y }]`, x and y between 0 and 1 | **feed only**. A validation error on carousels, ignored on reels and stories |
| `firstComment` | up to 2,200 characters | posted after publish, best effort: if it fails, the post still stands and nobody is told. Check it in the app |
| `shareToFeed` | boolean, default `true` | reels only |

What each `postType` needs:

| `postType` | media | what fails |
| --- | --- | --- |
| `feed` | exactly 1 image | video ("supports images only. Choose Reel for video"), 2+ items ("Use postType carousel") |
| `carousel` | 2 to 10 items, images and videos can mix, in `mediaItems` order | `userTags` |
| `reel` | exactly 1 video | any image |
| `story` | exactly 1 image or video | a caption, quietly: Meta ignores story captions and PostZen does not send one |

Stories through the API are the bare frame. No poll, question box, link,
quiz, slider or countdown. `/ig-story` knows this and keeps sticker frames
manual.

An example, a scheduled reel:

```json
{
  "content": "Proposals used to take me five hours. Twenty minutes now.\n\nThe template is in the first comment.\n\n#freelance #proposals #agencyowner",
  "mediaItems": [{ "url": "https://media.postzen.dev/.../reel.mp4" }],
  "platforms": [{
    "platform": "instagram",
    "accountId": "<_id from listAccounts>",
    "settings": {
      "postType": "reel",
      "shareToFeed": true,
      "firstComment": "Template: https://example.com/proposal-template"
    }
  }],
  "scheduledFor": "2026-10-07T19:30:00-04:00",
  "x-request-id": "ig-reel-2026-10-07-proposals"
}
```

`x-request-id` is an idempotency key. Repeating a call with the same value
returns the original post instead of creating a second one, which is the
only safe way to retry.

**Hashtags.** PostZen does not count them. `/ig-caption` enforces the cap of
five, so lint there, not here.

## Step 4: pick exactly one mode

| mode | set | notes |
| --- | --- | --- |
| **draft** | `isDraft: true` | saved in PostZen, nothing goes out. `platforms` is optional. No confirmation needed |
| **now** | `publishNow: true` | publishes inside the request |
| **schedule** | `scheduledFor: "<ISO 8601>"` | at least 60 seconds in the future. An offset (`-04:00`) or `Z` both work. Always write the offset for the user's timezone rather than converting to UTC in your head |
| **queue** | `queuedFromProfile: "<profileId>"`, optional `queueId` | PostZen claims the profile's next free slot and returns it as `scheduledFor`. The profile needs a queue with slots; `/ig-plan` sets those up |

Two modes in one call is a 400. No mode at all is an error too. Never call
`getNextQueueSlot` and feed its answer back as `scheduledFor`; it is a
preview and reserves nothing. Use queue mode.

**When to post.** If the user has not chosen a time, offer
`getBestTimeToPost({ platform: "instagram", accountId })`. It returns
`slots[]` of `day_name`, `hour` (**UTC**), `avg_engagement` and
`post_count`, ranked by engagement, from the account's own last 366 days of
posts with analytics. Convert every hour to the user's timezone before you
show it, and show `post_count`: a slot built on one post can rank first.
"Engagement" here is likes + comments + shares + saves. It knows nothing
about reach or when followers are awake. An account with ten posts of
history gets a guess, and you should call it one.

Every time shown to the user carries a timezone. Silent UTC is how a reel
goes out at 3 AM.

## Step 5: the gate

Before `createPost` with `publishNow` or `scheduledFor` or
`queuedFromProfile`, show all of this and get an explicit yes in this
conversation:

```
PUBLISH  ·  @handle (instagram)

type:       reel, shareToFeed on
media:      reel.mp4, 48 MB, 1080x1920, 28s
caption:    "Proposals used to take me five hours. Twenty minutes now." (389 chars, 3 tags)
first comment: the template link
when:       Tue Oct 7, 7:30 PM EDT (2026-10-07T19:30:00-04:00)

Reply "yes" to schedule it, or tell me what to change.
```

"Looks good" about the script earlier in the conversation is not a yes to
this. The yes is to this block, with this account and this time. Drafts
(`isDraft: true`) do not need it.

If the user wants to change the caption, change it and show the block again.
Do not send a version they have not seen.

## Step 6: verify, because the message lies by omission

`createPost` returns `{ post, message }`. The `message` is chosen by mode,
not by outcome: "Post published successfully" comes back even when Instagram
rejected the container, because the provider error is caught and logged and
the request still returns 201. So:

1. Read `post.platforms[].status`. It is one of `draft`, `scheduled`,
   `pending`, `publishing`, `published`, `failed`, `canceled`.
2. If `failed`, read `post.platforms[].error` and tell the user the actual
   text. Do not retry blindly. If a retry makes sense (a transient network
   error, say), reuse the same `x-request-id`.
3. If `publishing`, Instagram is still processing the container. Reels and
   carousels do this routinely. Poll `getPost({ postId: post._id })` about
   every 30 seconds. PostZen gives up after 20 minutes and marks the target
   `failed` with `container_timeout`. Tell the user what you are waiting on
   rather than going quiet.
4. If `published`, take `platformPostId` (the Instagram media id, which
   `/ig-reply` and `/ig-audit` use) and `platformPostUrl` (the permalink).
5. If `scheduled`, repeat the time back with its timezone and stop. The post
   goes out later; nothing to verify yet.

A `402` with `freePostLimitExceeded` means the month's 20 are used. Say so,
save the post as a draft if the user wants, and leave the upgrade decision
to them.

## Step 7: log it

On `published` or `scheduled`, append one line to
`~/.claude/instagram/log.md`. `/ig-audit` reads it to match hook formulas to
results and `/ig-plan` reads it to avoid repeating a theme:

```
2026-10-07  REEL  #5 Time Collapse  "Proposals used to take me five hours."  postzen:<post._id>  https://www.instagram.com/reel/XXXX/
```

Date, format, hook formula if the content skill named one, the first line,
the PostZen post id, and the permalink if there is one yet. Create the file
if it does not exist. If `/ig-reel` already logged this post when the script
was approved, add `postzen:<post._id>` and the permalink to that line rather
than writing a second one; one post, one line.

## Afterwards

- **Change a scheduled post:** `updatePost({ postId, ... })`. Same fields as
  create, every one optional, omitted fields keep their values. Works on
  `draft`, `scheduled`, `queued`, `failed`, `partially_failed` and
  `canceled`. Published and publishing posts cannot be edited, by PostZen or
  by the Instagram API.
- **Cancel a scheduled post:** `deletePost({ postId })`. Same statuses.
  Deleting a published post in PostZen is not possible, and nothing PostZen
  does removes a post from Instagram. That is done in the app.
- **See what is scheduled:** `listPosts({ platform: "instagram", accountId,
  status: "scheduled", sortBy: "scheduledFor" })`. It covers posts created
  in PostZen only.

## What this skill cannot do

The Instagram API, through PostZen or anyone else, does not offer these.
Say so when asked rather than attempting a workaround:

- Reel cover image or thumbnail choice. Instagram picks a frame; the user
  changes it in the app after publishing.
- Audio or music selection, trial reels, going live.
- Location tags, product tags, alt text. Alt text is added in the app after
  publishing, and it is worth the 20 seconds.
- User tags anywhere but single-image feed posts.
- Stickers of any kind on stories, and story captions.
- Editing a post after it is published, or deleting one from Instagram.
- Hashtag counting. That is `/ig-caption`'s linter.
- Checking aspect ratio or video length before Instagram does.

## Cautions

- Publishing is a real, outward-facing action. Never call `createPost` with
  `publishNow: true`, a `scheduledFor`, or `queuedFromProfile` without the
  user's explicit yes to the exact content, account and time in this
  conversation.
- Never invent `accountId` or `profileId` values. They come from
  `listAccounts` and `listProfiles` in this session.
- If a `createPost` call fails, report the actual error. Blind retries can
  double-post if the first attempt landed; a retry with the same
  `x-request-id` cannot.
- "Post published successfully" is a mode label. `platforms[].status` is
  the truth.
- One post, one call. For a week of posts, `/ig-repurpose` collects one
  confirmation that lists every post, then this skill sends them one at a
  time and verifies each.
