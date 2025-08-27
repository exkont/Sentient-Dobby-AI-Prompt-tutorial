DISCORD: ilyakudyavykh

X.com: https://x.com/IKucheriavykh

# Dobby API Tutorial for AI Teacher

This document explains how to interact with Dobby (Fireworks AI) to power the AI Teacher experience: strict-JSON answers, language control, never-ending follow-up questions, and robust parsing in production.

## Endpoints and Model
- HTTP Endpoint: `https://api.fireworks.ai/inference/v1/chat/completions`
- Model: `accounts/sentientfoundation-serverless/models/dobby-mini-unhinged-plus-llama-3-1-8b`
- Auth: Bearer token via `Authorization: Bearer <YOUR_KEY>`

## JSON Contracts
Always request STRICT JSON. We use two shapes.

- Teacher (interactive lesson):
```json
{
  "answer": "string",
  "follow_up_questions": [
    { "id": "string", "label": "string" }
  ]
}
```

- Strategies (DeFi helper):
```json
{
  "summary": "string",
  "strategies": {
    "conservative": { "title": "", "description": "", "riskLevel": "", "expectedAPY": "", "links": [""], "guide": "" },
    "balanced": { "title": "", "description": "", "riskLevel": "", "expectedAPY": "", "links": [""], "guide": "" },
    "risky": { "title": "", "description": "", "riskLevel": "", "expectedAPY": "", "links": [""], "guide": "" }
  }
}
```

## Prompt Design (Patterns that work reliably)

- Enforce language strictly in the System prompt
  - Include: “Respond STRICTLY in <lang>. No markdown or extra text.”
- Restate schema in the System prompt
  - Paste the TypeScript-like type the model must follow.
- Keep User prompt concise and structured
  - For Teacher: include `Topic`, `Path` context, and optionally `Previous answer` (shortened) to maintain coherence.
- Never-ending follow-ups
  - Instruct the model to always return 3–5 follow_up_questions. If a branch is exhausted, propose adjacent subtopics.
- Keep answers short
  - Ask for ≤ 120 words to improve consistency and speed.

Example System (Teacher):
```text
You are an AI crypto teacher for interactive lessons.
Respond ONLY with valid minified JSON matching this TypeScript type:
type FollowUp = { id: string; label: string }
type TeacherResponse = { answer: string; follow_up_questions: FollowUp[] }
Language STRICTLY: en.
No markdown or extra text.
ALWAYS return 3-5 follow_up_questions related to the current topic or suggest new adjacent topics when the current branch is exhausted.
```

Example User (Teacher):
```text
Topic: What is blockchain?.
Path: root
Generate an explanation "answer" and 3-5 "follow_up_questions" to continue the dialogue.
If there is nothing else to ask about this branch, propose other related subtopics so the learning never stops.
Each follow_up_questions item must have a stable id and a human-readable label.
Keep the answer short (<= 120 words).
```

## Code Examples

### TypeScript (browser/fetch)
```ts
const apiKey = import.meta.env.VITE_FIREWORKS_API_KEY
const API_URL = 'https://api.fireworks.ai/inference/v1/chat/completions'

const system = `You are an AI crypto teacher for interactive lessons.\n` +
`Respond ONLY with valid minified JSON matching this TypeScript type:\n` +
`type FollowUp = { id: string; label: string }\n` +
`type TeacherResponse = { answer: string; follow_up_questions: FollowUp[] }\n` +
`Language STRICTLY: en.\n` +
`No markdown or extra text.\n` +
`ALWAYS return 3-5 follow_up_questions related to the current topic or suggest new adjacent topics when the current branch is exhausted.`

const user = `Topic: What is blockchain?.\n` +
`Path: root\n` +
`Generate an explanation "answer" and 3-5 "follow_up_questions" to continue the dialogue.\n` +
`If there is nothing else to ask about this branch, propose other related subtopics so the learning never stops.\n` +
`Each follow_up_questions item must have a stable id and a human-readable label.\n` +
`Keep the answer short (<= 120 words).`

const response = await fetch(API_URL, {
  method: 'POST',
  headers: {
    'Accept': 'application/json',
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${apiKey}`
  },
  body: JSON.stringify({
    model: 'accounts/sentientfoundation-serverless/models/dobby-mini-unhinged-plus-llama-3-1-8b',
    max_tokens: 1024,
    temperature: 0.7,
    messages: [
      { role: 'system', content: system },
      { role: 'user', content: user }
    ]
  })
})
const json = await response.json()
const raw = json.choices?.[0]?.message?.content ?? ''
// See the safeExtractJson utility below
const cleaned = safeExtractJson(raw)
const data = JSON.parse(cleaned) as { answer: string; follow_up_questions: Array<{ id: string; label: string }> }
```

### cURL
```bash
curl 'https://api.fireworks.ai/inference/v1/chat/completions' \
  -H 'accept: application/json' \
  -H 'authorization: Bearer YOUR_KEY' \
  -H 'content-type: application/json' \
  --data-raw '{
  "model":"accounts/sentientfoundation-serverless/models/dobby-mini-unhinged-plus-llama-3-1-8b",
  "max_tokens":1024,
  "temperature":0.7,
  "messages":[
    {"role":"system","content":"You are an AI crypto teacher for interactive lessons.\nRespond ONLY with valid minified JSON matching this TypeScript type:\ntype FollowUp = { id: string; label: string }\ntype TeacherResponse = { answer: string; follow_up_questions: FollowUp[] }\nLanguage STRICTLY: en.\nNo markdown or extra text.\nALWAYS return 3-5 follow_up_questions related to the current topic or suggest new adjacent topics when the current branch is exhausted."},
    {"role":"user","content":"Topic: What is blockchain?.\nPath: root\nGenerate an explanation \"answer\" and 3-5 \"follow_up_questions\" to continue the dialogue.\nIf there is nothing else to ask about this branch, propose other related subtopics so the learning never stops.\nEach follow_up_questions item must have a stable id and a human-readable label.\nKeep the answer short (<= 120 words)."}
  ]
}'
```

### Node (axios)
```ts
import axios from 'axios'

