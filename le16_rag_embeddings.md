# LE 08 — RAG & Embeddings
> Lernblock 2: Kernkonzepte · 1 Stunde

---

## 1. Das Problem: Knowledge Cutoff & Private Daten (10 min)

```
Claude's Wissen endet Aug 2025.
Deine Codebasis kennt Claude nicht.
Deine internen Docs kennt Claude nicht.

Lösung A: Fine-Tuning       → teuer, langsam, starr
Lösung B: Alles in Context  → zu gross, zu teuer
Lösung C: RAG               → ✓ flexibel, aktuell, skalierbar
```

**RAG = Retrieval-Augmented Generation**
Relevante Dokumente werden **zur Laufzeit** in den Context geladen — nicht im Training eingebettet.

---

## 2. Embeddings verstehen (20 min)

### Was ist ein Embedding?

```
Embedding = Text → Zahlenvektor (hochdimensional)

"UserService.java"  → [0.23, -0.41, 0.87, 0.12, ...]  (1536 Werte)
"CustomerService"   → [0.24, -0.39, 0.85, 0.14, ...]  (ähnlich!)
"Bestellverwaltung" → [0.21, -0.38, 0.82, 0.11, ...]  (auch ähnlich)
"Datenbankschema"   → [0.89,  0.12, -0.3, 0.67, ...]  (anders)

Semantische Ähnlichkeit ≠ Zeichenähnlichkeit

"UserService" und "CustomerService" klingen ähnlich → nahe im Raum
"login()" und "authenticate()" klingen verschieden → trotzdem nahe!
```

### Cosine-Similarity

```
Wie ähnlich sind zwei Vektoren?

cos(θ) = (A · B) / (|A| × |B|)

Wert zwischen -1 und 1:
  1.0  = identisch
  0.9  = sehr ähnlich
  0.5  = mittel ähnlich
  0.0  = kein Zusammenhang
 -1.0  = Gegenteil

Praxis: alles > 0.8 = relevant für RAG
```

### Embeddings in Python

```python
import anthropic

client = anthropic.Anthropic()

def get_embedding(text: str) -> list[float]:
    response = client.embeddings.create(
        model="voyage-3",   # Anthropic's Embedding-Modell
        input=[text]
    )
    return response.embeddings[0].embedding

# Ähnlichkeit berechnen
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# Test:
emb1 = get_embedding("UserService handles authentication")
emb2 = get_embedding("login and user verification")
emb3 = get_embedding("database connection pool size")

print(cosine_similarity(emb1, emb2))  # → ~0.87 (ähnlich)
print(cosine_similarity(emb1, emb3))  # → ~0.31 (verschieden)
```

---

## 3. RAG-Pipeline bauen (20 min)

### Architektur

```
┌─────────────────────────────────────────────────────────────┐
│                     RAG PIPELINE                            │
│                                                             │
│  OFFLINE (einmalig, beim Start):                            │
│                                                             │
│  Deine Docs/Code                                            │
│       │                                                     │
│       ▼ Chunking                                            │
│  [Chunk 1] [Chunk 2] [Chunk 3] ... [Chunk N]               │
│       │                                                     │
│       ▼ Embedding (Voyage / text-embedding-3)               │
│  [Vec 1]  [Vec 2]  [Vec 3]  ... [Vec N]                    │
│       │                                                     │
│       ▼ Speichern                                           │
│  Vector-DB (ChromaDB / Pinecone / pgvector)                 │
│                                                             │
│  ONLINE (pro Anfrage):                                      │
│                                                             │
│  User-Frage                                                 │
│       │                                                     │
│       ▼ Embedding                                           │
│  Query-Vektor                                               │
│       │                                                     │
│       ▼ Similarity-Search in Vector-DB                      │
│  Top-K relevante Chunks (z.B. K=5)                         │
│       │                                                     │
│       ▼ Prompt-Konstruktion                                 │
│  System: "Beantworte nur anhand dieser Docs: [Chunk 1-5]"  │
│  User:   "Wie funktioniert die Auth?"                       │
│       │                                                     │
│       ▼                                                     │
│  Claude → Antwort (grounded, keine Halluzination)          │
└─────────────────────────────────────────────────────────────┘
```

