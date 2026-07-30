# LangSecret — System Architecture, Database Design & Core Algorithms

> This repository is a **showcase** of the system architecture, database schema (46 models), core algorithms, and technical decisions behind [LangSecret](https://langsecret.com) — an EdTech platform for learning English vocabulary through spaced repetition, adaptive reading, and gamification.
>
> The production source code is **private**. This repo contains only documentation and architectural descriptions — no source code, no credentials.

---

## Live Product

**https://langsecret.com** — English learning platform with vocabulary training, adaptive reading with parallel translation, and gamification.

| Word Cards | Reading with Translation | User Profile |
|:-:|:-:|:-:|
| ![Word Cards](screenshots/word-cards.png) | ![Reading](screenshots/reading.png) | ![Profile](screenshots/profile.png) |

---

## Two Core Modules

The product is built around **two primary learning modules**, unified by a personal vocabulary and spaced repetition system.

### 1. Word Cards

A vocabulary test engine with 1,000+ English words. The user translates each word, response time is measured.

**Scoring mechanics:**

| Action | Word Level Change | Points Change |
|--------|:-:|:-:|
| Correct answer | +1 | +1 |
| Incorrect answer | −2 (min 0) | −2 |

Word levels range from 0 (new) to 10+ (learned). Words at level 10+ are excluded from the basic test.

**Two selection modes:**
- **Smart** — weighted random sampling prioritizes weak words (see Smart Word Selection algorithm below)
- **Random** — uniform random from the full word pool

**Test size:** 20 cards for authenticated users, 30 for guests.

### 2. Reading with Parallel Translation

Adapted English texts organized by **CEFR level** (A1–C2) and category, with sentence-level parallel translation (en↔ru).

**Interactive features:**
- Click any word → popup with lemma, IPA transcription, translation, part of speech, and context sentence
- "Add to vocabulary" button directly from the popup
- Text statistics: total words, unique words, estimated reading time
- Progress tracking per text (started / completed)

**Both modules feed into the same personal vocabulary**, where words are reviewed using the SM-2 spaced repetition algorithm (5 review modes). LLM-powered grammatical analysis is available on demand in reading mode — users can click for a sentence breakdown and ask follow-up questions.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Backend** | Python 3.12, Flask 3.1, Flask Blueprints |
| **ORM** | SQLAlchemy 2.0, Flask-SQLAlchemy 3.1 |
| **Database** | MySQL 8 (via mysqlclient 2.2) |
| **Auth** | Flask-Bcrypt 1.0, Telegram OAuth, token-based email activation & password reset |
| **AI/LLM** | OpenAI-compatible API (DeepSeek via OpenRouter), Pydantic-validated responses, Fernet-encrypted API keys |
| **NLP** | BNC/CEFR-J frequency lists, wordfreq Zipf scores (offline content preparation only) |
| **Visualization (backend)** | Matplotlib 3.10 (server-side PNG charts, ring/progress SVGs) |
| **Visualization (frontend)** | Plotly.js (interactive charts on profile page — points, attempts, activity calendar, speed) |
| **Media** | Pillow 11.1 (image processing, thumbnails, watermarks, social share cards) |
| **TTS** | Qwen3-TTS 1.7B (standalone PySide6 desktop generator, not in web app) |
| **Games** | JavaScript (Rocket mini-game) |
| **Frontend** | Jinja2 templates, jQuery, Bootstrap, PWA (manifest.json, service-worker.js) |
| **Admin** | Flask-Admin 1.6 (darkly theme, IP-restricted), TinyMCE WYSIWYG |
| **Email** | Flask-Mail 0.9 (local Postfix), newsletter system with Telegram integration |
| **Deployment** | Gunicorn 20.1, Nginx reverse proxy |

---

## Core Algorithms

### 1. SM-2 Spaced Repetition

Implements the [SuperMemo-2 algorithm](https://supermemo.guru/wiki/SuperMemo_1.0_for_DOS_(1987)#Algorithm_SM-2) with extensions for vocabulary review.

**Input:** quality (0–5), current `easiness_factor`, current `interval`, `repetitions`

**On failure (quality < 3):**

| Quality | Action |
|---------|--------|
| 0–1 | Repeat today (`interval = 0`) |
| 2 | Repeat tomorrow (`interval = 1`) |

```
repetitions = 0
new_ef = max(1.3, current_ef - 0.2)
```

**On success (quality >= 3):**

```
new_ef = current_ef + 0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02)
new_ef = clamp(new_ef, 1.3, 2.5)

repetitions=0 → interval = 1 day
repetitions=1 → interval = 6 days
repetitions≥2 → interval = round(current_interval * new_ef)

If quality == 5: interval *= 1.3  (perfect answer bonus)
repetitions += 1
```

**Mastery criteria:** A word transitions to `status = mastered` when all three conditions are met:
1. `repetitions >= 6`
2. `easiness_factor >= 2.0`
3. `status == 'learning'`

On mastery: `next_review_date = today + 90 days`, user earns 3 bonus points.

**5 Review Modes:**
1. **Flashcard** — see the word, recall translation, self-rate quality
2. **Typing** — type the translation (with Levenshtein-based typo tolerance)
3. **Multiple Choice** — select from distractors generated from user's own vocabulary
4. **Listening** — hear TTS audio, type the word
5. **Matching** — pair English words with translations

Users can set preferred mode per word or use `auto`.

**Review History Tracking:** Every review records `easiness_factor_before/after` and `interval_before/after` for analytics.

---

### 2. Smart Word Selection

When generating a test set of 20 cards, the algorithm balances **new words** vs **repeats** based on the user's collection size:

| Collection Size | Repeats | New |
|:-:|:-:|:-:|
| < 100 | 0 | 20 |
| 100–399 | 4 | 16 |
| 400–699 | 8 | 12 |
| 700–999 | 12 | 8 |
| 1000–1499 | 16 | 4 |
| 1500+ | 18 | 2 |

**Repeat priority weights** (lower level = higher priority):

| Word Level | Weight |
|:-:|:-:|
| 1–2 | 5x |
| 3–4 | 3x |
| 5–9 | 2x |
| 10+ | 1x |

Selection uses weighted random sampling **without replacement**, implemented via `weighted_random_sample()`.

**Typing answer validation:** Exact match → quality 5. Levenshtein distance ≤ 1 (short words) or ≤ 2 (5+ chars) → quality 3 (typo tolerance). Otherwise → quality 0.

---

### 3. Content Pipeline for Reading Module

Reading texts are prepared **offline** before publication. No NLP processing occurs at runtime — all lemmas, POS tags, and translations are pre-generated by LLM and stored directly in the database.

**Content preparation (offline):**

1. **LLM generation:** ChatGPT/DeepSeek processes raw English text and produces a JSON with:
   - `sentences`: array of `{en, ru}` parallel sentence pairs
   - `words`: array of `{word, lemma, pos, transcription, translation_lemma, translation_context, sentence_index, word_index}`
2. **`TextProcessor` import:** The Flask app imports the ready-made JSON into the database — no NLP processing at runtime. Two formats supported: compact (by index) and legacy (by full sentence text)
3. **Storage:** Sentences stored in `ReadingText.content_pairs` (JSON), words in `ReadingTextWord` table with `word_lemma`, `pos_tag`, `transcription` (IPA), and sentence context for tooltip display

**CEFR level assignment (offline script):**

`bnc_vocab_loader.py` is an **offline utility** (not part of the running app) that assigns CEFR levels to words using a 3-tier priority system:
- Priority 1: Manual overrides (78 curated modern words)
- Priority 2: CEFR-J Wordlist v1.6
- Priority 3: `wordfreq` Zipf frequency — thresholds: ≥5.7 → A1, ≥5.1 → A2, ≥4.5 → B1, ≥3.8 → B2, below → C1

**AI-assisted analysis (runtime):**

LLM (DeepSeek V4 Flash via OpenRouter) performs on-demand sentence-level grammatical analysis when users click words in reading mode. Results cached in `SentenceAnalysis` with `cache_key` (SHA-256). Rate-limited to 50 requests/user/day via `UserAICounters`. Users can also ask follow-up questions stored in `AnalysisQuestion`.

---

### 4. Gamification System

**Level Thresholds** — exponential curve:

```
points_for_level(n) = int(1200 * 1.035^(n-1) - 1200)
```

| Level | Points Required |
|:-:|:-:|
| 1 | 0 |
| 2 | 42 |
| 5 | 165 |
| 10 | 413 |
| 20 | 1,279 |
| 50 | 4,518 |
| 100 | 7,735 |

**Active Day:** 5+ points earned in a day = active day (recorded in `UserActivity`).

**Streak Calculation:** Sort active dates descending. Must start from today or yesterday (otherwise streak = 0). Count consecutive dates backward.

**Daily Ring Progress (SVG):** `offset = circumference * (1 - min(daily_points, 5) / 5)`

**Achievement System:** Achievements are **dynamically generated** — they don't exist as a fixed set in the database, but are created on-the-fly when a user first reaches a threshold. This means the system scales automatically as users progress. Checked via `check_and_award_achievements()` after each test.

| Category | Thresholds | Example |
|----------|-----------|---------|
| Collected cards (level > 0) | 100, 200, 300… 20,000 (step 100) | "500 cards collected" |
| Learned words (level ≥ 5) | 50, 100, 150… 20,000 (step 50) | "200 words learned" |
| User level reached | 10, 20, 30… 300 (step 10) | "Reached level 50" |
| Correct answers | 100–10,000 (step 100), then 10,500–1,000,000 (step 500) | "1,000 correct answers" |
| Active days | 10, 20, 30… 5,000 (step 10) | "Active for 100 days" |
| Streak | 10, 20, 30… 5,000 (step 10) | "30-day streak" |
| Perfect runs (0 mistakes) | 10–5,000 (step 10), then 5,050–200,000 (step 50) | "50 perfect tests" |
| Daily points record | dynamic (beats previous personal best) | "New daily record: N points" |

---

## Social & Notifications

### Friends

- Send / accept / reject friend requests
- Friend list with pagination on profile
- Remove friends

### Congratulations

- Friends can congratulate on achievements or level-ups
- Duplicate protection: one congratulations per user per event
- Notification shows all congratulators

### Online Status

- Green dot on avatar if user was active in the last 30 minutes
- "Online now" and "Were online today" counters

### Notification System

**Types:** welcome, daily, achievement, level-up, friend request, friend accepted, congratulation, custom.

- Bell icon with unread counter in header
- Mark as read (individual or all)
- **300+ motivational messages** organized by time of day:
  - 80 morning messages
  - 88 afternoon messages
  - 61 evening messages
  - 70 night messages
  - 15 welcome messages
- Daily notification sent on first visit of the day, matched to time of day

---

## CMS / Articles

Full-featured blog system for publishing English learning articles.

**For users:**
- Categories and tags for navigation
- Full-text search across articles
- Nested comments with moderation (auto-approve for admins, pre-moderation for users)
- SEO: meta tags, Open Graph, `sitemap.xml`

**For admins (Flask-Admin):**
- TinyMCE WYSIWYG editor with image upload
- Workflow statuses: draft → scheduled → published → archived
- Scheduled publishing: articles with `status = scheduled` auto-publish when `published_at <= now()`
- Auto-generated slugs and preview thumbnails (3 sizes: small, medium, large)
- Content Planner for editorial scheduling
- Tag and category management, comment moderation

---

## Database Architecture

### High-Level ER Diagram

```mermaid
erDiagram
    User ||--o{ UserWordProgress : "vocabulary progress"
    User ||--o{ UserAttempts : "test attempts"
    User ||--o{ UserActivity : "daily activity"
    User ||--o{ UserLastActivity : "last seen"
    User ||--o{ UserAchievement : "achievements"
    User ||--o{ PointLog : "points history"
    User ||--o{ UserLogin : "login log"
    User ||--o{ Notification : "notifications"
    User ||--o{ Friendship : "friends"
    User ||--o{ UserReadingProgress : "reading progress"
    User ||--o{ UserReadingStats : "reading stats"
    User ||--o{ UserVocabulary : "personal vocabulary"
    User ||--o{ VocabularyReviewHistory : "review history"
    User ||--o{ GameSession : "game sessions"
    User ||--o{ Article : "authored articles"
    User ||--o{ ArticleComment : "comments"

    WordCard ||--o{ UserWordProgress : "progress"
    WordCard ||--o{ UserAttempts : "attempts"
    WordCard }o--o{ WordTopic : "categorized"

    ReadingCategory ||--o{ ReadingText : "contains"
    ReadingText ||--o{ ReadingTextWord : "words"
    ReadingText ||--o{ UserReadingProgress : "progress"
    ReadingText ||--o{ SentenceAnalysis : "AI analysis"

    UserVocabulary ||--o{ VocabularyReviewHistory : "review log"

    ArticleCategory ||--o{ Article : "contains"
    Article }o--o{ ArticleTag : "tagged"
    Article ||--o{ ArticleComment : "comments"
    Article ||--o{ ArticleImage : "images"

    Achievement ||--o{ UserAchievement : "earned by"

    LLMProvider ||--o{ SentenceAnalysis : "generates"
```

### Model Groups

| Group | Models | Key Fields | Notes |
|-------|--------|------------|-------|
| **User Core** | `User`, `ActivationToken`, `PasswordResetToken`, `UserLogin`, `UnsubscribeToken` | `email`, `username`, `telegram_id`, `oauth_provider`, `level`, `total_points` | OAuth (Telegram), bcrypt password hashing, SHA-256 token hashing, 24h/1h expiry |
| **Word Cards** | `WordCard`, `UserWordProgress`, `UserAttempts`, `WordTopic`, `WordTopicRelation`, `WordPageSEO` | Composite PK `(user_id, word_id)`, `level` (0–10+), `correct_attempts`, `incorrect_attempts` | 1,000+ words in base; smart selection with weighted sampling |
| **Reading / NLP** | `ReadingCategory`, `ReadingText`, `ReadingTextWord`, `ReadingSettings`, `ReadingCategoryLevelSEO`, `UserReadingProgress`, `UserReadingStats`, `SentenceAnalysis`, `LLMProvider`, `UserAICounters`, `AnalysisQuestion` | `content_pairs` (JSON), `word_lemma`, `pos_tag`, `cache_key` (SHA-256) | CEFR levels A1–C2; LLM-cached analysis; 50 req/user/day rate limit |
| **Vocabulary / SM-2** | `UserVocabulary`, `VocabularyReviewHistory` | `easiness_factor`, `interval_days`, `repetitions`, `status` (learning/mastered/archived), `preferred_review_mode` | 5 review modes; Levenshtein typo tolerance |
| **Social / Notifications** | `Friendship`, `Notification`, `UserCongratulations`, `Achievement`, `UserAchievement`, `UserLevelNotification`, `LevelNotifications` | `status` enum (pending/accepted/rejected), `is_system` flag | Friends graph; duplicate-safe congratulations; 300+ timed messages |
| **CMS** | `Article`, `ArticleCategory`, `ArticleTag`, `ArticleTagRelation`, `ArticleComment`, `ArticleImage`, `Page` | `slug`-based URLs, thumbnail sizes (small/medium/large), SEO fields, `status` (draft/scheduled/published/archived) | Moderation workflow; auto-publishing; Content Planner |
| **Gamification** | `GameSession`, `RocketGuessedWord`, `RocketMissedWord`, `PointLog` | `game_type`, `tier_reached`, `points_change`, `source_type` | Rocket mini-game with tier-based difficulty |
| **Newsletter** | `EmailCampaign`, `EmailCampaignRecipient` | `status` enum (draft/sending/paused/completed/cancelled), `filter_json` | Throttled sending with `send_delay_seconds` |

---

## API & Modular Structure

### Blueprint Registry

| Blueprint | URL Prefix | Key Endpoints | Purpose |
|-----------|-----------|---------------|---------|
| `routes` | `/` | `/`, `/learn-english/`, `/learn-english/test`, `/learn-english/results` | Home, word test, results |
| `auth_bp` | `/` | `/login`, `/register`, `/activate/<token>`, `/reset_password`, Telegram OAuth | Authentication & OAuth |
| `profile_bp` | `/` | `/@<username>`, `/update_avatar`, `/load_wordcards` | User profile & data loading |
| `stats_bp` | `/api/` | `/api/points_history`, `/api/activity_history`, `/api/speed_history` | Statistics API (AJAX) |
| `social_bp` | `/api/` | `/api/congratulate`, `/api/friend/*`, `/api/active-users` | Social interactions |
| `notifications_bp` | `/api/` | `/api/notifications`, `/api/notifications/mark-read/<id>` | Notification system |
| `reading_bp` | `/reading/` | `/reading/<cat>/<text>`, `/reading/vocabulary/`, `/api/review-session`, `/api/review-answer` | Reading module & vocabulary review |
| `english_words_bp` | `/english-words/` | `/english-words/`, `/english-words/<slug>`, `/english-words/letter/<l>/` | Word catalog with SEO pages |
| `articles_bp` | `/` | `/<cat_slug>/<article_slug>`, `/search`, `/sitemap.xml` | Article CMS |
| `games_bp` | `/games/` | `/games/rocket`, `/api/rocket/submit`, `/api/rocket/rating` | Rocket mini-game |
| `newsletter_bp` | `/` | `/unsubscribe/<token>`, `/profile/toggle_newsletter` | Newsletter management |
| `pages_bp` | `/` | `/contact/`, `/<slug>/` | Static pages |

---

## Architectural Patterns

### Context Processors

`inject_level_progress()` runs on **every request**, computing: current level, total_points, points_to_next_level, progress_percentage, active_streak, daily_points (for ring SVG offsets), collected_cards, texts_completed, and pending level-up notifications. This data is available in all Jinja2 templates.

### Dual Visualization Strategy

- **Backend (Matplotlib):** Server-side generation of PNG bar charts (correct/incorrect per session) and SVG ring/progress indicators (accuracy, time, speed, points). No client-side rendering required for results page.
- **Frontend (Plotly.js):** Interactive charts on the profile page — points over 30 days, attempts history, word level distribution, reading level progress, activity calendar heatmap, typing speed. Charts auto-resize on window change.

### Media Processing Pipeline

**Pillow** handles the full image lifecycle:
1. Upload validation (max 5MB, allowed extensions: jpg/jpeg/png/webp)
2. Optimization: downscale if >1920×1080
3. Thumbnail generation: 3 sizes (560×500, 800×440, 800×620)
4. Watermark overlay (configurable position/opacity)
5. Social share card: 1080×1080 gradient card with word image, logo, and text

### Encryption at Rest

API keys stored in the database are encrypted using **Fernet** (symmetric encryption from `cryptography` package). The `LLMProvider` model uses getter/setter methods (`get_api_key` / `set_api_key`) that transparently encrypt/decrypt via `crypto.py` — a singleton `Fernet` instance initialized from `FERNET_KEY` (environment variable).

### Blueprint Modularity

Each functional area is a separate Flask Blueprint with its own `routes.py`. Blueprints are registered in `app.py` with endpoint aliases for backward compatibility (e.g., `url_for('test')` instead of `url_for('routes.test')`).

### Scheduled Publishing

Articles and reading texts with `status = scheduled` auto-publish when `published_at <= now()`. The check runs at the top of the relevant route handlers — no background worker required.

### Cascade Delete

All relationships on `User` have `cascade='all, delete-orphan'`. Deleting a user removes all their data across 25+ tables.

### Database Migrations

Raw SQL files (34 migrations) — no Alembic. Each migration covers a feature area: reading system, vocabulary/SM-2, newsletter, SEO fields, OAuth, article thumbnails, game sessions, etc.

### PWA Support

`manifest.json` + `service-worker.js` in `/static/` enable install-as-app on mobile devices.

---

## Database Schema Summary

**46 models** across 8 logical groups, with:
- **Composite primary keys** for many-to-many relations (`UserWordProgress`, `UserLevelNotification`, `ArticleTagRelation`, `WordTopicRelation`)
- **Cascade delete-orphan** on all User relationships (deleting a user removes all their data)
- **Soft deletion** patterns via status enums (`learning`/`mastered`/`archived`, `pending`/`accepted`/`rejected`, `draft`/`scheduled`/`published`/`archived`)
- **Slug-based URLs** for SEO (`WordCard.slug`, `ReadingCategory.slug`, `ReadingText.slug`, `Article.slug`)
- **Per-entity SEO fields** (`seo_title`, `seo_description`, `seo_keywords`) on content models
- **Timezone-aware timestamps** using Moscow timezone (`pytz`) as the application standard

---

## Next Steps / Roadmap

**Functional:**
- Mobile app / enhanced PWA with offline mode and push notifications
- Extended OAuth (VK, Yandex, Google — model fields already support `oauth_provider`/`oauth_id`)
- Audio content: TTS infrastructure exists (`/static/audio/`), can be extended to full text narration
- C1/C2 reading levels — model supports them, content needed
- Grammar exercises as a new activity type on top of existing points/achievements
- Real-time multiplayer card games (WebSocket)

**Technical:**
- CSRF protection (Flask-WTF / CSRFProtect)
- Database migrations via Alembic / Flask-Migrate
- Redis caching for sessions, leaderboards, and statistics
- Asynchronous tasks via Celery (email sending, preview generation, scheduled publishing)
- API-first architecture (REST API + separate frontend)
- Automated test suite (pytest)
- Docker containerization
- Rate limiting (Flask-Limiter)
- WebSocket for real-time notifications
- Full-text search (Elasticsearch or MySQL FULLTEXT)
