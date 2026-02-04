# Field & Fen — Automation Roadmap

**Business:** Nature/wildlife photography → AI art → print-on-demand sales
**Owner:** Jonathan Wilson (J)
**Assistant:** Claude Ledbetter 🦝

---

## The Vision

Minimal human touch after creative work is done. Drop a finished art file into a folder → it gets upscaled, named, titled, and published to every sales channel and social platform automatically.

---

## Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│  YOU (Creative Work)                                                     │
│  Drone shots / Trail cam → Nano Banana render → Finished art            │
└─────────────────────┬───────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STAGE 1: INPUT                                                          │
│  Drop file into Google Drive "Incoming" folder                          │
└─────────────────────┬───────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STAGE 2: AI PROCESSING                                                  │
│  • Upscale to 300dpi (print-ready)                                      │
│  • Generate filename: species-description.jpg                           │
│  • Generate dramatic product title                                      │
│  • Generate product description + hashtags                              │
│  • Generate short video/reel from still                                 │
│  • Output to "Ready to Process" folder with metadata                    │
└─────────────────────┬───────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STAGE 3: HUMAN REVIEW (Last Touch)                                      │
│  • Review filename + title in "Ready to Process"                        │
│  • Edit if needed                                                        │
│  • Move to correct species folder → triggers automation                 │
└─────────────────────┬───────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STAGE 4: PUBLISH EVERYWHERE                                             │
│                                                                          │
│  E-commerce:                    Social:                                  │
│  • Shopify (via Printful)       • Instagram (post/story/reel)           │
│  • Etsy                         • TikTok (video)                        │
│  • eBay                         • Facebook (post/marketplace)           │
│  • Amazon                       • Pinterest (pin)                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Foundation (Week 1-2)

### 1.1 Google Drive Structure
- [ ] Create folder structure:
  ```
  Field & Fen/
  ├── 01-Incoming/           ← You drop files here
  ├── 02-Ready-to-Process/   ← AI outputs + metadata here
  ├── 03-Species/            ← Final sorted folders
  │   ├── Deer/
  │   ├── Squirrel/
  │   ├── Fox/
  │   ├── Birds/
  │   ├── Bear/
  │   └── [other species]/
  ├── 04-Published/          ← Archive of published work
  └── 05-Metadata/           ← Tracking spreadsheet
  ```
- [ ] Set up Google Drive API credentials
- [ ] Connect Zapier to Google Drive

### 1.2 Shopify + Printful
- [ ] Finish Shopify store setup
- [ ] Connect Printful to Shopify
- [ ] Configure product templates (sizes, pricing w/ 100% markup)
- [ ] Set up product categories matching species folders

### 1.3 Basic Webhook Pipeline
- [ ] Zapier: Google Drive trigger → Clawdbot webhook
- [ ] Test: File dropped → Claude notified

**Milestone:** Files can flow from Drive to Claude's awareness

---

## Phase 2: AI Processing (Week 2-3)

### 2.1 Canvas Ratio Standards
Reference: `docs/CANVAS-RATIOS.md`

**Supported ratios (create art in these):**
| Ratio | Use Case | Printful Sizes |
|-------|----------|----------------|
| 3:2 | Standard wildlife/landscape | 7 sizes (8×12 → 40×60) |
| 1:1 | Square, centered subjects | 15 sizes (6" → 37") |
| 4:3 | Alternative standard | 5 sizes (9×12 → 30×40) |
| 2:1 | Panoramic | 6 sizes (10×20 → 30×60) |
| 3:1 | Ultra-wide panoramic | 3 sizes (12×36 → 20×60) |

- [ ] Create ratio validation in pipeline (reject/flag unsupported ratios)
- [ ] Add ratio detection to incoming file processor

### 2.2 Image Upscaling
- [ ] Research/select upscaling solution:
  - Option A: Real-ESRGAN (free, self-hosted)
  - Option B: Topaz Gigapixel API (paid, high quality)
  - Option C: Replicate API (pay-per-use)
- [ ] Set up upscaling pipeline
- [ ] Test: 72dpi → 300dpi output

### 2.2 AI Metadata Generation
- [ ] Claude analyzes image and generates:
  - Filename suggestion (species-description)
  - Dramatic product title
  - Product description (100-200 words)
  - Hashtag set (20-30 tags)
  - Social caption variants (IG, TikTok, Pinterest)
- [ ] Output metadata to sidecar JSON or spreadsheet row
- [ ] Move processed files to "Ready to Process"

### 2.3 Video/Reel Generation
- [ ] Set up still → video conversion:
  - Ken Burns effect (slow zoom/pan)
  - 5-15 second duration
  - Music track options (royalty-free)
- [ ] Tools: ffmpeg, Canva API, or Lumen5

**Milestone:** Drop file → get back print-ready image + all metadata + video

---

## Phase 3: Human Review Workflow (Week 3)

### 3.1 Review Interface
- [ ] Metadata spreadsheet in "Ready to Process"
  - Thumbnail | Filename | Title | Description | Status
- [ ] Or: Simple approval via Telegram
  - Claude sends preview + suggested title
  - J replies "approved" or edits

