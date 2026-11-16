# PayCore Review-Regeln

Zentrale und verbindliche Quelle für alle Reviews an PayCore, egal ob durch einen Menschen oder durch einen Review-Agent. Die Agent-Definitionen in `.claude/agents/` enthalten diese Regeln bewusst nicht als Kopie, sondern lesen diese Datei zu Beginn jedes Reviews.

Jede Regel hat eine ID (zum Beispiel `TEN-2`). Findings nennen die verletzte Regel über diese ID.

## Geltungsbereich und Grenzen

- Die Regeln beschreiben, **was geprüft wird**. Sie legen keine Architektur fest. Maßgeblich ist immer das Modell, das im Repository tatsächlich vorhanden ist (Isolation-Konzept, Ledger-Modell, Money-Repräsentation).
- Fehlt im Repository die Grundlage für eine Prüfung (zum Beispiel weil es noch kein Ledger gibt), dann meldet der Reviewer das als offene Annahme und erfindet kein Soll-Modell.
- Ein Review ohne Findings ist kein Beweis für Sicherheit oder Korrektheit. Es ersetzt keine automatisierten Tests.

## TEN: Tenant-Isolation

- **TEN-1** Der Tenant-Kontext stammt aus einer vertrauenswürdigen, autorisierten Quelle (zum Beispiel aus den geprüften Credentials), nie aus frei wählbaren Request-Daten wie Body, Query-Parameter oder Header ohne Autorisierung.
- **TEN-2** Jeder Read und jeder Write auf mandantenbezogene Daten ist auf den Tenant eingeschränkt. Ein Zugriff nur über eine ID ohne Tenant-Bezug ist ein Finding.
- **TEN-3** Beziehungen und Constraints verhindern mandantenübergreifende Verknüpfungen (zum Beispiel ein Datensatz von Tenant A, der auf ein Objekt von Tenant B zeigt).
- **TEN-4** Background Jobs, Webhook-Handler und administrative Pfade sind berücksichtigt. Sie haben oft keinen Request-Kontext und brauchen eine eigene, nachvollziehbare Herleitung des Tenants.
- **TEN-5** Ein Stripe Connected Account ist eindeutig genau einem Tenant zugeordnet, und diese Zuordnung wird bei der Verwendung geprüft.
- **TEN-6** Es gibt Negativtests mit mindestens zwei Tenants, die zeigen, dass Tenant A nicht auf Daten von Tenant B zugreifen kann.
- **TEN-7** Nur falls Row Level Security (RLS) eingesetzt wird: Die Anwendung verbindet sich mit einer Datenbankrolle, für die die Policies tatsächlich gelten, und der Tenant-Kontext wird bei Connection Pooling pro Transaktion gesetzt und kann nicht in eine fremde Session durchsickern.

Diese Regeln schreiben keine RLS-Architektur vor. Geprüft wird zuerst das vorhandene Isolation-Konzept.

## IDEM: Idempotenz und Concurrency

- **IDEM-1** Der Scope eines Idempotency Keys ist fachlich definiert und schließt mindestens Tenant und Operation ein.
- **IDEM-2** Das Verhalten bei demselben Key mit abweichenden Request-Daten ist definiert und getestet.
- **IDEM-3** Ein Datenbankmechanismus (zum Beispiel ein Unique Constraint oder ein Lock) verhindert doppelte Verarbeitung auch bei gleichzeitigen Requests. Eine vorherige Existenzprüfung allein (check-then-act) reicht nicht aus. Der Reviewer verlangt eine konkrete Begründung des verwendeten Mechanismus.
- **IDEM-4** Wiederholte und verspätete Webhooks werden korrekt behandelt und führen nicht zu einer zweiten fachlichen Verarbeitung.
- **IDEM-5** Crash- und Retry-Fälle sind berücksichtigt: Was passiert, wenn der Prozess mitten in der Verarbeitung abbricht und der Vorgang wiederholt wird?
- **IDEM-6** Ein Fehler zwischen einer externen Stripe-Operation und dem lokalen Commit darf nicht zu einem dauerhaft inkonsistenten Zustand führen. Der Weg zurück in einen konsistenten Zustand ist beschrieben.

## LED: Double-Entry-Ledger

