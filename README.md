# EduAI

An AI-powered educational content generation platform for teachers and educators. EduAI automates the creation of multiple-choice questions, lesson plans, and doubt resolutions using Google Gemini 1.5 Pro, with support for eight Indian regional languages.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [How Data Flows End-to-End](#2-how-data-flows-end-to-end)
3. [Tech Stack & Why](#3-tech-stack--why)
4. [Project Structure](#4-project-structure)
5. [Key Modules Explained](#5-key-modules-explained)
6. [Database & Storage](#6-database--storage)
7. [APIs & Integrations](#7-apis--integrations)
8. [AI / LLM Integration](#8-ai--llm-integration)
9. [Authentication](#9-authentication)
10. [Key Design Decisions](#10-key-design-decisions)
11. [Environment Variables](#11-environment-variables)
12. [Error Handling & Logging](#12-error-handling--logging)
13. [Quick Setup](#13-quick-setup)

---

## 1. Architecture Overview

EduAI is a **two-tier monolith** with a clear client/server split:

```
┌─────────────────────────────────┐     HTTP/REST      ┌──────────────────────────────────┐
│         CLIENT (Next.js)        │ ←────────────────→ │        SERVER (Express.js)        │
│                                 │                     │                                  │
│  App Router + React 18          │                     │  Route handlers → Utils → LLM    │
│  Tailwind + shadcn/ui           │                     │  Mongoose ODM → MongoDB Atlas    │
│  Firebase Auth (Google OAuth)   │                     │  Firebase Storage (images)       │
│  Axios for API calls            │                     │  Tesseract.js (OCR)              │
└─────────────────────────────────┘                     └──────────────────────────────────┘
         │                                                          │
         │ Firebase SDK                                  Gemini API │ (via LangChain)
         ↓                                                          ↓
  ┌─────────────┐                                        ┌──────────────────┐
  │  Firebase   │                                        │  Google Gemini   │
  │  (Auth +    │                                        │   1.5 Pro        │
  │  Storage)   │                                        └──────────────────┘
  └─────────────┘
```

**Pattern:** The backend follows a **loose MVC** style — routes act as thin controllers, and all business logic (LLM calls, scoring, OCR, file upload) lives in a dedicated `utils/` layer. There is no service layer or repository pattern; Mongoose models are used directly in utilities.

The frontend is a **page-component model** using Next.js App Router. Global state is minimal (just the authenticated user) and managed via React Context. All other state is local to the page component.

Both client and server are deployed on **Vercel** as separate projects.

---

## 2. How Data Flows End-to-End

### MCQ Generation Flow

```
User fills subject/topic/difficulty form (dashboard/page.tsx)
        ↓
Axios POST /test/generate  {userId, subjects, topics, difficulty, language}
        ↓
server/routes/test.js  →  generateQuestion(difficulty, subject, topic, language)
        ↓
server/utils/generateQuestion.js  →  LangChain ChatGoogleGenerativeAI.invoke(prompt)
        ↓
Gemini 1.5 Pro returns raw text (possibly wrapped in ```json ... ```)
        ↓
Sanitization: .trim() → strip markdown fences → JSON.parse()
        ↓
new Test({ userId, questions }).save()  →  MongoDB
        ↓
Response: full Test document  →  Axios  →  React state
        ↓
Dashboard renders MCQ modal with question cards
```

---

### Test Submission & Analysis Flow

```
User selects answers and clicks Submit
        ↓
Axios POST /test/submit  {testId, answers: [0,2,1,3,...]}
        ↓
server/routes/test.js  →  analyseTest(testId, answers)
        ↓
server/utils/analyseTest.js:
  1. Test.findById(testId)
  2. Iterate questions: compare answers[i] vs questions[i].answer
  3. Build topicsPerformance Map<topic, {correct, total}>
  4. Compute score / total
  5. Call analyseTestusingAi(test)
        ↓
server/utils/analyseTestusingAi.js:
  Serialize topicsPerformance → Gemini prompt (JEE expert persona)
  → JSON output: {Performance_Analysis, Strengths, Weaknesses, Targeted_areas_for_Improvement}
  → Sanitize → JSON.parse → test.summeryByAi = result
        ↓
test.save()  →  MongoDB (updated with answers, score, topicsPerformance, summeryByAi)
        ↓
Response: enriched Test document  →  React state
        ↓
Dashboard conditionally renders <Analysis /> component
  - Recharts circular progress (score %)
  - Per-topic progress bars (from topicsPerformance Map)
  - AI insight cards (4 sections)
  - Color-coded question-by-question review
```

---

### Doubt Solver Flow

```
User uploads image (dashboard/page.tsx)
        ↓
FormData with image file  →  Axios POST /doubt/create  (multipart/form-data)
        ↓
server/routes/doubt.js
        ↓
  ┌─── extractTextFromImage(req.files.image.data)
  │       Tesseract.js createWorker("eng")
  │       → imageText: string
  │
  ├─── solveDoubt(imageText)
  │       Gemini 1.5 Pro → solution string
  │
  └─── handleFileUpload(req.files.image.data)
          Firebase Storage: doubt/{Date.now()}
          → imageUrl: string (download URL)
        ↓
new Doubt({ userId, imageUrl, imageText, answer }).save()  →  MongoDB
        ↓
Response: Doubt document  →  solution displayed in UI
```

All three side-effects (OCR, LLM call, Firebase upload) in the doubt solver run **sequentially**, meaning total latency is additive: OCR time + Gemini time + Firebase upload time.

---

### Lesson Plan Generation Flow

```
User inputs subject, topic, grade  →  Axios POST /lessonPlan/create
        ↓
server/routes/lessonPlan.js  →  generateLessonPlan(subjects, topics, grade)
        ↓
server/utils/generateLessonPlan.js:
  Detailed schema-aware prompt sent to Gemini 1.5 Pro
  → Raw JSON → Sanitize → JSON.parse
  → Field-level fallback defaults applied for any missing keys
        ↓
Response: lesson plan object (NOT saved to MongoDB, returned directly)
        ↓
<LessonPlan /> modal renders structured plan sections
```

> Lesson plans are **not persisted** to MongoDB. Each generation is ephemeral — the result lives only in React state until the modal is closed.

---

## 3. Tech Stack & Why

### Frontend

| Technology | Version | Why |
|---|---|---|
| **Next.js** | 15.2.0 | App Router enables file-system routing with layout nesting; Turbopack for fast local dev |
| **React** | 18.3.1 | Component model; concurrent features for smoother UI updates |
| **TypeScript** | 5 | Type safety across components, context, and API response shapes |
| **Tailwind CSS** | 3.4.1 | Utility-first styling with custom color tokens and animation config |
| **shadcn/ui** | — | Radix UI primitives (accessible, unstyled) + Tailwind + Lucide icons, composable without a heavy runtime |
| **Axios** | 1.8.3 | Promise-based HTTP with interceptor support; cleaner than native `fetch` for multipart uploads |
| **React Hook Form** | 7.54.2 | Uncontrolled form state with Zod integration; avoids re-renders on every keystroke |
| **Zod** | 3.24.2 | Runtime schema validation for form inputs |
| **Recharts** | 2.15.1 | Composable React chart library for the analysis dashboard |
| **Firebase SDK** | 11.5.0 | Google OAuth via `signInWithPopup`; no need to run an OAuth server |

### Backend

| Technology | Version | Why |
|---|---|---|
| **Node.js + Express** | 4.21.2 | Lightweight, non-blocking server; fits an LLM-heavy workload where latency is network-bound |
| **Mongoose** | 8.12.1 | Schema enforcement and document validation on top of MongoDB's schemaless nature |
| **LangChain** | 0.3.42 | Abstracts LLM provider details; makes swapping models a one-line change in `llmProvider.js` |
| **@langchain/google-genai** | 0.1.10 | LangChain adapter for Gemini; handles retries and response formatting |
| **bcryptjs** | 3.0.2 | Password hashing; pure JS (no native bindings), portable across Vercel's serverless runtime |
| **express-fileupload** | 1.5.1 | Parses `multipart/form-data` into `req.files` with buffer access; avoids temporary disk writes |
| **Tesseract.js** | 6.0.0 | Pure JS OCR (WASM); no binary dependencies, works in Vercel's serverless environment |
| **Firebase Admin** | 11.4.0 | Server-side Firebase Storage access for image uploads |
| **cors** | 2.8.5 | Enables cross-origin requests from the separately-deployed frontend |

### External Services

| Service | Purpose |
|---|---|
| **Google Gemini 1.5 Pro** | All LLM generation tasks (MCQs, analysis, lesson plans, doubt solving) |
| **Firebase Auth** | Google OAuth sign-in (browser-side popup flow) |
| **Firebase Storage** | Persistent storage for doubt images |
| **MongoDB Atlas** | Primary application database |
| **@google-cloud/vision** | Listed as a dependency; Tesseract.js is used in practice for OCR |

---

## 4. Project Structure

```
EduAI-main/
│
├── client/                         # Next.js 15 frontend
│   ├── next.config.ts              # Image domains: Unsplash, Firebase, Google avatars
│   ├── tailwind.config.ts          # Custom colors + accordion/spin animations
│   ├── components.json             # shadcn/ui config (component registry)
│   └── src/
│       ├── app/                    # Next.js App Router
│       │   ├── layout.tsx          # Root layout: UserContextProvider wrapper + Inter font
│       │   ├── page.tsx            # Landing page (hero, features, testimonials, FAQ, CTA)
│       │   ├── dashboard/
│       │   │   └── page.tsx        # Main app: all 4 AI tools in one page (~845 lines)
│       │   ├── sign-in/page.tsx
│       │   ├── sign-up/page.tsx
│       │   ├── features/page.tsx
│       │   ├── about-us/page.tsx
│       │   ├── contact/page.tsx
│       │   └── ClientBody.tsx      # Prevents hydration mismatch on body classes
│       │
│       ├── components/
│       │   ├── layouts/
│       │   │   ├── header.tsx      # Nav + user avatar from context
│       │   │   └── footer.tsx      # Links + social icons
│       │   ├── home/               # Landing page sections
│       │   │   ├── hero-quiz-creator.tsx
│       │   │   ├── feature-section.tsx
│       │   │   ├── testimonial-section.tsx
│       │   │   ├── faq-section.tsx
│       │   │   └── call-to-action.tsx
│       │   └── ui/                 # shadcn/ui primitives (accordion, button, card, dialog…)
│       │
│       ├── context/
│       │   ├── UserContext.ts          # TypeScript interface + createContext
│       │   └── UserContextProvider.tsx # useState wrapper, exported as provider
│       │
│       └── lib/
│           ├── routeProvider.ts    # Centralized API route string constants
│           ├── Firebase.ts         # Firebase app init + GoogleAuthProvider
│           ├── utils.ts            # cn() helper (clsx + tailwind-merge)
│           ├── Analysis.tsx        # Full test result/analysis component (~393 lines)
│           ├── LessonPlan.tsx      # Lesson plan modal renderer (~171 lines)
│           └── loading.tsx         # LoadingSpinner + FullPageLoader
│
└── server/                         # Express.js backend
    ├── vercel.json                 # maxDuration: 60s (LLM latency buffer)
    ├── api/
    │   └── server.js              # Entry point: Express init, CORS, routes, DB connect
    ├── routes/
    │   ├── auth.js                # POST /signup, POST /signin
    │   ├── test.js                # POST /generate, /generate/content, /submit
    │   ├── doubt.js               # POST /create
    │   └── lessonPlan.js          # POST /create
    ├── models/
    │   ├── userModel.js           # Mongoose User schema
    │   ├── testModel.js           # Mongoose Test schema (questions, answers, analysis)
    │   ├── doubtModel.js          # Mongoose Doubt schema
    │   └── lessonPlanModel.js     # Mongoose LessonPlan schema (~65 lines)
    └── utils/
        ├── llmProvider.js              # Singleton Gemini client (shared across all utils)
        ├── dbConnect.js                # Mongoose.connect() wrapper
        ├── Firebase.js                 # Firebase Admin app init
        ├── generateQuestion.js         # LLM: MCQs from topic
        ├── generateQuestionwithContext.js  # LLM: MCQs from content text
        ├── analyseTest.js              # Score computation + topicsPerformance map
        ├── analyseTestusingAi.js       # LLM: performance insights from topic map
        ├── generateLessonPlan.js       # LLM: structured lesson plan (~90 lines)
        ├── solveDoubt.js               # LLM: doubt resolution from text
        ├── extractTextfromImage.js     # Tesseract.js OCR pipeline
        └── uploadImage.js              # Firebase Storage upload → download URL
```

---

## 5. Key Modules Explained

### `server/api/server.js` — Entry Point

The Express app is configured here in this order:

1. Register middleware (`cors()`, `fileUpload()`)
2. Mount route modules at path prefixes (`/test`, `/doubt`, `/auth`, `/lessonPlan`)
3. Call `dbConnect()` — Mongoose connects to MongoDB Atlas
4. Start listening on `process.env.PORT`

Vercel invokes this as a serverless function, so `dbConnect()` runs on every cold start.

---

### `server/utils/llmProvider.js` — LLM Singleton

```js
const llm = new ChatGoogleGenerativeAI({
  model: "gemini-1.5-pro",
  temperature: 0,
  maxRetries: 2,
});
export default llm;
```

Every generation utility imports the same instance. `temperature: 0` makes responses deterministic — critical for JSON output consistency. `maxRetries: 2` means LangChain silently retries failed Gemini API calls twice before throwing.

---

### `server/utils/analyseTest.js` — Score Engine

The most algorithmically substantive backend utility. Execution order:

1. Fetch Test document by ID
2. Zip `test.questions` with `answers` array by index position
3. Build a `topicsPerformance` object dynamically (topic names come from the LLM-generated question data):
   ```js
   if (!topicsPerformance[topic]) topicsPerformance[topic] = { correct: 0, total: 0 };
   topicsPerformance[topic].total++;
   if (userAnswer === correctAnswer) topicsPerformance[topic].correct++;
   ```
4. Assign `test.score`, `test.total`, `test.answers`, `test.topicsPerformance`
5. Call `analyseTestusingAi(test)` — adds `summeryByAi` to the test object
6. `test.save()` — persists everything to MongoDB

---

### `client/src/lib/Analysis.tsx` — Result Dashboard (~393 lines)

Receives the full Test document as a prop and renders:

- **Circular score indicator:** A `div` with inline `conic-gradient` CSS computed from `(score/total)*100`
- **Topic performance bars:** Iterates `Object.entries(topicsPerformance)`, renders a progress bar with dynamic `width` style per topic
- **AI insight cards:** Destructures `summeryByAi` and renders each of the 4 fields in a styled card
- **Question review:** Maps over `questions`, compares with `answers[i]`, applies conditional Tailwind classes (`bg-green-50` for correct, `bg-red-50` for wrong, `bg-yellow-50` for unattempted), and shows the solution text

---

### `client/src/app/dashboard/page.tsx` — The Hub (~845 lines)

All four AI tools live in this single page file. Structure:

```
State:
  activeTab       — which tool panel is visible
  mcqMode         — "simple" | "advanced"
  [form inputs]   — controlled inputs per tool
  currentTest     — Test doc returned from /test/generate
  analysisData    — Test doc returned from /test/submit
  showAnalysis    — boolean that swaps the dashboard for <Analysis />
  [modal booleans]— showTest, showLessonPlan

Render:
  if (showAnalysis) → <Analysis data={analysisData} />
  else:
    <Sidebar> tool selector tabs
    <MainContent>
      switch(activeTab):
        "mcq"        → Simple/Advanced tab form + MCQ modal
        "lessonPlan" → subject/topic/grade form + <LessonPlan> modal
        "doubt"      → image upload + solution display
        "indic"      → language selector + MCQ form
```

---

## 6. Database & Storage

### MongoDB Atlas — Primary Database

Four collections managed via Mongoose:

#### `users`
```
_id           ObjectId  (PK)
name          String    required
email         String    required, unique index
password      String    bcrypt hash; omitted for OAuth users
profilePhoto  String    URL
authProvider  String    "GOOGLE" | null
class         String
tests         [ObjectId]  references tests collection
aiMsg         String
```

#### `tests`
```
_id       ObjectId  (PK)
userId    ObjectId  required, references users
questions [{
  topic:    String
  question: String
  options:  [String]  (always 4 elements)
  answer:   Number    (0–3, index of correct option)
  solution: String
}]
answers           [Number]   user's submitted option indices
score             Number
total             Number
topicsPerformance Map<String, { correct: Number, total: Number }>
summeryByAi {
  Performance_Analysis:           String
  Strengths:                      String
  Weaknesses:                     String
  Targeted_areas_for_Improvement: String
}
```

#### `doubts`
```
_id        ObjectId  (PK)
userId     ObjectId  required, references users
imageUrl   String    Firebase Storage download URL
imageText  String    Tesseract OCR output
answer     String    Gemini-generated solution
```

#### `lessonplans`
```
_id           ObjectId  (PK)
title         String
gradeLevel    String
subject       String
timeAllotment String
objective       { overall: String, specific: [String] }
prerequisites   [String]
introduction    { hook: String, overview: String }
contentOutline  [{ day: Number, topic: String, details: String }]
activities      [{ day: Number, activity: String, materials: String }]
assessment      { formative: [String], summative: [String] }
```

> The LessonPlan Mongoose schema is fully defined but the route currently returns plans without calling `.save()`. The schema is ready for future persistence.

**Relationships:**
- `User.tests[]` → `Test._id` (one-to-many)
- `Test.userId` → `User._id` (back-reference)
- `Doubt.userId` → `User._id`

---

### Firebase Cloud Storage — Image Storage

Used exclusively for doubt solver images. Server uses the Firebase **Admin SDK** to write buffers directly, bypassing client-side security rules. Upload path: `doubt/{Date.now()}`. The returned download URL is stored in the Doubt document alongside the OCR text and AI solution.

---

## 7. APIs & Integrations

### Internal REST API

All endpoints use `POST`. The client reads the server base URL from `NEXT_PUBLIC_Backend_Route`.

| Endpoint | File | Request Body | Response |
|---|---|---|---|
| `POST /auth/signup` | `routes/auth.js` | `{name, email, password?, profilePhoto?, authProvider?}` | User document |
| `POST /auth/signin` | `routes/auth.js` | `{email, password?, authProvider?}` | User document |
| `POST /test/generate` | `routes/test.js` | `{userId, subjects[], topics[], difficulty, language}` | Test document |
| `POST /test/generate/content` | `routes/test.js` | `{userId, content, difficulty}` | Test document |
| `POST /test/submit` | `routes/test.js` | `{testId, answers[]}` | Enriched Test document |
| `POST /doubt/create` | `routes/doubt.js` | `multipart/form-data: image` | Doubt document |
| `POST /lessonPlan/create` | `routes/lessonPlan.js` | `{subjects, topics, grade}` | Lesson plan object |

No `GET`, `PUT`, or `DELETE` endpoints exist. The API is RPC-style rather than strictly RESTful — all operations travel as `POST` with a body.

---

### Third-Party Integrations

#### Google Gemini 1.5 Pro (via LangChain)
- **Client:** `ChatGoogleGenerativeAI` from `@langchain/google-genai`
- **Auth:** API key via environment variable, passed to LangChain at initialization
- **Invocation pattern:** `await llm.invoke([new SystemMessage(...), new HumanMessage(...)])`
- **No streaming** — all calls are fire-and-wait; Vercel's `maxDuration: 60` accommodates the latency

#### Firebase
- **Frontend Auth:** `signInWithPopup(auth, new GoogleAuthProvider())` — browser popup, returns credential with `displayName`, `email`, `photoURL`
- **Backend Storage:** Firebase Admin SDK writes image buffers server-side; returns a permanent download URL

#### Tesseract.js (OCR)
- Runs server-side as WASM
- Input: raw file buffer from `express-fileupload`
- Language: English (`eng`)
- Output: plain text string, passed directly to the Gemini doubt-solving prompt

---

## 8. AI / LLM Integration

### How Every LLM Call Is Structured

```js
import llm from "./llmProvider.js";
import { SystemMessage, HumanMessage } from "@langchain/core/messages";

const response = await llm.invoke([
  new SystemMessage("You are an expert educator. Generate exactly 5 MCQs..."),
  new HumanMessage(`Subject: ${subject}, Topic: ${topic}`),
]);
```

### Response Sanitization (Applied in All 5 Utils)

Gemini often wraps JSON output in markdown code fences. Every utility strips these before parsing:

```js
const raw = response.content;
const cleaned = raw
  .trim()
  .replace(/^```json\n?/, '')
  .replace(/^```\n?/, '')
  .replace(/```$/, '');
const parsed = JSON.parse(cleaned);
```

### Prompt Design by Feature

| Feature | LLM Persona | Output Schema | Key Constraint |
|---|---|---|---|
| MCQ (topic) | Expert educator for the given subject | `[{topic, question, options[4], answer(int 0-3), solution}]` | Exactly 5 questions |
| MCQ (content) | Expert professor | Same schema | Questions only from provided content |
| Test analysis | JEE expert analyst | `{Performance_Analysis, Strengths, Weaknesses, Targeted_areas_for_Improvement}` | Ground analysis in the topic performance data |
| Doubt solver | Implied subject expert | Free-form text explanation | Step-by-step solution |
| Lesson plan | Curriculum designer | 12-field nested JSON | Adhere to explicit schema; grade-appropriate |

### Lesson Plan Fallback Defaults

`generateLessonPlan.js` applies `||` defaults after every parse to prevent frontend rendering errors:

```js
lessonPlan.title = lessonPlan.title || `${topic} Lesson Plan`;
lessonPlan.prerequisites = lessonPlan.prerequisites || [];
lessonPlan.contentOutline = lessonPlan.contentOutline || [];
// ... and so on for every field
```

---

## 9. Authentication

### Two Paths, One User Model

```
Password Auth:                          Google OAuth:
──────────────────────────────          ──────────────────────────────────────
POST /auth/signup                       Firebase signInWithPopup(GoogleAuthProvider)
  bcrypt.hash(password, 10)               ↓
  User.save()                           Extract: displayName, email, photoURL
                                          ↓
POST /auth/signin                       POST /auth/signup {authProvider: "GOOGLE"}
  User.findOne({email})                   Skip password hashing → User.save()
  bcrypt.compareSync(pw, hash)            ↓
  Return user doc                       POST /auth/signin {authProvider: "GOOGLE"}
                                          Skip bcrypt → Return user doc
```

The `authProvider` string field is the discriminator: its presence causes the backend to skip all password logic.

### Session Handling

No server-side session or JWT is issued at login. After a successful auth call, the User document is stored in:
- **React Context** (`UserContextProvider`) for the current browser session
- **`localStorage`** for persistence across page refreshes

`jsonwebtoken` is listed in `server/package.json` but unused in any route — all endpoints that accept `userId` implicitly trust the client-supplied value.

---

## 10. Key Design Decisions

### All Endpoints Are POST

Every route uses `POST`, even logically read-like operations such as generating questions. This allows all parameters — including potentially large content blobs — to travel in the request body, avoiding URL-length limits and query-string encoding issues.

### Monolithic Dashboard Page

All four AI tools are rendered inside a single `dashboard/page.tsx` (~845 lines) rather than split across separate routes. This avoids cross-page state sharing (e.g., holding the current test object while navigating to an analysis page), at the cost of a large, harder-to-test component.

### LangChain as the LLM Abstraction Layer

Rather than calling the Gemini REST API directly, LangChain is used as an intermediary. This adds a dependency but provides: automatic retry logic (`maxRetries: 2`), a unified message format, and a one-line migration path if the model changes. Only `llmProvider.js` needs updating to swap providers.

### Temperature = 0

All LLM calls use `temperature: 0` for fully deterministic output. This is critical for structured JSON generation: higher temperatures increase the chance of the model deviating from the required schema, causing `JSON.parse()` failures. The trade-off is less varied MCQ output for repeated identical inputs.

### In-Memory File Processing

The doubt solver processes image files entirely in memory using the buffer from `express-fileupload`. Nothing touches disk. This is required for Vercel's read-only serverless filesystem, but means large images consume server RAM for the duration of the request.

### Application-Level Topic Aggregation

Topic performance stats are computed by iterating the questions array in JavaScript (`analyseTest.js`) rather than using a MongoDB aggregation pipeline. This simplifies the query logic and is appropriate for a fixed set of 5 questions per test, but would not scale to large question counts.

### Lesson Plans Not Persisted

The `LessonPlan` Mongoose schema is fully defined (65 lines), but the `/lessonPlan/create` route returns the plan without saving it. This appears to be an incomplete feature — the infrastructure for persistence is in place but the `.save()` call is absent.

---

## 11. Environment Variables

### Server (`server/.env`)

| Variable | Controls |
|---|---|
| `MONGODB_URI` | MongoDB Atlas connection string (credentials embedded) |
| `PORT` | Express server listen port (Vercel sets this automatically) |
| `GEMINI_API_KEY` | Google Gemini API key passed to LangChain |
| `FIREBASE_API_KEY` | Firebase project API key |
| `FIREBASE_AUTH_DOMAIN` | Firebase auth domain |
| `FIREBASE_PROJECT_ID` | Firebase project ID |
| `FIREBASE_STORAGE_BUCKET` | Firebase Storage bucket name |
| `FIREBASE_MESSAGING_SENDER_ID` | Firebase messaging sender ID |
| `FIREBASE_APP_ID` | Firebase app ID |
| `GOOGLE_APPLICATION_CREDENTIALS` | Path to Google Cloud service account JSON (Vision API) |

### Client (`client/.env.local`)

| Variable | Controls |
|---|---|
| `NEXT_PUBLIC_Backend_Route` | Base URL of the Express server (e.g., `https://your-api.vercel.app`) |
| `NEXT_PUBLIC_FIREBASE_API_KEY` | Firebase API key for the client-side SDK |
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | Firebase auth domain |
| `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | Firebase project ID |
| `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` | Firebase storage bucket |
| `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | Firebase messaging sender ID |
| `NEXT_PUBLIC_FIREBASE_APP_ID` | Firebase app ID |
| `NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID` | Firebase Analytics measurement ID |

All `NEXT_PUBLIC_` variables are bundled into client-side JavaScript at build time and are visible to end users. Do not put secrets in `NEXT_PUBLIC_` variables.

---

## 12. Error Handling & Logging

### Backend

- Route handlers use `try/catch`; errors are returned as JSON error responses or passed to Express's default error handler.
- LangChain's `maxRetries: 2` silently retries transient Gemini API failures before surfacing an error.
- The JSON sanitization pipeline (`JSON.parse`) throws if Gemini returns malformed output — this propagates to the route handler uncaught at the utility level.
- No structured logging library (Winston, Pino) is used. Only `console.log` / `console.error`.
- No request IDs, correlation IDs, or distributed tracing.

### Frontend

- Axios errors are caught in `try/catch` blocks inside event handlers on the dashboard page.
- API failures typically result in a blank response or a silent console error — there is no toast or alert system for user-facing error messages.
- `<FullPageLoader />` blocks the UI during in-flight requests to prevent duplicate submissions.

### Background Jobs & Workers

None. There are no cron jobs, message queues, or background workers. Every operation is triggered synchronously by a user request.

---

## 13. Quick Setup

### Prerequisites

- Node.js 18+
- MongoDB Atlas cluster (free tier works)
- Google Cloud project with Gemini API enabled
- Firebase project with Authentication (Google provider) and Storage enabled

### Server

```bash
cd server
npm install
cp .env.example .env   # fill in all variables from Section 11
npm run dev            # nodemon watches for changes
```

### Client

```bash
cd client
npm install
cp .env.local.example .env.local   # fill in NEXT_PUBLIC_* variables
npm run dev                         # http://localhost:3000
```

### Deployment (Vercel)

Deploy `client/` and `server/` as two separate Vercel projects. The `server/vercel.json` sets `maxDuration: 60` seconds to accommodate Gemini API latency. Set `NEXT_PUBLIC_Backend_Route` on the client project to point at the server's Vercel deployment URL.
