# Weekly Blog Automation

An n8n workflow that writes one blog post a week from a Google Sheet of topics. It researches the topic, writes the post, makes three images, saves the post in Shopify as a **hidden draft**, posts a shorter version to LinkedIn, and emails a review summary.

Nothing goes live on the store until a person publishes the draft in Shopify. The LinkedIn post, however, is published **straight away**.

---

## What happens each week

Runs weekly, or by hand with **Run manually (test)**.

1. **Pick a topic.** Reads the queue sheet and takes the highest-priority row with status `Queued`. If none is left, it emails "queue empty" and stops.
2. **Check Shopify.** Gets an access token, confirms the app has the scopes it needs, finds the blog, and lists the posts already on it (used for internal links, to avoid repeating a post, and to keep URLs unique).
3. **Research.** One AI call with web search gathers facts, each with its source link.
4. **Write.** A second AI call writes the post in the brand voice, using only researched facts, and picks one to three tags from a fixed list.
5. **Check the article.** Cleans the HTML, removes links not on the allowed list, styles tables for phones, and runs automatic checks for the review email. A too-short or cut-off article stops the run here, before anything is saved.
6. **Images.** Generates a featured image and two in-body images, uploads them to Shopify Files, and waits until Shopify has processed them. If images fail, the post is still saved without them.
7. **Save the draft.** Creates the post in Shopify as hidden, then sets its SEO title and meta description.
8. **Update the sheet.** Sets the row to `Drafted` with the admin link and date.
9. **LinkedIn.** Writes a 350–450 word LinkedIn post from the blog post and publishes it with the featured image.
10. **Review email.** One email with the draft link, word count, tags, automatic checks, figures to verify, the images, the SEO fields, the research sources, and what happened on LinkedIn.

---

## Setup

### Credentials

