# Explorify Trips — AI-Powered Travel Planner

An end-to-end travel planning platform that uses **Gemini AI with real-time tool-calling** to generate complete trip itineraries — flights, trains, hotels, and day-by-day sightseeing — streamed live to the user.

> **Live Demo:** Frontend on **Vercel** • Backend on **AWS Lambda** (ap-south-1)

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [How It Works](#how-it-works)
- [Backend — Rust Lambda](#backend--rust-lambda)
  - [AI Agent Loop](#ai-agent-loop)
  - [Tool Declarations (Function Calling)](#tool-declarations-function-calling)
  - [Token Map System](#token-map-system)
  - [External API Integrations](#external-api-integrations)
  - [Structured Output Schema](#structured-output-schema)
- [Frontend — Next.js](#frontend--nextjs)
  - [WASM Session Manager](#wasm-session-manager)
  - [Real-Time Streaming UI](#real-time-streaming-ui)
- [Deployment](#deployment)
- [Environment Variables](#environment-variables)
- [Local Development](#local-development)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          USER (Browser)                             │
│  ┌──────────────┐   ┌──────────────────────────────────────────┐   │
│  │ Trip Wizard   │──▶│  Streaming Dashboard (partial-json)      │   │
│  │ (Form Input)  │   │  • Transport  • Hotels  • Itinerary      │   │
│  └──────────────┘   └─────────────────┬────────────────────────┘   │
│                                       │ WASM SessionManager        │
│                                       │ (gemini-client-api)        │
└───────────────────────────────────────┼─────────────────────────────┘
                                        │ POST /api/travel-planner/ask
                                        ▼
                              ┌──────────────────┐
                              │   Vercel (Edge)   │
                              │   Next.js 16      │
                              │   API Route Proxy │
                              └────────┬─────────┘
                                       │ Streams body through
                                       ▼
                         ┌───────────────────────────┐
                         │   AWS Lambda (ap-south-1)  │
                         │   Rust • RESPONSE_STREAM   │
                         │                            │
                         │  ┌──────────────────────┐  │
                         │  │   Gemini 3 Flash      │  │
                         │  │   (Function Calling)  │  │
                         │  └──────┬───────────────┘  │
                         │         │ tool calls        │
                         │         ▼                   │
                         │  ┌──────────────────────┐  │
                         │  │  RapidAPI / Google    │  │
                         │  │  • Google Flights     │  │
                         │  │  • Booking.com        │  │
                         │  │  • IRCTC Trains       │  │
                         │  │  • Google Places      │  │
                         │  └──────────────────────┘  │
                         └───────────────────────────┘
```

---
<img width="1908" height="882" alt="image" src="https://github.com/user-attachments/assets/d9b6b7de-a366-4328-b5bd-817303fa3f5d" />


## Tech Stack

| Layer | Technology | Details |
|-------|-----------|---------|
| **AI Model** | Gemini 3 Flash (Preview) | Structured output + function calling with thinking enabled |
| **Backend** | Rust | Compiled for `aarch64-unknown-linux-gnu`, runs on AWS Lambda |
| **Backend Runtime** | AWS Lambda | Response streaming mode (`RESPONSE_STREAM`) for real-time output |
| **Frontend** | Next.js 16 | React 19, TypeScript, TailwindCSS 4, Turbopack |
| **Session Management** | Rust → WASM | `gemini-client-api` compiled to WebAssembly for client-side session tracking |
| **External APIs** | RapidAPI + Google | Google Flights, Booking.com, IRCTC, Google Places |
| **Frontend Hosting** | Vercel | Edge runtime for API proxy routes |

---

## Project Structure

```
first-commit-hackathon/
├── backend/                          # Rust Lambda (AI Agent)
│   ├── Cargo.toml                    # Dependencies: gemini-client-api, lambda_runtime, reqwest
│   └── src/
│       ├── main.rs                   # Lambda entry point, stream handler
│       ├── constants.rs              # System prompt, structured output schema definitions
│       ├── function.rs               # Gemini AI setup, tool registration, retry logic
│       ├── execute_functions.rs      # Concurrent function call execution engine
│       ├── utils.rs                  # Date helpers, IATA code validation
│       └── api_requests/
│           ├── mod.rs
│           ├── flights.rs            # Google Flights API (search, booking details, booking URL)
│           ├── hotel.rs              # Booking.com API (search, details, availability)
│           ├── trains.rs             # IRCTC API (search trains, seat availability)
│           └── site_seen.rs          # Google Places API (text search, photos)
│
└── frontend/                         # Next.js 16 App
    ├── app/
    │   ├── page.tsx                  # Root redirect → /travel-planner/details
    │   ├── layout.tsx                # Root layout (Geist font, ThemeProvider, Navbar, Footer)
    │   ├── globals.css               # Global styles
    │   ├── travel-planner/
    │   │   ├── details/page.tsx      # Trip input wizard form
    │   │   ├── page.tsx              # Streaming dashboard (main AI interaction page)
    │   │   └── layout.tsx            # Travel planner layout wrapper
    │   ├── api/
    │   │   ├── travel-planner/
    │   │   │   ├── ask/route.ts      # Edge proxy → Lambda (streams response)
    │   │   │   └── resolve-photos/   # Google Places photo URL resolver
    │   │   └── places/autocomplete/  # Places autocomplete API
    ├── components/
    │   ├── travel-planner/
    │   │   ├── TripFormWizard.tsx     # Multi-step trip input form
    │   │   ├── output-box.tsx        # Streaming output container
    │   │   ├── plan-types.ts         # TypeScript types matching backend schema
    │   │   ├── response-to-html.ts   # Markdown → HTML renderer
    │   │   ├── travel-planner-context.tsx  # React context + WASM initialization
    │   │   └── sections/
    │   │       ├── TransportSection.tsx    # Flights & trains cards
    │   │       ├── HotelSection.tsx        # Hotel cards with images
    │   │       ├── ItinerarySection.tsx    # Day-by-day activity cards
    │   │       └── MessageThread.tsx       # Chat message thread
    │   ├── common/
    │   │   ├── nav/                   # Navbar + Footer
    │   │   └── theme-provider.tsx     # Dark/light theme
    │   └── ui/                        # Radix UI primitives (button, dialog, dropdown)
    ├── lib/
    │   ├── s3.ts                     # S3 presigned URLs, file management
    │   ├── utils.ts                  # Utility functions
    │   └── travel-planner/session/   # Rust → WASM session manager
    │       ├── Cargo.toml            # gemini-client-api (no_default_features), wasm-bindgen
    │       └── src/lib.rs            # SessionManager: tracks chat history client-side
    ├── types/
    │   └── api.ts                    # API response types
    ├── next.config.ts                # Image remote patterns (S3, Google, Unsplash)
    └── package.json
```

---

## How It Works

### 1. User Input → Trip Wizard
The user fills in a multi-step form (`TripFormWizard`) with:
- Source & destination (with Google Places autocomplete)
- Travel dates, number of travellers
- Budget and preferences

### 2. WASM Session Initialization
A **Rust-compiled WebAssembly** module (`SessionManager`) manages the Gemini conversation session entirely on the client side. This keeps the full chat history in the browser and avoids server-side session storage.

### 3. Streaming Request to Lambda
The frontend sends the serialized session + token map to the Next.js Edge API route (`/api/travel-planner/ask`), which proxies it to the **AWS Lambda function URL**. The secret is injected server-side to prevent direct API abuse.

### 4. AI Agent Loop (Lambda)
The Lambda function runs an **agentic loop**:

```
User Message → Gemini → [Function Calls?]
                              ↓ Yes
                Execute tools concurrently (flights, hotels, trains, places)
                              ↓
                Feed results back → Gemini → [More Function Calls?]
                              ↓ No
                Stream final structured JSON response
```

Each intermediate function call response and model reply is streamed back as newline-delimited JSON chunks.

### 5. Real-Time UI Updates
The frontend reads the stream, feeds each chunk into the WASM `SessionManager`, and uses `partial-json` to parse incomplete JSON. This allows the UI sections (transport, hotels, itinerary) to render **progressively** as the AI generates them.

### 6. Photo Resolution
After streaming completes, a post-processing step resolves Google Places photo references into actual image URLs via `/api/travel-planner/resolve-photos`.

---

## Backend — Rust Lambda

### AI Agent Loop

The core loop in [`main.rs`](backend/src/main.rs) works as follows:

1. **Receive** the serialized `Session` + `token_map` + `secret` in the request body
2. **Validate** the API secret against the `API_SECRET` environment variable
3. **Loop**: Call `plan_tour()` which sends the session to Gemini 3 Flash
4. **Stream** each response chunk back through Lambda's response streaming
5. If the model response contains **function calls**, execute them concurrently and loop back
6. If no function calls remain, send the final token map and close the stream

### Tool Declarations (Function Calling)

The AI has access to **9 tools** declared via the `gemini-client-api`'s `#[gemini_function]` macro:

| Tool | API Source | Purpose |
|------|-----------|---------|
| `flights_between` | Google Flights (RapidAPI) | Search one-way flights between airports |
| `flight_booking_details` | Google Flights (RapidAPI) | Get booking options for a specific flight |
| `flight_booking_link` | Google Flights (RapidAPI) | Get the final booking checkout URL |
| `trains_between` | IRCTC (RapidAPI) | Search trains between stations |
| `train_seats_available` | IRCTC (RapidAPI) | Check seat availability for a specific train |
| `get_hotel_by_coordinates` | Booking.com (RapidAPI) | Search hotels near GPS coordinates |
| `get_hotel_details` | Booking.com (RapidAPI) | Get detailed hotel info, rooms, photos |
| `get_room_availability` | Booking.com (RapidAPI) | Check room availability and pricing |
| `get_about_place` | Google Places API | Search for landmarks, attractions, restaurants |

All tool calls within a single model response are executed **concurrently** using `futures::join_all`.

### Token Map System

Flight booking tokens from Google Flights are extremely long strings (thousands of characters). To keep them out of the Gemini context window:

1. Real tokens are stored in a `Vec<String>` (the **token map**)
2. The AI sees only short placeholders like `TOKEN_0`, `TOKEN_1`, etc.
3. When the AI calls a tool with a placeholder, the backend resolves it to the real token
4. The final token map is sent back to the frontend for session persistence

### External API Integrations

| API | Provider | Host |
|-----|----------|------|
| Google Flights | RapidAPI | `google-flights2.p.rapidapi.com` |
| Booking.com Hotels | RapidAPI | `booking-com15.p.rapidapi.com` |
| IRCTC Trains | RapidAPI | `irctc1.p.rapidapi.com` |
| Google Places | Google Maps Platform | `places.googleapis.com` |

### Structured Output Schema

The AI is constrained to output JSON matching `PlanOutputSchema`:

```
PlanOutputSchema
├── outbound: Transports
│   ├── flights?: TransportDetails[]      # Flight options for getting there
│   └── trains?: TransportDetails[]       # Train options for getting there
├── inbound: Transports
│   ├── flights?: TransportDetails[]      # Flight options for return
│   └── trains?: TransportDetails[]       # Train options for return
├── hotels?: HotelDetails[]              # Hotel recommendations with photos & booking links
├── itinerary?: Activity[]               # Day-by-day activities with Google Maps links & images
└── message: string                       # AI's conversational message to the user
```

**Null semantics**: A `null` field means "keep the previous value" — only non-null fields overwrite. This enables incremental plan updates across multiple conversation turns.

---

## Frontend — Next.js

### WASM Session Manager

The [`session`](frontend/lib/travel-planner/session/) crate compiles the `gemini-client-api` Session type to **WebAssembly** using `wasm-bindgen`. This provides:

- **Client-side session tracking** — no server state needed for the chat
- **Function call inspection** — the UI can display which tools the AI is currently calling
- **Session serialization** — save/restore to `localStorage` for page refreshes
- **Chat history management** — append model and function responses from streamed chunks

The WASM binary (`session_bg.wasm`) is served from the `public/` directory and loaded synchronously via `initSync()`.

### Real-Time Streaming UI

The streaming pipeline:

```
Lambda stream → Edge proxy → ReadableStream → TextDecoder → split('\n')
    → SessionManager.add_chat(chunk) → partial-json parse → mergePlan()
    → React state update → UI sections render progressively
```

During streaming, animated "status pills" show the current function calls:
- *"Searching flights"*, *"Searching hotels"*, *"Finding best scenery"*, etc.

---

## Deployment

### Frontend (Vercel)
- Deployed as a standard Next.js app on **Vercel**
- Edge runtime enabled for the `/api/travel-planner/ask` proxy route
- Environment variables configured in Vercel dashboard

### Backend (AWS Lambda)
- **Function Name:** `travel-planner`
- **Region:** `ap-south-1` (Asia Pacific — Mumbai)
- **Runtime:** Custom (Rust compiled to AL2023)
- **Invoke Mode:** `RESPONSE_STREAM` — enables streaming responses via Lambda Function URL
- **Function URL:** Public (Auth type: `NONE`), secured by `API_SECRET` in request body
- **CORS:** Allow origin `*`, Allow methods `POST`, Allow headers `content-type`

To build and deploy the Lambda:
```bash
# Install cross-compilation target
rustup target add aarch64-unknown-linux-gnu

# Build release binary
cargo build --release --target aarch64-unknown-linux-gnu

# Package for Lambda
cp target/aarch64-unknown-linux-gnu/release/travel-planner ./bootstrap
zip lambda.zip bootstrap

# Deploy via AWS CLI
aws lambda update-function-code \
  --function-name travel-planner \
  --zip-file fileb://lambda.zip
```

---

## Environment Variables

### Backend (Lambda)
| Variable | Description |
|----------|-------------|
| `GEMINI_API_KEY` | Google AI Studio API key for Gemini 3 Flash |
| `RAPIDAPI_KEY` | RapidAPI key for Flights, Hotels, and Trains APIs |
| `GOOGLE_MAPS_API_KEY` | Google Maps Platform key (Places API) |
| `API_SECRET` | Shared secret between frontend and backend |

### Frontend (Vercel)
| Variable | Description |
|----------|-------------|
| `API_SECRET` | Must match the Lambda's `API_SECRET` |

---

## Local Development

### Frontend
```bash
cd frontend
npm install
npm run dev          # Starts Next.js dev server with Turbopack
```

### Backend
```bash
cd backend
# Create .env file with required variables (GEMINI_API_KEY, RAPIDAPI_KEY, etc.)
cargo test           # Run integration tests against live APIs
cargo build          # Build for local architecture
```

### WASM Session Module
```bash
cd frontend/lib/travel-planner/session
wasm-pack build --target web --out-dir pkg
# Copy session_bg.wasm to frontend/public/
cp pkg/session_bg.wasm ../../public/
```

---

## Key Libraries

| Library | Language | Purpose |
|---------|----------|---------|
| [`gemini-client-api`](https://crates.io/crates/gemini-client-api) | Rust | Gemini API client with streaming, function calling, sessions |
| [`lambda_runtime`](https://crates.io/crates/lambda_runtime) | Rust | AWS Lambda runtime with response streaming support |
| [`wasm-bindgen`](https://crates.io/crates/wasm-bindgen) | Rust | Compiles Rust to WASM for browser use |
| [`partial-json`](https://www.npmjs.com/package/partial-json) | JS | Parses incomplete/streaming JSON for progressive rendering |
| [`marked`](https://www.npmjs.com/package/marked) | JS | Markdown to HTML conversion |

---

<p align="center">Built with ☕ and Rust at First Commit Hackathon</p>