### 3.2 Species Sorting Trigger
- [ ] Zapier: File moved to species folder → trigger publish workflow
- [ ] Track moved files to prevent duplicates

**Milestone:** Clean handoff from review to publish

---

## Phase 4: E-commerce Publishing (Week 4-5)

### 4.1 Shopify + Printful (Primary)
- [ ] Auto-create product via Shopify API
- [ ] Attach to Printful for fulfillment:
  - Detect image aspect ratio
  - Auto-select ALL matching canvas sizes from ratio family
  - Generate properly-sized print files for each variant (300 DPI)
  - Push to Printful API with all variants
- [ ] Apply species-based categorization
- [ ] Set pricing (Printful cost + 100% per variant)
- [ ] Publish as active listing with all size options

### 4.2 Etsy
- [ ] Connect Etsy API (or use Etsy integration tool)
- [ ] Map product data → Etsy listing format
- [ ] Handle Etsy-specific requirements (tags, attributes)

### 4.3 eBay
- [ ] Connect eBay API
- [ ] Fixed-price listings
- [ ] Map categories

### 4.4 Amazon
- [ ] Evaluate options:
  - Amazon Merch on Demand
  - Amazon Handmade
  - Standard seller account
- [ ] Set up chosen integration

### 4.5 Multi-channel Sync (Alternative)
- [ ] Consider: Sellbrite, Listing Mirror, or Printful's native sync
- [ ] May simplify multi-marketplace management

**Milestone:** One file → products live on 4 platforms

---

## Phase 5: Social Media Publishing (Week 5-6)

### 5.1 Instagram
- [ ] Connect Meta Business Suite API
- [ ] Auto-post feed image with caption + hashtags
- [ ] Auto-post to Stories
- [ ] Auto-post Reel (video version)
- [ ] Product tagging (link to shop)

### 5.2 TikTok
- [ ] Connect TikTok Business API
- [ ] Post video version with caption
- [ ] Link in bio to shop

### 5.3 Facebook
- [ ] Post to Page
- [ ] Facebook Marketplace listing (if applicable)
- [ ] Cross-post strategy with Instagram

### 5.4 Pinterest
- [ ] Connect Pinterest API
- [ ] Create Pin with product link
- [ ] Assign to relevant boards (by species/season)

### 5.5 Scheduling & Timing
- [ ] Stagger posts across platforms (avoid spam feel)
- [ ] Optimal posting times by platform
- [ ] Queue system for multiple pieces

**Milestone:** One file → social presence everywhere

---

## Phase 6: Monitoring & Optimization (Ongoing)

### 6.1 Sales Tracking
- [ ] Dashboard: Sales by platform, by species
- [ ] Printful order monitoring
- [ ] Profit margin tracking

### 6.2 Social Engagement
- [ ] Monitor comments/messages
- [ ] Claude alerts J to high-engagement posts
- [ ] Track follower growth

### 6.3 Facebook Monitoring (Original Request)
- [ ] Monitor FB page comments
- [ ] Monitor FB Marketplace inquiries
- [ ] Alert on potential buyer messages

### 6.4 Iteration
- [ ] A/B test titles/descriptions
- [ ] Track which species/styles sell best
- [ ] Adjust pricing strategy

---

## Tools & Services Summary

| Service | Purpose | Status |
|---------|---------|--------|
| Google Drive | File storage + workflow hub | Needs setup |
| Zapier | Automation orchestration | Connected ✓ |
| Printful | Print fulfillment | Needs setup |
| Shopify | Primary storefront | In progress |
| Etsy | Marketplace | Needs setup |
| eBay | Marketplace | Needs setup |
| Amazon | Marketplace | Needs research |
| Meta Business Suite | Instagram/Facebook | Needs setup |
| TikTok Business | TikTok posting | Needs setup |
| Pinterest API | Pinterest posting | Needs setup |
| Upscaler (TBD) | 300dpi conversion | Needs research |
| Video generator (TBD) | Reels/TikToks | Needs research |

---

## Credentials & API Keys Needed

- [ ] Google Drive API (Service Account)
- [ ] Shopify API key
- [ ] Printful API key
- [ ] Etsy API key
- [ ] eBay API key
- [ ] Amazon credentials (depends on chosen path)
- [ ] Meta Business Suite access token
- [ ] TikTok Business API key
- [ ] Pinterest API key

---

## Questions to Resolve

1. **Etsy/eBay/Amazon** — Do you have seller accounts set up already?
2. **Instagram/TikTok** — Business accounts ready?
3. **Music for reels** — Any preferred royalty-free source?
4. **Species list** — Final list of categories/folders?
5. **Pricing tiers** — Different markups for different sizes or flat 100%?
6. **Noah's role** — What parts is Noah handling vs. Claude automating?

---

## Next Steps

1. Set up Google Drive folder structure
2. Finish Shopify + Printful integration
3. Test the first manual end-to-end flow
4. Start automating piece by piece

---

*Last updated: 2026-02-04*
*Created by: Claude Ledbetter 🦝*