- **LED-1** Jede fachliche Ledger-Transaktion wird atomar gebucht: alle Einträge oder keiner.
- **LED-2** Die Balance-Invariante gilt pro fachlicher Ledger-Transaktion und pro Währung. Bei einem Signed-Amount-Modell ergibt die Summe null. Bei getrennten Debit-/Credit-Feldern stimmen die jeweiligen Summen überein. Geprüft wird nach dem vorhandenen Modell.
- **LED-3** Beträge unterschiedlicher Währungen werden nie einfach gegeneinander summiert.
- **LED-4** Alle beteiligten Accounts gehören zum zulässigen Tenant-Kontext.
- **LED-5** Gebuchte Einträge sind vor nachträglicher fachlicher Veränderung geschützt (kein Update oder Delete auf gebuchte Einträge).
- **LED-6** Korrekturen werden über nachvollziehbare Gegenbuchungen modelliert.
- **LED-7** Wiederholungen (Retries, doppelte Webhooks, doppelte Requests) erzeugen keine doppelten Buchungen.

## MON: Geldbeträge

- **MON-1** Keine binären Floats in Berechnungen oder Konvertierungspfaden für Geld. Das gilt auch für Zwischenschritte wie Parsing, Division oder Serialisierung.
- **MON-2** Geldbeträge werden einheitlich als Integer Minor Units oder als Decimal mit passender Datenbankrepräsentation geführt, nicht gemischt.
- **MON-3** Währung, Skalierung und Rundungsregeln sind explizit. Jeder Betrag trägt seine Währung.
- **MON-4** Keine pauschale Annahme, jede Währung habe zwei Nachkommastellen.

Reviews wählen kein neues Money-Modell. Geprüft wird die Einhaltung des vorhandenen.

## WH: Stripe-Webhooks

- **WH-1** Die Signatur wird gegen den unveränderten Request Body (raw bytes) geprüft, nicht gegen ein bereits geparstes und neu serialisiertes Objekt.
- **WH-2** Es wird das Secret verwendet, das zu diesem Endpoint und zu dieser Umgebung gehört.
- **WH-3** Vor erfolgreicher Signaturprüfung findet keine fachliche Verarbeitung statt.
- **WH-4** Account- und Tenant-Kontext eines Events werden zusätzlich korrekt zugeordnet. Eine gültige Signatur ersetzt keine Tenant-Zuordnung.
- **WH-5** Duplikate und nicht garantierte Event-Reihenfolgen sind berücksichtigt.
- **WH-6** Antwort- und Retry-Verhalten passen zur tatsächlichen Persistenz beziehungsweise Verarbeitung. Ein Erfolg wird erst bestätigt, wenn das Event dauerhaft gesichert oder verarbeitet ist.

## SEC: Secrets und sensible Daten

- **SEC-1** Keine Secrets (API-Keys, Webhook-Secrets, Passwörter, Tokens) in Code, Logs, Testdaten, Dokumentation oder Agent-Ausgaben.
- **SEC-2** Keine unnötigen sensiblen Daten in Logs, Fehlermeldungen oder Testdaten.
- **SEC-3** Für Validierung werden ausschließlich synthetische Daten und isolierte Testumgebungen verwendet.
- **SEC-4** Ein Reviewer, der auf ein mutmaßliches Secret stößt, nennt Datei und Stelle, gibt den Wert aber nicht wieder.

## Finding-Format

Jedes Finding enthält genau diese Angaben:

| Feld | Inhalt |
| --- | --- |
| Severity | `CRITICAL`, `HIGH`, `MEDIUM` oder `LOW` |
| Datei und Stelle | Pfad und Zeile, bei übergebenem Diff die Stelle im Diff |
| Auslösender Fall | Konkrete Eingabe, konkreter Zustand oder Ablauf, der das Problem auslöst |
| Mögliche Auswirkung | Was dann falsch läuft |
| Verletzte Regel | Regel-ID aus dieser Datei oder die allgemeine Invariante |
| Korrekturrichtung | Sinnvolle Richtung für den Fix oder der fehlende Test |

Zusätzlich gilt:

- Annahmen und Unsicherheiten werden ausdrücklich als solche markiert, getrennt von belegten Findings.
- Ein Finding ohne zitierbare Stelle und ohne konkreten auslösenden Fall wird nicht als `HIGH` oder `CRITICAL` gemeldet.
- Das Ergebnis eines Reviews ist eine Liste von Findings. Es ist keine Freigabe. Reviewer geben kein Approve und mergen nicht.
- Lautet das Ergebnis „keine Findings“, dann steht dabei, was geprüft wurde und was nicht geprüft werden konnte.
