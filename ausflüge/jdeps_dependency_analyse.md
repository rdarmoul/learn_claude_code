# Ausflug: jdeps — Java Dependency Analyse

> Vertiefung zu Tag 2 · Kontext: Token-sparende Discovery-Phase

---

## Was ist jdeps?

`jdeps` ist ein Java-Kommandozeilentool das seit Java 8 im JDK mitgeliefert wird — kein separater Download nötig.

**Analysiert `.class` Dateien** (nicht `.java`) — also nach dem Compile-Schritt.

---

## Grundnutzung

```bash
# Einzelne Klasse analysieren
jdeps OrderService.class

# Ausgabe:
OrderService.class -> java.base
OrderService.class -> com.example.model
OrderService.class -> javax.persistence
   com.example.service.OrderService
      -> com.example.model.Order
      -> javax.persistence.EntityManager
      -> javax.persistence.PersistenceContext

# Gesamtes Projekt (alle .class Dateien)
jdeps target/classes/

# Als JSON-ähnliche Ausgabe (dot-Format für Graphen)
jdeps -dotoutput deps/ target/classes/
```

---

## Warum besser als Grep auf Sourcecode?

```
Grep auf .java:                     jdeps auf .class:
  ✗ Auskommentierte Imports           ✓ Nur echte Abhängigkeiten
  ✗ Kein Lombok-generierter Code      ✓ Auch generierter Code
  ✗ String-Referenzen nicht erkannt   ✓ Reflection-Hints
  ✓ Kein Compile nötig                ✗ Compile vorher nötig
```

---

## Token-Ersparnis in der Dark Factory

```
Ohne jdeps (Read ganzer Sourcecode):
  OrderService.java    → 2.000 Tokens
  PaymentService.java  → 1.500 Tokens
  UserRepository.java  → 800 Tokens
  Gesamt:                4.300 Tokens

Mit jdeps (nur Dependency-Output):
  jdeps output für alle 3 Klassen → ~150 Tokens
  Ersparnis: ~97%
```

---

## Integration in die Dark Factory Pipeline

```python
import subprocess
import json

def get_dependencies(class_path: str) -> dict:
    """Extrahiert Abhängigkeiten via jdeps ohne Sourcecode zu laden."""

    result = subprocess.run(
        ["jdeps", "-verbose:class", class_path],
        capture_output=True, text=True
    )

    # Output parsen → nur relevante Abhängigkeiten
    deps = []
    for line in result.stdout.splitlines():
        if "->" in line:
            dep = line.strip().split("->")[1].strip().split()[0]
            if not dep.startswith("java."):  # JDK-intern ignorieren
                deps.append(dep)

    return {"class": class_path, "dependencies": deps}

# Ergebnis:
# {
#   "class": "OrderService.class",
#   "dependencies": ["com.example.model.Order", "com.example.repository.OrderRepository"]
# }
```

---

## Verwandte Tools

| Tool | Zweck |
|------|-------|
| `jdeps` | Dependency-Analyse auf .class Ebene |
| `javap` | Dekompiliert .class zu Signaturen (ohne Implementierung) |
| `jdepend` | Paket-Level Metriken (Kopplung, Instabilität) |
| SonarQube | Statische Analyse inkl. Complexity Score |
| `grep` auf .java | Schnell, kein Compile nötig, weniger präzise |

---

*Entstanden in Tag 2 · Bezug: Dark Factory Discovery Phase*
