# LE 19 — Fine-tuning: Wann und Wie
> Lernblock 4: Vertiefung · 1 Stunde

---

## 1. Was ist Fine-tuning? (15 min)

```
Fine-tuning = Weiteres Training auf eigenen Daten
→ Die Model-Weights werden angepasst
→ Ergebnis: neues Modell das deine spezifischen Daten "kennt"

NICHT Fine-tuning:
  - RAG (Dokumente zur Laufzeit laden)
  - System Prompts (Verhalten zur Laufzeit anpassen)
  - Few-Shot (Beispiele im Prompt)

Analogie:
  Base Model:   Generalist-Entwickler von der Uni
  Fine-tuned:   Entwickler der 2 Jahre bei deiner Firma arbeitet
                und eure Konventionen, Codebase, Prozesse kennt
  RAG:          Generalist mit Zugriff auf eure Intranet-Doku
  
  RAG ist oft günstiger und flexibler als Fine-tuning!
```

### Fine-tuning vs. Alternativen

```
┌─────────────────┬───────────┬───────────┬──────────────────────┐
│                 │ Prompt    │ RAG       │ Fine-tuning          │
│                 │ Engineering│           │                      │
├─────────────────┼───────────┼───────────┼──────────────────────┤
│ Kosten Setup    │ ~0        │ niedrig   │ hoch ($10 - $1000+)  │
│ Kosten Inference│ mittel    │ mittel    │ niedriger (kürzerer  │
│                 │           │           │ Prompt nötig)        │
│ Dynamische Daten│ ✓         │ ✓         │ ✗ (statisch)         │
│ Private Daten   │ ✓         │ ✓         │ ✓                    │
│ Konsistenz      │ mittel    │ gut       │ sehr gut             │
│ Aufwand         │ klein     │ mittel    │ gross                │
└─────────────────┴───────────┴───────────┴──────────────────────┘
```

---

## 2. Wann Fine-tuning sinnvoll ist (15 min)

### Gute Use Cases

```
1. KONSISTENTER OUTPUT-STIL (grösster Nutzen)
   Dein firmeninternes Format für:
   - API-Responses (spezifische JSON-Struktur)
   - Code-Kommentare (firmeninternes Template)
   - Fehlermeldungen (einheitliches Format)
   
   Prompt Engineering schafft es nicht konsistent zu halten?
   → Fine-tuning hilft

2. SPEZIFISCHES FACHVOKABULAR
   - Proprietäre Terminologie
   - Interne Produktnamen / Systemnamen
   - Branchenspezifische Abkürzungen
   
   RAG löst das oft billiger → trotzdem zuerst versuchen!

3. GROSSE INFERENCE-VOLUMEN
   1 Million API-Calls/Tag × langer System-Prompt
   → Fine-tuning macht System-Prompt überflüssig
   → Einsparung bei Inference-Kosten

4. AUFGABEN MIT FIXEM FORMAT
   Klassifikation, Extraktion, strukturierte Outputs
   → Fine-tuning schlägt oft Prompt Engineering
```

### Schlechte Use Cases

```
✗ "Claude soll unsere 10.000 internen Docs kennen"
  → RAG ist besser (Docs ändern sich, sind zu gross für Training)

✗ "Claude soll sich an User erinnern"
  → Memory-System ist besser (Fine-tuning lernt keine Fakten)

✗ "Wir haben 50 Beispiele"
  → Zu wenig für gutes Fine-tuning (Few-Shot reicht)

✗ "Schnell mal ausprobieren"
  → Fine-tuning dauert Stunden/Tage, kostet Geld

✗ "Safety durch Fine-tuning erhöhen"
  → RLHF und Constitutional AI im Base Model sind robuster
```

---

## 3. Fine-tuning Prozess (20 min)

### Daten vorbereiten

```python
# Format für Anthropic Fine-tuning (Messages-Format)
import json

# Jedes Trainingsbeispiel = eine vollständige Konversation
training_data = [
    {
        "messages": [
            {
                "role": "user",
                "content": "Erstelle eine Fehlermeldung für ungültige User-ID"
            },
            {
                "role": "assistant",
                "content": """{
  "error": {
    "code": "INVALID_USER_ID",
    "message": "Die angegebene User-ID ist ungültig oder nicht vorhanden.",
    "timestamp": "2025-04-17T10:30:00Z",
    "traceId": "req-abc123"
  }
}"""
            }
        ]
    },
    # ... mehr Beispiele
]

# Daten speichern
with open("training_data.jsonl", "w") as f:
    for example in training_data:
        f.write(json.dumps(example) + "\n")

print(f"Training examples: {len(training_data)}")
# Minimum empfohlen: 100-500 Beispiele
# Optimal: 1000+ Beispiele
```

