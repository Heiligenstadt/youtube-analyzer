# YouTube Brand Analyzer - Architecture Specification (Updated)

## Project Overview
A multi-agent system that analyzes YouTube videos for brand mentions and sentiment, then generates strategic engagement recommendations. Built for interview demonstration at Plot Technologies.

## Core Functionality
**Input:** YouTube video URL(s) + brand website URL  
**Output:** 
- Brand relevance analysis
- Sentiment assessment
- Engagement recommendations (draft comments/tweets)

**Current Status:** Core analysis working, chat feature and caching not yet implemented.

---

## Architecture

### System Components
```
User Input (video URL + brand website URL)
    ↓
URL Validation (hard-coded function)
    ↓
Brand Document Fetching (per-request)
├─ Fetch brand website content
├─ Chunk with RecursiveCharacterTextSplitter
└─ Embed and store in MemoryVectorStore
    ↓
Video Data Fetching (parallel - hard-coded functions)
├─ fetchVideoTranscript() → chunks (via Supadata API)
├─ fetchComments() → array of strings
└─ fetchAnalytics() → stats object
    ↓
Agent 1: Analyst
├─ Tool: brand_knowledge (MemoryVectorStore RAG)
├─ Analyzes: video chunks, comments, stats with brand context
└─ Outputs: analysis object
    ↓
Agent 2: Evaluator
├─ Reviews Agent 1's output (structured response)
├─ Returns: { approved: boolean, output: string }
└─ Approves OR requests revision
    ↓
Return to User
```

---

## Agent Details

### Agent 1: Analyst (Worker Agent)
**Role:** Analyze video content for brand relevance

**Tools:**
- `brand_knowledge(query)` - RAG search in MemoryVectorStore for brand content

**Input:**
- Video transcript chunks (from Supadata API)
- Comments array (strings)
- Video stats object
- Brand website URL (triggers brand doc loading)

**Output:**
- Relevance: high/medium/low/none
- Key insights (3-5 bullet points)
- Sentiment: positive/neutral/negative
- (Optional) Draft comment/tweet if user requests

**System Prompt:**
```
You analyze YouTube videos for brand relevance and sentiment.

Given video content, comments, and brand context:
1. Determine if the brand is mentioned or relevant
2. Assess the tone and sentiment
3. Provide actionable insights
4. If requested, draft an engaging comment or tweet

Use the brand_knowledge tool to understand brand values and voice.
```

**Implementation:**
```typescript
import { createAgent } from 'langchain';

const analystAgent = createAgent({
  model: new ChatOpenAI({ model: 'gpt-4o-mini', temperature: 0.7 }),
  tools: [brandKnowledgeTool],
  systemPrompt: `You analyze YouTube videos for brand relevance...`
});
```

### Agent 2: Evaluator (Quality Control Agent)
**Role:** Review Agent 1's output before showing to user

**Tools:** None (reviews only)

**Input:** Agent 1's analysis

**Output (Structured):**
```typescript
{
  approved: boolean,
  output: string
}
```

**System Prompt:**
```
You evaluate analysis quality before it reaches the user.

Check:
- Accuracy: Does the analysis match the data?
- Brand alignment: Does it reflect brand values?
- Completeness: Are all key points covered?
- Tone: Is it professional and helpful?

Return approved: true if good, or approved: false with specific feedback.
```

**Implementation:**
```typescript
import { createAgent } from 'langchain';
import { z } from 'zod';

const evaluatorSchema = z.object({
  approved: z.boolean(),
  output: z.string()
});

const evaluatorAgent = createAgent({
  model: new ChatOpenAI({ model: 'gpt-4o-mini' }),
  tools: [],
  systemPrompt: `You evaluate analysis quality...`,
  responseFormat: {
    type: 'json_object',
    schema: evaluatorSchema
  }
});
```

---

## Tech Stack

