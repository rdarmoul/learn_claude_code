# Ausflug — Java Tooling Übersicht: Welches Tool wofür?

> **Kontext:** Diese Übersicht zeigt für welche Aufgabenfelder im Java-Alltag welche Tools
> besonders hilfreich sind — und wie deren Output als Input für Claude-Prompts genutzt werden kann.
>
> **Grundprinzip:** Tool-Output ist messbarer Kontext. Damit wird Symptom-First konkret:
> statt "mein Code ist langsam" gibst du einen JMH-Report — Claude analysiert statt zu raten.

---

## 1. Testing & Testqualität

### JUnit 5
Unit- und Integrationstests schreiben und strukturieren.

```
"Schreibe JUnit 5 Tests für OrderService.java.
 - @ExtendWith(MockitoExtension.class)
 - @Nested-Klassen pro Methode
 - @DisplayName für lesbare Test-Namen
 - AssertJ für Assertions
 - Teste: Happy Path, nicht gefunden, ungültige Eingabe"
```

### Mockito
Abhängigkeiten mocken, Interaktionen verifizieren.

```
"In OrderServiceTest mocke ich PaymentService und InventoryService.
 Zeige mir wie ich mit Mockito verifiziere dass:
 1. paymentService.charge() genau einmal aufgerufen wird
 2. inventoryService.reserve() nicht aufgerufen wird wenn die Zahlung fehlschlägt
 Nutze BDDMockito (given/willReturn/verify) statt Mockito.when()."
```

### AssertJ
Lesbare, verkettbare Assertions statt JUnit-Standard.

```
"Ersetze in OrderServiceTest alle assertEquals/assertTrue durch AssertJ.
 Nutze:
 - assertThat(list).hasSize(2).extracting(Order::getStatus).containsOnly(PENDING)
 - assertThatThrownBy(...).isInstanceOf(...).hasMessageContaining(...)
 - assertThat(optional).isPresent().hasValue(...)"
```

### Testcontainers
Echte Datenbank / Services im Test statt H2 In-Memory.

```
"Schreibe einen Integration-Test für OrderRepository mit Testcontainers.
 - PostgreSQL 15
 - @DataJpaTest mit @AutoConfigureTestDatabase(replace = NONE)
 - @DynamicPropertySource für Datenbankverbindung
 - Teste: save(), findByUserId(), deleteOlderThan(LocalDate)"
```

### JaCoCo
Test Coverage messen und Lücken schliessen.

JaCoCo kann auf zwei Arten mit Claude genutzt werden:

**Ansatz 1 — Assistent: Du lieferst den Report, Claude analysiert**
Du hast JaCoCo bereits ausgeführt und kopierst relevante Findings in den Prompt.

```
"JaCoCo meldet folgende nicht abgedeckten Branches in OrderService.java:
 - Zeile 47: Exception-Zweig in validateStock()
 - Zeile 89: Admin-Check in applyDiscount()
 Hier ist der Code: [Code]
 Schreibe Tests die genau diese Branches treffen."
```

Wann: Report liegt bereits vor, du weisst welche Klasse dich interessiert.

---

**Ansatz 2 — Agent: Claude führt JaCoCo selbst aus (Plan-Mode empfohlen)**
Claude führt den Maven-Befehl aus, liest den XML-Report und generiert die Tests — alles in einem Schritt.

```
"Führe mvn test jacoco:report für das aktuelle Projekt aus.
 Lies danach target/site/jacoco/jacoco.xml und identifiziere
 alle nicht abgedeckten Branches und Lines in OrderService.java.
 Liste die Lücken zuerst auf — ich bestätige bevor du Tests generierst."
```

Claude macht dann:
1. `mvn test jacoco:report` ausführen
2. `target/site/jacoco/jacoco.xml` lesen (maschinenlesbar, besser als HTML)
3. Lücken in OrderService.java identifizieren und auflisten
4. Nach Bestätigung: Tests generieren