### Qualität der Trainingsdaten

```
GOLDENE REGELN FÜR TRAININGSDATEN:

1. KONSISTENZ
   Alle Beispiele müssen dem gleichen Format/Stil folgen.
   Inkonsistente Daten = schlechteres Modell als Base!

2. DIVERSITÄT
   Viele verschiedene Inputs, nicht 1000 Varianten des gleichen Prompts.
   → Echte Produktionsdaten sind oft besser als synthetische

3. QUALITÄT > QUANTITÄT
   100 perfekte Beispiele > 1000 schlechte Beispiele

4. NEGATIVBEISPIELE (optional aber hilfreich)
   Zeige auch was das Modell NICHT tun soll:
   {"role": "user", "content": "ungültige Anfrage"}
   {"role": "assistant", "content": "Ich kann das nicht verarbeiten weil..."}

5. SYSTEM PROMPT EINBAUEN (wenn du einen nutzt)
   System Prompt muss in JEDEM Trainingsbeispiel identisch sein!
```

### Kosten-Schätzung

```
Anthropic Fine-tuning Preise (ungefähr, Preise ändern sich):

Training:
  Haiku:  ~$6 pro Million Tokens (Trainingsdaten)
  Sonnet: Auf Anfrage / höher

Inference (fine-tuned Modell):
  Etwas teurer als Base-Modell
  Aber: kürzere Prompts nötig → oft trotzdem günstiger

Beispiel-Kalkulation:
  1000 Trainingsbeispiele × 500 Tokens/Beispiel = 500k Tokens
  500k × $6/MTok = $3 für Training

  Inference: 100k Calls/Tag × 200 Tokens Input × $0.25/MTok = $5/Tag
  Ohne Fine-tuning: 100k Calls × 600 Tokens (inkl. System Prompt) × $0.25/MTok = $15/Tag
  Einsparung: $10/Tag → ROI: <1 Tag!

  → Bei hohem Volumen und langen System-Prompts: Fine-tuning rentiert schnell
```

---

## 4. Evaluation nach Fine-tuning (10 min)

```python
# Evaluation-Framework
def evaluate_fine_tuned_model(
    base_model: str,
    fine_tuned_model: str,
    test_cases: list[dict]
) -> dict:
    """Vergleicht Base- vs. Fine-tuned Modell."""
    
    results = {"base": [], "fine_tuned": []}
    
    for test in test_cases:
        for model_name, model_id in [("base", base_model), ("fine_tuned", fine_tuned_model)]:
            response = client.messages.create(
                model=model_id,
                max_tokens=512,
                messages=[{"role": "user", "content": test["input"]}]
            )
            output = response.content[0].text
            
            # Automatische Bewertung (Format-Konformität)
            score = evaluate_output(output, test["expected_format"])
            results[model_name].append(score)
    
    return {
        "base_avg": sum(results["base"]) / len(results["base"]),
        "fine_tuned_avg": sum(results["fine_tuned"]) / len(results["fine_tuned"]),
        "improvement": sum(results["fine_tuned"]) / len(results["fine_tuned"]) - 
                       sum(results["base"]) / len(results["base"])
    }
```

---

## Zusammenfassung LE 19

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Fine-tuning                              │
│                                                         │
│  WANN: Konsistenter Stil, hohes Volumen, fixes Format   │
│  WANN NICHT: Dynamische Daten, wenige Beispiele, RAG    │
│               löst es schon                             │
│                                                         │
│  Prozess: Daten → JSONL → Training → Evaluation         │
│  Qualität > Quantität bei Trainingsdaten                │
│                                                         │
│  ROI-Check: Wie viel spare ich pro Tag bei Inference?   │
│  Wenn < Training-Kosten → RAG/Prompting bevorzugen      │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 18 — RAG vertieft](le18_rag_vertieft.md)*
*Weiter: [LE 20 — Claude API für eigene Produkte](le20_claude_api_produkte.md)*