### Backend
- **Runtime:** Node.js + TypeScript
- **Execution:** tsx (ESM-compatible)
- **Framework:** Express (planned, not yet implemented)
- **LLM:** LangChain + OpenAI (gpt-4o-mini)
- **Vector DB:** MemoryVectorStore (in-memory, local)
- **Embeddings:** OpenAI text-embedding-3-small (512 dimensions)
- **APIs:** 
  - Supadata API (video transcripts)
  - YouTube Data API v3 (comments, stats)

### Frontend (Not Yet Built)
- React (planned)
- Or just use Postman/curl for testing

### Not Yet Implemented
- ❌ Upstash Redis (caching)
- ❌ Chat endpoint
- ❌ Session management

---

## Folder Structure
```
youtube-analyzer/
├── server/
│   ├── agents/
│   │   ├── analyst.ts          # Agent 1
│   │   └── evaluator.ts        # Agent 2
│   ├── tools/
│   │   └── brand-knowledge.ts  # MemoryVectorStore RAG tool (hyphenated!)
│   ├── utils/
│   │   ├── validation.ts       # URL validation
│   │   ├── chunking.ts         # RecursiveCharacterTextSplitter wrapper
│   │   ├── fetch-video.ts      # Supadata transcript fetcher
│   │   ├── fetch-comments.ts   # YouTube comments
│   │   └── fetch-analytics.ts  # YouTube stats
│   ├── test-utils.ts           # Testing script
│   └── server.ts               # Express app (planned)
├── .env                         # API keys
├── .gitignore
├── .cursorrules
├── package.json
└── tsconfig.json
```

**Key Changes from Original Plan:**
- ✅ Hyphenated filenames: `brand-knowledge.ts`, `fetch-video.ts`, etc.
- ✅ Fetch functions in `utils/` not `tools/` (tools are for agent tools only)
- ✅ Using `tsx` for execution instead of `ts-node`

---

## Data Flow

### Current Implementation (No API Yet)

**Testing Flow:**
```typescript
// Direct function calls (no HTTP)
const videoData = await fetchVideoTranscript(videoUrl);
const comments = await fetchComments(videoUrl);
const stats = await fetchAnalytics(videoUrl);

// Load brand content per-request
await embedAndStore(brandUrl, vectorStore);

// Run agents
const analysis = await analystAgent.invoke({...});
const evaluation = await evaluatorAgent.invoke({...});

// Return result
console.log(evaluation.output);
```

---

## API Endpoints (Planned)

### POST /api/analyze (Not Yet Implemented)

**Request Schema:**

| Property | Type | Required | Description | Example |
|----------|------|----------|-------------|---------|
| `videoUrl` | `string` | ✅ Yes | Valid YouTube URL | `"https://youtube.com/watch?v=dQw4w9WgXcQ"` |
| `brandUrl` | `string` | ✅ Yes | Brand website URL to analyze | `"https://www.nike.com"` |

**Request Example:**
```json
{
  "videoUrl": "https://youtube.com/watch?v=dQw4w9WgXcQ",
  "brandUrl": "https://www.nike.com"
}
```

**Response Schema (Success - 200):**

| Property | Type | Description |
|----------|------|-------------|
| `relevance` | `"high" \| "medium" \| "low" \| "none"` | Brand relevance level |
| `insights` | `AnalysisInsights` | Analysis details |

**AnalysisInsights Object:**

| Property | Type | Description |
|----------|------|-------------|
| `summary` | `string` | Brief summary of findings |
| `sentiment` | `"positive" \| "neutral" \| "negative"` | Overall sentiment |
| `keyPoints` | `string[]` | Array of key insights (3-5 items) |
| `videoStats` | `VideoStats` | Video statistics |

**VideoStats Object:**

| Property | Type | Description |
|----------|------|-------------|
| `views` | `number` | View count |
| `likes` | `number` | Like count |
| `commentCount` | `number` | Comment count |

---