Wann: Frischer Start, kein Report vorhanden, du willst keinen manuellen Schritt.
Plan-Mode: **ja** — Claude verändert anschliessend Dateien.

---

| | Ansatz 1 (Assistent) | Ansatz 2 (Agent) |
|--|---------------------|-----------------|
| Voraussetzung | Report liegt vor | Build läuft lokal |
| Aufwand für dich | Report lesen, relevantes kopieren | Nur Prompt schreiben |
| Kontrolle | Hoch — du filterst vorab | Mittel — Liste vor Generierung anfordern |
| Plan-Mode nötig | nein | ja |

### PIT (Pitest) — Mutation Testing
Schwache Tests identifizieren — findet Tests die zwar grün sind aber nichts wirklich prüfen.

```
"Analysiere OrderServiceTest.java mit Blick auf Mutation Testing.
 Welche Tests würden immer noch grün bleiben wenn ich:
 - In Zeile 47 das > durch >= ersetze?
 - In Zeile 89 die if-Bedingung invertiere?
 Schreibe stärkere Assertions für diese Fälle."
```

### WireMock
Externe HTTP-APIs mocken — kein echter API-Call im Test.

```
"Schreibe einen WireMock-Test für PaymentClient.charge().
 Der Client ruft POST /v1/charges auf einem externen Payment-Provider auf.
 Mocke:
 - Erfolgsfall: HTTP 200 mit { 'status': 'succeeded' }
 - Fehlerfall: HTTP 402 mit { 'error': 'insufficient_funds' }
 - Timeout: WireMock simuliert 5s Verzögerung"
```

### REST Assured
HTTP-Ebene testen — Controller bis Response.

```
"Schreibe REST Assured Tests für POST /api/orders.
 - Spring Boot Test mit @SpringBootTest(webEnvironment = RANDOM_PORT)
 - Teste: 201 Created mit Location-Header bei gültiger Order
 - Teste: 400 Bad Request wenn Pflichtfelder fehlen
 - Teste: 409 Conflict wenn Artikel nicht auf Lager
 Nutze given().body(...).when().post(...).then().statusCode(201)"
```

---

## 2. Code-Modernisierung & Automatische Migration

### OpenRewrite
Automatisierte Code-Transformation — migriert ganze Codebases regelbasiert.
OpenRewrite führt Änderungen selbst aus (Recipes), Claude hilft beim Auswählen,
Konfigurieren und Nachbearbeiten von Fällen die OpenRewrite nicht automatisch lösen kann.

**Typische Einsatzfelder:**

| Recipe | Was es macht |
|--------|-------------|
| `Java17Migration` | Java 8/11 → Java 17 (Records, sealed classes, text blocks) |
| `SpringBoot3xMigration` | Spring Boot 2 → 3 (javax → jakarta, Security-Config) |
| `JUnit5Migration` | JUnit 4 → JUnit 5 (Annotationen, Assertions, Rules) |
| `Quarkus3Migration` | Quarkus 2 → 3 |
| `UpgradeDependencyVersion` | Einzelne Library auf neue Version heben |

**Schritt 1 — OpenRewrite ausführen:**
```bash
mvn rewrite:run -Drewrite.activeRecipes=org.openrewrite.java.migrate.Java17Migration
```

**Schritt 2 — Claude für den Rest:**
```
"OpenRewrite hat unsere Codebase von Java 11 auf Java 17 migriert.
 Der Build ist grün, aber in folgenden Klassen hat OpenRewrite
 keine Änderungen vorgenommen:
 - OrderProcessor.java — nutzt noch anonymous inner classes
 - ReportBuilder.java — StringBuilder-Ketten die text blocks wären
 Hier sind die Klassen: [Code]
 Modernisiere diese manuell auf Java 17 Style (Records, text blocks,
 pattern matching instanceof, switch expressions wo sinnvoll)."
```

