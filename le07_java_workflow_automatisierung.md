# LE 07 — Java Workflow-Automatisierung: Wiederkehrende Aufgaben mit Claude
> Lernblock 1: Sofort produktiv · 1 Stunde

---

## Lernziel

Wiederkehrende Java-Entwicklungsaufgaben (Feature vorbereiten, Release schneiden,
Abhängigkeiten aktualisieren, …) durch Claude-Kommandos teilautomatisieren —
weniger Kontextwechsel, mehr Konsistenz.

---

## 1. Das Prinzip: Aufgaben-Templates (10 min)

### Warum Teilautomatisierung?

Viele Aufgaben folgen einem festen Muster, brauchen aber Kontext:
```
Feature vorbereiten:
  1. Git-Branch anlegen (feat/TICKET-123-kurzbeschreibung)
  2. JIRA-Ticket-Nummer in Commits referenzieren
  3. Changelog-Eintrag vorbereiten
  4. Skeleton-Klassen anlegen (Service, Repository, DTO, Test)
  5. TODO-Kommentare für Implementierung setzen

→ Claude kennt das Muster, du gibst nur Ticket + Beschreibung
```

### Wo liegt das Template?

Entweder als **Custom Slash Command** (`.claude/commands/`) oder direkt
als CLAUDE.md-Anweisung für wiederkehrende Prompts.

---

## 2. Typische Java-Workflows automatisieren (35 min)

### Workflow 1: Feature vorbereiten

```bash
# Prompt in Claude Code:
"Bereite ein neues Feature vor:
 Ticket: PROJ-456
 Beschreibung: Benutzer-Benachrichtigungen per E-Mail
 Package: com.example.notification

 1. Erstelle Git-Branch feat/PROJ-456-email-notifications
 2. Lege folgende Klassen an (nur Skeleton mit TODOs):
    - NotificationService.java
    - NotificationRepository.java (extends JpaRepository)
    - NotificationDto.java (als Record)
    - NotificationServiceTest.java (JUnit 5 + Mockito)
 3. Erstelle einen leeren Changelog-Eintrag in CHANGELOG.md"
```

**Ergebnis:** Branch existiert, Klassen sind angelegt, du kannst sofort
mit der Implementierung beginnen — ohne Boilerplate-Tipperei.

---

### Workflow 2: Dependency-Update durchführen

```bash
# Prompt:
"Analysiere pom.xml und prüfe welche Dependencies veraltet sind.
 Nutze mvn versions:display-dependency-updates.
 Zeige mir:
 1. Welche Updates sind sicher (Patch-Version)?
 2. Welche brauchen Vorsicht (Minor/Major)?
 3. Führe die sicheren Patch-Updates direkt durch."
```

**Warum nützlich:** Claude unterscheidet Patch/Minor/Major und aktualisiert
nur was sicher ist — du entscheidest über Breaking Changes.

---

### Workflow 3: Release vorbereiten

```bash
# Prompt:
"Bereite Release 2.3.0 vor:
 1. Prüfe ob alle Tests grün sind (mvn test)
 2. Aktualisiere die Version in pom.xml auf 2.3.0
 3. Erstelle CHANGELOG-Eintrag aus den Commits seit letztem Tag
    (git log v2.2.0..HEAD --oneline)
 4. Erstelle einen Commit 'chore: prepare release 2.3.0'
 5. Erstelle Git-Tag v2.3.0"
```

---

### Workflow 4: Code-Smell-Scan vor PR

```bash
# Prompt:
"Bevor ich diesen PR stelle, prüfe:
 1. Gibt es TODO/FIXME-Kommentare in den geänderten Dateien? (git diff main)
 2. Sind alle neuen public Methoden getestet?
 3. Gibt es offensichtliche Null-Safety-Lücken?
 4. Sind Logging-Statements auf dem richtigen Level (kein System.out.println)?
 Gib mir eine kurze Checkliste mit Status."
```

---

### Workflow 5: Boilerplate für neue Entität generieren

```bash
# Prompt:
"Erstelle vollständigen CRUD-Stack für Entität 'Product':
 - Product.java (@Entity, Felder: id, name, price, category)
 - ProductRepository.java (JpaRepository + 2 Custom Queries)
 - ProductService.java (findAll, findById, create, update, delete)
 - ProductDto.java (Record)
 - ProductMapper.java (MapStruct)
 - ProductController.java (REST, @RequestMapping('/api/products'))
 - ProductServiceTest.java (Unit Tests mit Mockito)
 Konventionen: Google Java Style, Lombok wo sinnvoll, Java 17"
```

**Zeitersparnis:** ~2 Stunden Boilerplate → ~2 Minuten.

---

## 3. Als Slash Command verankern (10 min)

Häufige Workflows einmal als Command definieren, dann immer verfügbar:

```bash
# .claude/commands/new-feature.md
Bereite ein neues Feature vor für Ticket $ARGUMENTS.

1. Extrahiere Ticket-Nummer und Beschreibung aus "$ARGUMENTS"
   (Format: "PROJ-123 Kurzbeschreibung des Features")
2. Erstelle Git-Branch: feat/{ticket-lower}-{beschreibung-kebab-case}
3. Lege Skeleton-Klassen an basierend auf dem Package-Pattern des Projekts
4. Füge leeren Changelog-Eintrag ein
5. Zeige eine Zusammenfassung was erstellt wurde
```

**Aufruf danach:**
```
/new-feature PROJ-456 E-Mail Benachrichtigungen
```

---

### Weitere sinnvolle Commands für Java-Teams

| Command | Zweck |
|---------|-------|
| `/new-feature TICKET Beschreibung` | Branch + Skeleton + Changelog |
| `/review-pr` | Pre-PR Checkliste (TODOs, Tests, Null-Safety) |
| `/update-deps` | Sichere Dependency-Updates |
| `/prep-release VERSION` | Version bumpen, Changelog, Tag |
| `/new-entity Name Felder` | Vollständiger CRUD-Stack |
| `/explain-error` | Stack Trace analysieren und Ursache erklären |

---

## 4. Grenzen der Automatisierung (5 min)

```
Claude automatisiert GUT:
  ✓ Strukturierte, wiederholbare Aufgaben mit klarem Muster
  ✓ Boilerplate nach bekannten Konventionen
  ✓ Checks und Reports (was fehlt, was ist veraltet)
  ✓ Commit-Messages und Changelogs aus Diff generieren

Claude automatisiert SCHLECHT:
  ✗ Business-Logik (kennt eure Domain nicht)
  ✗ Entscheidungen mit Seiteneffekten (z.B. welche DB-Migration sicher ist)
  ✗ Anything mit Produktionsdaten oder laufenden Systemen
  → Immer: Review before merge, nie blind pushen
```

---

## Zusammenfassung

```
┌──────────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Workflow-Automatisierung                      │
│                                                              │
│  1. Muster erkennen: Was machst du >3x pro Woche?            │
│  2. Prompt schreiben: Schritte explizit beschreiben          │
│  3. Als Slash Command speichern: einmal definieren, immer    │
│     nutzen                                                   │
│  4. Grenzen kennen: Claude macht Struktur, du machst Logik   │
│                                                              │
│  Goldene Regel: "Zeig mir was du tust, bevor du es tust"     │
│  → Claude listet Schritte, du bestätigst, dann Ausführung    │
└──────────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 06 — Praxis: Dark Factory Migration](le06_praxis_dark_factory.md)*
*Weiter: [LE 08 — Tokens, Context Window & Prompts](le07_tokens_context_prompts.md)*
