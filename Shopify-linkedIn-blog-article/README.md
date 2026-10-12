# Blog Article Creation (n8n workflow)

An n8n workflow that writes one researched, SEO-ready blog post per week for a Shopify store. It takes the next topic from a Google Sheet, researches it on the web, writes the post, generates three photos, saves everything to Shopify as a **hidden draft**, adapts it into a LinkedIn post, and emails an editor a review checklist.

Nothing goes live on the blog without a person pressing Publish in Shopify.

## How it works

```
Schedule (Thu 9am) / Manual run
  → Config → Check Config
  → Read topic queue (Google Sheets) → Pick next "Queued" row
      └─ none queued → email "queue is empty" → stop
  → Shopify: find the blog + list existing posts
  → AI research (web search, sourced facts only)
  → AI write article (structured tool call) → Parse & check article
  → Build 3 image prompts → AI generate images → Collect images
      └─ upload to Shopify Files → wait until processed
  → Assemble draft → Shopify: create hidden article
  → Set SEO title & meta description → Mark sheet row "Drafted"
  → AI write LinkedIn post → Publish to LinkedIn (company page)
  → Email the editor a review summary
```

### Stage 1: Queue and research
- **Pick Next Topic** chooses the first row with status `Queued`, lowest `priority` first, then top to bottom. Rows with any other status (Hold, Drafted, Published…) are skipped.
- **Check Shopify Setup** confirms the blog handle exists and pulls up to 200 existing posts, which the writer links to and avoids repeating.
- **AI: Research the Topic** searches the web (up to `maxSearches`) and returns notes in four sections: Key facts, Answers to the searches, Brand figures, and Gaps. Every fact carries its source URL. If the research is empty or has no sources, the run stops and the row stays `Queued`.

### Stage 2: Write and illustrate
- **Build Writing Request** holds the voice, structure, accuracy and image-brief rules. The model can only state figures that appear in the research, and can only link to allowed internal pages and research sources.
- **Parse Article** cleans the HTML, removes disallowed links, styles tables for mobile, enforces the tag list and a unique URL handle, and runs automatic checks (banned claims, health wording, missing sections, word count). These checks are reported in the email and never block the draft.
- **Build Image Prompts** combines `imageStyle` from Config with the writer's scene for each slot: `featured`, `inline_1` and `inline_2`. If image generation fails, the post is still saved without images.

### Stage 3: Save, follow up, notify
- Images are uploaded to Shopify Files and polled until ready, for up to 12 checks.
- The post is created with `isPublished: false`. The SEO title and meta description are then set as metafields.
- The sheet row is updated to `Drafted`, with `draft_link` and `date_drafted`.
- The LinkedIn branch writes a 350–450 word post from the article. It links to the article if it is already live, and otherwise to the blog front page. If that branch fails, the blog draft is unaffected.
- **Email: Draft Ready** sends the draft link, automatic checks, every figure to verify with its source, the images, the SEO fields, the LinkedIn result, and the pages the research read.

## Setup

1. **Import** `Blog-Article-Creation.json` into n8n.
2. **Config node**: replace the sample values (see below).
3. **Credentials**: attach your own to these nodes.

   | Credential | Nodes |
   |---|---|
   | Google Sheets OAuth2 | Sheets: Read Queue, Sheets: Mark Row Drafted |
   | Shopify OAuth2 | All `Shopify:` nodes |
   | OpenRouter | AI: Research the Topic, AI: Write Article, AI: Generate Image, AI: Write LinkedIn Article |
   | LinkedIn OAuth2 | LinkedIn: Publish Post |
   | Gmail OAuth2 | Email: Draft Ready, Email: Queue Empty |

4. **Shopify app scopes**: `read_content`, `write_content`, `read_files`, `write_files`. If you add scopes to an existing app, release a new version and approve it on the store.
5. **Edit the prompts** in **Build Writing Request** and **Build LinkedIn Request**. They contain brand-specific selling points, voice and claims rules, so rewrite them for your own brand before the first run.
6. Click **Run manually (test)**, review the draft in Shopify, then activate the workflow.

## Config reference

| Field | Purpose | Sample value |
|---|---|---|
| `shop` | Store handle, the part before `.myshopify.com` | `example-store` |
| `shopifyApiVersion` | Shopify Admin API version | `2026-07` |
| `blogHandle` | Handle of the Shopify blog to post to | `articles` |
| `authorName` | Author shown on the post | `ABC Company` |
| `storeUrl` | Public store address | `https://www.example.com` |
| `notifyEmail` | Who receives review emails | `editor@example.com` |
| `sheetId` / `sheetTab` | The topic queue | sample ID / `Blog Queue` |
| `researchModel` / `writerModel` / `imageModel` | OpenRouter model IDs; each must contain a `/` | `anthropic/…`, `openai/…` |
| `maxSearches` | Web searches allowed in research | `10` |
| `wordCount` / `maxTokens` | Target post length and writer token limit | `1500` / `16000` |
| `imageQuality` / `imageSize` | Image options. Only the one the image model supports is sent | `medium` / `1920x1080` |
| `brandFacts` | The **only** things the AI may say about the brand, including a "never say" list | placeholder text |
| `linkTargets` | Internal pages the post may link to, one per line: `URL \| label` | example.com pages |
| `imageStyle` | Fixed photographic style added to every image prompt | coffee editorial style |

**Check Config** stops the run before any AI spend if a required field is missing or still a placeholder.

## Topic queue (Google Sheet)

Row 1 must hold these headings. They are case-insensitive, and columns can be in any order.

| Column | Required | Notes |
|---|---|---|
| `id` | yes | Your reference |
| `status` | yes | `Queued` gets picked; anything else is skipped |
| `working_title` | yes | |
| `primary_keyword` | yes | Main search phrase the post answers |
| `brief` | yes | What the post should cover |
| `priority` | no | Lower number goes first; blank is treated as 99 |
| `research_query` | no | Defaults to the keyword |
| `supporting_searches` | no | Related searches, separated by `\|` |
| `internal_links` | no | Extra allowed URLs, separated by `\|` |
| `names_other_brands` | no | `yes` lets the post cite other brands' figures |
| `health_sensitive` | no | `yes` applies stricter health-claim rules |
| `draft_link`, `date_drafted` | no | Filled in by the workflow |

## Safety rails

- Posts are always saved hidden. A person publishes them.
- Facts and figures must come from the research step, and each is listed with its source for the editor.
- Brand claims are limited to `brandFacts`, and the automatic checks flag banned words and health claims.
- Off-list internal links, links to AI tools and `javascript:` links are stripped.
- Image or LinkedIn failures never stop the blog draft from being saved.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| "Finish the Config node first" | A Config field still has a placeholder value |
| "missing these column headings" | Row 1 of the sheet was renamed |
| "research was not usable" | The research model returned no sources; check the research node's output |
| "article was cut off" | Raise `maxTokens` or lower `wordCount` |
| "Shopify would not accept the image uploads" | The app is missing `write_files`, or was not reinstalled after adding it |
| Same topic written twice | The sheet update failed; set the row to `Drafted` by hand (the email warns you) |
