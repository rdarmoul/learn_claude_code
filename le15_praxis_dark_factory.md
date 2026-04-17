# LE 15 — Praxis: Dark Factory Migration
> Lernblock 3: Claude Code im Java-Alltag · 1 Stunde (Hands-on)

---

## Einführung

Diese Lerneinheit ist eine **Praxis-Session**. Wir wenden alles aus Block 3 (LE 11-14) auf ein reales Szenario an: die Migration einer fiktiven "Dark Factory" Java-Anwendung.

Die Ausgangssituation und Aufgaben sind in den Arbeitsergebnissen dokumentiert:
→ [aufgaben/legacy_migration_dark_factory.md](../aufgaben/legacy_migration_dark_factory.md)
→ [aufgaben/CLAUDE_template_wildfly_springboot.md](../aufgaben/CLAUDE_template_wildfly_springboot.md)

---

## Szenario: Dark Factory

```
Dark Factory = vollautomatisierte Fabrik ohne Bedienpersonal.

Legacy-System (IST):
  - Java 8, JBoss/WildFly 10, EJB 3.x, JSF 2.x
  - Monolith: 300 Klassen, 80.000 Zeilen Code
  - Keine Tests
  - Deployment: manuell per SSH + War-File
  - Uptime-Anforderung: 99.9% (Fabrik läuft 24/7)

Zielsystem (SOLL):
  - Java 17, Spring Boot 3, REST APIs, Docker
  - Microservices-Architektur (oder Modular Monolith)
  - >80% Test-Coverage
  - CI/CD mit automatischem Deployment
  - Zero-Downtime Deployment (Blue/Green)
```

---

## Session-Ablauf (60 min)

### Phase 1: Analyse (15 min)

**Aufgabe:** Nutze Claude Code um das Projekt zu analysieren.

```
Prompt 1 (Überblick):
  "Analysiere das Legacy-Projekt in /legacy-dark-factory/
   und erstelle:
   1. Architektur-Übersicht (Packages, Hauptklassen, Abhängigkeiten)
   2. Kritische Pfade (welche Klassen sind am riskantesten?)
   3. Dependency-Analyse (welche EJB/JSF Klassen müssen zuerst weg?)
   4. Schätzung: wie viele Klassen können 1:1 migriert werden?"

Prompt 2 (Risiko):
  "Identifiziere alle Stellen wo:
   - @EJB Injektionen genutzt werden
   - EntityManager direkt genutzt wird (statt Repository)
   - @Stateful Beans existieren (schwierigste Migration)
   - JSF-spezifischer Code ist (FacesContext, Managed Beans)"
```

### Phase 2: Migration-Plan erstellen (15 min)

```
Prompt:
  "Erstelle einen detaillierten Migrationsplan in Phasen.
   Berücksichtige:
   - Strangler Fig Pattern (keine Big-Bang Migration)
   - Uptime-Anforderung (99.9% → Zero-Downtime)
   - Team-Grösse: 3 Entwickler
   - Zeitrahmen: 6 Monate
   
   Format:
   Phase 1 (Monat 1-2): [Was, Warum, Risiko, Rollback-Plan]
   Phase 2 (Monat 2-4): ...
   Phase 3 (Monat 4-6): ..."
```

### Phase 3: Erste Klasse migrieren (20 min)

**Praktische Übung:** Migriere `SensorDataService` (EJB → Spring Service)

```java
// VORHER: EJB-Stil (WildFly)
@Stateless
@Local(SensorDataServiceLocal.class)
public class SensorDataServiceBean implements SensorDataServiceLocal {
    
    @PersistenceContext
    private EntityManager em;
    
    @EJB
    private MachineRegistryBean machineRegistry;
    
    public SensorReading getLatestReading(String machineId) {
        Machine machine = em.find(Machine.class, machineId);
        if (machine == null) {
            return null;  // ← Bad Practice
        }
        
        Query q = em.createQuery(
            "SELECT s FROM SensorReading s WHERE s.machine = :m " +
            "ORDER BY s.timestamp DESC"
        );
        q.setParameter("m", machine);
        q.setMaxResults(1);
        
        List results = q.getResultList();
        return results.isEmpty() ? null : (SensorReading) results.get(0);
    }
}
```

```
Prompt:
  "Migriere SensorDataServiceBean von EJB/WildFly auf Spring Boot 3 / Java 17.
   Anforderungen:
   - @Stateless → @Service
   - @PersistenceContext EntityManager → Spring Data JpaRepository
   - @EJB → @Autowired
   - null-Rückgaben → Optional
   - Raw Query → typisierte JPQL oder Spring Data Methode
   - Schreibe auch den passenden JUnit 5 Test"
```

Claude migriert zu:

```java
// NACHHER: Spring Boot 3 / Java 17
@Service
@RequiredArgsConstructor
public class SensorDataService {
    
    private final SensorReadingRepository readingRepository;
    private final MachineRegistry machineRegistry;
    
    public Optional<SensorReading> getLatestReading(String machineId) {
        return machineRegistry.findById(machineId)
            .flatMap(machine -> 
                readingRepository.findTopByMachineOrderByTimestampDesc(machine)
            );
    }
}

// Repository (Spring Data — kein manuelles JPQL nötig):
public interface SensorReadingRepository extends JpaRepository<SensorReading, Long> {
    Optional<SensorReading> findTopByMachineOrderByTimestampDesc(Machine machine);
}
```

### Phase 4: CLAUDE.md für das Migrationsprojekt (10 min)

```
Prompt:
  "Erstelle eine CLAUDE.md für das Dark-Factory Migrationsprojekt.
   Sie soll enthalten:
   - Stack (Legacy und Ziel)
   - Migration-Regeln (welche Patterns migrieren auf was)
   - Verbotene Patterns (was darf nicht mehr rein)
   - Test-Anforderungen
   - Deployment-Kontext (Zero-Downtime, Blue/Green)"
```

---

## Ergebnis-Checkliste

Nach dieser Session solltest du:

```
✓ Einen vollständigen Analyse-Report des Legacy-Projekts haben
✓ Einen Migrations-Phasenplan (6 Monate, 3 Entwickler)
✓ Mindestens eine Klasse erfolgreich migriert haben
✓ Tests für die migrierte Klasse geschrieben haben
✓ Eine CLAUDE.md für das Projekt erstellt haben

Arbeitsergebnisse ablegen in:
  aufgaben/dark_factory_analyse.md
  aufgaben/dark_factory_migrationsplan.md
  aufgaben/dark_factory_migrated_services.md
```

---

## Reflexion

```
Was haben wir in Block 3 gelernt?

LE 11: Code Review → CLAUDE.md + spezifische Prompts
LE 12: Debugging & Testing → Stacktrace-Analyse, JUnit 5 Generator
LE 13: Legacy Migration → Strangler Fig, Characterization Tests
LE 14: CI/CD → GitHub Actions, Automatisches Review
LE 15: Praxis → alles zusammen auf realem Szenario

Key Insight:
  Claude Code ist kein Zauberstab.
  Es ist ein Pair-Programming-Partner der:
  ✓ schnell analysiert
  ✓ Boilerplate schreibt
  ✓ Muster erkennt
  ✓ Optionen aufzeigt
  
  Du entscheidest:
  ✓ Architektur
  ✓ Business-Logik
  ✓ Qualitäts-Massnahmen
  ✓ Go/No-Go
```

---

*Zurück: [LE 14 — CI/CD Integration](le14_cicd_integration.md)*
*Weiter: [LE 16 — Prompt Engineering vertieft](le16_prompt_engineering_vertieft.md)*
