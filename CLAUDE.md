# PayCore: Hinweise für Claude Code

PayCore ist ein mandantenfähiger Payment-Service. Diese Datei enthält nur den Arbeitsablauf und die Review-Regeln für die Entwicklung. Architektur und Meilensteininhalte stehen hier bewusst nicht: Sie gehören in eigene Dokumente im Repository, sobald sie dort abgelegt sind.

## Verbindliche Regeln

- Die PayCore Review-Regeln stehen in `docs/review/paycore-review-rules.md`. Sie gelten für jede Änderung an Tenants, Idempotenz, Ledger, Geldbeträgen, Stripe-Webhooks und Secrets.
- Keine Secrets in Code, Logs, Testdaten, Dokumentation oder Ausgaben. Für Tests nur synthetische Daten und isolierte Testumgebungen.
- Ein Pull Request hat genau ein Thema. Tooling, Refactoring und Payment-Features werden nicht gemischt.

## Arbeitsablauf für Änderungen

1. Aufgabe dem vorhandenen Meilenstein zuordnen und Akzeptanzkriterien klären. Ist die Zuordnung aus dem Repository nicht belegbar, wird das im PR gesagt.
2. Bei mehrstufigen oder unklaren Änderungen einen kurzen Plan mit dem Agent `planner` erstellen.
3. Änderung implementieren und die passenden Tests ausführen.
4. Die relevanten Reviewer beauftragen und ihnen Diff, Akzeptanzkriterien und benötigten Kontext mitgeben.
5. Findings prüfen und berechtigte Probleme beheben. Unberechtigte Findings werden mit Begründung verworfen.
6. Betroffene Checks erneut ausführen.
7. PR mit nachvollziehbarer Zusammenfassung und offenen Punkten bereitstellen (`.github/pull_request_template.md`).

## Review-Agents

Die Agents liegen in `.claude/agents/`. Herkunft, Aufruf und Grenzen stehen in `docs/dev-tooling/ecc.md`.

| Agent | Wann verwenden |
| --- | --- |
| `planner` | Bei mehrstufigen oder unklaren Änderungen, vor der Implementierung |
| `code-reviewer` | Nach Implementierungsänderungen |
| `security-reviewer` | Bei Änderungen an Tenant-Isolation, Authorization, API-Keys, Webhooks oder sensiblen Daten |
| `database-reviewer` | Bei Änderungen an Schema, Migrationen, Queries, Transactions oder Ledger-Persistenz |

Regeln für den Einsatz:

- Nur die Agents starten, die zur Änderung passen. Nicht pauschal alle Rollen ausführen.
- Die Agents haben keine Shell. Der Aufrufer übergibt den Diff oder die Liste der geänderten Dateien im Prompt.
- Die Agents liefern Findings und ändern keine Dateien. Die Umsetzung bleibt beim primären Implementierungsagenten.
- Keine rekursive Delegation: Review-Agents beauftragen keine weiteren Agents.
- Kein Agent gibt frei oder merged. „Keine Findings“ ist kein Beweis für Sicherheit und ersetzt keine Tests.