## Environment Variables
```bash
# .env
OPENAI_API_KEY=sk-...
YOUTUBE_API_KEY=AIza...
SUPADATA_API_KEY=...          # For video transcripts
PORT=3000

# Not yet implemented:
# UPSTASH_REDIS_REST_URL=https://...
# UPSTASH_REDIS_REST_TOKEN=...
```

---

## Key Implementation Details

### 1. URL Validation (Simple Function)
```typescript
// utils/validation.ts
export function validateYouTubeUrl(url: string): { valid: boolean; videoId?: string } {
  const regex = /(?:youtube\.com\/watch\?v=|youtu\.be\/)([^&\n?#]+)/;
  const match = url.match(regex);
  return match ? { valid: true, videoId: match[1] } : { valid: false };
}
```

### 2. Transcript Fetching (Supadata API)
```typescript
// utils/fetch-video.ts
export async function fetchVideoTranscript(url: string): Promise<string> {
  const { videoId } = validateYouTubeUrl(url);
  
  const response = await fetch(
    `https://api.supadata.ai/v1/youtube/transcript?url=https://youtube.com/watch?v=${videoId}`,
    {
      headers: {
        'x-api-key': process.env.SUPADATA_API_KEY!,
      }
    }
  );
  
  const data = await response.json();
  return data.transcript; // Full text
}
```

### 3. Text Chunking (RecursiveCharacterTextSplitter)
```typescript
// utils/chunking.ts
import { RecursiveCharacterTextSplitter } from '@langchain/textsplitters';

export async function chunkText(text: string, chunkSize: number = 500): Promise<string[]> {
  const splitter = new RecursiveCharacterTextSplitter({
    chunkSize,
    chunkOverlap: 100,
  });
  
  const chunks = await splitter.splitText(text);
  return chunks;
}
```

### 4. Comments Fetching
```typescript
// utils/fetch-comments.ts
import { google } from 'googleapis';

const youtube = google.youtube({
  version: 'v3',
  auth: process.env.YOUTUBE_API_KEY!,
});

export async function fetchComments(url: string): Promise<string[]> {
  const { videoId } = validateYouTubeUrl(url);
  
  const response = await youtube.commentThreads.list({
    part: ['snippet'],
    videoId,
    maxResults: 100,
  });
  
  return response.data.items?.map(
    item => item.snippet?.topLevelComment?.snippet?.textDisplay || ''
  ) || [];
}
```

### 5. Analytics Fetching
```typescript
// utils/fetch-analytics.ts
export async function fetchAnalytics(url: string) {
  const { videoId } = validateYouTubeUrl(url);
  
  const response = await youtube.videos.list({
    part: ['statistics'],
    id: [videoId],
  });
  
  const stats = response.data.items?.[0]?.statistics;
  
  return {
    views: parseInt(stats?.viewCount || '0'),
    likes: parseInt(stats?.likeCount || '0'),
    commentCount: parseInt(stats?.commentCount || '0'),
  };
}
```

### 6. Brand Knowledge Tool (MemoryVectorStore)
```typescript
// tools/brand-knowledge.ts
import { tool } from 'langchain';
import { MemoryVectorStore } from '@langchain/classic/vectorstores/memory';
import { OpenAIEmbeddings } from '@langchain/openai';
import { z } from 'zod';

const embeddings = new OpenAIEmbeddings({ 
  model: 'text-embedding-3-small'
});

export async function embedAndStore(brandUrl: string, vectorStore: MemoryVectorStore) {
  // Fetch brand website content
  const response = await fetch(brandUrl);
  const html = await response.text();
  
  // Extract text from HTML (simple approach - could be improved)
  const text = html.replace(/<[^>]*>/g, ' ').replace(/\s+/g, ' ');
  
  // Chunk the text
  const chunks = await chunkText(text, 500);
  
  // Create documents
  const docs = chunks.map(chunk => ({
    pageContent: chunk,
    metadata: { source: brandUrl }
  }));
  
  // Add to vector store
  await vectorStore.addDocuments(docs);
}