**Schritt 3 — Spring Boot 3 Migration, was OpenRewrite nicht kann:**
```
"Nach der OpenRewrite Spring Boot 3 Migration kompiliert der Code,
 aber folgende Tests schlagen fehl:
 - SecurityConfigTest: 'antMatchers() is no longer supported'
 - WebMvcTest: 'MockMvc setup incompatible with new SecurityFilterChain'
 Hier ist die alte SecurityConfig: [Code]
 Zeige mir die neue Spring Security 6 Konfiguration mit
 requestMatchers() und der Lambda-DSL."
```

**Claude als Recipe-Berater:**
```
"Ich will unsere Codebase von Spring Boot 2.7 auf 3.2 migrieren.
 Welche OpenRewrite Recipes soll ich in welcher Reihenfolge anwenden?
 Was kann OpenRewrite automatisch, was muss ich manuell nacharbeiten?"
```

---

## 4. Code-Qualität & Statische Analyse

### SonarLint / SonarQube
Code Smells, Bugs und Security-Findings.

```
"SonarQube meldet in UserService.java:
 - Critical: 'Possible NullPointerException' in Zeile 84
 - Major: Cognitive Complexity von 24 in processOrder() (Max: 15)
 - Minor: 'Use isEmpty() instead of size() == 0' in Zeile 102
 Hier ist der Code: [Code]
 Erkläre jeden Finding und refactore processOrder() auf Complexity ≤ 15."
```

### Checkstyle
Code-Style Regeln automatisch prüfen und beheben.

```
"Checkstyle meldet folgende Violations in OrderService.java:
 - Zeile 23: 'Line is longer than 120 characters'
 - Zeile 45: 'Missing Javadoc for public method'
 - Zeile 67: 'WhitespaceAround: '{'  not followed by whitespace'
 Behebe alle Violations ohne die Logik zu verändern."
```

### PMD
Komplexität messen, Duplikate und schlechte Patterns finden.

```
"PMD meldet für OrderService.java:
 - CyclomaticComplexity: Methode processOrder() hat Komplexität 18
 - ExcessiveMethodLength: processOrder() hat 87 Zeilen
 Hier ist der Code: [Code]
 Schlage Extract-Method Refactorings vor um die Methode aufzuteilen.
 Jede neue Methode soll Komplexität ≤ 5 haben."
```

### SpotBugs
Statische Analyse auf potenzielle Bugs zur Compile-Zeit.

```
"SpotBugs meldet in OrderService.java:
 - NP_NULL_ON_SOME_PATH: 'order' könnte null sein in Zeile 84
 - RCN_REDUNDANT_NULLCHECK_WOULD_HAVE_BEEN_A_NPE in Zeile 91
 Hier ist der Code: [Code]
 Erkläre warum SpotBugs das meldet und zeige den sicheren Fix."
```

### ArchUnit
Architektur-Regeln als ausführbare Tests definieren.

```
"Schreibe ArchUnit-Tests für unser Schichtenmodell:
 - Controller dürfen nur Services aufrufen (nicht Repositories direkt)
 - Services dürfen keine Spring MVC Annotationen haben
 - Klassen im Package 'domain' dürfen keine Spring-Abhängigkeiten haben
 - Exceptions müssen im Package 'exception' liegen
 Nutze ArchRuleDefinition.noClasses() und layeredArchitecture()."
```

---

## 5. Build & Dependency Management

### Maven
Build lifecycle, Dependency-Konflikte, Plugin-Konfiguration.

```
"mvn dependency:tree zeigt folgenden Konflikt:
 [WARNING] org.springframework:spring-core:jar:5.3.20:compile - omitted for conflict with 6.0.11
 Erkläre was dieser Konflikt bedeutet, welche Version gewinnt,
 und zeige wie ich in der pom.xml explizit die richtige Version erzwinge."
```

### Gradle
Build-Skripte verstehen und optimieren.

