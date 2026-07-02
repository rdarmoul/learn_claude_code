# LE 07 — Agent Loop & ReAct Pattern
> Lernblock 2: Kernkonzepte · 1 Stunde

---

## 1. Was ist ein Agent? (15 min)

Ein Agent ist kein spezielles Modell. Es ist ein **Ausführungs-Pattern**:

```
Agent = LLM + Tools + Schleife + Abbruchbedingung

Ohne Agent (Single-Call):
  User → Claude → Antwort. Fertig.
  Problem: Claude kann nur auf sein Wissen zugreifen.

Mit Agent (Loop):
  User → Claude → braucht Info → Tool → Ergebnis
              ↑                               │
              └───────────── zurück ──────────┘
              → Claude → braucht mehr → Tool → ...
              → Claude → fertig → Antwort

Analogie für Java-Entwickler:
  Single-Call = statische Methode
  Agent       = Objekt mit State, das in einer while-Schleife arbeitet
```

### Wann einen Agenten nutzen?

```
KEIN Agent nötig:                    AGENT sinnvoll:
  ✓ "Erkläre mir X"                    ✓ "Finde den Bug in meinem Projekt"
  ✓ "Schreibe Text Y"                  ✓ "Refactore diese Klasse + fixe Tests"
  ✓ "Übersetze Z"                      ✓ "Analysiere Logs + erstelle Report"
                                       ✓ "Durchsuche Codebase + dokumentiere API"

Faustregel:
  Braucht die Aufgabe mehrere Informationsquellen oder Schritte?
  → Agent
  Ist die Aufgabe mit einer Antwort erledigt?
  → Single-Call
```

---

## 2. Der Agent Loop im Code (20 min)

### Vollständige Python-Implementierung

```python
import anthropic
import json
from typing import Any

client = anthropic.Anthropic()

# ── Tools definieren ──────────────────────────────────────
tools = [
    {
        "name": "read_file",
        "description": "Liest eine Datei aus dem Dateisystem.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "Dateipfad"}
            },
            "required": ["path"]
        }
    },
    {
        "name": "list_directory",
        "description": "Listet alle Dateien in einem Verzeichnis.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"}
            },
            "required": ["path"]
        }
    }
]

# ── Tool-Ausführung ──────────────────────────────────────
def execute_tool(name: str, inputs: dict) -> Any:
    if name == "read_file":
        try:
            with open(inputs["path"], "r") as f:
                return f.read()
        except FileNotFoundError:
            return f"FEHLER: Datei nicht gefunden: {inputs['path']}"
    
    if name == "list_directory":
        import os
        try:
            return "\n".join(os.listdir(inputs["path"]))
        except FileNotFoundError:
            return f"FEHLER: Verzeichnis nicht gefunden: {inputs['path']}"
    
    return f"FEHLER: Unbekanntes Tool: {name}"

# ── Agent Loop ───────────────────────────────────────────
def run_agent(task: str, max_iterations: int = 10) -> str:
    messages = [{"role": "user", "content": task}]
    iteration = 0
    
    while iteration < max_iterations:
        iteration += 1
        print(f"\n[Iteration {iteration}]")
        
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )
        
        print(f"stop_reason: {response.stop_reason}")
        
        # ── Fertig ──────────────────────────────────────
        if response.stop_reason == "end_turn":
            texts = [b.text for b in response.content if b.type == "text"]
            return "\n".join(texts)
        
        # ── Tool-Calls ausführen ─────────────────────────
        if response.stop_reason == "tool_use":
            messages.append({
                "role": "assistant",
                "content": response.content
            })
            
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    print(f"  → Tool: {block.name}({json.dumps(block.input)})")
                    result = execute_tool(block.name, block.input)
                    print(f"  ← Ergebnis: {str(result)[:100]}...")
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })
            
            messages.append({
                "role": "user",
                "content": tool_results
            })
    
    return "ABBRUCH: Max Iterationen erreicht"

# ── Aufruf ───────────────────────────────────────────────
result = run_agent("Analysiere die Java-Dateien im src/ Verzeichnis und liste alle public Klassen.")
print("\nErgebnis:", result)
```

---

## 3. ReAct Pattern (15 min)