export function createBrandKnowledgeTool(vectorStore: MemoryVectorStore) {
  return tool(
    async ({ query }) => {
      const results = await vectorStore.similaritySearch(query, 3);
      return results.map(doc => doc.pageContent).join('\n\n');
    },
    {
      name: 'brand_knowledge',
      description: 'Search brand website content for relevant information',
      schema: z.object({
        query: z.string().describe('What to search for in brand content')
      })
    }
  );
}
```

### 7. Agent Setup
```typescript
// agents/analyst.ts
import { createAgent } from 'langchain';
import { ChatOpenAI } from '@langchain/openai';
import { createBrandKnowledgeTool } from '../tools/brand-knowledge.js';

export function createAnalystAgent(vectorStore: MemoryVectorStore) {
  const model = new ChatOpenAI({ 
    model: 'gpt-4o-mini',
    temperature: 0.7 
  });
  
  const brandKnowledgeTool = createBrandKnowledgeTool(vectorStore);
  
  return createAgent({
    model,
    tools: [brandKnowledgeTool],
    systemPrompt: `You analyze YouTube videos for brand relevance...`
  });
}
```
```typescript
// agents/evaluator.ts
import { createAgent } from 'langchain';
import { ChatOpenAI } from '@langchain/openai';
import { z } from 'zod';

const evaluatorSchema = z.object({
  approved: z.boolean().describe('Whether the analysis is approved'),
  output: z.string().describe('Final output or feedback for revision')
});

export const evaluatorAgent = createAgent({
  model: new ChatOpenAI({ model: 'gpt-4o-mini' }),
  tools: [],
  systemPrompt: `You evaluate analysis quality...`,
  responseFormat: {
    type: 'json_object',
    schema: evaluatorSchema
  }
});
```

---

## Performance Metrics (Current)
```
✅ Chunking:        ~1ms (4 chunks)
✅ Video stats:     ~150ms
✅ Comments:        ~240ms
✅ Transcript:      ~1s (Supadata API)
⚠️  Analyst agent:  ~16s (includes RAG queries + analysis)
✅ Evaluator:       ~2s
---
Total:              ~19.5s
```

**Performance Notes:**
- Analyst agent is the bottleneck (82% of total time)
- MemoryVectorStore queries are fast (in-memory)
- Most time spent in LLM inference, not data fetching

---

## Build Progress

### Completed ✅
- [x] Environment setup
- [x] URL validation
- [x] All data fetching functions (Supadata, YouTube API)
- [x] Text chunking with RecursiveCharacterTextSplitter
- [x] MemoryVectorStore setup
- [x] Brand knowledge tool
- [x] Agent 1 (Analyst)
- [x] Agent 2 (Evaluator) with structured output
- [x] Full integration working

### Not Yet Implemented ❌
- [ ] Redis caching
- [ ] Chat endpoint for follow-ups
- [ ] Express API (`/api/analyze`)
- [ ] Session management
- [ ] Error handling/retries
- [ ] Frontend

---

## Next Steps (Tomorrow - Feb 17)

### Priority 1: Optimization
1. Implement Redis caching
   - Cache transcripts (24h TTL)
   - Cache comments (1h TTL)
   - Cache brand embeddings (1h TTL)
   - Cache analysis results (1h TTL)

### Priority 2: Chat Feature
1. Implement chat agent for follow-ups
2. Store chat history in Redis
3. Reference cached analysis for context

### Priority 3: API Layer (If Time)
1. Set up Express server
2. Implement `POST /api/analyze`
3. Add proper error handling

---

## TypeScript Type Definitions
```typescript
// Request Types (for future API)
export interface AnalyzeRequest {
  videoUrl: string;
  brandUrl: string;  // Changed from brandName
}

export interface ChatRequest {
  sessionId: string;
  message: string;
}

