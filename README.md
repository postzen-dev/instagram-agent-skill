# The Instagram agent skill

Fourteen Claude skills that run an Instagram account, and with
[PostZen](https://www.postzen.dev) connected, can publish, schedule, reply to
comments and DMs, run comment-to-DM automations and read analytics through
the official Instagram API. Free, MIT. PostZen's free plan is $0 with no card.
Without PostZen, every skill still works the way it always did: it writes,
you post.

One of them writes your Reels off 26 hook formulas and scores the hook before
you waste a take on it. One goes and finds the reels that are actually working
in your niche and ranks them by how far each beat its own account. One writes
the caption and shows you exactly what the feed shows before the "... more".
One scores your profile out of 100 and rewrites what lost points. One plans the
week. One publishes, and then checks that it really published.

And one is the humanizer, which is the reason the rest are usable. It strips
the em dashes, the slop vocabulary and the invisible watermark characters out
of a draft, then scores what is left against a five-check panel before you ever
see it.

**Nothing gets posted until you say yes.** Not posted, not scheduled, not
replied, not hidden, not sent. Every skill shows you the exact content, the
account and the time, and waits.

## Install

As a plugin, in Claude Code:

```
/plugin marketplace add postzen-dev/instagram-agent-skill
/plugin install postzen-instagram
```

The plugin registers the PostZen MCP server for you. Then run `/mcp`, pick
`postzen`, and Authenticate. A browser tab opens on PostZen, you sign in there,
and nothing is pasted into Claude. Pick **Read & Write** if you want to
publish; Read Only is enough for audits.

Or copy the skills by hand:

```bash
git clone https://github.com/postzen-dev/instagram-agent-skill.git
cp -r instagram-agent-skill/skills/ig-* ~/.claude/skills/
claude mcp add --transport http postzen https://mcp.postzen.dev/mcp
```

Then `/mcp`, `postzen`, Authenticate, as above. Project-local instead of
global: copy the same folders into your repo's `.claude/skills/`. No Claude
Code at all? Paste any single `SKILL.md` at the top of a chat and it runs as a
mode. You lose the five Python tools and the PostZen tools, but the writing
works.

**Connecting Instagram.** Say "connect my Instagram" and `/ig-publish` walks
you through it: it asks PostZen for a connect link, you open it and complete
Instagram Login, done. The account has to be an Instagram **Business or
Creator** account; personal accounts cannot connect. No Facebook Page is
needed. PostZen's free plan covers 2 connected accounts and 20 posts a month,
no card required.

Then spend ten minutes on `templates/voice.md`. Copy it to
`~/.claude/instagram/voice.md` and fill it in, or send Claude three of your own
reels and say "write my voice.md from these". Every skill reads that file. It
matters more here than on other platforms, because you have to say the words
out loud.

## The fourteen

| command | what it does | with PostZen connected |
| --- | --- | --- |
| `/ig-reel` | One idea into a Reel. Three hooks from [26 formulas](skills/ig-reel/hooks.json), scored, then the script, the on-screen text and a timed beat sheet. | hands the finished video to `/ig-publish` |
| `/ig-viral` | Goes and finds what is working in your niche, ranks it by multiple over each account's own median, names the formula, writes the swipe file. | unchanged. It reads, it never scrapes |
| `/ig-caption` | The caption, linted. Shows you the 125 characters the feed actually shows before the tap. | caption and first comment go straight into the post |
| `/ig-carousel` | Swipe posts. The cover that earns the swipe, slide copy, and the 1080x1350 files. | publishes up to 10 slides, in order |
| `/ig-story` | The daily story sequence, which sticker does which job, and the DM funnel that starts with them moving first. | plain frames publish; sticker frames stay manual, because the API has no stickers |
| `/ig-profile` | Scores your profile against a [12-part rubric](skills/ig-profile/rubric.json) out of 100, then rewrites in fix-first order. | unchanged. You edit the fields |
| `/ig-plan` | The week. What to post, which format, when, and the 10 accounts to engage with. | best time from your own history, what is already scheduled, queue slots |
| `/ig-human` | The humanizer. Two scripts that actually run. See below. | unchanged |
| `/ig-comment` | Comments on other people's posts. Nine types, picked by what the post actually is. Never "🔥🔥🔥". | unchanged, deliberately. The API does not comment on other people's posts |
| `/ig-reply` | The thread under your own post. Sorts into keyword / lead / substance / question / support / noise, then writes in that order. | fetches the comments, sends the replies, hides the noise |
| `/ig-dm` | The keyword delivery, the first message, the collab pitch, and the two follow-ups. Two. | the keyword delivery becomes a real comment-to-DM automation; replies inside the 24-hour window go out from here |
| `/ig-repurpose` | One video, podcast or newsletter into a week of reels and carousels that each stand alone. | schedules the week on one confirmation |
| `/ig-audit` | Post-mortem on what you already posted. Ranks by outlier multiple and sends per reach, not views. | pulls views, reach, likes, comments, shares, saves and follower history itself |
| `/ig-publish` | The one skill that touches Instagram. Media in, `createPost`, the yes, then it verifies the post actually went out and logs it. | this is the PostZen skill |

## The five tools that actually run

No dependencies, no network, nothing uploaded. They run on your machine, on
your text.

### Hooks

```bash
python3 hookscore.py hooks.txt              # rank your options
python3 beats.py script.txt --target 30     # time it before you shoot it
```

```
HOOK RANKING
==============================================================================
->  85.6 STRONG  Nobody tells you that your first 30 reels are supposed to f...
        weakest: SPECIFICITY (75)
    81.4 STRONG  I lost $18,000 because of one missing contract.
        weakest: ADDRESS (70)
    54.4 OK      Stop scrolling if you want to grow on Instagram in 2026 🔥
        weakest: STAKES (70)
        dealbreaker: Opens with "stop scrolling". Asking for attention proves
                     you have not earned it.
     9.6 WEAK    Hey guys, today I wanted to talk about content strategy
        weakest: FRONTLOAD (0)
        dealbreaker: Greeting. Nobody came to the feed to be greeted.
```

`beats.py` estimates how long each line takes to say, stacks them into
timecodes, and flags the four things that kill a Reel in the edit: a hook that
runs past three seconds, a beat long enough for the viewer to leave, a run of
lines with nothing concrete in them, and no loop back to the first line.

```
BEAT SHEET  ·  73 words  ·  ~26.6s at 165 wpm  ·  target 30.0s
========================================================================
  0:00.0   2.9s  HOOK    I lost $18,000 because of one missing contract.
  0:02.9  14.6s  MID     Here is the exact clause I now put in every single...
                         ^ 14.6s on one beat
  0:17.4   2.9s          Section 4. Payment on delivery, not on approval.
  0:20.4   2.9s          Approval is a feeling. Delivery is a date.
  0:23.3   3.3s  CTA     That one word change is worth $18,000 to me.
------------------------------------------------------------------------
  - Beat 2 runs past 4s. Either split the line or change what is on screen
    inside it. A static frame is where people leave.
  - Loops: the last beat repeats "$18,000" from the hook. Second watches are
    free reach.
  - 3.5s under target. Either add 9 words or shoot it short. Short is usually
    right.
```

### The caption

```bash
python3 caption.py caption.txt --keywords "client contracts,freelance pricing"
```

Instagram gives a caption about 125 characters in the feed and hides the rest
behind a tap. Almost every caption that fails, fails there. So the first thing
this prints is that window, as a box, the way a scrolling stranger reads it:

```
  WHAT THE FEED SHOWS
  +------------------------------------------------------+
  | I lost $18,000 because of one missing clause in a    |
  | contract, and the worst part is that I had read the  |
  | thing twice before I si                              |
  +-------------------------------------------- ... more +

  PASS  LENGTH            389 / 2200 characters
  WARN  FIRST LINE        133 characters, so it gets cut at 125 mid-thought
  PASS  HOOK IS CONCRETE  2 numbers or names in the visible window
  PASS  HASHTAGS          3 tags: #freelance #contracts #agencyowner
  PASS  ONE ASK           one call to action: comment a keyword
  WARN  SEARCH TERMS      1/2 present. Missing: client contracts
```

It enforces the current hashtag cap, which is **five per post**, not thirty.
Instagram cut it on 18 December 2025.

### The humanizer

```bash
python3 humanize.py draft.txt --report      # clean it, show every change
python3 detect.py draft.txt                  # score it, five checks
python3 detect.py before.txt after.txt       # prove the delta
```

**What comes out automatically:**

- **Invisible characters.** Zero-width spaces and joiners, word joiners, soft
  hyphens, byte-order marks, Unicode tag characters, invisible separators,
  non-breaking and narrow spaces. Your keyboard does not make these. They
  survive copy-paste and they are invisible in every editor you own.
- **Typography.** Em dash to comma, en dash to hyphen, curly quotes to
  straight, ellipsis to three dots, and the orphaned punctuation that leaves.
- **The lexicon.** 154 stock words and phrases with plain-English replacements.
  The last block of it is Instagram-specific: "stop scrolling", "in today's
  video", "follow for more", "tag someone who needs this", "the algorithm
  loves", "run don't walk". It lives in
  [`slop.json`](skills/ig-human/slop.json) and it is meant to be edited.

**What gets flagged instead of fixed:** "It's not just X, it's Y", rule-of-three
triads, the video preamble, emoji bullet lists, three shouted words in a row,
hashtag walls, reflex follow bait, uniform sentence length. Changing the shape
of a sentence needs judgement, so those come back for a rewrite rather than
getting mangled by a regex.

Run against a caption written to be as bad as possible:

```
  BURSTINESS    ###################.....  78.4
  SPECIFICITY   #############...........  53.6    3.4 concrete markers per 100 words
  SLOP DENSITY  ........................   0.0    20 stock terms, 23.0 per 100 words
  FINGERPRINT   ################........  68.7    1 em dash
  VOICE         #################.......  70.0    9 structural tells
  ------------------------------------------------------------
  HUMAN SCORE   ########................  32.5   FLAGGED
```

After `humanize.py`, with the flagged structures still unrewritten:

```
  HUMAN SCORE   ###################.....  79.7   PASS    (+47.2)
```

### The swipe file

```bash
python3 swipe.py captured.tsv --out ~/.claude/instagram/swipe.md
```

Raw views are not evidence. A 2,000,000-follower account doing 400,000 views
had a quiet Tuesday. A 4,000-follower account doing 400,000 views found
something. `swipe.py` ranks on the multiple over each account's own median,
names the hook formula, and prints what separates the top third from the
bottom third.

```
SWIPE FILE  ·  4 reels  ·  4 accounts  ·  baseline: account median
==============================================================================
    60.0x  hook  57  #9  The Steal              @c                 180,000
           "steal this four line follow up it took me two years"
    37.5x  hook  86  #3  Nobody Tells You       @a                 412,000
           "nobody tells you that your first 30 reels are supposed to flop"
     1.3x  hook  13  -   unclassified           @b               1,200,000
           "in this video I am going to show you my morning routine"
```

## What PostZen adds, and what it does not

PostZen is a social media API with a hosted MCP server. It has already done
the Meta developer app, the app review and the token handling, so Claude gets
the official Content Publishing API without you building any of that.
Through it the skills can:

- **Publish, schedule, queue or draft** feed images, reels, carousels of 2 to
  10 items, and bare story frames. `/ig-publish` then reads the real status
  back, because a "published successfully" message is a mode label, not a
  result. Reels and carousels can sit in `publishing` for a while; it waits
  and checks.
- **Read and answer comments** on your own posts, including posts you
  published by hand. Hide the noise rather than delete it.
- **Reply to DMs** in an existing thread, inside Meta's 24-hour window.
- **Run comment-to-DM automations**: someone comments the keyword, they get
  the DM and a public reply, with keyword modes, rotating variations, link
  buttons or an image card, delays, an audience rule and an optional follow
  gate. Meta's rules apply: one private reply per comment, within 7 days.
- **Read analytics**: views, reach, likes, comments, shares, saves and
  engagement rate per post, the follower count over time, and a best-time
  table built from your own posts.

What it does not do, because the Instagram API does not:

- Choose a reel cover, pick audio, add a location, product tags or alt text.
  User tags work on single-image feed posts only.
- Put stickers on a story, or a caption. A poll frame is a manual post.
- Open a new DM thread, or message anyone outside the 24-hour window.
- Comment on other people's posts. That stays manual, and it should.
- Edit or delete a post once it is published.
- Hand over retention, watch time, non-follower reach, follows per post,
  profile visits, link taps or any story metric. Those stay a screenshot
  from the app.

## What was actually measured, which is the part worth reading

The hook scorer was not published on the claim that it feels right. It was
tested. The corpus is **74 real short-form hooks**: the first three seconds of
the auto-caption track from the top eight and bottom eight performing shorts
on each of five channels, view counts from 931 to 550,000.

They are YouTube Shorts rather than Reels, because Instagram does not hand you
view counts you can collect without logging into somebody's account, and
scraping it would violate the Terms this repo tells you not to violate. The
hook grammar is the same and the sourcing is public. That is a real limitation
and it is stated here rather than buried.

**Three results, two of them uncomfortable:**

**1. It catches bad hooks well.** Against ten hooks written deliberately badly,
AUC 0.83, and nine of the ten scored below the median of the real corpus. If
your hook opens on a greeting, a preamble or nothing concrete, this tells you.

**2. It does not pick winners.** Separating a good creator's hits from that
same creator's misses: **AUC 0.56, where 0.50 is a coin flip.** Of the five
checks, only SPECIFICITY separated the bands meaningfully, by 60 points of
median. STAKES and ADDRESS had identical medians in both bands, which means on
this corpus they measured nothing.

So the honest use is: kill the obviously weak hooks before you shoot them, then
trust your own retention graph. Nothing that reads text can tell you which of
two decent hooks will travel, because that is decided by your face, your edit,
your audio and who Instagram shows it to.

**3. The formula classifier was broken and the test is what caught it.** The
`match` regexes in `hooks.json` were first written off the formula templates
themselves, and they named **8%** of real hooks. People do not speak in
templates. Rewriting them against actual transcribed speech took it to
**49%**, and four formulas went into the set because they kept appearing and
were not there: Contrarian Flip, The Statistic, The Reveal, Someone Else's
Result, plus The Superlative. On the other 51% it abstains, which is correct:
a lot of short-form is podcast clips that have no hook formula at all.

One fixed bug worth naming: `SPECIFICITY` only counted digits, so "zero
dollars" and "three marketing books" scored as having nothing concrete in them.
Spoken hooks say their numbers out loud.

If you re-run this on a bigger or cleaner corpus and get a different answer,
open an issue. A different answer is worth knowing. The measurement script is
not in the repo because it depends on `yt-dlp`, but the method is four lines
and is written out in [`/ig-viral`](skills/ig-viral/SKILL.md).

## The fine print, which is the honest part

**Posting goes through the official API, and only through it.** PostZen uses
Instagram's Content Publishing API for Professional accounts, which is the
one sanctioned way to post by machine. It needs a Business or Creator account
and it cannot do everything the app does; the list above is the list.
Everything else people use to automate posting, commenting, following or
DMing is browser automation or a scraping tool, and both violate
[Instagram's Terms of Use](https://help.instagram.com/581066165581870) and get
accounts action-blocked. No skill here drives a browser to post, comment,
follow or message, with or without PostZen. Where the API has no door, the
skill ends in a copy-ready block and you post it.

**The approval gate is real, not a setting.** Before anything is published,
scheduled, replied, hidden or sent, the skill shows the exact content, the
account handle and the time, and waits for a yes to that. A "looks good" about
a draft three messages ago does not count. Drafts saved in PostZen need no
confirmation, because nothing leaves.

**`/ig-viral` reads, it does not scrape.** Ten accounts, a dozen reels each, at
human speed, with you driving your own browser. It never asks for your password
and never logs in as you. Automated collection at volume is the thing that gets
accounts restricted, and a crawler is not what this is.

**No skill handles your Instagram password.** Connecting goes through
Instagram Login in your own browser; PostZen's MCP server is authorized in
your browser too. Nothing is pasted into Claude.

**The five detection checks are local heuristics, not detector APIs.** They are
modelled on the signals public detectors key on and they run entirely on your
machine. They are not GPTZero, Originality, Copyleaks, Winston or Turnitin,
they do not call those services, and they cannot promise those verdicts. Fixing
what they measure tends to move those numbers, because they are measuring the
same underlying things. That is the whole claim. Nobody can honestly sell you
"undetectable", and anybody who does is selling you something.

**The invisible-character pass is real and it is narrow.** It removes the
zero-width and format characters that end up in generated text and survive a
copy-paste. That is a genuine, checkable fingerprint. It is not a claim about
defeating a cryptographic watermarking scheme, and this repo does not make one.

**Nothing here fabricates.** No invented metrics, clients or outcomes go under
your name. If a draft needs a number you have not given, it comes back with
`{{your number}}` in it and a flag, every time. The analytics numbers in
`/ig-audit` are the ones PostZen read from Instagram or the ones you
screenshotted, and the skill says which.

**Platform numbers go stale.** The hashtag cap moved from 30 to 5 in December
2025 while this repo was being written, and the linter had the old number in it
until the fact got checked. PostZen's limits and plan numbers in this README
were checked in October 2026. If something here contradicts what Instagram or
PostZen is doing when you read it, they are right.

## Files

```
.mcp.json                          registers the PostZen MCP server for the plugin
skills/ig-publish/SKILL.md         the only skill that touches Instagram
skills/ig-reel/hooks.json          26 hook formulas: template, example, on-screen line,
                                   what it is for, how it gets ruined, and a match regex
skills/ig-reel/hookscore.py        the five-property hook panel
skills/ig-reel/beats.py            script to timed beat sheet
skills/ig-caption/caption.py       the truncation preview and the caption linter
skills/ig-human/slop.json          the lexicon: 154 terms, 18 invisible classes, 16 tells
skills/ig-human/humanize.py        the three cleaning passes
skills/ig-human/detect.py          the five-check panel
skills/ig-viral/swipe.py           outlier ranking and formula classification
skills/ig-profile/rubric.json      the 100-point profile score
templates/voice.md                 your voice profile. Fill this in first.
```

## Credits

This pack is a fork of Jake Schincariol's
[instagram-agent-skill](https://github.com/Jakeschincariol/instagram-agent-skill)
(MIT). He wrote the original thirteen skills, the 26 hook formulas, the
humanizer and the five Python tools, and the measurement above is his.
PostZen added `/ig-publish` and the publishing, inbox, automation and
analytics layer across the other skills.

## License

MIT. Take it, change it, ship it.