| Credential | Used by |
|---|---|
| **OpenRouter** | AI: Research the Topic, AI: Write Article, AI: Generate Image, AI: Write LinkedIn Article |
| **Google Sheets OAuth2** | Sheets: Read Queue, Sheets: Mark Row Drafted |
| **Shopify** (custom auth with the app's client ID and secret) | Shopify: Get Access Token |
| **Gmail** | Email: Draft Ready, Email: Queue Empty |
| **LinkedIn OAuth2** | LinkedIn: Publish Post |

**Shopify app scopes:** `read_content`, `write_content`, `read_files`, `write_files`. After adding scopes, release a new app version and approve it on the store.

**LinkedIn app:** add the products **Share on LinkedIn** and **Sign In with LinkedIn using OpenID Connect**. In the n8n credential, turn **Organization Support** and **Legacy** off. On the LinkedIn node, set **Post As** to **Person** and choose yourself.

### Config node

| Field | Value |
|---|---|
| `shop` | The part before `.myshopify.com` |
| `shopifyApiVersion` | `2026-07` |
| `blogHandle` | `articles` |
| `authorName` | `Author name` |
| `storeUrl` | `https://store.com` |
| `notifyEmail` | Where the review email goes |
| `sheetId` / `sheetTab` | The queue sheet and its tab |
| `researchModel` | `anthropic/claude-sonnet-5.5` |
| `writerModel` | `anthropic/claude-opus-5.5` (also used for LinkedIn) |
| `imageModel` | `openai/gpt-image-2.5-sunburst` (or `google/gemini-nano-banana-2.1`) |
| `imageQuality` | OpenAI models only: `low` to `max`, default `high` |
| `imageSize` | Google models only: `1K`, `2K` or `4K`, default `1K` |
| `maxSearches` | Web searches per research run, default 10 |
| `wordCount` | Target post length, default 1500 |
| `maxTokens` | Writer output limit, default 16000 |
| `brandFacts` | The only things the AI may say about the brand |
| `linkTargets` | Store pages the post may link to, one `URL \| label` per line |
| `imageStyle` | The photographic style added to every image prompt |

Model IDs must include the provider prefix (`anthropic/…`, `openai/…`); the workflow stops early if one doesn't.

### The queue sheet

Columns: `id, priority, status, working_title, primary_keyword, research_query, brief, supporting_searches, internal_links, names_other_brands, health_sensitive, searches, clicks, impressions, draft_link, date_drafted`.

- Only rows with status `Queued` are picked; lowest `priority` number first.
- `supporting_searches` and `internal_links`: several values separated by `|`.
- `names_other_brands` / `health_sensitive`: `TRUE` adds extra accuracy rules to the prompt and extra checks to the email.
- The workflow fills in `status`, `draft_link` and `date_drafted`.

---

## Content rules (built into the prompts)

- **Selling points in every post:** 100% Arabica; single origin from one family farm; low acid ("bright, fruity flavor", not "acidity"); roasted weekly in Atlanta and shipped within 48 hours.
- **Wording:** "100% Kenyan Arabica". Never "AA". No organic, certification, lab-tested, roasted-to-order or cooperative claims, and no prices.
- **Accuracy:** figures only from the research, each linked to its source and listed for checking. No health claims (including stomach or acid reflux).
- **Structure:** a 40–60 word "Short answer", a table where useful, headings worded like searches, a section on our coffee, a call to action, an FAQ and a Sources list.
- **Tags:** one to three from this list only: Caffeine, Coffee & Health, Kenyan Coffee, Roast Guides, Flavor & Tasting, Brewing Guides, Coffee Brands, Coffee Basics, Decaf & Half-Caff. Edit `ALLOWED_TAGS` at the top of **Build Writing Request** to change it.
- **Images:** real-looking food photography of coffee, gear and beans. No faces, farms, people, packaging, logos or text.

---

## LinkedIn

- LinkedIn's API cannot publish long-form **articles**, so the workflow publishes a long **post** (up to 3,000 characters) with the featured image.
- It is built on one idea from the blog post, not a copy, and ends with a link and two or three hashtags.
- **The post goes out publicly with no human review.** The brand rules are in the prompt in **Build LinkedIn Request**; there are no wording checks after it.
- The blog post is usually still hidden when this runs, so the post links to the blog front page instead (a link to a hidden post gives "page not found"). If the post is already live, it links to the post itself.
- A LinkedIn problem (AI failure, post too long, LinkedIn refusing it) never blocks the review email. The email says what happened and includes the text to post by hand.
- Audience, length and reasoning effort are the first lines of **Build LinkedIn Request**.
- **Posting as the company page** needs LinkedIn's **Community Management API** approval (the `w_organization_social` scope). Until then, post as yourself.

---

## Each week, by hand

1. Open the review email and the draft in Shopify.
2. Read the post, check every figure in "Figures and claims to check", and swap any odd image for a real photo.
3. Publish the post in Shopify.

---

## Costs (observed)

Per run, through OpenRouter: research about $0.41–0.48 (9–10 searches), writing about $0.23, LinkedIn post about $0.09, plus three images. OpenRouter adds a 5.5% fee when you buy credits; there is no markup on model prices.

---

## Troubleshooting

| Problem | Cause and fix |
|---|---|
| "Node 'X' hasn't been executed" / "Referenced node doesn't exist" | n8n added a number to a node's name (for example "Build Research Request2") when it was pasted while an old copy existed. The latest node files cope with numbered names. The lasting fix: delete unused copies and rename live nodes to their plain names. |
| `tool_choice` error from the writer | Claude Opus 5.5 rejects a forced tool call. The current nodes use `tool_choice: 'auto'`. |
| Article cut off (`finish_reason: length`) | Thinking shares the token limit with the post. Keep `WRITER_REASONING = 'low'` in Build Writing Request, or raise `maxTokens`. |
| "Research was not usable" | The research returned no source links or almost no notes. The row stays `Queued`; check the output of AI: Research the Topic. |
| Image error "expected string, received undefined" | The image node must call `https://openrouter.ai/api/v1/images` with a `prompt`, which the current nodes do. |
| Shopify upload or scope errors | Add the missing scope to the Shopify app, release a new version, and approve it on the store. |
| LinkedIn `unauthorized_scope_error` for `w_organization_social` | Organization Support is on in the credential. Turn it off and post as a person. |
| Sheet not updated | The email says so in red. Set the row to `Drafted` by hand, or the same topic is written again next week. |

---

## Testing note

All nodes were tested against stand-in services (fake Shopify, Google Sheets, OpenRouter and LinkedIn), including failure cases and canvases with numbered node names. They were also run on real AI replies from the live workflow. The LinkedIn publish node itself has only been checked against n8n's own settings, so watch the first live post.
