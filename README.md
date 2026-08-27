# Wander

An anonymous social platform where people can share opinions, thoughts, hot takes, or
anything else on their mind — kept completely private, or posted publicly under a fresh,
untraceable name every time. No group to find first, no approval needed, no identity
attached. Public posts pass through AI moderation before anyone sees them; private posts
are never touched by AI at all.

---

## ✨ Features

- **Google-only authentication** — no password to manage, ID token verified server-side.
- **Private or public posting** — private posts are visible only to their author and
  never processed by AI in any way.
- **Fresh anonymous alias per public post** — generated locally, never reused or linkable
  back to the author across posts.
- **Category-organized public feed** — technology, music, fashion, study, relationships,
  and more, so posts land somewhere relevant instead of one undifferentiated stream.
- **AI content moderation before publishing** — text (toxicity/harassment, spam, and
  sensitive-content detection) and images (NSFW/violence) are checked automatically;
  flagged content stays visible only to its author, with the specific reason and an
  AI-suggested softer rewrite.
- **Hybrid semantic + typo-tolerant search** — find posts by meaning ("feeling stuck"
  matches a post that never uses that phrase) as well as by near-miss spelling/typos.
- **View history, saves, and reactions** — non-numeric reactions ("relate," "support"),
  a private saved list, and a self-expiring view history you can also search.
- **Related-post surfacing** — after publishing, surfaces 1-2 existing posts with a
  similar theme, never from the same anonymous session.
- **Format cleanup on request** — AI reformats spacing/paragraphs without touching wording.

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| Backend | ASP.NET Core Web API (.NET 9), C#, EF Core |
| Database | PostgreSQL (`pgvector` + `pg_trgm` extensions) |
| Cache | Redis (Upstash) |
| Auth | Google OAuth 2.0 + JWT |
| AI (text) | Google Gemini API — moderation, rewrite suggestions, embeddings |
| AI (image) | Google Cloud Vision SafeSearch Detection |
| Image storage | Cloudinary |
| Validation | FluentValidation |
| Logging | Serilog |
| Frontend | React (Vite), Tailwind CSS |
| API docs | Swagger / OpenAPI |
| Hosting | Render (backend), Vercel (frontend), Cloudflare (edge) |

---

## 🏗 Architecture

Clean Architecture, four layers, one direction of dependency:

```
Wander.Api  →  Wander.Application  →  Wander.Domain
Wander.Infrastructure  →  Wander.Application  →  Wander.Domain
```

`Domain` has zero dependencies. `Application` defines interfaces only
(`IPostRepository`, `ITextModerationStrategy`, `IImageModerationStrategy`,
`IEmbeddingService`, `IAliasFactory`, `ICacheService`) — it never references EF Core,
Gemini, or Redis directly. `Infrastructure` implements those interfaces. This means
swapping an AI provider, cache, or database is a change confined to one class and one
DI registration, not a change scattered across controllers.

```
src/
├── Wander.Domain/          # Entities, enums, value objects — no external dependencies
├── Wander.Application/     # Use cases, DTOs, interfaces
├── Wander.Infrastructure/  # EF Core, Gemini, SafeSearch, Redis, Cloudinary implementations
└── Wander.Api/             # Controllers, middleware, DI wiring
```

**Design patterns used, deliberately kept to a small set:**

| Pattern | Where |
|---|---|
| Repository | `Application/Interfaces` + `Infrastructure/Persistence/Repositories` |
| Strategy | `ITextModerationStrategy`, `IImageModerationStrategy` |  
| Factory | `IAliasFactory` → `WordListAliasFactory` |
| Dependency Injection | Throughout |

---

## 🛡 Content Moderation

Every public post and comment passes through automated checks before it's visible to
anyone else:

- **Text** — toxicity/harassment, spam, and sensitive content (e.g. self-harm risk),
  checked via Gemini. Flagged posts are never deleted — they stay visible only to their
  author, with the specific reason, the flagged excerpt, and an AI-suggested rewrite.
  Sensitive-content flags (e.g. self-harm risk) surface a distinct, gentle message with
  support resources rather than a generic "blocked" notice.
- **Images** — checked via Google Cloud Vision's SafeSearch Detection, a purpose-built
  classifier chosen over general-purpose LLM vision for more consistent NSFW/violence
  detection.

Private posts are never sent through moderation, or to any AI service, in any form.

---

## 🔐 Privacy & Anonymity Design

- Private posts are never sent to any AI service, in any form.
- Public posts get a freshly generated alias (adjective + noun + random number,
  generated locally — no external API), never reused, never linkable to a real user
  identity in any API response outside the owner's own authenticated request.
- Google ID tokens are verified server-side against Google's public keys — never trusted
  from a client-decoded payload alone.

---

## 🔍 Search

Two complementary techniques, merged and de-duplicated:

- **`pgvector`** — embeds post/query text via Gemini's embedding endpoint, compares by
  cosine similarity, catches different-wording-same-meaning matches.
- **`pg_trgm`** — trigram similarity, catches typos and spelling/diacritic variants that
  meaning-based search alone won't reliably catch.

The same hybrid search also runs over view history, saved posts, and reactions, so users
can re-find something they remember reading without recalling its exact title.

---

## 🗺 Roadmap / Future Features

Deliberately scoped out of the initial build, designed but not yet implemented:

- **Personalized feed ranking** — per-user "taste vector" (averaged embeddings of
  engaged-with posts) ranking the feed instead of a shared recency-sorted list, with
  cold-start fallback for new users.
- **Cross-device read-position sync** — persist last-viewed post/scroll position to the
  backend (Redis-buffered) so reading resumes across devices/sessions, beyond what
  client-side storage alone provides.
- **Full multi-language UI** — currently posts support any language via a per-post
  `language` field; a full UI language toggle is not yet built.
