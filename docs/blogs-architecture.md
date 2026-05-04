# CartAlchemy — Blog Module Architecture

The blog module powers content publishing across two Express/Handlebars apps — an admin app for content management and a public-facing website for readers. All blog content lives in a single MongoDB collection (`blogs`) differentiated by `documentType`, with images stored in S3 after processing through Sharp.

---

## 1. Module Structure

```mermaid
flowchart LR
    subgraph AdminApp["Admin App"]
        AR["routes/blog.js\n~1662 lines"]
        AT["views/blog/\nposts.hbs\npost.hbs\npost-edit.hbs\ncategories.hbs\ntags.hbs\nauthors.hbs\ncomments.hbs\nsettings.hbs"]
        AJ["public/javascripts/blog.js"]
    end

    subgraph WebsiteApp["Website App"]
        WR["routes/blog.js\n~810 lines"]
        WT["themes/mta/views/blog/\nposts.hbs\npost.hbs"]
        WJ["public/javascripts/blog.js"]
    end

    subgraph SharedStorage["Shared Storage"]
        DB[("MongoDB\nblogs collection")]
        S3["AWS S3\nuploads/blog/"]
    end

    AR --> DB
    AR --> S3
    WR --> DB
    AT --> AR
    WT --> WR
    AJ --> AR
    WJ --> WR
```

---

## 2. MongoDB Data Model

All document types reside in the single `blogs` collection — there are no separate collections per type. Relationships are embedded arrays of `{ _id, display fields }` rather than foreign-key joins; cascading deletes are handled in application logic via `$pull` operators.

```mermaid
classDiagram
    class Post {
        _id
        documentType: "post"
        title
        slug
        content (rich text HTML)
        contentOrigin (bool)
        page _id, title
        published (false | Date)
        thumbnail (S3 URL)
        thumbnailAlt
        header (S3 URL)
        headerAlt
        metaDescription
        shortDescription
        seoKeyword
        seoReport (HTML)
        showDate
        created
        updated
        moderator (ObjectId)
        preview (token)
    }

    class Category {
        _id
        documentType: "category"
        title
        slug (hierarchical)
        parent (parent slug)
        created
        moderator
    }

    class Tag {
        _id
        documentType: "tag"
        title (UPPERCASE)
        slug
        created
        moderator
    }

    class Author {
        _id
        documentType: "author"
        usersName
        profilePic (S3 URL)
        profile (bio)
        created
        moderator
    }

    class Comment {
        _id
        documentType: "comment"
        postId
        created
    }

    Post "1" --> "0..*" Category : categories[]
    Post "1" --> "0..*" Tag : tags[]
    Post "1" --> "0..*" Author : authors[]
    Post "1" --> "0..*" Comment : comments[]
    Post "1" --> "0..*" Post : crossSellPosts[]
    Category "1" --> "0..1" Category : parent slug
```

---

## 3. Post Lifecycle

Status is computed dynamically in aggregation pipelines — there is no separate status field. Publishing requires `thumbnail`, `metaDescription`, and `shortDescription` to be present.

```mermaid
stateDiagram-v2
    [*] --> Draft : Create post

    Draft : Draft\npublished = false
    Scheduled : Scheduled\npublished = future Date
    Published : Published\npublished = past Date

    Draft --> Scheduled : Set future publish date\n(requires thumbnail,\nmetaDescription,\nshortDescription)
    Draft --> Published : Set current date\n(requires thumbnail,\nmetaDescription,\nshortDescription)
    Scheduled --> Published : Scheduled date passes\n(time-based transition)
    Scheduled --> Draft : Clear publish date
    Published --> Draft : Clear publish date
    Scheduled --> Scheduled : Update scheduled date
    Published --> Published : Update publish date\n(keep in past)
```

---

## 4. Content Creation Flow

The admin form submission triggers sequential validation, optional image upload, and MongoDB persistence before the post becomes visible on the public website.

```mermaid
flowchart TD
    A([Admin submits post form]) --> B{Validation}
    B -- Invalid --> C[Return errors to form]
    B -- Valid --> D{Images attached?}

    D -- Yes --> E[multer parses multipart]
    E --> F[sharp resize & WebP convert]
    F --> G[Upload to S3]
    G --> H[Store S3 URL in document]
    H --> I

    D -- No --> I{New post?}

    I -- New --> J[MongoDB insert\ndocumentType: post\npublished: false]
    I -- Existing --> K[MongoDB update]

    J --> L{Publish requested?}
    K --> L

    L -- No --> M([Saved as Draft])
    L -- Yes, future date --> N([Saved as Scheduled])
    L -- Yes, current date --> O([Saved as Published])

    O --> P[Public website\nGET /blog/posts/:slug\nserves post]
```

---

## 5. Image Upload Pipeline

All three image types follow the same multer → sharp → S3 pipeline; they differ only in resize dimensions and S3 path. The resulting S3 URL is written back to the corresponding field on the post document.

```mermaid
flowchart LR
    subgraph Ingest["Ingest"]
        U([File upload\nPOST /blog/upload-image\n/:id/:type])
        M["multer\nmultipart parse"]
        U --> M
    end

    subgraph Processing["Sharp Processing"]
        direction TB
        H["Header\n1600×400px\nWebP q100"]
        T["Thumbnail\n1200×800px\nWebP q100"]
        C["Content image\nOriginal dims\nWebP convert"]
    end

    subgraph Storage["S3 Storage"]
        SH["uploads/blog/{postId}\n/header/{filename}"]
        ST["uploads/blog/{postId}\n/thumbnail/{filename}"]
        SC["uploads/blog/{postId}\n/content/{filename}"]
    end

    subgraph Persist["MongoDB"]
        FH["post.header = S3 URL"]
        FT["post.thumbnail = S3 URL"]
        FC["inline img src = S3 URL"]
    end

    M -- type=header --> H
    M -- type=thumbnail --> T
    M -- type=content --> C

    H --> SH --> FH
    T --> ST --> FT
    C --> SC --> FC
```

