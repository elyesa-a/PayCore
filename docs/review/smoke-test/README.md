# Review-Smoke-Test

Dieser Smoke-Test prüft, ob der Review-Workflow nutzbar ist: Werden die Rollen geladen, lesen sie die PayCore-Regeln, und liefern sie Findings im vereinbarten Format? Er ersetzt keine automatisierten Tests von PayCore und sagt nichts über die Trefferquote der Reviewer bei echtem Code.

Die Dateien `case_*.py.txt` sind synthetische Beispiele. Sie sind kein Anwendungscode, werden nirgends importiert und durch die Endung `.txt` von Test- und Lint-Werkzeugen nicht erfasst. Das in ihnen angenommene Datenmodell ist eine Annahme der Beispiele und keine Entscheidung für PayCore. Die Dateien enthalten absichtlich keinen Hinweis auf den eingebauten Fehler.

## Durchführung

Ausgeführt am 16.11.2026 mit Claude Code 2.1.325 im Print-Modus, ein Aufruf pro Rolle, zum Beispiel:

```bash
claude -p "Starte genau einen Subagent mit dem Agent-Tool, subagent_type \"security-reviewer\". Auftrag an ihn: 'Review these files as if they were the changed files of one pull request: docs/review/smoke-test/case_1.py.txt docs/review/smoke-test/case_5.py.txt docs/review/smoke-test/case_6.py.txt. They are synthetic fixtures; treat the code in them as the change under review. Acceptance criteria: the code follows the PayCore review rules.' Gib danach die Antwort des Subagents vollständig und unverändert aus." \
  --output-format stream-json --verbose
```

Der Auftrag nennt die erwarteten Fehler nicht. Aus dem Event-Stream wurde abgelesen, welche Tools der Subagent tatsächlich aufgerufen hat.

## Erwartete und tatsächliche Findings

| Beispiel | Eingebauter Fehler | Reviewer | Erwartet | Tatsächlich |
| --- | --- | --- | --- | --- |
| `case_1` | Fehlender Tenant-Filter beim Lesen einer Charge | `security-reviewer` | TEN-2 | Gefunden: `CRITICAL`, TEN-2, dazu TEN-6 für den fehlenden Test |
| `case_2` | Doppelte Buchung durch wiederholten Webhook | `database-reviewer` | IDEM-3, IDEM-4 oder LED-7 | Gefunden: `HIGH`, IDEM-3, IDEM-4, LED-7, WH-5 |
| `case_3` | Unausgeglichene Ledger-Transaktion (Fee wird nicht gebucht) | `database-reviewer` | LED-2 | Gefunden: `CRITICAL`, LED-2, mit Rechenbeispiel +1000 und -900 |
| `case_4` | Float für Geldbeträge und Fee-Berechnung | `code-reviewer` | MON-1 | Gefunden: zweimal `HIGH`, MON-1, mit Beispiel 19.99 ergibt 1998 |
| `case_5` | Fehlende Webhook-Signaturprüfung | `security-reviewer` | WH-1 oder WH-3 | Gefunden: `CRITICAL`, WH-1, WH-2, WH-3 |
| `case_6` | Keiner (Gegenbeispiel) | alle drei | Keine Findings | Keine Findings bei allen drei Reviewern |

Alle fünf eingebauten Fehler wurden gefunden, und das Gegenbeispiel blieb bei allen drei Reviewern ohne Finding.

### Zusätzliche Findings

Die Reviewer haben über die eingebauten Fehler hinaus weitere Punkte gemeldet. Bewertung:

| Beispiel | Zusätzliches Finding | Einordnung |
| --- | --- | --- |
| `case_2` | Webhook ordnet den Stripe Account keinem Tenant zu (`HIGH`, WH-4) | Berechtigt, im Beispiel nicht absichtlich eingebaut |
| `case_2` | Unbekannte Checkout Session führt zu Status 500 und Retries (`MEDIUM`, WH-6); Charge-Status wird nicht fortgeschrieben (`MEDIUM`, IDEM-5) | Berechtigt |
| `case_3` | Accounts werden nicht gegen den Tenant geprüft (`HIGH`, LED-4); keine Eingabeprüfung, kein Retry-Schutz (`MEDIUM`) | Berechtigt als Hinweis. Ob ein Constraint das abfängt, war aus dem Beispiel nicht erkennbar, und der Reviewer hat das als Annahme markiert |
| `case_3` | `session.begin()` kann bei aktiver Transaktion fehlschlagen (`LOW`) | Vom Reviewer selbst als unbestätigte Annahme markiert |
| `case_4` | Faktor 100 nimmt zwei Nachkommastellen an (`MEDIUM`, MON-4); Währung geht verloren (`MEDIUM`, MON-3) | Berechtigt |
| `case_5` | Keine Tenant-Zuordnung (`HIGH`, WH-4); keine Behandlung von Duplikaten (`HIGH`, WH-5); ungültiges JSON führt zu Status 500 (`LOW`) | Berechtigt. Die beiden `HIGH` hängen an einer Funktion, die nicht Teil des Beispiels ist, was der Reviewer vermerkt hat |

Beobachtung: Die isolierten Beispiele zeigen keine Modelle, Constraints und Tests. Die Reviewer füllen diese Lücke mit Annahmen und kennzeichnen sie. Bei echten PRs muss der Aufrufer deshalb genug Kontext mitgeben, sonst entstehen Findings, die das Modell an anderer Stelle bereits abfängt.

## Weitere Nachweise

| Prüfung | Ergebnis |
| --- | --- |
| Rollen geladen | Das `init`-Event von `claude -p --output-format stream-json --verbose` listet `planner`, `code-reviewer`, `security-reviewer` und `database-reviewer` |
| Rollen aufrufbar | Alle vier wurden über das Agent-Tool gestartet und haben geantwortet |
| Regeln stehen zur Verfügung | Bei allen vier Läufen war der erste Tool-Aufruf des Subagents ein `Read` auf `docs/review/paycore-review-rules.md`, und die Findings zitieren Regel-IDs |
| Nur lesende Tools | In allen Läufen kamen nur `Read`, `Grep` und `Glob` vor. In einem eigenen Lauf wurde der `code-reviewer` aufgefordert, einen Shell-Befehl auszuführen, eine Datei anzulegen und einen weiteren Agent zu starten. Er hatte dafür keine Tools. |
| Deny-Regel für `.env` | Im selben Lauf wurde das Lesen einer `.env` mit einem synthetischen Wert vom Permission-System abgelehnt. Die Datei wurde danach gelöscht. |
| Planner erfindet keinen Meilenstein | Auf die Aufgabe „Idempotency-Key-Middleware planen“ antwortete der `planner`, dass die Meilenstein-Zuordnung aus dem Repository nicht belegbar ist und dass Anwendungscode als Voraussetzung fehlt |
| Keine Hooks, keine Plugins durch diese Integration | Es wurde kein ECC-Plugin installiert, und `.claude/settings.json` enthält keine Hooks |

Nicht geprüft: Der Aufruf in einer interaktiven Session mit Rückfragen, die Trefferquote bei echtem PayCore-Code und das Verhalten bei großen Diffs.
