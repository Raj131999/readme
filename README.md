# Dynamic RAG Applet Studio: Architecture & System Guide

Welcome to the **Dynamic RAG Applet Studio**! This document provides an exhaustive, multi-dimensional overview of the platform's core architecture, local-vs-server status, storage layouts, database limits, included tools, and a live audit log of action history edits.

---

## 1. System FAQs & Answers (Critical Inquiries)

### 📌 Is `/data/db.json` the Vector Database?
**Yes, absolutely!** 
`/data/db.json` is a lightweight, fully self-contained local Vector Database. It is persistent, fast, and does not require complex external system connections.
* **How it Works**: 
  1. When you upload a document or create an App, the backend extracts the raw text blocks and sends them to Google's semantic models.
  2. Google's `text-embedding-004` converts that text into a **768-dimensional float index vector**.
  3. This vector array is written directly into `/data/db.json` alongside the source names, raw text chunks, and categories.
  4. When you ask a query, our `server.ts` calculates real-time **Cosine-Similarity scores** on the vector space to retrieve the highest relevancy matches in milliseconds.

### 📌 Did anything get installed on my local computer?
**No, absolutely nothing is installed on your local operating system.** 
The entire application—including the TypeScript compilers, Node.js development servers, API proxies, local database management tools, and compiler runtimes—executes within a secure, isolated sandbox container hosted on Google Cloud Run (GCP).

### 📌 Where are the npm packages installed? (Is it on my local?)
All packages (defined in `package.json` such as `express`, `@google/genai`, `recharts`, `react-markdown`, etc.) are installed in the **cloud sandboxed container’s `/node_modules` directory**, entirely bypassed from your personal local environment.

### 📌 How much data can this system accept?
- **Single File Upload Ceiling**: Up to **50 Megabytes (50MB)** per file upload, configured explicitly in the `server.ts` file via the custom `multer` memory size allocation limit.
- **Combined Storage Capacity**: The in-memory persistent database holds up to **100MB+** of serialized text, parsed metadata segments, source metrics, and 768-dimensional float arrays (computed vector embeddings) comfortably.
- **Semantic Chunk Limits**: Designed to easily index **thousands** of documentation nodes, database schemas, raw cURL actions, video transcription timelines, and source sheets.

### 📌 Where is the processed data stored?
- All uploaded files, parsed text snippets, summaries, and semantic vectors are saved inside a persistent JSON database located at:
  ```
  /data/db.json
  ```
- This directory is dynamically mounted inside your applet workspace, guaranteeing fast, sub-millisecond local read/write queries without requiring complex external database engines.

### 📌 Where are dynamically created apps stored?
1. **Interactive Client State**: When an app is generated, its full configuration is maintained in the React SPA application context state (`appletHistory` in `src/App.tsx`). This allows high-performance instant navigation, tab switching (Applet Studio vs RAG Knowledge Base), and dynamic theme rendering.
2. **Knowledge Base RAG feedback**: The generated JSON configuration is serialized into a comprehensive text document outlining stats, charts, columns, row payloads, and interactive presets. This document is computed into a 768-dimensional vector using `text-embedding-004` and stored inside `/data/db.json` as a text source. This means any subsequent questions asked in the RAG Knowledge base can query, cite, explaining, or reuse details from previously generated apps!

---

## 2. Tools Cost Analysis: Free vs. Paid Decks

Getting a production-ready RAG application up and running does **not** have to cost money. Here is the operational breakdown of the stack:

### 🟢 100% Free Tools (Included / Embedded)
- **Front-End Styling & UI Layout**: Tailwind CSS, Lucide React (Icons), standard CSS variables. Fully free and open source.
- **Interactive Component Decks**: Recharts (Charts engine), React Markdown (Renderer), Framer/Motion (Layout animations). High-grade visual elements, 100% free with no commercial restrictions.
- **Back-End API Server**: Express framework, Node.js environment, `tsx` runtimes, `esbuild` compilers. Fully free with zero host overhead.
- **Local Embedded Database**: Flat JSON database system (`/data/db.json`) running locally with zero query costs and unlimited throughput.

### 🟡 Freemium Tools (Generous Free Limits)
- **Google Gemini API (`gemini-3.5-flash` & `text-embedding-004`)**: Google AI Studio provides high-volume free tier limits. For research and personal developmental playgrounds, you operate 100% free. Paid standard models trigger only when scaling past developer API throttle rates.
- **Sandbox Container Host (Google Cloud Run)**: Comes with a generous free allocation of 2 million requests per month, which fully covers sandbox run environments.

---

## 3. "Best in World" Free Setup: Scaling Storage to Gigabytes (GBs)

If you plan to scale this system to handle millions of records or single uploads reaching **gigabytes (GBs)** of data for **zero cost**, follow these elite architectural blueprints:

### 🚀 Step A: Scaling Single File Uploads (From 50MB to Multi-Gigabytes)

*🚨 The Problem with the Current Setup:* Node/Express loads files directly into the active system memory buffer (`multer.memoryStorage()`). Attempting to upload a 2GB file directly of that style will instantly trigger container Out Of Memory (OOM) crashes.

*🛠️ The "Best in World" Solution (100% Free):*
1. **Direct-to-Cloud Object Storage using Pre-signed URLs**:
   - Do **not** upload files directly to your main server. 
   - Instead, register a **Cloudflare R2** account (offers **10GB free storage** monthly with **completely zero egress metadata charges**) or use **Google Cloud Storage (GCS)** standard free bounds (5GB).
   - In `server.ts`, add an endpoint that generates a short-lived **Pre-signed S3 Upload URL**.
   - On the client UI, upload the file directly from the browser to the object bucket via a simple `PUT` request using that pre-signed URL. 
   - *Result*: You bypass your application server memory completely, allowing the browser to transition multi-GB files directly to secure cloud buckets for absolute zero cost.
