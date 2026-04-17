# LE 09 — Memory-Systeme
> Lernblock 2: Kernkonzepte · 1 Stunde

---

## 1. Das Grundproblem (10 min)

```
Claude hat kein persistentes Gedächtnis.

Session A: "Mein Name ist Ramon."
Session B: "Wie heisse ich?" → Claude: "Das weiss ich nicht."

Die Weights sind READ-ONLY (eingefroren nach Training).
Context Window ist temporär — endet mit der Session.

Lösung: externe Memory-Systeme, die Informationen LADEN und SPEICHERN.
```

---

## 2. Die 4 Memory-Arten im Vergleich (25 min)

```
┌────────────────┬──────────────────────────────┬──────────────┬─────────┐
│ Art            │ Wie es funktioniert          │ Persistenz   │ Kosten  │
├────────────────┼──────────────────────────────┼──────────────┼─────────┤
│ In-Context     │ Text direkt im Context       │ Session      │ hoch    │
│                │ Window                       │              │         │
├────────────────┼──────────────────────────────┼──────────────┼─────────┤
│ File-based     │ Dateien lesen/schreiben,     │ permanent    │ niedrig │
│                │ beim Start laden             │              │         │
├────────────────┼──────────────────────────────┼──────────────┼─────────┤
│ Vector (RAG)   │ Embeddings + Ähnlichkeits-   │ permanent    │ mittel  │
│                │ suche, nur Relevantes laden  │              │         │
├────────────────┼──────────────────────────────┼──────────────┼─────────┤
│ Fine-tuning    │ Ins Modell eintrainiert      │ permanent    │ sehr    │
│                │ (neue Weights)               │              │ hoch    │
└────────────────┴──────────────────────────────┴──────────────┴─────────┘
```

### 2a. In-Context Memory

```python
# Einfachste Form: Verlauf mitschicken
messages = []

def chat(user_input: str) -> str:
    messages.append({"role": "user", "content": user_input})
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=messages
    )
    answer = response.content[0].text
    messages.append({"role": "assistant", "content": answer})
    return answer

chat("Ich analysiere das UserService-Modul.")
chat("Welche Methode soll ich als nächstes anschauen?")
# Claude "erinnert" sich an UserService — weil es im Context steht

# PROBLEM: Nach 100 Nachrichten = 50.000+ Tokens Input
# LÖSUNG: Summarization
```

### In-Context mit automatischer Zusammenfassung

```python
MAX_MESSAGES = 20

def chat_with_summary(user_input: str) -> str:
    messages.append({"role": "user", "content": user_input})
    
    # Bei zu vielen Nachrichten: zusammenfassen
    if len(messages) > MAX_MESSAGES:
        summary = summarize_conversation(messages[:-5])
        messages.clear()
        messages.append({
            "role": "user",
            "content": f"[Zusammenfassung bisheriges Gespräch]: {summary}"
        })
        messages.append({"role": "assistant", "content": "Verstanden."})
    
    # ... rest wie oben

def summarize_conversation(msgs: list) -> str:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Haiku für Zusammenfassung = günstig
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": f"Fasse die wichtigsten Punkte dieses Gesprächs zusammen:\n\n{str(msgs)}"
        }]
    )
    return response.content[0].text
```

### 2b. File-based Memory (wie dieses Projekt)

```
Konzept:
  1. Claude schreibt wichtige Fakten in Dateien
  2. Beim nächsten Start: Dateien lesen → in Context laden
  3. Claude "erinnert" sich durch das Laden

Dieses Projekt macht genau das:
  ~/.claude/projects/.../memory/MEMORY.md    ← Index
  ~/.claude/projects/.../memory/user_profile.md
  ~/.claude/projects/.../memory/feedback_*.md

Stärken:
  ✓ Kein API-Overhead für Memory
  ✓ Menschlich lesbar und editierbar
  ✓ Versionierbar mit Git

Schwächen:
  ✗ Skaliert nicht gut (>100 Fakten = Context zu gross)
  ✗ Kein semantisches Retrieval
```

### 2c. Vector Memory (RAG für Konversationen)

