# LE 18 — RAG vertieft: Chunking, Reranking, Evaluation
> Lernblock 4: Vertiefung · 1 Stunde

---

## 1. Chunking-Strategien im Detail (20 min)

### Warum Chunking kritisch ist

```
Schlechtes Chunking → schlechte Retrieval-Ergebnisse
→ Claude antwortet falsch OBWOHL die Antwort im Dokument steht!

Häufige Fehler:
  ✗ Zu grosse Chunks (zuviel irrelevanter Kontext)
  ✗ Zu kleine Chunks (kein Kontext für Verständnis)
  ✗ Mitten durch Methoden/Sätze schneiden
  ✗ Keine Metadaten (weiss nicht wo Chunk herkommt)
```

### Strategie 1: Recursive Character Splitting

```python
# Für Dokumentation und Prosa
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,        # max. Zeichen pro Chunk
    chunk_overlap=200,      # Überlappung für Kontext
    separators=["\n\n", "\n", ". ", " ", ""]
)

# Teilt bei: Absatz → Zeile → Satz → Wort → Zeichen
# Nie mitten im Wort!
chunks = splitter.split_text(documentation_text)
```

### Strategie 2: Semantic Chunking (für Code)

```python
import ast
import javalang  # pip install javalang

def chunk_java_file(file_path: str) -> list[dict]:
    """Zerlegt Java-Datei methodenweise."""
    with open(file_path) as f:
        source = f.read()
    
    chunks = []
    
    try:
        tree = javalang.parse.parse(source)
    except javalang.parser.JavaSyntaxError:
        # Fallback: zeilenbasiert
        return [{"id": file_path, "text": source, "type": "file"}]
    
    for _, node in tree.filter(javalang.tree.MethodDeclaration):
        # Methode als eigener Chunk
        chunks.append({
            "id": f"{file_path}#{node.name}",
            "text": f"""Klasse: {get_class_name(tree)}
Methode: {node.name}
Javadoc: {extract_javadoc(node)}
Signatuer: {get_signature(node)}
Body: {get_method_body(source, node)}""",
            "metadata": {
                "file": file_path,
                "method": node.name,
                "class": get_class_name(tree),
                "line": node.position.line if node.position else 0
            }
        })
    
    return chunks
```

### Strategie 3: Hierarchisches Chunking

```
Für grosse Codebases: Zwei-Ebenen-Indexierung

EBENE 1 (Grosse Chunks, für Groborientierung):
  Chunk: gesamte Klasse
  "UserService: Verwaltet User-Authentifizierung.
   Methoden: login, logout, register, resetPassword.
   Abhängigkeiten: UserRepository, EmailService."

EBENE 2 (Kleine Chunks, für Details):
  Chunk: einzelne Methode
  "UserService.login(String email, String password):
   Prüft credentials gegen DB. Gibt JWT zurück.
   Wirft AuthException bei falschen credentials."

Retrieval-Strategie:
  1. Finde relevante KLASSEN (Ebene 1)
  2. Innerhalb der Klassen: finde relevante METHODEN (Ebene 2)
  → Präzisere Treffer bei geringeren Kosten
```

---

## 2. Hybrid Search (15 min)

### Keyword + Semantic Search kombinieren

```
Nur Keyword-Suche:
  "UserService login" → findet nur exakte Treffer
  Problem: "authenticate" oder "signIn" werden nicht gefunden

Nur Semantic-Suche:
  "UserService login" → findet semantisch ähnliche Texte
  Problem: spezifische IDs, Dateinamen, Fehlercodes werden übersehen

Hybrid = beide kombinieren:

┌───────────────────────────────────────────────────┐
│  QUERY: "UserService login bug PROJ-456"           │
│                                                   │
│  Keyword-Suche:  → findet "PROJ-456", "login"     │
│  Semantic-Suche: → findet "authenticate", "auth"  │
│                                                   │
│  RRF-Fusion (Reciprocal Rank Fusion):             │
│  Beide Rankings kombinieren → bessere Ergebnisse  │
└───────────────────────────────────────────────────┘
```

