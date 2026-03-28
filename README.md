# 🛍️ AI Store Associate

> A conversational shopping assistant that understands natural language and recommends products with personalized reasoning — like having a knowledgeable store associate available 24/7.

![Demo](https://img.shields.io/badge/status-hackathon%20MVP-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue)
![Built with](https://img.shields.io/badge/built%20with-Gemini%20API-blue)

---

## 📌 The Problem

Traditional e-commerce search is broken for discovery. Filters and keyword search force customers to already know what they want. Millions of shoppers abandon carts because they can't find the right product — not because it doesn't exist.

**AI Store Associate** solves this by letting customers describe their need conversationally:

> *"I need a gift for my dad who loves Italian cooking, under $50"*

...and getting back 3 relevant product recommendations, each with a personalized reason why it fits.

---

## ✨ Features

- **Natural language shopping** — describe what you need, who it's for, your budget, any constraints
- **Clarifying questions** — the AI asks one smart follow-up when it needs more context, just like a real associate
- **Personalized reasoning** — every recommendation explains *why* it fits this specific customer's request
- **Refinement loop** — follow up with "show me cheaper options" or "actually he's vegetarian" and the results update
- **Catalog-grounded** — the AI only recommends real products from your inventory, no hallucinated items

---

## 🎥 Demo

| Customer says | AI recommends |
|---|---|
| "Gift for dad, loves Italian cooking, under $50" | Pasta maker attachment, cast iron grill pan, olive oil tasting set — each with a tailored reason |
| "Running shoes for someone with flat feet, mostly road running" | Stability shoes matched to pronation type and surface |
| "Something fun for a 6-year-old who likes dinosaurs" | Age-appropriate picks filtered to the toy catalog |

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
│           Gemini API (gemini-2.0-flash)      │
│  System prompt contains:                    │
│  • Store persona                            │
│  • Full product catalog (JSON)              │
│  • Response format instructions             │
└─────────────────────────────────────────────┘
```

**For production scale**, replace the in-prompt catalog with vector embeddings (e.g. Vertex AI Vector Search, pgvector) to support catalogs of 10,000+ products via semantic search before passing candidates to Gemini.

---

## 🚀 Quick Start

### Prerequisites

- Node.js 18+
- A [Google AI Studio API key](https://aistudio.google.com/app/apikey)

### Installation

```bash
git clone https://github.com/your-username/ai-store-associate
cd AiStoreAssociate
npm install
```

### Configuration

```bash
cp .env.example .env
```

Edit `.env`:

```env
GEMINI_API_KEY=your_api_key_here
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
│   │   └── gemini.js             # Gemini API wrapper + system prompt builder
│   └── App.jsx
├── .env.example
├── package.json
└── README.md
```

---

## 🧠 System Prompt Design

The core of the project is the system prompt. Here's the pattern:

```js
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

const buildSystemPrompt = (catalog) => `
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

export async function getRecommendations(catalog, conversationHistory) {
  const model = genAI.getGenerativeModel({
    model: "gemini-2.0-flash",
    systemInstruction: buildSystemPrompt(catalog),
  });

  const chat = model.startChat({ history: conversationHistory });
  const lastMessage = conversationHistory.at(-1).parts[0].text;
  const result = await chat.sendMessage(lastMessage);
  return result.response.text();
}
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React + Tailwind CSS |
| AI | Gemini API (`gemini-2.0-flash`) |
| AI SDK | `@google/generative-ai` |
| Product data | Static JSON (swap for any DB or API) |
| Deployment | Vercel / Netlify |

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
- **Multi-modal** — let customers upload a photo ("find me something like this") using Gemini's native vision capabilities
- **A/B testing** — log which recommendations lead to add-to-cart events and fine-tune the prompt accordingly

---

## 🏆 Hackathon Notes

This project was built as an MVP for a Google AI hackathon. The goal was to demonstrate that conversational, intent-aware product discovery is both feasible and significantly better than keyword search — using only a well-crafted system prompt and a static product catalog.

**Key insight:** The quality of the *reasoning* shown to the customer (not just the product match) is what drives trust and conversion. Gemini's ability to tie recommendations back to specific things the customer said is the core differentiator.

---

## 📄 License

MIT — see [LICENSE](LICENSE) for details.

---

## 🙏 Acknowledgements

Built with the [Google Gemini API](https://ai.google.dev). Inspired by the gap between how people naturally shop in stores vs. how they're forced to search online.