```python
# Konversations-Erinnerungen mit Embeddings speichern

class VectorMemory:
    def __init__(self):
        self.collection = chroma.create_collection("memory")
    
    def remember(self, fact: str, metadata: dict = {}):
        """Speichert einen Fakt."""
        import uuid
        self.collection.add(
            documents=[fact],
            ids=[str(uuid.uuid4())],
            metadatas=[metadata]
        )
    
    def recall(self, query: str, k: int = 5) -> list[str]:
        """Erinnert sich an die k relevantesten Fakten."""
        results = self.collection.query(
            query_texts=[query],
            n_results=k
        )
        return results["documents"][0]
    
    def get_context_for(self, question: str) -> str:
        memories = self.recall(question)
        return "Relevante Erinnerungen:\n" + "\n".join(f"- {m}" for m in memories)

# Nutzung:
memory = VectorMemory()
memory.remember("Der UserService hat einen Bug in logout() - NullPointerException")
memory.remember("Ramon bevorzugt Java 17 Features")
memory.remember("Das Projekt nutzt Spring Boot 3.2")

context = memory.get_context_for("Wie soll ich die Auth-Klassen schreiben?")
# → "Relevante Erinnerungen:
#    - Das Projekt nutzt Spring Boot 3.2
#    - Ramon bevorzugt Java 17 Features"
```

### 2d. Fine-tuning — wann sinnvoll?

```
Fine-tuning = Neue Gewichte durch weiteres Training

SINNVOLL wenn:
  ✓ Konsistenter Stil/Format (z.B. firmeninternes Template)
  ✓ Spezialisiertes Fachvokabular (sehr spezifisch, viel Wiederholung)
  ✓ Tausende von Beispielen vorhanden
  ✓ Kosten durch kürzere Prompts sparen (gross Volumen)

NICHT sinnvoll wenn:
  ✗ "Claude soll meine Docs kennen" → RAG besser
  ✗ "Claude soll Fakten lernen" → RAG besser
  ✗ Wenige Beispiele (<100) → Prompt Engineering reicht
  ✗ Daten ändern sich häufig → Modell müsste ständig neu trainiert werden

Faustregel:
  Prompt Engineering → klappt nicht? → RAG → klappt nicht? → Fine-tuning
```

---

## 3. Memory in der Praxis — Entscheidungsbaum (15 min)

```
Frage: Muss Claude sich etwas "merken"?
  │
  ▼
Handelt es sich um die AKTUELLE SESSION?
  │ Ja  → In-Context Memory (Messages-Array)
  │ Nein ↓
  │
Wie VIELE Informationen müssen gespeichert werden?
  │ Wenige (< 50 Fakten) → File-based Memory
  │ Viele (> 50, skalierbar) ↓
  │
Brauche ich SEMANTISCHE SUCHE?
  │ Ja  → Vector Memory (RAG)
  │ Nein → Strukturierte DB (SQL/JSON) + Tool Use
  │
Ändert sich das Wissen HÄUFIG?
  │ Ja  → RAG (aktualisierbar)
  │ Nein + konsistenter Stil + grosses Volumen → Fine-tuning
```

### Kombination in der Praxis

```
PRODUKTIONSSYSTEM: Support-Bot für Java-Framework

In-Context:   aktuelles Gespräch (letzte 20 Nachrichten)
File-based:   Nutzer-Präferenzen, Team-Konventionen
Vector (RAG): gesamte Framework-Dokumentation (10.000+ Seiten)
Fine-tuned:   firmenspezifischer Antwortstil, Template-Format

→ Jede Ebene löst ein anderes Problem
→ Zusammen = vollständiges Memory-System
```

---

## 4. Memory in Claude Code (10 min)

Claude Code nutzt genau dieses System:

```
Beim Start lädt Claude Code:
  1. CLAUDE.md (projekt-spezifische Regeln → In-Context)
  2. ~/.claude/projects/.../memory/MEMORY.md (persistent Facts → In-Context)
  3. Bisherige Tasks/Plans (falls vorhanden → In-Context)

Während der Session:
  4. Dateien werden on-demand gelesen (File Tool → temporär im Context)

Am Ende der Session:
  5. Neue Fakten werden in memory/*.md Dateien geschrieben
  6. Nächste Session: Schritt 1-3 wiederholen
```

---

## Zusammenfassung LE 09

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Memory                                   │
│                                                         │
│  4 Arten: In-Context, File, Vector, Fine-tuning         │
│                                                         │
│  Wahl nach:                                             │
│    Dauer (Session vs. permanent)                        │
│    Volumen (wenige vs. viele Fakten)                    │
│    Dynamik (ändert sich vs. statisch)                   │
│    Budget (günstig vs. teuer)                           │
│                                                         │
│  Goldene Regel:                                         │
│  Immer RAG vor Fine-tuning. Immer Prompt Engineering    │
│  vor RAG. In-Context ist am einfachsten — wenn es       │
│  reicht.                                                │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 08 — RAG & Embeddings](le14_rag_embeddings.md)*
*Weiter: [LE 10 — Halluzinationen & Safety](le11_halluzinationen_safety.md)*
