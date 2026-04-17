# LE 22 — Architektur-Muster: RAG + Memory + Multi-Agent
> Lernblock 4: Vertiefung · 1 Stunde

---

## 1. Das vollständige KI-System (15 min)

Einzelne Techniken kombiniert ergeben ein vollständiges System:

```
┌───────────────────────────────────────────────────────────────────┐
│                   VOLLSTÄNDIGES KI-SYSTEM                        │
│                                                                   │
│  USER REQUEST                                                     │
│       │                                                           │
│       ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  INPUT LAYER                                            │     │
│  │  - Input Validation (Länge, Injection-Schutz)          │     │
│  │  - User Authentication / Context                        │     │
│  └────────────────────────┬────────────────────────────────┘     │
│                           │                                       │
│       ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  CONTEXT ASSEMBLY                                       │     │
│  │  - Memory laden (User-Präferenzen, Session-History)     │     │
│  │  - RAG: relevante Docs/Code-Chunks retrieven            │     │
│  │  - System Prompt (gecacht)                              │     │
│  └────────────────────────┬────────────────────────────────┘     │
│                           │                                       │
│       ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  ORCHESTRATOR AGENT                                     │     │
│  │  (claude-sonnet-4-6)                                    │     │
│  │  - Analysiert Request                                   │     │
│  │  - Entscheidet: direkte Antwort oder Sub-Agents?        │     │
│  │  - Delegiert bei Bedarf                                 │     │
│  └──────┬───────────────────┬──────────────────────────────┘     │
│         │                   │                                     │
│    Worker Agents       Direct Response                            │
│    (Haiku × N)         (einfache Fragen)                         │
│         │                                                         │
│       ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  OUTPUT LAYER                                           │     │
│  │  - Memory aktualisieren                                 │     │
│  │  - Response streamen                                    │     │
│  │  - Logging & Metriken                                   │     │
│  └─────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────┘
```

---

## 2. Referenz-Implementierung: Java Dev Assistant (30 min)

Ein vollständiger AI-Assistent für Java-Entwickler.

