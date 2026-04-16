# Tag 1 — Wie Claude funktioniert
> Senior Developer Onboarding · 2 Stunden

---

## 1. LLMs von Grund auf (30 min)

### Was ist ein Large Language Model?

Ein LLM ist kein Datenbank-Lookup und keine Suchmaschine. Es ist ein **statistisches Modell über Sprache** — trainiert darauf, das nächste Token vorherzusagen.

```
Training (einmalig, sehr teuer):
┌─────────────────────────────────────────────────────────────┐
│  Billionen Textzeichen aus dem Internet, Bücher, Code ...   │
│                          │                                  │
│                          ▼                                  │
│         ┌─────────────────────────┐                        │
│         │   Neural Network        │                        │
│         │   (~100 Mrd. Parameter) │  ← "Gewichte" (weights)│
│         └─────────────────────────┘                        │
│                          │                                  │
│              Gradient Descent optimiert                     │
│         "Sag das nächste Wort voraus"                       │
└─────────────────────────────────────────────────────────────┘

Ergebnis: Eine Datei mit Milliarden von Zahlen (Weights).
Das ist "das Modell". Es ändert sich nach dem Training nicht mehr.
```

### Was sind Tokens?

Tokens sind **keine Wörter** — sie sind Subwort-Stücke. Ein Tokenizer zerlegt Text in Chunks, die häufig zusammen vorkommen.

```
Beispiel: "Entwicklungsumgebung"

Als Tokens:
┌──────────┬──────────┬──────────┬──────────┐
│ Entwick  │  lungs   │  umge    │  bung    │
│  (1234)  │  (5678)  │  (9012)  │  (3456)  │
└──────────┴──────────┴──────────┴──────────┘
= 4 Tokens für 1 Wort

Englisch: "development environment"
┌───────────┬──────────────┐
│development│ environment  │
│  (7890)   │   (1122)     │
└───────────┴──────────────┘
= 2 Tokens — Englisch ist oft "billiger"

Faustregel: 1 Token ≈ 0.75 Wörter (Englisch)
            1 Token ≈ 0.5  Wörter (Deutsch)
```

### Inference: Was passiert bei JEDER deiner Nachrichten?

```
Du tippst: "Was ist Rekursion?"
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  TOKENIZER                                                │
│  "Was ist Rekursion?" → [1023, 456, 7890]                │
└───────────────────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  TRANSFORMER (96 Layer, Attention, Feed-Forward)          │
│                                                           │
│  Input tokens → Matrix-Multiplikationen → Output-Vektor  │
│                                                           │
│  Für jede mögliche nächste Token: Wahrscheinlichkeit      │
│  z.B.: "Rekursion" → 0.8%, " Eine" → 12%, " In" → 8% ... │
└───────────────────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  SAMPLING                                                 │
│  Wähle nächsten Token (z.B. " Rekursion")                │
│  → füge ihn an Input an                                   │
│  → nächste Runde                                          │
└───────────────────────────────────────────────────────────┘
                │
         (wiederholen bis <stop>)
                │
                ▼
      "Rekursion ist eine Technik..."

WICHTIG: Jeder Token wird einzeln generiert. Es gibt kein
"Nachdenken" vor der Antwort — nur sequenzielles Vorhersagen.
```

---

## 2. Context Window (30 min)

### Das Context Window ist alles

Das Context Window ist der **einzige Speicher**, den ich während einer Konversation habe. Alles, was darin steht, "sehe" ich. Was rausfällt — vergesse ich.

```
┌─────────────────────────────────────────────────────────┐
│                   CONTEXT WINDOW                        │
│                 (z.B. 200.000 Tokens)                   │
│                                                         │
│  ┌─────────────────┐                                    │
│  │  System Prompt  │  ← Instruktionen, Persönlichkeit   │
│  │  (vom Betreiber)│                                    │
│  └────────┬────────┘                                    │
│           │                                             │
│  ┌────────▼────────┐                                    │
│  │  User: Hallo    │                                    │
│  │  Asst: Hallo!   │                                    │
│  │  User: Was ist..│  ← Gesamte Gesprächshistorie       │
│  │  Asst: Das ist.│                                    │
│  │  User: Und was.│                                    │
│  │  Asst: ...     │                                    │
│  └────────┬────────┘                                    │
│           │                                             │
│  ┌────────▼────────┐                                    │
│  │  Aktuelle       │  ← Hier stehst du gerade           │
│  │  User-Nachricht │                                    │
│  └─────────────────┘                                    │
└─────────────────────────────────────────────────────────┘

Wenn das Fenster voll wird:
→ Ältere Nachrichten werden komprimiert oder entfernt
→ Ich "vergesse" frühe Details der Konversation
```

### Context vs. Wissen

```
                  ┌──────────────────────────────┐
                  │    WAS ICH "WEISS"            │
                  │                              │
   ┌──────────────┤  Trainings-Wissen            │
   │   STATISCH   │  (eingefroren Aug 2025)      │
   │              │  Fakten, Code, Sprache,      │
   │              │  Konzepte, Reasoning         │
   └──────────────┤                              │
                  ├──────────────────────────────┤
   ┌──────────────┤  Context Window              │
   │   DYNAMISCH  │  (pro Konversation)          │
   │              │  Deine Nachrichten,          │
   │              │  Tool-Ergebnisse,            │
   │              │  hochgeladene Dateien        │
   └──────────────┴──────────────────────────────┘

Kein Lernen zur Laufzeit!
Ich kann nichts "behalten" zwischen Konversationen
(außer explizite Memory-Systeme wie meines hier).
```

