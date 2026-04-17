# LE 17 — Multi-Agent Systeme vertieft
> Lernblock 4: Vertiefung · 1 Stunde

---

## 1. Multi-Agent Architektur-Patterns (20 min)

### Pattern 1: Orchestrator + Workers

```
Das häufigste Pattern:

┌──────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR                              │
│                    (Claude Sonnet)                           │
│  - Empfängt Aufgabe                                          │
│  - Plant Schritte                                            │
│  - Delegiert an Worker-Agents                                │
│  - Aggregiert Ergebnisse                                     │
│  - Gibt finale Antwort                                       │
└──────┬──────────────────┬──────────────────┬─────────────────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  WORKER A   │  │  WORKER B   │  │  WORKER C   │
│  (Haiku)    │  │  (Haiku)    │  │  (Haiku)    │
│  Analyse    │  │  Code Gen   │  │  Testing    │
└─────────────┘  └─────────────┘  └─────────────┘

Vorteile:
  ✓ Workers können parallel laufen
  ✓ Günstigere Modelle für einfache Aufgaben
  ✓ Spezialisierung pro Worker
  ✓ Context-Schutz (Worker-Details bleiben beim Worker)
```

### Pattern 2: Pipeline

```
Aufgabe A → Agent 1 → Ergebnis 1 → Agent 2 → Ergebnis 2 → ...

Beispiel: Java-Migrations-Pipeline

Input: Legacy-Code-File
    │
    ▼
Agent 1 (Analyse):
  → "Welche EJB-Patterns werden verwendet?"
  → Output: Analyse-Report
    │
    ▼
Agent 2 (Migration):
  → Input: Analyse-Report + Original-Code
  → Output: Migrierter Spring-Code
    │
    ▼
Agent 3 (Test-Generierung):
  → Input: Migrierter Code
  → Output: JUnit 5 Tests
    │
    ▼
Agent 4 (Review):
  → Input: Code + Tests
  → Output: Review-Kommentare + finale Version
```

### Pattern 3: Debate / Critic

```
Zwei Agents streiten → besseres Ergebnis

AGENT A (Proposer):
  "Hier ist meine Architektur-Lösung: [Microservices]"

AGENT B (Critic):
  "Probleme mit diesem Ansatz:
   1. Team zu klein für Microservices
   2. Deployment-Komplexität unterschätzt
   3. Latenz bei Service-Calls"

AGENT A (Revised):
  "Revidierter Vorschlag basierend auf Kritik:
   Modularer Monolith mit klaren Boundaries..."

Nutzen: Vermeidet blinde Flecken eines einzelnen Agents
```

---

## 2. Multi-Agent in Python implementieren (25 min)

### Orchestrator-Pattern

```python
import anthropic
from typing import Callable

client = anthropic.Anthropic()

# ── Worker-Funktionen ────────────────────────────────────
def analyze_code(code: str) -> str:
    """Worker A: Code analysieren"""
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=1024,
        system="Du bist ein Java-Code-Analyst. Antworte präzise und strukturiert.",
        messages=[{
            "role": "user",
            "content": f"Analysiere diesen Java-Code. Identifiziere: Klassen, Methoden, Abhängigkeiten, Probleme.\n\n```java\n{code}\n```"
        }]
    )
    return response.content[0].text

def migrate_code(code: str, analysis: str) -> str:
    """Worker B: Code migrieren"""
    response = client.messages.create(
        model="claude-sonnet-4-6",  # Komplexere Aufgabe → besseres Modell
        max_tokens=2048,
        system="Du bist ein Java-Migrationsspezialist. Migriere EJB-Code auf Spring Boot 3.",
        messages=[{
            "role": "user",
            "content": f"Analyse:\n{analysis}\n\nOriginal-Code:\n```java\n{code}\n```\n\nMigriere auf Spring Boot 3 / Java 17."
        }]
    )
    return response.content[0].text

def generate_tests(migrated_code: str) -> str:
    """Worker C: Tests generieren"""
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=1024,
        system="Du bist ein Java-Test-Spezialist. Schreibe JUnit 5 Tests.",
        messages=[{
            "role": "user",
            "content": f"Schreibe JUnit 5 Tests für:\n```java\n{migrated_code}\n```"
        }]
    )
    return response.content[0].text

# ── Orchestrator ─────────────────────────────────────────
def migration_orchestrator(legacy_code: str) -> dict:
    """
    Orchestriert die komplette Migration:
    Analyse → Migration → Tests
    """
    print("Phase 1: Code-Analyse...")
    analysis = analyze_code(legacy_code)
    
    print("Phase 2: Migration...")
    migrated = migrate_code(legacy_code, analysis)
    
    print("Phase 3: Test-Generierung...")
    tests = generate_tests(migrated)
    
    # Orchestrator fasst zusammen
    summary_response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        messages=[{
            "role": "user",
            "content": f"""Erstelle eine kurze Migrations-Zusammenfassung:
            
Analyse: {analysis[:500]}...
Migriert: {migrated[:500]}...
Tests: {tests[:300]}...

Format: 3 Stichpunkte Was wurde gemacht, Was sind die Highlights, Was muss noch geprüft werden."""
        }]
    )
    
    return {
        "analysis": analysis,
        "migrated_code": migrated,
        "tests": tests,
        "summary": summary_response.content[0].text
    }

# Nutzung:
with open("SensorDataServiceBean.java") as f:
    legacy = f.read()

result = migration_orchestrator(legacy)
print("Summary:", result["summary"])
```