```python
from rank_bm25 import BM25Okapi  # pip install rank-bm25

class HybridRetriever:
    def __init__(self, chunks: list[dict]):
        self.chunks = chunks
        
        # BM25 für Keyword-Suche
        tokenized = [c["text"].lower().split() for c in chunks]
        self.bm25 = BM25Okapi(tokenized)
        
        # Vector DB für Semantic-Suche
        self.vector_collection = setup_chroma(chunks)
    
    def search(self, query: str, k: int = 10) -> list[dict]:
        # Keyword-Suche (BM25)
        keyword_scores = self.bm25.get_scores(query.lower().split())
        keyword_ranking = sorted(
            range(len(self.chunks)),
            key=lambda i: keyword_scores[i],
            reverse=True
        )[:k]
        
        # Semantic-Suche
        semantic_results = self.vector_collection.query(
            query_texts=[query],
            n_results=k
        )
        semantic_ids = semantic_results["ids"][0]
        
        # RRF-Fusion
        scores = {}
        for rank, idx in enumerate(keyword_ranking):
            scores[idx] = scores.get(idx, 0) + 1 / (rank + 60)
        
        for rank, chunk_id in enumerate(semantic_ids):
            idx = next(i for i, c in enumerate(self.chunks) if c["id"] == chunk_id)
            scores[idx] = scores.get(idx, 0) + 1 / (rank + 60)
        
        # Top-K nach fusionierten Scores
        top_k = sorted(scores, key=scores.get, reverse=True)[:k]
        return [self.chunks[i] for i in top_k]
```

---

## 3. Reranking (10 min)

### Warum Reranking?

```
Retrieval (Embedding-Suche): schnell, aber ungenau
  → findet 20 "möglicherweise relevante" Chunks

Reranking: langsamer, aber präzise
  → bewertet die 20 Chunks gegen die Query
  → gibt Top-5 zurück die wirklich relevant sind

┌─────────────────────────────────────────────────────┐
│  Stage 1: Candidate Retrieval                       │
│    Embedding Search → 20 Kandidaten (schnell)       │
│                                                     │
│  Stage 2: Reranking                                 │
│    Cross-Encoder bewertet Query+Chunk → Score       │
│    Top-5 nach Score → an Claude                     │
└─────────────────────────────────────────────────────┘
```

```python
# Reranking mit Claude (LLM-basiert)
def rerank_with_claude(query: str, candidates: list[str], top_k: int = 5) -> list[int]:
    """Nutzt Claude um Kandidaten nach Relevanz zu sortieren."""
    
    candidates_text = "\n".join(
        f"[{i}] {c[:200]}..." for i, c in enumerate(candidates)
    )
    
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Haiku für Speed
        max_tokens=256,
        messages=[{
            "role": "user",
            "content": f"""Frage: {query}

Kandidaten:
{candidates_text}

Antworte NUR mit den Indizes der {top_k} relevantesten Kandidaten,
komma-getrennt, nach Relevanz sortiert.
Beispiel: 3,1,7,0,4"""
        }]
    )
    
    indices = [int(x.strip()) for x in response.content[0].text.split(",")]
    return indices[:top_k]
```

---

## 4. RAG Evaluation (15 min)

```python
# Die 3 wichtigsten RAG-Metriken

# 1. FAITHFULNESS: Antwortet Claude nur basierend auf den Chunks?
def check_faithfulness(answer: str, chunks: list[str]) -> float:
    """Prüft ob die Antwort durch die Chunks gestützt ist."""
    claims_in_answer = extract_claims(answer)  # Claude oder NLP
    supported = 0
    for claim in claims_in_answer:
        if any(claim_supported_by(claim, chunk) for chunk in chunks):
            supported += 1
    return supported / len(claims_in_answer) if claims_in_answer else 0

# 2. ANSWER RELEVANCE: Beantwortet es wirklich die Frage?
def check_relevance(question: str, answer: str) -> float:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=10,
        messages=[{
            "role": "user",
            "content": f"Bewertet diese Antwort die Frage (0-10)?:\nFrage: {question}\nAntwort: {answer}\nNur Zahl:"
        }]
    )
    return float(response.content[0].text.strip()) / 10

# 3. CONTEXT PRECISION: Wurden die richtigen Chunks gefunden?
# → Manuell oder mit Golden-Dataset prüfen

# Goldenes Test-Set für Java-Projekt:
test_cases = [
    {
        "question": "Wie logge ich einen User aus?",
        "expected_chunks_containing": ["UserService", "logout"],
        "expected_answer_contains": ["logout", "UserService"]
    },
    # ... mehr Test-Fälle
]
```

---

## Zusammenfassung LE 18

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: RAG vertieft                             │
│                                                         │
│  Chunking: Für Java → methodenbasiert, mit Metadaten    │
│  Hybrid Search: BM25 + Embedding + RRF-Fusion           │
│  Reranking: 20 Kandidaten → Top-5 (präziser)           │
│  Evaluation: Faithfulness, Relevance, Precision         │
│                                                         │
│  Die häufigste Ursache für schlechte RAG-Ergebnisse:    │
│  → Schlechtes Chunking. Immer hier zuerst ansetzen!     │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 17 — Multi-Agent vertieft](le17_multi_agent_vertieft.md)*
*Weiter: [LE 19 — Fine-tuning](le19_fine_tuning.md)*
