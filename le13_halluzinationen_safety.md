# LE 10 — Halluzinationen & Safety
> Lernblock 2: Kernkonzepte · 1 Stunde

---

## 1. Halluzinationen — das wichtigste Problem (20 min)

### Was sind Halluzinationen?

```
Claude generiert Token für Token nach Wahrscheinlichkeit.
Es gibt kein internes "Prüf-Flag" für Wahrheit.

Hohe Wahrscheinlichkeit ≠ Korrektheit

Beispiel:
  "Welche Methoden hat die Spring-Klasse HttpEntityMethodProcessor?"
  Claude: "HttpEntityMethodProcessor hat folgende Methoden:
           - handleReturnValue(...)    ✓ (existiert)
           - processResponse(...)      ✗ (ERFUNDEN)
           - validateHttpEntity(...)   ✗ (ERFUNDEN)"

Das klingt plausibel → und ist falsch!
```

### Wann halluziniert Claude häufiger?

```
HOHES RISIKO:
  ✗ Spezifische API-Methoden & Signaturen (ändert sich mit Versionen)
  ✗ Library-Versionen & Kompatibilität
  ✗ Konkrete Zahlen & Statistiken
  ✗ Nischenwissen (wenige Trainingsdaten)
  ✗ Events nach Knowledge Cutoff (Aug 2025)
  ✗ Sehr spezifische Namen: Personen, Pakete, Issue-IDs

NIEDRIGES RISIKO:
  ✓ Allgemeines Programmierwissen (Java OOP, Patterns)
  ✓ Code schreiben (verifizierbar durch Compiler)
  ✓ Logisches Schlussfolgern
  ✓ Erklärungen von gut-dokumentierten Konzepten
  ✓ Wenn du echte Docs/Code im Context hast (RAG)
```

### Für dich als Java-Entwickler

```
GEFÄHRLICHE PROMPTS:
  "Welche Spring Boot Version ist kompatibel mit Java 21?"
  "Hat Jackson eine Methode readValueAsTree()?"
  "Was ist die genaue Signatur von ResponseEntity.ok()?"

SICHERER UMGANG:
  → Immer den generierten Code compilieren und testen
  → Bei API-Fragen: offizielle Docs prüfen (docs.spring.io, etc.)
  → Spezifische Fakten mit einem zweiten Lookup bestätigen
  → Claude mit echten Code-Snippets im Prompt füttern (statt fragen)

PRAKTISCHE REGEL:
  "Code von Claude ist ein Entwurf, kein Fertigprodukt."
  → Dein Review-Instinkt bleibt unverzichtbar.
```

### Halluzinationen reduzieren

```python
# Technik 1: Dokumente mitschicken (Context statt Gedächtnis)
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="""Du bist ein Java-Experte. 
Beantworte Fragen NUR basierend auf den bereitgestellten Docs.
Wenn etwas unklar ist, sage: 'Das steht nicht in den Docs.'""",
    messages=[{
        "role": "user",
        "content": f"""Javadoc:\n{javadoc_text}\n\nFrage: {question}"""
    }]
)

# Technik 2: Unsicherheit explizit einfordern
# Füge zum Prompt hinzu:
"Wenn du dir bei etwas nicht sicher bist, sage das explizit."
"Gib Konfidenz-Level an: [sicher / wahrscheinlich / unsicher]"

# Technik 3: Code immer testen lassen
"Schreibe einen JUnit-Test der diese Implementierung prüft."
→ Halluzinierter Code wird oft durch den Test entlarvt
```

---

## 2. Safety-Architektur (20 min)

### Die drei Safety-Ebenen

```
EBENE 1: Hard-coded (unveränderlich, im Modell eingebaut)
  Keine Ausnahmen, keine Überzeugung möglich:
  ✗ CBRN-Waffen (biologisch, chemisch, radiologisch, nuklear)
  ✗ CSAM (sexueller Missbrauch von Kindern)
  ✗ Angriffe auf kritische Infrastruktur (Strom, Wasser)
  ✗ Cyberangriffe auf echte Produktionssysteme

EBENE 2: Default-Behavior (änderbar durch Operator/System Prompt)
  Betreiber können diese anpassen:
  ✗ Explizite Gewaltdarstellung (default aus, für manche Plattformen erlaubbar)
  ✗ Medizinische Details (default vorsichtig, für Ärzte-Plattform lockerer)
  ✓ Sicherheitsforschung / CTF (default vorsichtig, mit Kontext erlaubt)

EBENE 3: Context-abhängig (per Prompt steuerbar)
  Je nach Kontext und Anfrage:
  → "Ich bin Pen-Tester, zeige mir..." → mit Kontext meist möglich
  → Ohne Kontext → vorsichtige Antwort
```

### Constitutional AI (Anthropics Ansatz)