```
"Mein Gradle-Build braucht 4 Minuten. Hier ist mein build.gradle.kts: [Inhalt]
 Analysiere wo Zeit verloren geht und schlage vor:
 1. Welche Tasks parallelisiert werden können
 2. Wo incremental compilation hilft
 3. Ob der Build-Cache sinnvoll konfiguriert ist"
```

### jdeps
Modul-Abhängigkeiten und zirkuläre Dependencies analysieren.

```
"jdeps --print-module-deps target/myapp.jar liefert:
 java.base,java.sql,java.naming,jdk.unsupported
 Erkläre warum jdk.unsupported auftaucht (das wollte ich vermeiden)
 und wie ich die Abhängigkeit entfernen kann."
```
> Ausführliches Beispiel: [Ausflug jdeps](jdeps_dependency_analyse.md)

### OWASP Dependency-Check
Sicherheitslücken in verwendeten Libraries finden.

```
"OWASP Dependency-Check meldet:
 CVE-2021-44228 (Log4Shell) in log4j-core:2.14.1 — CVSS 10.0 Critical
 CVE-2022-22965 (Spring4Shell) in spring-webmvc:5.3.17 — CVSS 9.8 Critical
 Erkläre das Risiko für unsere Spring Boot App und zeige die pom.xml-Änderungen
 um beide CVEs zu beheben."
```

---

## 6. Performance & Profiling

### JMH
Microbenchmarks — nanosekunden-genaue Messungen einzelner Methoden.

```
"Schreibe einen JMH-Benchmark für zwei Implementierungen von parseDate():
 - Variante A: new SimpleDateFormat() pro Aufruf
 - Variante B: DateTimeFormatter (thread-safe, wiederverwendet)
 Konfiguriere: 3 Warmup-Iterationen, 5 Messiterationen, Mode.AverageTime."
```

### VisualVM
Heap, Threads und CPU live analysieren.

```
"VisualVM zeigt für unsere Spring Boot App nach 2h Laufzeit:
 - Heap: steigt von 256MB auf 1.8GB (kein GC-Abfall)
 - Sampler: 67% Zeit in HashMap.resize()
 Was sind typische Ursachen für dieses Muster?
 Welche Code-Strukturen sollte ich zuerst untersuchen?"
```

### async-profiler
CPU und Memory Flame Graphs für Produktions-Profiling.

```
"async-profiler Flame Graph zeigt: 43% der CPU-Zeit in
 com.example.OrderService.enrichOrderData() →
   → Jackson ObjectMapper.readValue() (31%)
   → ReflectionUtils.findField() (12%)
 Hier ist enrichOrderData(): [Code]
 Was verursacht den ObjectMapper-Overhead und wie beheben?"
```

### JVM Flags
GC-Verhalten und Heap-Konfiguration verstehen.

```
"Unsere App läuft mit:
 -Xms512m -Xmx2g -XX:+UseParallelGC -XX:MaxGCPauseMillis=200
 GC-Log zeigt alle 30s einen Full GC mit 800ms Pause.
 Ist G1GC oder ZGC für uns besser?
 Welche Flags empfiehlst du für eine latenz-sensitive Spring Boot App?"
```

---

## 7. Datenbankmigrationen

### Flyway
Versionierte SQL-Migrationen — geordnet, nachvollziehbar, repeatable.

```
"Ich füge der orders-Tabelle eine neue NOT NULL Spalte 'tenant_id' (UUID) hinzu.
 Schreibe ein Flyway-Migrationsskript (V5__add_tenant_id_to_orders.sql) das:
 1. Spalte als nullable hinzufügt
 2. Bestehende Rows mit einem Default-UUID befüllt
 3. NOT NULL Constraint setzt
 Die Migration muss bei laufender Produktion (zero-downtime) sicher sein."
```

### Liquibase
XML/YAML-basierte Migrationen mit Rollback-Support.