const client = axios.create({
  baseURL: 'https://api.fireworks.ai/inference/v1',
  headers: { Authorization: `Bearer ${process.env.FIREWORKS_KEY}` }
})

const { data } = await client.post('/chat/completions', {
  model: 'accounts/sentientfoundation-serverless/models/dobby-mini-unhinged-plus-llama-3-1-8b',
  max_tokens: 1024,
  temperature: 0.7,
  messages: [
    { role: 'system', content: '...system prompt here...' },
    { role: 'user', content: '...user prompt here...' }
  ]
})
const raw = data.choices?.[0]?.message?.content ?? ''
const cleaned = safeExtractJson(raw)
const parsed = JSON.parse(cleaned)
```

## Robust JSON Parsing
Models sometimes wrap JSON in ```json code fences or add extra text. Use a robust extractor before parsing.

```ts
export const safeExtractJson = (raw: unknown): string => {
  const text = String(raw ?? '')
  const fenceMatch = text.match(/```json\s*([\s\S]*?)\s*```/i)
  if (fenceMatch && fenceMatch[1]) return fenceMatch[1].trim()
  const start = text.indexOf('{')
  const end = text.lastIndexOf('}')
  if (start !== -1 && end !== -1 && end > start) return text.slice(start, end + 1)
  return text
}
```

We’ve integrated this in `src/services/dobby.ts` for both Teacher and Strategies.

## Language Control
- Always pass the UI language to the System prompt (`Language STRICTLY: <code>`)
- In a router/app, wire your i18n language into each request
- For inputs (initial topic), use language-specific defaults (see `getDefaultTopic` in `Lesson.tsx`)

## Never-Ending Follow-ups
- Instruct the model to return 3–5 follow_up_questions on every turn
- If there’s nothing left in the current branch, propose adjacent topics
- Add a local fallback when the model returns an empty list
  - We localize fallback labels by language in `fetchDobbyTeacher`

## Parameter Tuning
- `temperature: 0.6–0.8` — balance creativity and determinism
- `max_tokens: 1024–2048` — depends on answer length and follow-ups
- `top_p: 1` and `top_k: 40` — keep defaults unless you have specific needs

## Common Bugs We Hit (and Fixes)
1) Model returns fenced JSON with extra prose after the block
- Symptom: `JSON.parse` errors intermittently
- Fix: `safeExtractJson` to extract fenced JSON or the first `{...}` block

2) Language drift (answers in the wrong language)
- Symptom: User sets EN but response arrives in RU
- Fix: Strengthen System prompt (“Language STRICTLY: <lang>”); pass `i18n.language` into every request; set language-sensitive default topic

3) Empty `follow_up_questions`
- Symptom: UI gets stuck with no follow-ups
- Fix: Instruct “ALWAYS return 3–5”; add localized fallback list client-side

4) Inconsistent follow-up IDs
- Symptom: Buttons lose identity between turns
- Fix: Ask for stable IDs in the prompt; derive path from labels; treat IDs as opaque but prefer stable label-based path

5) Occasional non-OK HTTP responses
- Symptom: 4xx/5xx
- Fix: check `response.ok` and throw; display user-friendly error; optionally add retry with backoff for 5xx

6) Unused imports breaking CI/build
- Symptom: TypeScript build fails (TS6133)
- Fix: remove unused imports (`useMemo` in `Lesson.tsx`)

## Developer Guidelines
- DO
  - Restate the JSON schema in the System prompt
  - Enforce language strictly and pass the runtime language
  - Keep answers short and actionable
  - Provide path context and optionally the previous answer (truncated)
  - Use `safeExtractJson` before parsing
  - Add client-side fallbacks for follow-ups
  - Log raw responses in non-PII logs for diagnostics (avoid tokens)

- DON’T
  - Don’t trust the model to always return clean JSON without fences
  - Don’t parse with regex-only; always use a two-step extraction then JSON.parse
  - Don’t rely on one-off prompts; keep prompts consistent per feature
  - Don’t expose API keys in public builds (use environment variables)

## Testing Tips
- Unit-test `safeExtractJson` with cases:
  - plain JSON, fenced ```json, extra text after, extra text before, malformed fences
- Mock the endpoint in integration tests
- Snapshot-test the UI rendering of `answer` and follow-up buttons

## Troubleshooting
- “Unexpected token … in JSON at position …”
  - Inspect raw response; likely fenced JSON or trailing prose; use extractor
- Wrong language output
  - Verify system prompt has `Language STRICTLY: <code>` and the code matches your locale
- Empty follow-ups
  - Ensure the instruction to always return 3–5; confirm fallback logic executes

---
Maintainer notes:
- The reference implementation lives in `src/services/dobby.ts` and `src/pages/Lesson.tsx`.
- Free chat forces language via system prompt: see `src/pages/ChatFree.tsx`. 
