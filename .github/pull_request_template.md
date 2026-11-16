## Zusammenfassung

<!-- Was ändert dieser PR und warum? Ein Thema pro PR. -->

## Meilenstein und Akzeptanzkriterien

<!-- Meilenstein und Beleg im Repository, oder "aus dem Repository nicht belegbar". -->

- [ ] 

## Entscheidungen

<!-- Welche Entscheidungen stecken in dieser Änderung, und welche Alternativen gab es? -->

## Prüfungen

<!-- Welche Tests und Checks wurden tatsächlich ausgeführt, mit welchem Ergebnis? -->

## Review-Checkliste

Regeln: `docs/review/paycore-review-rules.md`. Nicht betroffene Punkte mit „n/a“ markieren.

- [ ] **Tenant-Isolation (TEN):** Tenant aus vertrauenswürdiger Quelle, Reads und Writes eingeschränkt, Negativtest mit zwei Tenants
- [ ] **Idempotenz und Concurrency (IDEM):** Scope des Keys definiert, Schutz durch einen Datenbankmechanismus, Retry- und Crash-Fälle bedacht
- [ ] **Ledger (LED):** atomar, ausgeglichen pro Transaktion und Währung, keine Änderung gebuchter Einträge
- [ ] **Geldbeträge (MON):** keine Floats, Währung und Rundung explizit
- [ ] **Stripe-Webhooks (WH):** Signatur gegen den Raw Body vor jeder Verarbeitung, Tenant-Zuordnung, Duplikate
- [ ] **Secrets (SEC):** keine Secrets oder unnötigen sensiblen Daten in Code, Logs, Tests und Doku
- [ ] Kein sachfremder Umbau in diesem PR

## Agent-Reviews

<!-- Welche Reviewer liefen (code-reviewer, security-reviewer, database-reviewer)? Welche Findings wurden behoben, welche mit welcher Begründung verworfen? "Keine Findings" ist kein Beweis für Sicherheit. -->

## Offene Punkte

<!-- Bekannte Grenzen, Annahmen, Folgearbeiten. -->
