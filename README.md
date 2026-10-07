# The Instagram agent skill

Fourteen Claude skills for running an Instagram account. Connect [PostZen](https://www.postzen.dev) to publish, schedule, reply to comments and DMs, run comment-to-DM automations, and read analytics through the official Instagram API.

The skills cover Reel scripts, niche research, captions, profile reviews, weekly planning, and publishing. The Reel writer uses 26 hook formulas and scores your options. The research skill ranks Reels against each account’s own performance. The caption writer previews the text visible before “… more”. The profile reviewer scores your account out of 100 and rewrites the parts that lost points.

The humanizer removes em dashes, stock phrases, and invisible watermark characters. It then scores the draft against five checks before showing it to you.

**Nothing gets posted until you say yes.** Each skill shows you the exact content, account, and time.

## Install

Install the plugin in Claude Code:

```text
/plugin marketplace add postzen-dev/instagram-agent-skill
/plugin install postzen-instagram
```

The plugin registers the PostZen MCP server. Run `/mcp`, select `postzen`, and choose Authenticate. Sign in through the browser tab that opens on PostZen. Select **Read & Write** to enable publishing.

Or copy the skills by hand:

```bash
git clone https://github.com/postzen-dev/instagram-agent-skill.git
cp -r instagram-agent-skill/skills/ig-* ~/.claude/skills/
claude mcp add --transport http postzen https://mcp.postzen.dev/mcp
```

Then run `/mcp`, select `postzen`, and authenticate.

For a project-local installation, copy the same folders into your repo’s `.claude/skills/`.

You can also paste a single `SKILL.md` at the top of a chat to use it as a mode. The writing works without Claude Code, but the Python and PostZen tools are unavailable.

**Connect Instagram:** Say “connect my Instagram”. `/ig-publish` requests a connect link from PostZen. Open it and complete Instagram Login. You need an Instagram **Business or Creator** account; personal accounts cannot connect. PostZen’s free plan covers 2 connected accounts.

Spend ten minutes filling in `templates/voice.md`. Copy it to `~/.claude/instagram/voice.md`, or send Claude three of your own Reels and say “write my voice.md from these”. All fourteen skills read that file. For Reel scripts, the wording needs to sound like something you would say out loud.

## The fourteen skills

| Command | What it does | With PostZen connected |
| --- | --- | --- |
| `/ig-reel` | Turns one idea into a Reel. Generates and scores three hooks from [26 formulas](skills/ig-reel/hooks.json), then writes the script, on-screen text, and timed beat sheet. | Passes the finished video to `/ig-publish`. |
| `/ig-viral` | Finds Reels working in your niche. Ranks them by their multiple over each account’s median, identifies the formula, and writes a swipe file. | Unchanged. It reads without scraping. |
| `/ig-caption` | Writes and lints the caption. Previews the 125 characters visible in the feed before the tap. | Adds the caption and first comment to the post. |
| `/ig-carousel` | Creates the cover, slide copy, and 1080x1350 files for swipe posts. | Publishes up to 10 slides in order. |
| `/ig-story` | Plans the daily story sequence, assigns a purpose to each sticker, and builds a DM funnel where the viewer initiates contact. | Publishes plain frames. Sticker frames require manual posting because the API does not support stickers. |
| `/ig-profile` | Scores your profile out of 100 against a [12-part rubric](skills/ig-profile/rubric.json). Rewrites it in priority order. | Unchanged. You edit the profile fields. |
| `/ig-plan` | Plans the week: posts, formats, times, and 10 accounts to engage with. | Uses your posting history to find the best time, checks scheduled posts, and uses queue slots. |
| `/ig-human` | Cleans and scores drafts with two Python scripts. | Unchanged. |
| `/ig-comment` | Writes comments on other people’s posts. Chooses from nine types based on the post’s content. Avoids generic “🔥🔥🔥” replies. | Unchanged. The API cannot comment on other people’s posts. |
| `/ig-reply` | Sorts comments on your posts into keyword / lead / substance / question / support / noise, then writes replies in that order. | Fetches comments, sends replies, and hides noise. |
| `/ig-dm` | Writes keyword deliveries, first messages, collaboration pitches, and two follow-ups. | Creates comment-to-DM automations and sends replies within the 24-hour window. |
| `/ig-repurpose` | Turns one video, podcast, or newsletter into a week of Reels and carousels. Each post stands on its own. | Schedules the week after one confirmation. |
| `/ig-audit` | Reviews published posts. Ranks them by outlier multiple and sends per reach rather than views. | Pulls views, reach, likes, comments, shares, saves, and follower history. |
| `/ig-publish` | Handles Instagram writes. Takes the media, prepares `createPost`, asks for approval, then verifies and logs publication. | Provides the PostZen publishing workflow. |

## Command-line tools

### Hooks

```bash
python3 hookscore.py hooks.txt              # rank your options
python3 beats.py script.txt --target 30     # time it before you shoot it
```

```text
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

`beats.py` estimates how long each line takes to say and assigns timecodes. It flags hooks longer than three seconds, beats that drag, consecutive lines without concrete details, and endings that do not loop back to the opening.

```text
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

### Captions

```bash
python3 caption.py caption.txt --keywords "client contracts,freelance pricing"
```

Instagram shows about 125 caption characters in the feed and hides the rest behind a tap. The script starts with a boxed preview of that visible text:

```text
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

It enforces the **five-hashtag limit per post**. Instagram reduced the cap from thirty on 18 December 2025.

### The humanizer

```bash
python3 humanize.py draft.txt --report      # clean it, show every change
python3 detect.py draft.txt                  # score it, five checks
python3 detect.py before.txt after.txt       # prove the delta
```

The cleaner handles:

- **Invisible characters:** Zero-width spaces and joiners, word joiners, soft hyphens, byte-order marks, Unicode tag characters, invisible separators, non-breaking spaces, and narrow spaces. These can survive copy-paste and go unnoticed in an editor.
- **Typography:** Converts em dashes to commas, en dashes to hyphens, curly quotes to straight quotes, and ellipses to three dots. Cleans up punctuation left behind by those changes.
- **Stock wording:** Replaces 154 words and phrases with plain-English alternatives. The Instagram-specific entries include “stop scrolling”, “in today’s video”, “follow for more”, “tag someone who needs this”, “the algorithm loves”, and “run don’t walk”. Edit the list in [`slop.json`](skills/ig-human/slop.json).

The detector flags sentence structures that need a rewrite: “It’s not just X, it’s Y”, rule-of-three triads, video preambles, emoji bullet lists, three shouted words in a row, hashtag walls, reflex follow bait, and uniform sentence lengths. It leaves these for review because regex replacements can damage the meaning.

A caption written to trigger the checks scored:

```text
  BURSTINESS    ###################.....  78.4
  SPECIFICITY   #############...........  53.6    3.4 concrete markers per 100 words
  SLOP DENSITY  ........................   0.0    20 stock terms, 23.0 per 100 words
  FINGERPRINT   ################........  68.7    1 em dash
  VOICE         #################.......  70.0    9 structural tells
  ------------------------------------------------------------
  HUMAN SCORE   ########................  32.5   FLAGGED
```

After `humanize.py`, with the flagged structures still awaiting a rewrite:

```text
  HUMAN SCORE   ###################.....  79.7   PASS    (+47.2)
```

### The swipe file

```bash
python3 swipe.py captured.tsv --out ~/.claude/instagram/swipe.md
```

View counts need context. A post with 400,000 views means something different on an account with 2,000,000 followers than on one with 4,000.

`swipe.py` ranks Reels by their multiple over each account’s median. It identifies the hook formula and compares the top third with the bottom third.

```text
SWIPE FILE  ·  4 reels  ·  4 accounts  ·  baseline: account median
==============================================================================
    60.0x  hook  57  #9  The Steal              @c                 180,000
           "steal this four line follow up it took me two years"
    37.5x  hook  86  #3  Nobody Tells You       @a                 412,000
           "nobody tells you that your first 30 reels are supposed to flop"
     1.3x  hook  13  -   unclassified           @b               1,200,000
           "in this video I am going to show you my morning routine"
```

## PostZen integration

PostZen is a social media API with a hosted MCP server. It handles the Meta developer app, app review, and tokens. Claude can use the official Content Publishing API without you setting those up.

With PostZen, the skills can:

- **Publish, schedule, queue, or draft** feed images, Reels, carousels of 2 to 10 items, and plain story frames. `/ig-publish` reads back the post’s status to verify publication. A “published successfully” message can describe the requested mode without confirming the result. Reels and carousels may remain in `publishing` while processing, so the skill waits and checks again.
- **Read and answer comments** on your own posts, including posts you published by hand. It hides unwanted comments instead of deleting them.
- **Reply to DMs** in existing threads within Meta’s 24-hour window.
- **Run comment-to-DM automations.** A keyword comment triggers a DM and public reply. Options include keyword modes, rotating variations, link buttons or an image card, delays, an audience rule, and an optional follow gate. Meta allows one private reply per comment, sent within 7 days.
- **Read analytics:** Views, reach, likes, comments, shares, saves, and engagement rate per post. It also retrieves follower counts over time and a best-time table based on your posts.

The integration has these Instagram API limits:

- It cannot choose a Reel cover, select audio, or add a location, product tags, or alt text. User tags work on single-image feed posts only.
- It cannot add story stickers or captions. Poll frames require manual posting.
- It cannot open a new DM thread or message someone outside the 24-hour window.
- It cannot comment on other people’s posts. You post those comments by hand.
- It cannot edit or delete a published post.
- It does not provide retention, watch time, non-follower reach, follows per post, profile visits, link taps, or story metrics. Supply those as screenshots from the Instagram app.

## Testing and limitations

The hook scorer was tested against **74 real short-form hooks**. The corpus uses the first three seconds of the auto-caption track from the top eight and bottom eight performing Shorts on each of five channels. View counts range from 931 to 550,000.

The corpus contains YouTube Shorts rather than Reels. Instagram does not provide view counts for collection without logging into someone’s account, and scraping would violate the Terms this repo requires users to follow. YouTube provides public sourcing for comparable hook structures. Testing on Shorts remains a limitation.

### 1. It catches weak hooks

Against ten intentionally bad hooks, the scorer achieved **AUC 0.83**. Nine of the ten scored below the real corpus’s median. It can flag greetings, preambles, and openings without concrete details.

### 2. It does not predict winners

For separating a creator’s hits from that same creator’s misses, the scorer achieved **AUC 0.56**, where **0.50 is a coin flip**.

Of the five checks, only SPECIFICITY meaningfully separated the bands, with a 60-point difference between their medians. STAKES and ADDRESS had identical medians in both bands. Those two checks showed no separation in this corpus.

Use the scorer to remove weak hooks before filming, then assess results through your retention graph. Text scoring alone cannot reliably choose between two decent hooks. Delivery, editing, audio, and the audience Instagram shows the Reel to also affect performance.

### 3. Testing exposed a broken formula classifier

The original `match` regexes in `hooks.json` followed the formula templates. They classified just **8%** of real hooks.

Rewriting the regexes against transcribed speech raised coverage to **49%**. Four missing formulas appeared often enough to add: Contrarian Flip, The Statistic, The Reveal, and Someone Else’s Result. The Superlative was added too.

The classifier abstains on the remaining **51%**. Many short-form videos are podcast clips without a recognizable hook formula.

Testing also caught a `SPECIFICITY` bug: it counted digits but missed written numbers. Hooks containing “zero dollars” or “three marketing books” scored as though they had no concrete details.

Open an issue if you rerun the test on a larger or cleaner corpus and get different results. The measurement script is absent from the repo because it depends on `yt-dlp`. The four-line method is documented in [`/ig-viral`](skills/ig-viral/SKILL.md).

## Files

```text
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

This pack is a fork of Jake Schincariol’s [instagram-agent-skill](https://github.com/Jakeschincariol/instagram-agent-skill), licensed under MIT.

PostZen added `/ig-publish` and integrated publishing, inbox management, automations, and analytics across the other skills.

## License

MIT.