### Parallele Worker

```python
import concurrent.futures

def run_parallel_analyses(files: list[str]) -> list[dict]:
    """Analysiert mehrere Dateien parallel."""
    
    def analyze_file(file_path: str) -> dict:
        with open(file_path) as f:
            code = f.read()
        analysis = analyze_code(code)
        return {"file": file_path, "analysis": analysis}
    
    # Parallele Ausführung (ThreadPoolExecutor für I/O-bound API-Calls)
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        futures = {executor.submit(analyze_file, f): f for f in files}
        results = []
        for future in concurrent.futures.as_completed(futures):
            results.append(future.result())
    
    return results
```

---

## 3. Fehlerbehandlung & Resilience (10 min)

```python
import time
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60)
)
def resilient_agent_call(prompt: str, model: str = "claude-haiku-4-5-20251001") -> str:
    """Agent-Call mit automatischem Retry bei Rate-Limit."""
    try:
        response = client.messages.create(
            model=model,
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.content[0].text
    except anthropic.RateLimitError:
        print("Rate limit — warte...")
        raise  # tenacity retry
    except anthropic.APIError as e:
        print(f"API Fehler: {e}")
        raise

# Timeout-Schutz für Agents
import signal

class TimeoutError(Exception):
    pass

def timeout_handler(signum, frame):
    raise TimeoutError("Agent timeout!")

def run_with_timeout(func, timeout_seconds: int, *args, **kwargs):
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(timeout_seconds)
    try:
        return func(*args, **kwargs)
    except TimeoutError:
        return {"error": "Agent-Timeout nach {timeout_seconds}s"}
    finally:
        signal.alarm(0)
```

---

## 4. Kosten-Kontrolle (5 min)

```python
# Token-Tracking über mehrere Agent-Aufrufe
class CostTracker:
    PRICES = {
        "claude-opus-4-6":         (15.0, 75.0),   # input, output per MTok
        "claude-sonnet-4-6":       ( 3.0, 15.0),
        "claude-haiku-4-5-20251001": ( 0.25,  1.25),
    }
    
    def __init__(self):
        self.total_cost = 0.0
        self.calls = []
    
    def track(self, model: str, input_tokens: int, output_tokens: int):
        in_price, out_price = self.PRICES.get(model, (3.0, 15.0))
        cost = (input_tokens * in_price + output_tokens * out_price) / 1_000_000
        self.total_cost += cost
        self.calls.append({"model": model, "in": input_tokens, "out": output_tokens, "cost": cost})
    
    def report(self):
        print(f"Gesamt-Kosten: ${self.total_cost:.4f}")
        for call in self.calls:
            print(f"  {call['model']}: {call['in']}in + {call['out']}out = ${call['cost']:.4f}")

# Budget-Limit
MAX_BUDGET_USD = 1.0
tracker = CostTracker()

if tracker.total_cost > MAX_BUDGET_USD:
    raise RuntimeError(f"Budget überschritten: ${tracker.total_cost:.2f} > ${MAX_BUDGET_USD}")
```

---

## Zusammenfassung LE 17

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Multi-Agent                              │
│                                                         │
│  Patterns:                                              │
│    Orchestrator + Workers → häufigste Architektur       │
│    Pipeline → sequenzielle Verarbeitung                 │
│    Debate → bessere Qualität durch Kritik               │
│                                                         │
│  Praxis:                                                │
│    → Haiku für einfache Worker (günstig)                │
│    → Sonnet für Orchestrator & Kern-Logik               │
│    → Parallele Ausführung mit ThreadPoolExecutor        │
│    → Immer: Timeout + Retry + Budget-Limit              │
│                                                         │
│  Kosten: Agents multiplizieren Token-Verbrauch —        │
│  immer Budget-Limit setzen!                             │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 16 — Prompt Engineering vertieft](le10_prompt_engineering.md)*
*Weiter: [LE 18 — RAG vertieft](le24_rag_vertieft.md)*