// Response Types
export type RelevanceLevel = 'high' | 'medium' | 'low' | 'none';
export type SentimentType = 'positive' | 'neutral' | 'negative';

export interface VideoStats {
  views: number;
  likes: number;
  commentCount: number;
}

export interface AnalysisInsights {
  summary: string;
  sentiment: SentimentType;
  keyPoints: string[];
  videoStats: VideoStats;
  draftComment?: string;
}

export interface AnalyzeResponse {
  relevance: RelevanceLevel;
  insights: AnalysisInsights;
}

export interface ChatResponse {
  reply: string;
  draftComment?: string;
}

// Evaluator structured output
export interface EvaluatorResponse {
  approved: boolean;
  output: string;
}
```

---

## Interview Talking Points

### Architecture Decisions

**Brand Input Change:**
> "I changed from brandName to brandUrl so the system can dynamically load brand content from their website. This makes it more flexible - you don't need pre-loaded brand docs, just point it at any brand's site."

**MemoryVectorStore vs Pinecone:**
> "For the MVP, I used MemoryVectorStore because it's simpler and faster for single-user scenarios. In production, I'd migrate to Pinecone for persistent storage and multi-user support, but the tool interface stays the same."

**Supadata API:**
> "YouTube's transcript APIs were unreliable (youtube-transcript broke, LangChain's loader had issues). Supadata provides a stable, paid API with better reliability. In production, I'd add fallback logic to try multiple sources."

**Structured Output for Evaluator:**
> "The evaluator returns a structured response with Zod validation - this ensures I always get a predictable format with `approved` boolean and `output` string. Makes error handling and routing much cleaner than parsing unstructured text."

**Per-Request Brand Loading:**
> "Since users provide brandUrl in each request, I load and embed brand content per-request rather than at startup. For production, I'd cache embeddings in Redis with brandUrl as the key to avoid re-processing."

**No Caching Yet:**
> "I built the core functionality first to prove the concept. Next step is Redis caching for transcripts, embeddings, and results - this will cut the 16-second analyst time dramatically for repeat analyses."

### Cost Optimization
> "The analyst agent takes 16 seconds, mostly from LLM inference. I plan to add caching for repeat videos and pre-filter chunks before sending to the LLM to reduce token usage."

### Production Roadmap
> "Current state: Working prototype with in-memory storage. Production additions would be: Redis for caching, Pinecone for persistent vector storage, session management for multi-user, and rate limiting."

---

## Changes from Original Plan

| Original | Implemented | Reason |
|----------|-------------|--------|
| `brandName: string` | `brandUrl: string` | Dynamic brand content loading |
| Pinecone | MemoryVectorStore | Simpler for MVP, faster iteration |
| youtube-transcript | Supadata API | Reliability issues with free libraries |
| `ts-node` | `tsx` | Better ESM support |
| Pinecone setup in startup | Brand loading per-request | User provides URL dynamically |
| Tools in `tools/` | Fetch utils in `utils/` | Better separation: tools = agent tools only |
| camelCase files | hyphenated-files | Consistent naming convention |
| createReactAgent | createAgent | Simpler API, matches current LangChain docs |
| Plain text evaluator | Structured Zod schema | Type-safe, predictable responses |

---

## Running the Project

### Install Dependencies
```bash
npm install --legacy-peer-deps
```

### Environment Setup
```bash
# Create .env
OPENAI_API_KEY=sk-...
YOUTUBE_API_KEY=AIza...
SUPADATA_API_KEY=...
```

### Run Tests
```bash
# Test utility functions
tsx server/test-utils.ts

# Test full agent flow (manual test script)
tsx server/test-agents.ts
```

### Future: Run Server
```bash
# Not yet implemented
npm run dev
```

---

**Last Updated:** Feb 16, 2026, 7:28 PM  
**Status:** Core functionality complete, optimization and API layer pending