ReAct = **Re**asoning + **Act**ing. Das bekannteste Agent-Pattern aus der Forschung.

```
THOUGHT → ACTION → OBSERVATION → THOUGHT → ...

┌─────────────────────────────────────────────────────┐
│  THOUGHT:  "Um den Bug zu finden, muss ich zuerst   │
│             die Exception verstehen. Ich lese den   │
│             Stacktrace."                             │
│                                                      │
│  ACTION:   read_file("logs/app.log")                 │
│                                                      │
│  OBSERVATION: "NullPointerException in              │
│                UserService.java:47"                  │
│                                                      │
│  THOUGHT:  "Ich schaue mir Zeile 47 an."             │
│                                                      │
│  ACTION:   read_file("src/UserService.java")         │
│                                                      │
│  OBSERVATION: "user.getAddress() ohne null-Check"   │
│                                                      │
│  THOUGHT:  "Gefunden. Ich fixe es mit Optional."    │
│                                                      │
│  ACTION:   edit_file(...)                            │
│                                                      │
│  ANSWER:   "Bug gefunden und gefixt in Zeile 47..." │
└─────────────────────────────────────────────────────┘
```

### ReAct vs. Pure Reasoning

```
Pure Reasoning (kein Tool Use):
  Prompt: "Was ist in UserService.java in Zeile 47?"
  Problem: Claude kennt deine Datei nicht → halluziniert

ReAct (mit Tools):
  Claude erkennt: "Ich muss die Datei lesen" → read_file
  → echte Daten im Context → korrekte Antwort

Für dich als Entwickler:
  Claude Code = ReAct Agent der automatisch läuft
  Du siehst: "Let me read this file" = THOUGHT + ACTION
```

---

## 4. Sub-Agents & Parallelisierung (10 min)

Claude Code nutzt Sub-Agents für unabhängige Teilaufgaben:

```
HAUPT-AGENT: "Analysiere das gesamte Projekt"
     │
     ├──► SUB-AGENT A (Explore)       läuft parallel
     │      → durchsucht alle .java Dateien
     │      → gibt Liste + Zusammenfassung zurück
     │
     ├──► SUB-AGENT B (Explore)       läuft parallel
     │      → analysiert pom.xml + Dependencies
     │      → gibt Dependency-Tree zurück
     │
     ▼ (wartet auf beide)
     
HAUPT-AGENT: kombiniert Ergebnisse → Gesamtanalyse
```

### Warum Sub-Agents?

```
1. Context-Schutz:
   Sub-Agent durchsucht 500 Dateien → gibt nur Zusammenfassung zurück
   Haupt-Agent bekommt nicht 500 Dateien in seinen Context!

2. Parallelisierung:
   Zwei unabhängige Analysen laufen gleichzeitig → 2x schneller

3. Spezialisierung:
   Explore-Agent für Suche, Plan-Agent für Architektur
```

### Kosten-Warnung bei Sub-Agents

```
WICHTIG: Jeder Sub-Agent = eigener API-Call mit eigenem Context

Haupt-Agent Context: 10.000 Tokens
Sub-Agent A läuft:   + 5.000 Tokens Input
Sub-Agent B läuft:   + 8.000 Tokens Input
─────────────────────────────────────────
Gesamt Input:        ~23.000 Tokens (nur für diese eine Runde!)

Bei langen, tiefen Agenten-Ketten: Kosten explodieren schnell.
→ max_iterations setzen
→ Sub-Agent-Tiefe begrenzen
→ Ergebnisse zusammenfassen statt rohe Daten weitergeben
```

---

## Zusammenfassung LE 07

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Agents                                   │
│                                                         │
│  Agent = while(stop_reason != "end_turn") { tool(); }   │
│                                                         │
│  ReAct = Thought → Action → Observation (Schleife)      │
│                                                         │
│  Sub-Agents = Context-Schutz + Parallelisierung         │
│               aber: Kosten aufpassen!                   │
│                                                         │
│  Wann Agent? → Mehrstufige Aufgaben mit Infobedarf      │
│  Wann nicht? → Einmalige Fragen ohne externe Daten      │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 06 — Tool Use Deep Dive](le12_tool_use_deep_dive.md)*
*Weiter: [LE 08 — RAG & Embeddings](le14_rag_embeddings.md)*
