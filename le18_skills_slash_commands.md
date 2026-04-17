# LE — Custom Slash Commands & Skill Templates
> Lernblock 4: AI-Fähigkeiten zentralisieren · 1 Stunde

---

## 1. Was sind Slash Commands? (10 min)

Slash Commands sind **wiederverwendbare Prompt-Templates** die du mit `/name` aufrufen kannst. Sie sind Markdown-Dateien — kein Code, kein Framework.

```
Ohne Slash Commands:
  Du tippst denselben langen Prompt 10× am Tag:
  "Reviewe diese Java-Klasse mit Fokus auf Null-Safety,
   Exception-Handling, Transaktionen und N+1-Queries.
   Zeige konkrete Verbesserungen als Code."

Mit Slash Command /java-review:
  Du tippst: /java-review UserService.java
  → Claude führt den vollständigen Review durch
```

### Wo leben Slash Commands?

```
Persönlich (nur du):
  ~/.claude/commands/         ← funktioniert in allen Projekten

Projekt-spezifisch (geteilt via Git):
  <projekt>/.claude/commands/ ← für alle im Team!

Priorität:
  Projekt-Commands überschreiben persönliche Commands
  → Team kann eigene Standards setzen
```

---

## 2. Einen Slash Command erstellen (15 min)

### Grundstruktur

```markdown
<!-- .claude/commands/java-review.md -->

Führe einen vollständigen Java Code Review für die angegebene Datei durch.

Analysiere mit Fokus auf:
1. **Null-Safety**: Optional statt null, fehlende null-Checks
2. **Exception-Handling**: Zu breite catches, fehlende spezifische Exceptions
3. **Transaktionen**: @Transactional korrekt gesetzt, Grenzen beachtet
4. **Performance**: N+1 Queries, fehlende Indizes, ineffiziente Streams
5. **Clean Code**: God Methods, zu tiefe Verschachtelung, magische Zahlen

Für jedes gefundene Problem:
- Zeige den problematischen Code
- Erkläre warum es ein Problem ist
- Zeige die korrigierte Version

Datei: $ARGUMENTS
```

Aufruf: `/java-review UserService.java`

Claude liest die Datei und führt den Review durch. `$ARGUMENTS` wird durch alles ersetzt was nach dem Command-Namen kommt.

### Komplexeres Beispiel mit Kontext

```markdown
<!-- .claude/commands/migrate-ejb.md -->

Migriere die angegebene EJB-Klasse auf Spring Boot 3 / Java 17.

## Migrations-Regeln (IMMER einhalten)
- @Stateless → @Service
- @Stateful  → @Service + State in DB oder Session auslagern
- @EJB       → @Autowired (Constructor Injection bevorzugen)
- @PersistenceContext EntityManager → Spring Data JpaRepository
- javax.*    → jakarta.*
- null-Rückgaben → Optional<T>

## Output-Format
1. Migrierter Code (vollständig, kompilierbar)
2. Neue Repository-Interfaces (falls benötigt)
3. JUnit 5 Tests für die migrierte Klasse
4. Kurze Liste was manuell geprüft werden muss

Eingabe-Datei: $ARGUMENTS
```

---

## 3. Argument-Patterns & Variablen (15 min)

### $ARGUMENTS — das wichtigste Konstrukt

```markdown
<!-- Alles nach dem Command-Namen wird zu $ARGUMENTS -->

/mein-command foo bar baz
→ $ARGUMENTS = "foo bar baz"

/java-review src/main/java/UserService.java --strict
→ $ARGUMENTS = "src/main/java/UserService.java --strict"
```

### Mehrere Argumente strukturieren

```markdown
<!-- .claude/commands/add-tests.md -->

Generiere JUnit 5 Tests für eine Java-Klasse.

Argumente (Format: <Datei> [Fokus]):
- $ARGUMENTS enthält den Datei-Pfad und optionalen Fokus
- Beispiel: UserService.java "exception handling"
- Beispiel: OrderService.java

## Test-Anforderungen
- @ExtendWith(MockitoExtension.class)
- Given/When/Then Struktur
- @DisplayName für Lesbarkeit
- Happy Path + alle Fehlerfälle

Ziel: $ARGUMENTS
```

### Dynamische Prompts mit Datei-Kontext