---

## 6. Content Authoring Modes

The post editor (built with Vue.js) offers two content modes, toggled via a `contentOrigin` checkbox. The choice determines what is stored in MongoDB and how the public website renders the post.

- **CKEditor 5 mode** (`contentOrigin: false`, default) — admin writes directly in the WYSIWYG rich text editor. Content is stored as an HTML string in `post.content`. Inline images are uploaded to S3 via a custom CKEditor upload adapter that calls the same Sharp pipeline as header/thumbnail images.
- **Page Builder mode** (`contentOrigin: true`) — admin selects an existing page built in the page builder. `post.page` stores a reference `{ _id, title }` to that page; `post.content` is set to `null`. The public website renders the page builder output instead.

```mermaid
flowchart LR
    subgraph Editor["Post Editor (Vue.js)"]
        TOGGLE["contentOrigin toggle\n(checkbox)"]
    end

    TOGGLE -->|"false — default\nwrite in editor"| CK["CKEditor 5\nWYSIWYG rich text"]
    TOGGLE -->|"true\nuse existing page"| PB["Page Builder\npage selector"]

    subgraph CKMode["CKEditor mode"]
        CK --> CK_IMG["Inline image upload\nPOST /blog/upload-image/{id}/content\nSharp → S3 → URL embedded in HTML"]
        CK --> CK_STORE["post.content = HTML string\npost.page = null"]
    end

    subgraph PBMode["Page Builder mode"]
        PB --> PB_STORE["post.page = { _id, title }\npost.content = null"]
    end

    CK_STORE -->|"public website"| RENDER_CK["Renders post.content\ndirectly as HTML"]
    PB_STORE -->|"public website"| RENDER_PB["Renders page builder\npage output"]
```

---

## 7. SEO Analysis Engine

I built a custom SEO and readability analyser (`admin/lib/seo.js`) triggered from the post editor via `POST /blog/posts/checkSeo`. It runs two independent analysis tracks — 14 SEO checks and 7 readability checks — each producing a weighted score out of 10. The overall score is the average of both tracks. The result is rendered as an HTML report with colour-coded indicators and stored on the post document in `seoReport`.

Each check returns one of four statuses, which map to score weights:

| Status | Weight | Indicator |
|--------|--------|-----------|
| `success` | 1.0 | Green dot |
| `warning` | 0.5 | Amber dot |
| `danger` | 0.0 | Red dot |
| `invalid` | 0.0 | Grey dot |

### Analysis Tracks

```mermaid
flowchart TD
    INPUT["Post content + focus keyword\n(title · content HTML · metaDescription · slug)"]

    INPUT --> SEOTRACK
    INPUT --> READTRACK

    subgraph SEOTRACK["SEO Track — 14 checks"]
        S1["Keyword in title"]
        S2["Title length 35–65 chars"]
        S3["Keyword in meta description"]
        S4["Meta description length 120–160 chars"]
        S5["Keyword density in content > 2%"]
        S6["Keyword in first paragraph"]
        S7["Keyword in a subheading (H1–H6)"]
        S8["Keyword in image alt text"]
        S9["Keyphrase distribution 0.5–2.5%"]
        S10["Keyphrase length ≤ 4 words"]
        S11["Content length ≥ 300 words"]
        S12["SEO title width 35–65 chars"]
        S13["Outbound links ≥ 1"]
        S14["Internal links ≥ 2"]
    end

    subgraph READTRACK["Readability Track — 7 checks"]
        R1["Flesch-Kincaid reading level\n(syllable library — grade 5 to Professional)"]
        R2["Sentence length\n< 30% of sentences > 20 words"]
        R3["Transition words\n≥ 3 (however, moreover, therefore…)"]
        R4["Consecutive sentences\nno identical adjacent sentences"]
        R5["Passive voice\n< 2 occurrences"]
        R6["Subheading distribution\n≥ 3 subheadings"]
        R7["Paragraph length\nno paragraph > 150 words"]
    end

    SEOTRACK --> SEOSCORE["SEO score\n(weighted sum / 14) × 10"]
    READTRACK --> READSCORE["Readability score\n(weighted sum / 7) × 10"]

    SEOSCORE --> OVERALL["Overall score\naverage of both tracks\n0–10"]
    READSCORE --> OVERALL

    OVERALL --> STATUS{"Score threshold"}
    STATUS -->|"≥ 7"| GREEN["success — green"]
    STATUS -->|"≥ 4"| AMBER["warning — amber"]
    STATUS -->|"< 4"| RED["danger — red"]

    GREEN & AMBER & RED --> REPORT["generateSeoReport()\nHTML with coloured dot per check\nstored in post.seoReport"]
```

### Flesch-Kincaid Reading Level

The readability track uses the Flesch-Kincaid reading ease formula, implemented with the `syllable` library for syllable counting:

```
score = 206.835 − 1.015 × (words / sentences) − 84.6 × (syllables / words)
```

| Score range | Grade | Status |
|-------------|-------|--------|
| 90–100 | 5th grade | success |
| 80–90 | 6th grade | success |
| 70–80 | 7th grade | success |
| 60–70 | 8th–9th grade | success |
| 50–60 | 10th–12th grade | warning |
| 30–50 | College | warning |
| 10–30 | College graduate | danger |
| 0–10 | Professional | danger |