### Einfaches RAG mit ChromaDB

```python
# pip install chromadb anthropic

import chromadb
import anthropic

chroma = chromadb.Client()
collection = chroma.create_collection("java_docs")
claude = anthropic.Anthropic()

# ── Dokumente einlesen & speichern ──────────────────────
def index_documents(docs: list[dict]):
    """docs = [{"id": "...", "text": "...", "metadata": {...}}]"""
    texts = [d["text"] for d in docs]
    ids   = [d["id"]   for d in docs]
    
    collection.add(
        documents=texts,
        ids=ids
    )
    # ChromaDB berechnet Embeddings automatisch (oder du gibst sie mit)

# ── Suchen & Antworten ──────────────────────────────────
def rag_query(question: str, k: int = 5) -> str:
    # Ähnlichste Chunks finden
    results = collection.query(
        query_texts=[question],
        n_results=k
    )
    
    chunks = results["documents"][0]
    context = "\n\n---\n\n".join(chunks)
    
    # Claude mit Kontext aufrufen
    response = claude.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system=f"""Du bist ein hilfreicher Assistent für Java-Entwickler.
Beantworte Fragen NUR basierend auf dem bereitgestellten Kontext.
Wenn die Antwort nicht im Kontext steht, sage das klar.

Kontext:
{context}""",
        messages=[{"role": "user", "content": question}]
    )
    
    return response.content[0].text

# ── Beispiel ─────────────────────────────────────────────
# Javadoc-Kommentare aller Klassen indexieren:
index_documents([
    {"id": "UserService", "text": "UserService: Verwaltet User-Authentifizierung. login(String, String), logout(String), resetPassword(String)..."},
    {"id": "OrderService", "text": "OrderService: Bestellverwaltung. createOrder(Cart), cancelOrder(Long), getOrderHistory(Long userId)..."},
])

answer = rag_query("Wie logge ich einen User aus?")
print(answer)
# → "Mit UserService.logout(String userId) lässt sich ein User abmelden."
```

---

## 4. Chunking-Strategien (10 min)

Das häufigste RAG-Problem: schlechte Chunks.

```
STRATEGIE 1: Fixed-Size (simpel, oft schlecht)
  chunk_size = 500 Zeichen, overlap = 50
  Problem: schneidet mitten durch Methoden/Sätze

STRATEGIE 2: Semantic Chunking (für Docs)
  Jeder Paragraph = ein Chunk
  Jeder Javadoc-Block = ein Chunk
  → viel besser für Code-Dokumentation

STRATEGIE 3: Hierarchisch (für grosse Codebases)
  Chunk 1 (grob):  "UserService — Klasse für Auth"
  Chunk 2 (fein):  "UserService.login() — prüft credentials"
  Chunk 3 (fein):  "UserService.logout() — invalidiert session"
  → erst Klasse finden, dann Methode

Für Java-Projekte empfohlen:
  → Ein Chunk pro public Methode + Javadoc
  → Metadaten: Klasse, Package, Zeile
```

---

## Zusammenfassung LE 08

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: RAG                                      │
│                                                         │
│  Embedding = Text zu Zahlenvektor                       │
│  Ähnliche Texte → ähnliche Vektoren → Cosine > 0.8     │
│                                                         │
│  RAG-Pipeline:                                          │
│  Docs → Chunks → Embeddings → Vector-DB → Suche        │
│  → Top-K Chunks → Prompt → Claude → Antwort            │
│                                                         │
│  Chunking ist der kritischste Schritt:                  │
│  Für Java: ein Chunk pro Methode/Klasse mit Javadoc     │
│                                                         │
│  RAG löst: Knowledge Cutoff & private Codebases        │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 07 — Agent Loop & ReAct](le13_agent_loop_react.md)*
*Weiter: [LE 09 — Memory-Systeme](le15_memory_systeme.md)*