```python
import anthropic
import chromadb
import json
from pathlib import Path

class JavaDevAssistant:
    """
    Vollständiger Java-Entwickler-Assistent mit:
    - RAG für Codebasis und Dokumentation
    - File-based Memory für Nutzerpräferenzen
    - Multi-Turn Konversation
    - Streaming
    """
    
    def __init__(self, project_path: str):
        self.client = anthropic.Anthropic()
        self.project_path = Path(project_path)
        self.chroma = chromadb.Client()
        self.collection = self.chroma.get_or_create_collection("java_codebase")
        
        # Memory (File-based)
        self.memory_file = self.project_path / ".ai-memory.json"
        self.memory = self._load_memory()
        
        # Konversations-Verlauf
        self.messages = []
        
        # System Prompt (wird gecacht)
        self.system_prompt = self._build_system_prompt()
        
        # Codebase indexieren wenn nötig
        if self.collection.count() == 0:
            self._index_codebase()
    
    def _build_system_prompt(self) -> list:
        """System Prompt mit Cache-Marker."""
        preferences = self.memory.get("preferences", {})
        
        base_prompt = f"""Du bist ein Java Senior Developer Assistent.

Projekt: {self.project_path.name}
Stack: {self.memory.get("stack", "Java, Spring Boot")}
Stil: {preferences.get("style", "Google Java Style Guide")}

Regeln:
1. Code immer in vollständigen, compilierbaren Snippets
2. JUnit 5 für alle Tests
3. Optional statt null
4. Erkläre wichtige Design-Entscheidungen kurz
5. Bei Unsicherheit: sagst du es explizit"""
        
        return [
            {
                "type": "text",
                "text": base_prompt,
                "cache_control": {"type": "ephemeral"}  # System Prompt cachen
            }
        ]
    
    def _load_memory(self) -> dict:
        if self.memory_file.exists():
            return json.loads(self.memory_file.read_text())
        return {}
    
    def _save_memory(self):
        self.memory_file.write_text(json.dumps(self.memory, indent=2))
    
    def _index_codebase(self):
        """Indexiert alle Java-Dateien im Projekt."""
        print("Indexiere Codebasis...")
        for java_file in self.project_path.rglob("*.java"):
            content = java_file.read_text()
            self.collection.add(
                documents=[content[:2000]],  # Erste 2000 Chars
                ids=[str(java_file)],
                metadatas=[{"path": str(java_file)}]
            )
        print(f"Indexiert: {self.collection.count()} Dateien")
    
    def _retrieve_context(self, query: str, k: int = 3) -> str:
        """Holt relevante Code-Chunks für die Anfrage."""
        if self.collection.count() == 0:
            return ""
        
        results = self.collection.query(
            query_texts=[query],
            n_results=min(k, self.collection.count())
        )
        
        if not results["documents"][0]:
            return ""
        
        chunks = []
        for doc, meta in zip(results["documents"][0], results["metadatas"][0]):
            chunks.append(f"// Datei: {meta['path']}\n{doc[:500]}")
        
        return "Relevanter Code aus dem Projekt:\n\n" + "\n\n---\n\n".join(chunks)
    
    def _compress_history(self):
        """Komprimiert lange Konversationen."""
        if len(self.messages) <= 10:
            return
        
        old = self.messages[:-8]
        recent = self.messages[-8:]
        
        summary_resp = self.client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=300,
            messages=[{
                "role": "user",
                "content": f"Fasse in 3-4 Sätzen zusammen: {json.dumps(old)}"
            }]
        )
        
        self.messages = [
            {"role": "user", "content": f"[Kontext]: {summary_resp.content[0].text}"},
            {"role": "assistant", "content": "Verstanden."}
        ] + recent
    
    def chat(self, user_input: str) -> str:
        """Verarbeitet eine User-Nachricht mit RAG + Memory + Streaming."""
        
        # RAG: relevanten Code holen
        context = self._retrieve_context(user_input)
        
        # Nachricht mit Kontext anreichern
        enriched_input = user_input
        if context:
            enriched_input = f"{context}\n\n---\n\nFrage: {user_input}"
        
        self.messages.append({"role": "user", "content": enriched_input})
        self._compress_history()
        
        # Streaming-Response
        full_response = ""
        with self.client.messages.stream(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            system=self.system_prompt,
            messages=self.messages
        ) as stream:
            for token in stream.text_stream:
                print(token, end="", flush=True)
                full_response += token
        
        print()  # Neue Zeile nach Stream
        
        self.messages.append({"role": "assistant", "content": full_response})
        
        # Memory: Präferenzen extrahieren und speichern
        self._update_memory(user_input, full_response)
        
        return full_response
    
    def _update_memory(self, user_input: str, response: str):
        """Extrahiert und speichert relevante Fakten."""
        # Einfache Heuristik: Stack-Informationen merken
        if "spring boot" in user_input.lower() or "java" in user_input.lower():
            if "stack" not in self.memory:
                self.memory["stack"] = "Java 17, Spring Boot 3"
                self._save_memory()

# ── Nutzung ──────────────────────────────────────────────────
if __name__ == "__main__":
    assistant = JavaDevAssistant("/path/to/my/java/project")
    
    print("Java Dev Assistant (Ctrl+C zum Beenden)\n")
    while True:
        try:
            user_input = input("Du: ")
            if user_input.strip():
                print("\nAssistent: ", end="")
                assistant.chat(user_input)
                print()
        except KeyboardInterrupt:
            print("\nTschüss!")
            break
```

---

## 3. Architektur-Entscheidungen (15 min)

### Wann welches Muster?

```
┌──────────────────┬───────────────────────────────────────────────┐
│ Pattern          │ Wann nutzen                                    │
├──────────────────┼───────────────────────────────────────────────┤
│ Single Call      │ Einfache Fragen, kein externe Daten nötig      │
├──────────────────┼───────────────────────────────────────────────┤
│ RAG              │ Grosse Dokumenten-Basis, private Codebasis     │
├──────────────────┼───────────────────────────────────────────────┤
│ Agent Loop       │ Mehrstufige Aufgaben, unbekannte Schrittanzahl │
├──────────────────┼───────────────────────────────────────────────┤
│ Multi-Agent      │ Parallelisierbare Teilaufgaben,                │
│                  │ Spezialisierung nötig                          │
├──────────────────┼───────────────────────────────────────────────┤
│ Memory + RAG     │ Persistente Assistenten, lang laufende         │
│                  │ Projekte mit wachsender Wissensbasis           │
├──────────────────┼───────────────────────────────────────────────┤
│ Fine-tuning      │ Konsistenter Output-Stil, hohes Volumen,       │
│                  │ RAG reicht nicht mehr                          │
└──────────────────┴───────────────────────────────────────────────┘
```

---

## Zusammenfassung LE 22

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Vollständiges KI-System                  │
│                                                         │
│  Input → Context Assembly → Agent → Output              │
│                                                         │
│  Context Assembly = RAG + Memory + System Prompt        │
│  Agent = Loop mit Tools oder direkte Antwort            │
│  Output = Stream + Memory Update + Logging              │
│                                                         │
│  Die Muster sind kombinierbar:                          │
│  RAG + Memory + Streaming + Multi-Agent                 │
│  = vollständiger, produktionsreifer Assistent           │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 21 — Production Deployment](le21_production_deployment.md)*
*Weiter: [LE 23 — Wiederholung & nächste Schritte](le23_wiederholung.md)*
