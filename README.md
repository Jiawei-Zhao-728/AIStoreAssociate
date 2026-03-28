# 🛍️ AI Store Associate

> A conversational shopping assistant that understands natural language and recommends products with personalized reasoning — like having a knowledgeable store associate available 24/7.

![Demo](https://img.shields.io/badge/status-hackathon%20MVP-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue)
![Built with](https://img.shields.io/badge/built%20with-Vertex%20AI-orange)

---

## 📌 The Problem

Traditional e-commerce search is broken for discovery. Filters and keyword search force customers to already know what they want. Millions of shoppers abandon carts because they can't find the right product — not because it doesn't exist.

**AI Store Associate** solves this by letting customers describe their need conversationally:

> _"I need a gift for my dad who loves Italian cooking, under $50"_

...and getting back 3 relevant product recommendations, each with a personalized reason why it fits.

---

## ✨ Features

- **Natural language shopping** — describe what you need, who it's for, your budget, any constraints
- **Clarifying questions** — the AI asks one smart follow-up when it needs more context, just like a real associate
- **Personalized reasoning** — every recommendation explains _why_ it fits this specific customer's request
- **Refinement loop** — follow up with "show me cheaper options" or "actually he's vegetarian" and the results update
- **Catalog-grounded** — the AI only recommends real products from your inventory, no hallucinated items

---

## 🎥 Demo

| Customer says                                                   | AI recommends                                                                                    |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| "Gift for dad, loves Italian cooking, under $50"                | Pasta maker attachment, cast iron grill pan, olive oil tasting set — each with a tailored reason |
| "Running shoes for someone with flat feet, mostly road running" | Stability shoes matched to pronation type and surface                                            |
| "Something fun for a 6-year-old who likes dinosaurs"            | Age-appropriate picks filtered to the toy catalog                                                |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│                 Frontend (React)             │
│  ┌──────────────┐    ┌─────────────────────┐ │
│  │  Chat pane   │    │  Product card grid  │ │
│  │  (input +    │    │  (recommendations + │ │
│  │   history)   │    │   reasoning badges) │ │
│  └──────┬───────┘    └──────────▲──────────┘ │
└─────────┼──────────────────────┼─────────────┘
          │                      │
          ▼                      │
┌─────────────────────────────────────────────┐
│        Backend API (Cloud Run, Node)         │
│  • Loads catalog (JSON)                      │
│  • Builds system prompt                      │
│  • Calls Vertex AI                           │
│  • Validates JSON and grounds to catalog     │
└─────────┬────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────┐
│        Vertex AI (Gemini/Claude on GCP)      │
│  Models via Model Garden (e.g.,              │
│  publishers/anthropic/claude-3.5-sonnet      │
│  or gemini-1.5-pro)                          │
└─────────────────────────────────────────────┘
```

**For production scale**, replace the in-prompt catalog with embeddings-based retrieval on Google Cloud (Vertex AI Matching Engine or Cloud SQL/Postgres + pgvector). Retrieve top‑k candidates server‑side before calling Vertex AI.

---

## 🚀 Quick Start

### Prerequisites

- Node.js 18+
- A Google Cloud project with billing/credits
- gcloud CLI installed and authenticated
- Vertex AI and Secret Manager APIs enabled

### Installation

```bash
git clone https://github.com/your-username/ai-store-associate
cd ai-store-associate
npm install
```

### Configuration

```bash
cp .env.example .env
```

Edit `.env` for local development (server uses ADC or a service account in production):

```env
GCP_PROJECT_ID=your_project_id
GCP_LOCATION=us-central1
# Choose one model:
VERTEX_MODEL_ID=gemini-1.5-pro
# or: VERTEX_MODEL_ID=publishers/anthropic/models/claude-3.5-sonnet

# Optional for local dev with a service account:
# GOOGLE_APPLICATION_CREDENTIALS=/absolute/path/to/service-account.json
```

### Run

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

---

## 📦 Project Structure

```
ai-store-associate/
├── src/
│   ├── components/
│   │   ├── ChatPane.jsx          # Conversational input + message history
│   │   ├── ProductCard.jsx       # Recommendation card with reasoning badge
│   │   └── ProductGrid.jsx       # Layout for recommendation results
│   ├── data/
│   │   └── catalog.json          # Product inventory (name, price, tags, description)
│   ├── lib/
│   │   └── vertex.js             # Vertex AI client + system prompt builder
│   ├── server/
│   │   └── index.js              # Cloud Run API (catalog load, call Vertex, validate/ground)
│   └── App.jsx
├── .env.example
├── package.json
└── README.md
```

---

## 🧠 System Prompt Design

The core of the project is the system prompt. Here's the pattern:

```js
import { VertexAI } from "@google-cloud/vertexai";

const vertex = new VertexAI({
  project: process.env.GCP_PROJECT_ID,
  location: process.env.GCP_LOCATION || "us-central1",
});

export const buildSystemPrompt = (catalog) => `
You are a helpful and knowledgeable store associate for [Store Name].

Your product catalog is:
${JSON.stringify(catalog, null, 2)}

When a customer describes what they need:
1. Ask ONE clarifying question if the request is ambiguous.
2. Otherwise, recommend exactly 3 products from the catalog above.
3. For each recommendation, include:
   - Product name
   - Price
   - A 1–2 sentence reason that ties back to something specific the customer said.

Rules:
- Only recommend products that exist in the catalog above.
- Never invent product details, features, or prices.
- If nothing in the catalog fits, say so honestly and ask what else might help.
- Keep your tone friendly and concise.

Respond in this JSON format:
{
  "clarifying_question": null,
  "recommendations": [
    { "name": "...", "price": "...", "reason": "..." }
  ]
}
`;

export async function getRecommendations({ catalog, history, userMessage }) {
  const model = vertex.getGenerativeModel({
    model: process.env.VERTEX_MODEL_ID || "gemini-1.5-pro",
    systemInstruction: buildSystemPrompt(catalog),
  });
  const contents = [
    ...(history || []),
    { role: "user", parts: [{ text: userMessage }] },
  ];
  const resp = await model.generateContent({ contents });
  const text = resp.response.candidates?.[0]?.content?.parts?.[0]?.text || "";
  return text;
}
```

---

## 🛠️ Tech Stack

| Layer          | Technology                                              |
| -------------- | ------------------------------------------------------- |
| Frontend       | React + Tailwind CSS                                    |
| AI             | Vertex AI (Gemini or Anthropic Claude via Model Garden) |
| AI SDK         | `@google-cloud/vertexai`                                |
| Product data   | Static JSON → Cloud SQL/AlloyDB or Matching Engine      |
| Hosting/Deploy | Firebase Hosting (frontend) + Cloud Run (API)           |

---

## 📋 Sample Catalog Format

```json
[
  {
    "id": "001",
    "name": "Pasta Maker Attachment",
    "price": 44.99,
    "category": "kitchen",
    "tags": ["cooking", "italian", "gift", "equipment", "pasta"],
    "description": "KitchenAid-compatible pasta roller and cutter set. Makes fresh fettuccine, spaghetti, and lasagna sheets."
  },
  {
    "id": "002",
    "name": "Cast Iron Grill Pan",
    "price": 39.99,
    "category": "kitchen",
    "tags": ["cooking", "grilling", "italian", "equipment", "gift"],
    "description": "Pre-seasoned 10-inch cast iron grill pan. Perfect for proteins, vegetables, and paninis."
  }
]
```

---

## 🔮 Future Improvements

- **Vector search** — embed product descriptions with Google's `text-embedding-004` model and use Vertex AI Vector Search for semantic retrieval at scale
- **Inventory awareness** — surface stock levels and flag low-inventory items
- **Personalization** — store past purchases and preferences across sessions
- **Multi-modal** — let customers upload a photo ("find me something like this") using Gemini’s vision via Vertex AI
- **A/B testing** — log which recommendations lead to add-to-cart events and fine-tune the prompt accordingly

---

## 🏆 Hackathon Notes

This project was built as an MVP for a Google Cloud hackathon. The goal was to demonstrate that conversational, intent-aware product discovery is both feasible and significantly better than keyword search — using only a well-crafted system prompt and a static product catalog.

**Key insight:** The quality of the _reasoning_ shown to the customer (not just the product match) is what drives trust and conversion. Vertex AI models’ ability to tie recommendations back to specific things the customer said is the core differentiator.

---

## 📄 License

MIT — see [LICENSE](LICENSE) for details.

---

## 🙏 Acknowledgements

Built with Google Cloud Vertex AI. Inspired by the gap between how people naturally shop in stores vs. how they're forced to search online.
