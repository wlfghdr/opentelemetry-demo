# Diff Analyse: Fork vs Upstream

## Schnellübersicht

Dieser Fork (`wlfghdr/opentelemetry-demo`) enthält **zusätzliche Observability-Driven Development (ODD) Dokumentation**, die im offiziellen Upstream-Repository (`open-telemetry/opentelemetry-demo`) nicht vorhanden ist.

### Zusätzliche Dateien im Fork

| Datei | Zeilen | Zweck |
|-------|--------|-------|
| `.github/copilot-instructions.md` | 272 | GitHub Copilot Anweisungen für ODD |
| `.kiro/steering/odd.md` | 161 | ODD Workflow und Best Practices |

### Inhalt der zusätzlichen Dateien

#### 1. `.github/copilot-instructions.md`

Diese Datei enthält detaillierte Anweisungen für GitHub Copilot, um Observability-Driven Development zu unterstützen:

- **Agentic expectations:** Wie Copilot bei Änderungen vorgehen soll
- **Repo context:** Regeln zur Erkundung des Repositories
- **ODD value proposition:** Warum ODD wichtig ist
- **Default workflow:** Schritte für sichere Änderungen
- **Risk tiers:** Bewertung von Änderungsrisiken
- **Dynatrace-assisted development:** Integration mit Telemetrie-Tools
- **Hot-path safety rules:** Schutz kritischer Pfade
- **Observability requirements:** Instrumentation-Standards

**Beispiel-Inhalt (Auszug):**
```
When the user asks to implement a change:
- Prefer taking action over theorizing
- Use a short plan for non-trivial work
- Treat observability as part of the feature
- If the request is ambiguous, ask precise clarifying questions
```

#### 2. `.kiro/steering/odd.md`

Diese Datei definiert den ODD-Workflow:

- **Default workflow:** 5-Schritte-Prozess für Änderungen
- **Risk tiers:** Kategorisierung nach Auswirkung
- **Baseline-first approach:** Telemetrie vor Änderungen erfassen
- **Guardrails:** Timeouts, Retries, Circuit Breakers
- **Instrumentation:** Spans, Metrics, Logs

**Beispiel-Inhalt (Auszug):**
```
1) Identify the critical path
2) Baseline first (before change)
3) Implement with guardrails
4) Instrument and verify
5) After-change verification
```

## Vollständige Dokumentation

Für den vollständigen Diff und alle Details siehe:

- **UPSTREAM_DIFF_SUMMARY.md** - Ausführliche Analyse
- **UPSTREAM_DIFF.patch** - Anwendbarer Git-Diff

## Empfohlene Nächste Schritte

1. **Upstream-Updates integrieren:**
   ```bash
   git checkout main
   git merge upstream/main
   ```
   
2. **ODD-Dokumentation beibehalten oder aktualisieren** nach Bedarf

3. **Bei Konflikten:** Die ODD-Dateien sind einzigartig für diesen Fork und sollten wahrscheinlich beibehalten werden