```markdown
<!-- .claude/commands/explain.md -->

Erkläre den folgenden Code für einen Senior Java-Entwickler der
NEU in diesem Projekt ist.

Fokus auf:
- Was tut es (Business-Logik, nicht nur Technik)
- Warum wurde es so implementiert (Design-Entscheidungen)
- Was könnte verbessert werden
- Wo sind potenzielle Fallstricke

Bitte lies die Datei und erkläre: $ARGUMENTS
```

---

## 4. Skill Templates für Teams bauen (15 min)

### Team-spezifische Command-Bibliothek

```
Typische Sammlung für ein Java-Backend-Team:

.claude/commands/
├── review.md           → Standard Code Review
├── java-review.md      → Java-spezifischer Review
├── migrate-ejb.md      → EJB → Spring Boot Migration
├── add-tests.md        → JUnit 5 Test-Generierung
├── fix-bug.md          → Bug-Analyse und Fix
├── explain.md          → Code erklären (für Onboarding)
├── document.md         → Javadoc generieren
├── refactor.md         → Refactoring nach SOLID
├── security-check.md   → OWASP Security Review
└── sprint-prep.md      → User Story in Tasks aufteilen
```

### Beispiel: `/sprint-prep` für Entwickler

```markdown
<!-- .claude/commands/sprint-prep.md -->

Analysiere diese User Story und bereite sie für die Entwicklung vor.

## Output
1. **Akzeptanzkriterien** (falls nicht vorhanden: ableiten)
2. **Technische Tasks** (2-4h pro Task, als Checkliste)
3. **Betroffene Klassen** (aus unserem Stack: Spring Boot 3, Java 17)
4. **Risiken & Unklarheiten** (was muss noch geklärt werden?)
5. **Schätzung** (Story Points: 1/2/3/5/8)

## Unser Stack
- Backend: Java 17, Spring Boot 3, PostgreSQL
- Frontend: Angular (falls relevant)
- CI/CD: GitLab CI, Docker, Kubernetes

User Story: $ARGUMENTS
```

### Beispiel: `/security-check` für sichere Reviews

```markdown
<!-- .claude/commands/security-check.md -->

Führe einen Security Review nach OWASP Top 10 durch.

Prüfe auf:
- A01 Broken Access Control: fehlende Autorisierungs-Checks
- A02 Cryptographic Failures: schwache/keine Verschlüsselung
- A03 Injection: SQL-, Command-, LDAP-Injection
- A04 Insecure Design: fehlende Validierung, unsichere Defaults
- A05 Security Misconfiguration: Debug-Modi, Default-Credentials
- A07 Auth Failures: schwache Sessions, keine Brute-Force-Schutz
- A08 Software Integrity: unsichere Deserialisierung

Für jeden Fund:
- Schweregrad: CRITICAL / HIGH / MEDIUM / LOW
- CWE-Nummer
- Konkreter Fix

Datei: $ARGUMENTS
```

---

## 5. Versionierung & Distribution (5 min)

```
Slash Commands = normale Dateien → Git-versioniert!

Empfohlener Workflow:
  1. Commands in .claude/commands/ anlegen
  2. In Git committen
  3. Kollegen clonen das Repo → haben sofort alle Commands

Für unternehmensweite Verteilung:
  Option A: Zentrales "AI-Toolbox" Repository
    → Teams clonen oder als Git-Submodul einbinden

  Option B: Persönliche Commands via Dotfiles
    → ~/.claude/commands/ in persönlichem Dotfiles-Repo verwalten

Wichtig: Commands enthalten keinen Code → kein Sicherheits-Review nötig
         Commands sind Plaintext → einfach zu reviewen und anzupassen
```

---

## Zusammenfassung

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Slash Commands                           │
│                                                         │
│  Command = Markdown-Datei mit Prompt-Template           │
│  $ARGUMENTS = Platzhalter für Eingabe beim Aufruf       │
│                                                         │
│  Ablage:                                                │
│    ~/.claude/commands/      → persönlich                │
│    .claude/commands/        → Projekt/Team (Git!)       │
│                                                         │
│  Aufruf: /command-name Argumente                        │
│                                                         │
│  Power-Tipp: Gute Commands = dokumentierte Team-        │
│  Standards die jeder sofort nutzen kann                 │
└─────────────────────────────────────────────────────────┘
```

---

*Weiter: [MCP — Tool-Server bauen](le19_mcp_server.md)*
