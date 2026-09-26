# Glossa

**A vocabulary-learning web application: a personal dictionary, flashcards and adaptive tests, with AI used where it makes practice better.**

Formerly developed and published under the name _Smart Dictionary_.

[Live demo](https://umenrechnik.vercel.app) · [Published paper](https://jsee.uctm.edu/index.php/see/article/view/193/182) · [Deployment guide](docs/DEPLOYMENT.md)

---

## Publication and recognition

- **Published paper:** N. Nikolaev, G. Hristov, _"Smart Dictionary – Original Development of an Online Dictionary Powered by GPT-4o Mini"_, **Science, Engineering & Education**, Vol. 11, No. 1, 2026. University of Chemical Technology and Metallurgy, Sofia.
  [Full text (PDF)](https://jsee.uctm.edu/index.php/see/article/view/193/182) · DOI: [10.59957/see.v11.i1.2026.17](https://doi.org/10.59957/see.v11.i1.2026.17)
- Presented at the **4th National Conference for Pupils, Students and PhD Candidates "Information Technologies and Automation" (ITA 2025)**, organized by the Department of Industrial Automation at UCTM, Sofia.
- **Awarded by Schwarz IT.**

---

## Overview

Most vocabulary apps give every learner the same word lists and the same drills. Glossa starts from the learner's own words. A user builds a personal dictionary, typed in by hand or imported from their own notes and documents, and the application turns those words into flashcards and tests.

AI is not the product. It is used for the specific parts of practice that are hard to do well by hand: writing wrong answers that are actually plausible, building reading passages and gap-fill sentences around the learner's own words, turning an unstructured file into clean dictionary entries, and answering questions in context.

## Features

**Dictionary**

- Personal dictionary with folders, search and filters.
- Each entry holds the word, definition, part of speech and an example sentence.
- Pronunciation through the browser's speech synthesis.

**Import and export**

- Import from `.txt`, `.md`, `.csv`, `.json`, `.rtf`, `.doc`, `.docx`, `.html` and `.xml`.
- Unstructured text is converted into structured entries (word, definition, part of speech) by the model, then shown in a preview with duplicate detection before anything is written.
- Export to TXT, CSV or JSON.

**Practice**

- Flashcards over any selection of words.
- Five test formats, each at three difficulty levels:

| Format                | What the AI contributes                                         |
| --------------------- | --------------------------------------------------------------- |
| Multiple choice       | Wrong answers that are contextually plausible, not random words |
| Open answer           | Prompts written around the target word                          |
| Reading comprehension | A short passage using the learner's words, with questions       |
| Gap fill              | Sentences with the target word removed                          |
| Gap fill, verb form   | Sentences that require the correct inflected form               |

- Test history with per-session results.

**Assistant**

- A chat assistant that knows the learner's dictionary and current screen.
- It can act inside the application: start a test of a given type and length, open flashcards, switch views or add a word. The model appends a structured directive to its reply, which the client parses, validates and executes.

**General**

- Google sign-in, or guest mode that keeps everything in the browser.
- Interface in six languages: English, Bulgarian, German, French, Spanish and Chinese.
- Installable as a progressive web app, with light and dark themes and responsive layouts.

## Architecture

```
Browser (React SPA)
  ├── Supabase Auth (Google OAuth)
  ├── PostgREST ──────────────► PostgreSQL, row-level security on every table
  └── Edge Function ai-chat ──► quota check in Postgres ──► OpenRouter
                                                               ├── chat model
                                                               ├── test-generation model
                                                               ├── extraction model
                                                               └── fallback model
```

Guests never touch the backend: their dictionary lives in `localStorage` and tests are generated locally.

## Engineering highlights

- **AI output is validated, never trusted.** One validation module (`supabase/functions/_shared/aiValidation.ts`) is shared by the Edge Function and the client. Every response is parsed and checked against the expected shape; a response that fails is retried once on a fallback model before the user sees an error.
- **Task-based model routing.** Chat, test generation and extraction are routed to different models through OpenRouter. Each can be changed through an environment variable without a redeploy.
- **Quotas enforced in the database.** Daily AI limits are consumed inside a `SECURITY DEFINER` function that checks the caller's identity, so a client cannot reset or spend another user's quota. Oversized or malformed requests are rejected before a quota unit is spent.
- **Database security is tested.** A test suite applies every migration to an in-process Postgres (PGlite) and attempts the attacks the policies are meant to stop.
- **Bounded cost and latency.** Request bodies are size-limited, model calls have timeouts and token caps, and test generation keeps at most three requests in flight.
- **Privacy-aware logging.** The Edge Function logs one structured line per event with metadata only, never prompts, replies or dictionary content.
- **Accessibility.** ESLint's `jsx-a11y` rules, keyboard-operable dialogs with focus management, and automated axe checks in the end-to-end suite.

## Tech stack

| Layer    | Technologies                                                                                             |
| -------- | -------------------------------------------------------------------------------------------------------- |
| Frontend | React 18, TypeScript (strict), Vite, Tailwind CSS, TanStack Query, React Router                          |
| Backend  | Supabase: PostgreSQL with RLS, Auth, Deno Edge Functions                                                 |
| AI       | OpenRouter, with per-task model routing and fallback                                                     |
| Testing  | Vitest, Testing Library, MSW, PGlite, Playwright, axe-core (150+ tests)                                  |
| Tooling  | ESLint, Prettier, GitHub Actions (lint, format, typecheck, test, build, e2e), scheduled database backups |
| Hosting  | Vercel                                                                                                   |

## Running locally

**Prerequisites:** Node.js 20+, a Supabase project, an OpenRouter API key.

```bash
npm install
cp .env.example .env          # VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY, VITE_OAUTH_REDIRECT_URL
npm run dev                   # http://localhost:5173
```

Database and Edge Function:

```bash
supabase db push                                     # apply migrations
supabase secrets set OPENROUTER_API_KEY=...          # see supabase/functions/.env.example
supabase functions deploy ai-chat
```

Quality checks, the same as CI:

```bash
npm run lint
npm run typecheck
npm test
npm run e2e
```

Full setup, including Google OAuth configuration, is in [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md).

## Author

**Nikolay Nikolaev**