```
"Schreibe ein Liquibase-Changeset (YAML) das:
 - Eine neue Tabelle 'audit_log' mit id, entity_type, entity_id, changed_at, changed_by anlegt
 - Einen Index auf (entity_type, entity_id) setzt
 - Ein rollback-Statement enthält das die Tabelle wieder löscht
 Nutze changeSet mit author und id."
```

---

## 8. Code-Generierung & Boilerplate-Reduktion

### Lombok
Boilerplate durch Annotationen eliminieren.

```
"Konvertiere diese Order-Entity Klasse zu Lombok.
 Nutze:
 - @Data für Getter/Setter/equals/hashCode/toString
 - @Builder für den Builder-Pattern
 - @NoArgsConstructor und @AllArgsConstructor
 - @Slf4j statt private static final Logger log = ...
 Behalte die JPA-Annotationen unverändert."
```

### MapStruct
Typ-sicheres Objekt-Mapping zwischen DTO und Entity.

```
"Schreibe einen MapStruct-Mapper für Order (Entity) ↔ OrderDto (DTO).
 - OrderDto hat 'customerName' statt der separaten Customer-Entity
 - 'totalAmount' soll aus items.stream().mapToDouble(...).sum() berechnet werden
 - Ignoriere das Feld 'internalNotes' beim Mapping zu DTO
 Nutze @Mapper(componentModel = 'spring')."
```

### OpenAPI / Swagger
API-Spezifikation schreiben und Stubs generieren.

```
"Schreibe eine OpenAPI 3.0 Spezifikation (YAML) für den Order-Endpunkt:
 POST /api/v1/orders — Order anlegen
 GET  /api/v1/orders/{id} — Order abrufen
 Definiere: Request/Response-Schemas, Fehlercodes (400, 404, 409),
 und einen Bearer-Token Security Scheme."
```

---

## 9. Logging & Observability

### SLF4J + Logback
Strukturiertes Logging und MDC-Kontext.

```
"Unsere Logs sind schwer durchsuchbar. Konfiguriere Logback (logback-spring.xml)
 für strukturiertes JSON-Logging mit:
 - timestamp, level, logger, message als feste Felder
 - MDC-Felder: requestId, userId, tenantId automatisch in jedem Log-Eintrag
 - Separates Appender-Profil für Konsole (Dev) und JSON-File (Prod)"
```

### Micrometer
Custom Metriken für Prometheus / Grafana.

```
"Füge Micrometer-Metriken zu OrderService.processOrder() hinzu:
 - Counter: orders.created mit Tags status=success/failure
 - Timer: orders.processing.duration für die Gesamtdauer
 - Gauge: orders.pending.count — aktuell offene Orders
 Nutze MeterRegistry per Constructor Injection."
```

### OpenTelemetry
Distributed Tracing über Service-Grenzen hinweg.

```
"Instrumentiere OrderService und PaymentClient für OpenTelemetry Tracing.
 - Span für processOrder() mit Attributen: orderId, customerId, amount
 - Child-Span für den PaymentClient HTTP-Call
 - Bei Fehler: span.setStatus(ERROR) und Exception aufzeichnen
 Nutze die OpenTelemetry Java API (kein Vendor-Lock-in)."
```

---

## Das Wichtigste: Tool-Output → Claude-Input

```
┌──────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Tools als Claude-Kontext                  │
│                                                          │
│  Ohne Tool:  "Mein Code ist langsam, was tun?"           │
│              → Claude rät ins Blaue                      │
│                                                          │
│  Mit Tool:   JMH-Report + Code → Claude analysiert       │
│              JaCoCo-Gaps + Code → Claude schreibt Tests  │
│              SonarQube-Findings + Code → Claude fixt     │
│              OWASP-Report → Claude priorisiert CVEs      │
│                                                          │
│  Tool-Output ist messbarer Kontext — damit wird          │
│  Symptom-First konkret und überprüfbar.                  │
└──────────────────────────────────────────────────────────┘
```

---

*Entstanden in LE 04 — Java Debugging & Testing*