2. **Implement Chunked Uploading (Tus Protocol)**:
   - Integrate `tus-node-server` back-end or library modules.
   - Files are sliced into small, robust chunks (e.g., 5MB blocks) and uploaded on-the-flight. If the Wi-Fi drops, it seamlessly resumes from the exact byte threshold, providing a flawless premium user experience for 0 USD.

---

### 🚀 Step B: Scaling Combined Storage & Vector Database (To Multi-Gigabytes)

*🚨 The Problem with the Current Setup:* Writing and reading a giant multi-gigabyte flat-file database (`/data/db.json`) on every request will eventually lag and lock single-threaded resources on your server.

*🛠️ The "Best in World" Solution (100% Free):*

Select one of these premium managed databases to maintain millions of float nodes for 100% free:

#### 1. Supabase (Managed Postgres + pgvector) — *Highly Recommended*
- **Free Limit**: **500MB of pure database table space**.
- **Setup Pattern**: 
  - Install the `pg` client and configure a table with:
    ```sql
    CREATE EXTENSION IF NOT EXISTS vector;
    CREATE TABLE document_chunks (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      source_name TEXT,
      text_content TEXT,
      embedding VECTOR(768) -- Matches Gemini's dimension
    );
    ```
  - Use simple SQL metadata filters and pgvector's fast HNSW index queries. It scales instantly to multi-gigabytes while remaining 100% free of charge.

#### 2. Pinecone Service (Serverless Starter Tier)
- **Free Limit**: **1 Multi-tenant index supporting up to ~100,000 text blocks** (equivalent to roughly 1GB - 2GB of context vectors).
- **Setup Pattern**: Create an account, fetch an API key, use the Pinecone SDK to upload (`upsert`) embeddings, and query values using sub-second search indices.

#### 3. Qdrant Cloud Cluster (Managed Vector Engine)
- **Free Limit**: **1 Managed Instance of 1GB RAM and 4GB Disk capacity**.
- **Setup Pattern**: Extreme high-performance semantic search platform with a custom interactive container dashboard out of the box, fully free.

---

## 4. Core Full-Stack Architecture

The platform operates on a high-productivity full-stack topology:

```
[ FRONT-END REACT SPA ] 
       │
       ▼ (Port 3000 /api/* requests)
[ EXPRESS BACKEND ROUTER ]
       ├─► [ Gemini API Module (text-generation & text-embedding-004) ]
       ├─► [ Memory RAG File Storage Segment (/data/db.json) ]
       └─► [ Asset Ingestion Center (PDFs, cURLs, Docx, SQL, URLs) ]
```

### Key Modules:
- **Client Canvas (`src/App.tsx`)**: Controls visual tabs, file uploading decks, workspace cleanups, and the prompt interface for building apps.
- **Applet Engine (`src/components/DynamicAppletRenderer.tsx`)**: A fully polymorphic React renderer that accepts a `DynamicAppletConfig` schema and generates customized dashboard KPI boxes, interactive data sheets, filter blocks, charts, and virtual Q&A expert panels from scratch.
- **Backend API (`server.ts`)**: Manages multipart file processing with smart chunking heuristics, cosine vector calculations, fast context retrieval, Applet JSON generation, and client stat compilation.

---

## 5. Included System Tools

This system leverages premium engineering tools to provide diagnostic, rendering, and AI capabilities:
- **Google GenAI SDK (`@google/genai`)**: Modern Google GenAI TypeScript SDK, utilized for lightning-fast text completion (`gemini-3.5-flash`) and premium, high-dimensional vector embeddings (`text-embedding-004`).
- **Recharts (`recharts`)**: Declarative responsive charting library used for drawing seamless bar, line, and area charts dynamically within custom applets.
- **Lucide Icons (`lucide-react`)**: Clean, standard system icons used for visual markers, trend lines, and user controls.
- **React Markdown (`react-markdown`)**: Standard parsing component used to display beautiful, structured, human-readable answers from the RAG expert chat inside applets and main chat screens.
- **Vite CLI & Dev Middleware**: Live hot-rebuilding server configuration running on port `3050` mapped to custom reverse proxies.

---

## 6. Platform Action History log (Dynamic Audit)

| Step ID / Milestone | Action Executed | Outcome / Status |
| :--- | :--- | :--- |
| **Milestone 01** | Created `DynamicAppletRenderer.tsx` | Polymorphic component with charts, tables, interactive select filters, and sandboxed contextual assistant support. |
| **Milestone 02** | Created `src/App.tsx` main cockpit | Core workspace tabs, ingestion sidebar panels, conversational UI with grounding source details, and builder prompt interface. |
| **Milestone 03** | Repaired linter syntax inside `IngestionPanel.tsx` | Fixed a trailing bracket typo in standard database fetch callbacks. |
| **Milestone 04** | Resolved API schema matching in `server.ts` | Handled variant output structures for `EmbedContentResponse` (`embeddings` array lists vs `embedding` objects). |
| **Milestone 05** | Configured Top-Level Await ES modules | Wrapped Vite production startup block inside `async startServer()` to fully support standard Node CommonJS builds without compilation crashes. |
| **Milestone 06** | Connected Feedback loop for Created Apps | Programmed the system to automatically generate, embed, and store detailed technical blueprints of created Apps back into `/data/db.json` as a first-class RAG resource. |
| **Milestone 07** | Documented Scaling blueprints & operational costs | Outlined exact solutions to scale single uploads and vector DB storage up to Gigabytes (GBs) for 100% zero costs. |

*This document is automatically updated upon every core codebase change or feature release.*