```
RLHF allein hat ein Problem:
  "Maximiere menschliches Feedback" → manchmal: sage was man hören will

Constitutional AI löst das:
  1. Prinzipien definieren ("sei hilfreich, sei harmlos, sei ehrlich")
  2. Claude bewertet eigene Antworten gegen diese Prinzipien
  3. Claude verbessert eigene Antworten (self-critique)
  4. Dieses Feedback trainiert das Reward-Modell

Resultat:
  → Claude kann begründen WARUM es etwas ablehnt
  → Konsistentere Safety-Entscheidungen
  → Weniger "Jailbreak"-anfällig
```

### Der "Reversal Curse"

```
Training: "A ist B" → Modell lernt "A → B"
Nicht automatisch: "B → A"

Praktische Auswirkung:
  Claude kennt: "Spring Boot 3 erfordert Java 17"
  Frage: "Welche Spring Boot Version läuft auf Java 17?"
  → Claude kann unsicher sein oder halluzinieren!

Warum relevant?
  → Immer beide Richtungen explizit testen wenn kritisch
  → Bi-direktionale Fakten extra prüfen
```

---

## 3. Grenzen im Entwickleralltag (20 min)

### Technische Grenzen

```
┌─────────────────────────────────────────────────────────────┐
│ GRENZE              │ AUSWIRKUNG           │ LÖSUNG         │
├─────────────────────┼──────────────────────┼────────────────┤
│ Context Window      │ Grosse Repos nicht   │ RAG, Sub-Agent │
│ (~200k Tokens)      │ vollständig lesbar   │ Explore        │
├─────────────────────┼──────────────────────┼────────────────┤
│ Knowledge Cutoff    │ Neue Libraries       │ Docs in Context│
│ (Aug 2025)          │ unbekannt            │ laden (RAG)    │
├─────────────────────┼──────────────────────┼────────────────┤
│ Kein persistentes   │ Vergisst zwischen    │ Memory-System  │
│ Gedächtnis          │ Sessions             │ (Datei/Vector) │
├─────────────────────┼──────────────────────┼────────────────┤
│ Kein Internet       │ Kann nicht googlen   │ WebFetch Tool  │
│ (ohne Tool)         │                      │ oder RAG       │
├─────────────────────┼──────────────────────┼────────────────┤
│ Kein exaktes Rechnen│ Grosse Zahlen        │ Code-Interpreter│
│                     │ unzuverlässig        │ (Bash + Python)│
└─────────────────────┴──────────────────────┴────────────────┘
```

### "Trust but Verify" — dein Umgang mit Claude

```
Als Senior Developer: dein Code-Review-Instinkt bleibt unverzichtbar.

Was du IMMER tun solltest:
  ✓ Generierten Code compilieren und testen
  ✓ Library-Versionen und APIs selbst prüfen
  ✓ Security-relevanten Code besonders genau reviewen
  ✓ Komplexe Algorithmen auf Korrektheit prüfen
  ✓ SQL-Queries auf Injection prüfen (auch Claude macht Fehler!)

Was meist sicher ist:
  ✓ Boilerplate-Code (Getter/Setter, Builder, etc.)
  ✓ Standardmässige Spring-Konfigurationen
  ✓ Unit-Test-Strukturen
  ✓ Dokumentation und Kommentare
  ✓ Refactoring-Vorschläge (als Input, nicht blind übernehmen)

Goldene Regel:
  Je kritischer der Code (Auth, Zahlung, Daten),
  desto gründlicher dein Review.
```

### Sicherheits-Anti-Patterns

```java
// Claude kann diese Fehler machen — IMMER prüfen:

// 1. SQL Injection (selten aber möglich)
// Schlecht (Claude könnte so generieren):
String query = "SELECT * FROM users WHERE name = '" + input + "'";

// Gut:
String query = "SELECT * FROM users WHERE name = ?";
stmt.setString(1, input);

// 2. Path Traversal
// Schlecht:
File file = new File(baseDir + "/" + userInput);

// Gut:
Path resolved = Paths.get(baseDir).resolve(userInput).normalize();
if (!resolved.startsWith(Paths.get(baseDir))) throw new SecurityException();

// 3. Unsichere Deserialisierung
// → Prüfe immer ob Jackson-Config sicher ist
// → ObjectMapper ohne enableDefaultTyping()
```

---

## Zusammenfassung LE 10

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Grenzen kennen = besser nutzen           │
│                                                         │
│  Halluzinationen:                                       │
│    → API-Details, Versionen, Zahlen immer prüfen        │
│    → Mit echten Docs im Context drastisch reduzierbar   │
│    → Code immer testen                                  │
│                                                         │
│  Safety:                                                │
│    → Hard-coded: unveränderlich                         │
│    → Default: per Kontext justierbar                    │
│    → Constitutional AI: prinzipienbasiert               │
│                                                         │
│  Grenzen:                                               │
│    → Context, Cutoff, Gedächtnis, Rechnen               │
│    → Für jede Grenze gibt es eine Lösung                │
│                                                         │
│  "Claude ist ein mächtiges Tool im Werkzeugkasten.     │
│   Dein Urteil ersetzt es nicht."                        │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 09 — Memory-Systeme](le15_memory_systeme.md)*
*Weiter: [LE 11 — Java Code Review & Refactoring](le03_java_review_refactoring.md)*