---

## 3. Prompt Engineering (30 min)

### Anatomie eines guten Prompts

```
┌──────────────────────────────────────────────────────────┐
│  SYSTEM PROMPT (optional, vom Entwickler gesetzt)        │
│                                                          │
│  "Du bist ein erfahrener Go-Entwickler. Antworte         │
│   präzise, ohne Erklärungen die nicht gefragt wurden.    │
│   Nutze nur stdlib, keine externen Packages."            │
└──────────────────────────────────────────────────────────┘
                         │
┌──────────────────────────────────────────────────────────┐
│  USER PROMPT                                             │
│                                                          │
│  Rolle:     "Als Senior-Reviewer:"                       │
│  Kontext:   "Ich habe diese Funktion geschrieben:"       │
│             [code]                                       │
│  Aufgabe:   "Finde Bugs und erkläre sie."                │
│  Format:    "Antworte als nummerierte Liste."            │
│  Einschrän: "Maximal 5 Punkte."                          │
└──────────────────────────────────────────────────────────┘
```

### Die wichtigsten Techniken

**Zero-shot** — Einfach fragen:
```
"Erkläre Deadlocks."
```

**Few-shot** — Beispiele mitgeben (sehr wirkungsvoll):
```
Übersetze diese Fehlermeldungen ins Deutsche:

Input:  "connection refused"
Output: "Verbindung abgelehnt"

Input:  "permission denied"
Output: "Zugriff verweigert"

Input:  "file not found"
Output: ???
```

**Chain-of-Thought** — Schritt-für-Schritt denken lassen:
```
"Denke Schritt für Schritt nach, bevor du antwortest."
"Analysiere zuerst das Problem, dann gib eine Lösung."
→ Signifikant bessere Ergebnisse bei Logik/Mathe/Code
```

**Rollen-Prompting:**
```
"Du bist ein skeptischer Security-Auditor. Deine Aufgabe
ist es, Schwachstellen zu finden, nicht Lob zu verteilen."
→ Verändert Tonalität und Fokus der Antwort stark
```

### Was mein Output NICHT ist

```
❌ FALSCH gedacht:    Claude "denkt nach" → gibt Antwort
✅ RICHTIG gedacht:   Claude generiert Token für Token,
                      jeder auf Basis aller vorherigen

Das bedeutet:
- Ich kann mich "verrennen" und dann weiterlügen
- Längere Antworten sind nicht zwingend besser
- "Denke laut nach" (CoT) hilft, weil es Zwischen-
  schritte als Tokens erzwingt, auf denen aufgebaut wird
```

---

## 4. Hands-on (30 min)

### Übung 1: Tokens beobachten

Geh auf https://platform.openai.com/tokenizer (funktioniert auch für Claude-Tokenizer-Verständnis) und tokenisiere:
- Einen deutschen Satz
- Den gleichen Satz auf Englisch
- Code (Python vs. Go)

Beobachte: Wie viele Tokens verbraucht welche Sprache?

### Übung 2: Prompt-Qualität vergleichen

Stelle mir diese Fragen nacheinander und beachte den Unterschied:

```
Version A (schlecht):
"Erkläre mir Goroutines"

Version B (gut):
"Ich bin ein erfahrener Java-Entwickler und verstehe Threads.
Erkläre mir Go Goroutines im Vergleich zu Java Threads.
Fokus: Scheduling-Unterschiede. Maximal 5 Sätze."
```

### Übung 3: Context Window testen

Führe eine lange Konversation. Stelle früh eine Frage ("Meine Lieblingsfarbe ist Blau"). Nach 20+ Nachrichten: Frag wieder. Beobachte ob und wann ich es vergesse.

### Übung 4: Chain-of-Thought

```
Ohne CoT:
"Was ist 17 * 24 + 89 / 7?"

Mit CoT:
"Was ist 17 * 24 + 89 / 7? Rechne Schritt für Schritt."
```

---

## Zusammenfassung Tag 1

```
┌─────────────────────────────────────────────────────────┐
│                  MENTAL MODEL                           │
│                                                         │
│  1. Ich bin ein sehr großes, gefrorenes Sprachmodell    │
│     → kein Lernen zur Laufzeit                          │
│                                                         │
│  2. Jede Antwort = Token für Token generiert            │
│     → Wahrscheinlichkeiten, kein "Wissen abrufen"       │
│                                                         │
│  3. Mein einziger Speicher = Context Window             │
│     → Was drin steht, sehe ich. Sonst nichts.           │
│                                                         │
│  4. Prompt-Qualität beeinflusst Output-Qualität massiv  │
│     → Kontext + Rolle + Format = bessere Ergebnisse     │
└─────────────────────────────────────────────────────────┘
```

---

*Weiter mit: [Tag 2 — Claude als Werkzeug](tag2_claude_als_werkzeug.md)*
