# ECC-Integration: Review-Agents für PayCore

Diese Anleitung beschreibt, welche Teile von [ECC](https://github.com/affaan-m/ECC) PayCore für Entwicklung und Reviews nutzt, wie sie aufgerufen werden und wie sie wieder entfernt werden.

ECC ist hier ausschließlich Development Tooling. Es gibt keine Runtime-Abhängigkeit: Kein Anwendungscode importiert ECC, und es steht in keiner Dependency-Datei.

## Gewählter Ansatz

PayCore übernimmt **vier Agent-Definitionen als lokale Kopie** aus einem festen ECC-Stand nach `.claude/agents/`. Das native ECC-Plugin wird nicht installiert.

Begründung: Das Plugin lässt sich nicht auf eine kleine Auswahl beschränken. Im geprüften Stand bringt es laut eigenem Manifest 71 Agents, 314 Skills und 98 Command Shims mit, dazu eine `hooks/hooks.json` mit 24 Hook-Kommandos auf `PreToolUse`, `PostToolUse`, `SessionStart`, `Stop`, `SessionEnd` und `PreCompact`, die Claude Code bei einer Plugin-Installation automatisch lädt. Darunter sind Hooks für Memory Persistence, also genau die Art automatischer Memory-Systeme, die für diesen Einstieg nicht gewünscht ist. Die Hooks lassen sich über die Plugin-Option `hooks_enabled` abschalten, Skills, Commands und die übrigen 67 Agents kämen aber trotzdem mit. Außerdem nennt die ECC-README für den nativen Weg nur `/plugin marketplace add` und `/plugin install ecc@ecc` ohne Versionsangabe. Eine Festlegung der Plugin-Installation auf einen bestimmten Commit ist dort nicht dokumentiert und wurde hier nicht verifiziert.

Die lokale Kopie hat diese Eigenschaften:

- Der Stand ist durch Git vollständig festgelegt. Es gibt keine automatischen Updates.
- Es kommen keine Hooks, keine MCP-Server, keine Skills und keine Commands mit.
- Jeder Entwickler bekommt die Agents mit dem Checkout. Es ist keine lokale Installation nötig.

## Herkunft und Lizenz

| | |
| --- | --- |
| Quelle | https://github.com/affaan-m/ECC |
| Release | `v2.2.3` |
| Commit | `c05b2d6614f62f6db0047669aa4eefb223d478f9` |
| Lizenz | MIT, Copyright (c) 2026 Affaan Mustafa, Volltext in `docs/dev-tooling/ECC-LICENSE.txt` |
| Übernommen | `agents/planner.md`, `agents/code-reviewer.md`, `agents/security-reviewer.md`, `agents/database-reviewer.md` |

Jede übernommene Datei trägt einen Herkunftskommentar mit Release, Commit und Lizenzhinweis. Der Hinweis des ECC-Originals von `database-reviewer.md` auf Supabase Agent Skills (MIT) ist dort erhalten.

Die Dateien sind **angepasste** Kopien, keine unveränderten. Ein Vergleich mit dem Original ist so möglich:

```bash
git clone https://github.com/affaan-m/ECC /tmp/ecc && git -C /tmp/ecc checkout c05b2d6614f62f6db0047669aa4eefb223d478f9
diff /tmp/ecc/agents/security-reviewer.md .claude/agents/security-reviewer.md
```

### Anpassungen gegenüber ECC v2.2.3

| Anpassung | Grund |
| --- | --- |
| `tools` der drei Reviewer von `Read, Grep, Glob, Bash` auf `Read, Grep, Glob` reduziert | Reviewer sollen technisch nur lesen können. Mit `Bash` wäre „read-only“ nur eine Bitte. |
| `description` neu formuliert, ohne „Use PROACTIVELY“ und „MUST BE USED for all code changes“ | Agents sollen gezielt gestartet werden, nicht pauschal bei jeder Änderung. |
| Abschnitt „PayCore Context“ ergänzt | Jeder Agent liest zuerst `docs/review/paycore-review-rules.md` und zitiert Regel-IDs. |
| Abschnitt „Boundaries“ ergänzt | Nur Findings, keine Dateiänderung, keine Delegation, keine Freigabe, kein Merge. |
| Ausgabeformat auf das Finding-Format der PayCore-Regeln umgestellt, Verdict `APPROVE` entfernt | Reviewer geben nicht frei. „Keine Findings“ nennt, was geprüft und was nicht geprüft wurde. |
| JavaScript-, React-, Next.js- und Node-Abschnitte samt Beispielen entfernt, ebenso `npm audit` und `eslint` | Passt nicht zu einem Python-Service. |
| Im `database-reviewer`: Supabase-Muster `auth.uid()`, die RLS-Pflicht, `numeric` für Geld und die `psql`-Diagnosebefehle entfernt | PayCore schreibt weder RLS noch ein Money-Modell über das Tooling vor, und Reviewer bekommen keinen Datenbankzugriff. |
| Im `security-reviewer`: Abschnitt „Emergency Response“ entfernt | Er verlangt Handlungen wie Secret Rotation, die nicht Aufgabe eines lesenden Reviewers sind. |
| Im `planner`: Worked Example (Next.js und Supabase) entfernt, Felder für Meilenstein, Regel-IDs und benötigte Reviewer ergänzt | Framework-fremd. Der Planner soll Meilenstein-Lücken benennen, statt sie zu füllen. |
| Verweise auf ECC-Skills (`security-review`, `postgres-patterns`) entfernt | Diese Skills sind nicht installiert. |
| Prompt Defense Baseline gekürzt | Die Zeile, die jede Ausgabe von Code verbietet, widerspricht einem Reviewer, der Code-Stellen zitiert. |

`model` ist unverändert übernommen: `opus` für den `planner`, `sonnet` für die Reviewer.

## Versionierte Dateien und lokale Schritte

Versioniert im Repository:

| Datei | Zweck |
| --- | --- |
| `.claude/agents/*.md` | Die vier Rollen |
| `.claude/settings.json` | Verbietet Claude Code das Lesen von `.env` und `.env.local` |
| `CLAUDE.md` | Arbeitsablauf und Einsatzregeln für die Agents |
| `docs/review/paycore-review-rules.md` | Zentrale Review-Regeln und Finding-Format |
| `docs/review/smoke-test/` | Synthetische Beispiele und Ergebnis des Smoke-Tests |
| `docs/dev-tooling/ecc.md`, `docs/dev-tooling/ECC-LICENSE.txt` | Diese Anleitung und die ECC-Lizenz |
| `.github/pull_request_template.md` | Kompakte PR-Checkliste |

Lokal pro Entwickler:

- Claude Code installieren und anmelden. Geprüft wurde mit Version 2.1.325.
- Sonst nichts. ECC wird nicht installiert, weder als Plugin noch über `install.sh` oder npm.
- Wer ECC zusätzlich als Plugin im User-Scope installiert hat, bekommt Agents mit denselben Namen doppelt und dazu die ECC-Hooks. Für PayCore wird das nicht empfohlen. Welche Definition dann Vorrang hat, wurde hier nicht geprüft.

Es wurden keine persönlichen globalen Einstellungen verändert.

## Rollen

| Agent | Aufgabe | Wann verwenden | Tools |
| --- | --- | --- | --- |
| `planner` | Aufgabe, Umfang, Meilenstein und Akzeptanzkriterien klären | Bei mehrstufigen oder unklaren Änderungen | Read, Grep, Glob |
| `code-reviewer` | Korrektheit, Verständlichkeit, Regressionen | Nach Implementierungsänderungen | Read, Grep, Glob |
| `security-reviewer` | Tenant-Isolation, Authorization, Webhooks, sensible Daten | Bei sicherheitsrelevanten Änderungen | Read, Grep, Glob |
| `database-reviewer` | Constraints, Transactions, Concurrency, Ledger-Persistenz | Bei Schema-, Query- und Ledger-Änderungen | Read, Grep, Glob |

Die PayCore-Regeln stehen nur in `docs/review/paycore-review-rules.md`. Die Agents lesen diese Datei beim Start und enthalten keine Kopie davon.

## Aufruf

Verifiziert in Claude Code 2.1.325: Der Hauptagent delegiert, wenn man ihn in normaler Sprache darum bittet und den Agent beim Namen nennt. Beispiel:

```text
Beauftrage den Subagent security-reviewer mit einem Review der geänderten Dateien
src/paycore/api/charges.py und tests/test_charges.py.
Akzeptanzkriterien: Ein Tenant kann nur eigene Charges lesen.
Hier ist der Diff: <Ausgabe von git diff main...HEAD>
```

Das funktioniert in einer interaktiven Session und im Print-Modus (`claude -p "..."`). Ob die Agents geladen sind, zeigt das Feld `agents` im ersten Event von `claude -p "ok" --output-format stream-json --verbose`.

Die Agents haben keine Shell. Sie können `git diff` nicht selbst ausführen. Der Aufrufer übergibt deshalb den Diff oder die Liste der geänderten Dateien, die Akzeptanzkriterien und den nötigen Kontext im Auftrag.

Diese Integration definiert keine eigenen Slash Commands.

## Erlaubte Tools, Nebenwirkungen und Grenzen

Technisch durchgesetzt und im Smoke-Test nachgewiesen:

- Die Agents haben nur `Read`, `Grep` und `Glob`. Sie können keine Dateien schreiben, keine Shell-Befehle ausführen und keine weiteren Agents starten.
- `.claude/settings.json` verweigert das Lesen von `.env` und `.env.local`. Das gilt für die ganze Claude-Code-Session im Projekt, nicht nur für die Reviewer.

Verbleibende Grenzen:

- Die Agents können jede andere Datei im Projekt lesen. Secrets, die an anderer Stelle liegen, sind nicht geschützt. Die Deny-Liste muss erweitert werden, sobald PayCore weitere Orte für lokale Secrets bekommt.
- Der Hauptagent, der die Reviewer startet, hat weiterhin alle seine Tools. Die Beschränkung gilt nur für die Subagents.
- Ohne Shell und Datenbank bleiben Aussagen über Laufzeitverhalten, Query-Pläne und Testergebnisse Annahmen. Die Agents sind angewiesen, sie so zu kennzeichnen.
- Die Agents sind Sprachmodelle. Sie können Fehler übersehen und Fehler melden, die keine sind. Ein Review ersetzt keine automatisierten Tests.
- Es sind keine Hooks aktiv. Diese Integration führt nichts automatisch aus: keine Commits, Pushes, Deployments oder Datenbankänderungen.

## Updates

Es gibt keinen automatischen Update-Weg, und das ist Absicht. Ein Update ist ein eigener, kleiner Pull Request:

1. Neuen ECC-Release auswählen und dessen Commit notieren.
2. Die vier Originaldateien des alten und des neuen Stands vergleichen (`git diff c05b2d66..<neuer Commit> -- agents/planner.md agents/code-reviewer.md agents/security-reviewer.md agents/database-reviewer.md`).
3. Sinnvolle Änderungen von Hand in `.claude/agents/` übernehmen. Die Tabelle der Anpassungen oben bleibt dabei gültig.
4. Release und Commit in den Herkunftskommentaren und in dieser Datei aktualisieren, bei Bedarf auch `ECC-LICENSE.txt`.
5. Den Smoke-Test aus `docs/review/smoke-test/README.md` wiederholen.

## Entfernen

Die Integration besteht nur aus Dateien in diesem Repository:

```bash
git rm .claude/agents/planner.md .claude/agents/code-reviewer.md \
       .claude/agents/security-reviewer.md .claude/agents/database-reviewer.md
git rm docs/dev-tooling/ecc.md docs/dev-tooling/ECC-LICENSE.txt
git rm -r docs/review/smoke-test
```

Danach den Abschnitt „Review-Agents“ und Schritt 2 und 4 des Arbeitsablaufs in `CLAUDE.md` anpassen. `docs/review/paycore-review-rules.md`, `.github/pull_request_template.md` und `.claude/settings.json` hängen nicht von ECC ab und können bleiben. Außerhalb des Repositorys ist nichts zu bereinigen, weil nichts installiert wurde.
