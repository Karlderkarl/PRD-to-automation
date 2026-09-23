# PRD: governance-pipeline (Claude Code Skill)

**Status:** Entwurf
**Datum:** 2026-09-22
**Zielgruppe:** Der Mensch, der das neue Repo aufsetzt; der govern-Modus des Skills selbst (diese PRD ist die Eingabe für die Governance des neuen Repos); menschliche Reviewer.
**Name:** entschieden am 2026-09-22: Repo `Karlderkarl/PRD-to-automation`, Skill `prd-to-automation` (Abschnitt 6). `governance-pipeline` im Text ist der ursprüngliche Vorschlag.

---

## 1. Problem

Die Kette "Idee zu Governance zu Automation" für Claude Code besteht heute aus zwei Skills in zwei Repos:

| Skill | Repo | Version | Aufgabe |
|---|---|---|---|
| `prd-to-governance` | Karlderkarl/prd-to-governance | 1.2.0 | Erzeugt und auditiert `SOUL.md`, `AGENTS.md`, `CLAUDE.md`, `MEMORY.md` aus einer PRD und dem Repo-Zustand |
| `governance-to-automation` | Karlderkarl/governance-to-automation | 1.2.2 | Generiert aus diesen vier Dateien ein projektspezifisches `auto-develop.sh` samt Task-Quelle, Prompt-Buildern und Logging |

Daraus folgen vier Schwächen:

1. **Ein Vertrag, zwei Quellen.** Beide Skills teilen Unsicherheitsmarker, Prioritätsstufen, die Memory-Regeln, die Skill Policy, die Testfelder `TEST_POLICY` / `TEST_ELIGIBILITY` und `TARGETED_TEST_CMD`. Jede Änderung muss in beiden Repos von Hand nachgezogen werden. Das Repo-`CLAUDE.md` von `governance-to-automation` sagt wörtlich "Keep these invariants aligned".
2. **Prozessbrüche für den Nutzer.** Zwei Installationen, zwei Namen, und Anweisungen wie "route back to `prd-to-governance`" oder "recommend `governance-to-automation`" statt eines Moduswechsels.
3. **Lange Skill-Texte.** Die `SKILL.md` beider Skills umfassen 393 beziehungsweise 242 Zeilen und werden bei jedem Trigger vollständig geladen, obwohl jeweils nur ein Teil gebraucht wird.
4. **Das Muster existiert schon, aber nicht für Claude Code.** `pi-governance-pipeline` (Version 1.2.4) bietet beide Stufen als einen Skill mit den Modi govern, automate und audit an. Es ist jedoch ein pi-Paket mit einer versionierten Engine statt eines generierten Skripts, verlangt Reviewer auf mindestens zwei Providern und hat nur einen experimentellen Claude-Code-Adapter. Für reine Claude-Code-Nutzer ist es keine Option.

## 2. Ziel

Ein einziger Claude-Code-Skill **`governance-pipeline`** in einem neuen Repo **`Karlderkarl/governance-pipeline`**, der beide Stufen als Modi anbietet, nach dem Strukturmuster von `pi-governance-pipeline`, bei **unverändertem Verhalten** der beiden Ursprungsskills.

**Nicht-Ziele:**

- Keine Engine wie in pi. Das Skript `auto-develop.sh` wird weiterhin pro Projekt generiert.
- Keine Änderung der Pipeline-Semantik: Review-Schleife, Refactor-Pass, Memory-Disziplin, Skill- und Testauflösung, Sicherheits-Defaults bleiben, wie sie in 1.2.x sind.
- Keine Zusammenlegung mit `pi-governance-pipeline`. Das bleibt das Schwesterprojekt für pi.
- Keine Unterstützung weiterer Harnesses, kein Metering, keine Registry- oder Netzwerksuche für Skills.

## 3. Architektur

### 3.1 Repository

Ein Skill-Authoring-Repo ohne Anwendungscode: Markdown plus eine Bash-Beispieldatei. Keine Laufzeitabhängigkeiten.

```
governance-pipeline/
  skills/governance-pipeline/
    SKILL.md                         kurz: Grundregeln, Modusauswahl, Boundaries
    references/
      contract.md                    der gemeinsame Vertrag, einmal definiert (siehe 4.3)
      govern.md                      Workflow des govern-Modus (heute prd-to-governance SKILL.md, Schritt 0 bis 11)
      soul-template.md
      agents-template.md
      claude-template.md
      memory-template.md
      completed-phases-template.md
      automate.md                    Workflow des automate-Modus (heute governance-to-automation SKILL.md, Schritt 0 bis 7)
      auto-develop-template.md
      prompt-builders.md
      task-list-template.md
      extraction-checklist.md
      audit.md                       Lesemodus: Governance-Drift plus Skript-Drift plus Validierung
    agents/openai.yaml               Codex-Metadaten (Anzeigename, Default-Prompt)
  examples/
    auto-develop.payload-sample.sh   Fixture, generiert für ein Node/pnpm + Payload-Projekt
    refact-todo.md                   Fixture, lokale Task-Liste (Option B)
  docs/
    PRD.md                           dieses Dokument
    parity.md                        Nachweistabelle: jede Regel der alten Checklisten und ihr neuer Ort
  README.md  CLAUDE.md  AGENTS.md  CHANGELOG.md  LICENSE  SECURITY.md  skills.sh.json
```

### 3.2 Modusmodell

| Modus | Schreibt | Entspricht heute |
|---|---|---|
| **govern** | `SOUL.md`, `AGENTS.md`, `CLAUDE.md`, `MEMORY.md`, `memory/completed-phases.md` | `prd-to-governance`: Generate und Update/Merge |
| **automate** | `auto-develop.sh`, Task-Quelle, `.gitignore`-Eintrag, Run-Guide, eine Zeile in `MEMORY.md` | `governance-to-automation`: Generate und Audit/Sync |
| **audit** | nichts | `prd-to-governance` Schritt 9 (Drei-Eimer-Bericht) plus `governance-to-automation` Audit/Sync-Bericht plus Validate (`bash -n`, `shellcheck`, `--dry-run`) |

Aufruf: `/governance-pipeline <modus> [argumente]`, zum Beispiel `/governance-pipeline govern docs/PRD.md`. Ohne genannten Modus gelten die Auswahlregeln:

| Situation | Modus |
|---|---|
| PRD vorhanden, keine der vier Governance-Dateien | govern |
| Governance vorhanden, Nutzer sagt erzeugen, aktualisieren, neu generieren | govern, mit Audit zuerst und Rückfrage pro Datei (overwrite, merge, skip) |
| Governance vollständig, kein `auto-develop.sh` | automate |
| `auto-develop.sh` vorhanden, Nutzer sagt synchronisieren, anpassen | automate (Sync) |
| Nutzer sagt prüfen, review, audit, drift, dry-run | audit |
| Governance fehlt, Nutzer will automatisieren | govern zuerst; automate startet nie ohne die vier Dateien |

Der Skill nennt den gewählten Modus in einer Zeile, bevor er arbeitet. Ein Modus lädt nur die Referenzen, die er braucht.

### 3.3 Abgrenzung zu pi-governance-pipeline

Gleiches Modusmodell, andere Mechanik. Die Unterschiede sind gewollt und werden im README benannt:

| Aspekt | governance-pipeline (Claude Code) | pi-governance-pipeline (pi) |
|---|---|---|
| Pipeline | generiertes Bash-Skript pro Projekt | versionierte Node-Engine, Wrapper im Projekt |
| Harness-Datei | `CLAUDE.md` | `SYSTEM.md` und `.pi/APPEND_SYSTEM.md` |
| Vertrag | Prosa-Abschnitte in `AGENTS.md` / `CLAUDE.md` | YAML-Block `pipeline-contract` in `AGENTS.md` |
| Review | Reviewer A/B, Fix-Schleife, Refactor-Pass | drei Reviewer, Controller, Master, Multi-Provider |
| MEMORY.md | Implement-Schritt schreibt die "Next Up"-Zeile, Memory-Schritt schreibt das Archiv | nur die Engine schreibt, Modellrollen nie |

## 4. Anforderungen

### 4.1 Skill-Struktur

- **R1** Ein Skill, drei Modi (govern, automate, audit). Die Modusauswahl folgt der Tabelle in 3.2 und wird in einer Zeile ausgegeben.
- **R2** Das Frontmatter von `SKILL.md` verwendet nur die Standardschlüssel `name`, `description`, `license` und `metadata` (mit `metadata.version`). Modus und Argumente kommen aus dem Anfragetext, nicht aus harness-spezifischen Frontmatter-Feldern.
- **R3** `SKILL.md` bleibt kurz: Ziel unter 150 Zeilen, hartes Limit 200. Die Schritt-für-Schritt-Workflows und die Qualitäts-Checklisten liegen in `references/govern.md`, `references/automate.md` und `references/audit.md`. Templates und Blueprints werden unverändert übernommen.
- **R4** Die Beschreibung im Frontmatter löst für beide Aufgabenfamilien aus (PRD zu Governance, Governance zu Pipeline, Audit von beidem) und schließt explizit aus, dass der Skill allein wegen einer vorhandenen `AGENTS.md` anspringt.
- **R5** Skill-Texte und README bleiben auf Englisch, wie die Ursprungsskills. Diese PRD und interne Notizen dürfen deutsch sein.

### 4.2 Verhaltenserhalt

- **R6** Der govern-Modus verhält sich wie `prd-to-governance` 1.2.0: Projektwurzel-Regel (ein Projekt gleich ein Ordner), Modusstatus, PRD lesen, Repo-Realität erkunden, Interview, Erzeugungsreihenfolge SOUL, AGENTS plus CLAUDE, MEMORY, Merge-Strategie, Präsentation und Bestätigung, Qualitäts-Checkliste.
- **R7** Der automate-Modus verhält sich wie `governance-to-automation` 1.2.2: Extraktion aus allen vier Dateien, Wahl genau einer Task-Quelle, Bestätigung der Parameter mit expliziter Modellwahl pro Schritt, Generierung aus dem Template, Artefakte, Validierung ohne Ausführung der echten Schleife, ein Memory-Eintrag, Audit/Sync-Bericht in den sechs Drift-Klassen.
- **R8** `auto-develop-template.md`, `prompt-builders.md`, `task-list-template.md` und `extraction-checklist.md` werden inhaltlich unverändert übernommen. Einzige erlaubte Änderung: Verweise auf den jeweils anderen Skill werden zu Modusverweisen.
- **R9** Der audit-Modus schreibt nichts. Er fasst den Governance-Drift-Bericht (PRD sagt X, Governance sagt Y; Governance sagt X, Repo tut Y; Governance schweigt) und den Skript-Drift-Bericht (Checks, Rollen und Modelle, Memory-Regeln, Konventionen, Skill Policy, Test-Policy, Vertragslücken) in einem Bericht zusammen. Änderungen erfolgen erst nach Wechsel in govern oder automate und nach Freigabe.
- **R10** Bestehende Projekte, deren Governance mit `prd-to-governance` 1.2.0 und deren Skript mit `governance-to-automation` 1.2.2 erzeugt wurde, laufen ohne Änderung weiter. Ein audit meldet allein wegen des Skill-Wechsels keinen Drift.
- **R11** Die Rückverweise zwischen den Skills werden intern: "route back to `prd-to-governance`" wird zu "wechsle in den govern-Modus", "recommend `governance-to-automation`" wird zu "wechsle in den automate-Modus". Die Boundary bleibt: der automate-Modus editiert `SOUL.md`, `AGENTS.md`, `CLAUDE.md` niemals; Korrekturen laufen durch govern.

### 4.3 Der gemeinsame Vertrag

- **R12** `references/contract.md` definiert einmal, was govern schreibt, automate liest und audit prüft. Beide Modus-Referenzen verweisen darauf statt es zu wiederholen. Inhalt:
  - die Unsicherheitsmarker `[NEEDS PRD CLARIFICATION]`, `[NEEDS CODEBASE DISCOVERY]`, `[USER DECISION REQUIRED]`, `[GOVERNANCE DRIFT]`, `[NEEDS GOVERNANCE]`;
  - die Prioritätsstufen Critical, Required, Advisory;
  - die Memory-Regeln aus 4.4;
  - das Format der AGENTS.md Skill Policy (`label:` / `title:` Matcher) und wie sie `SKILL_MAP` speist;
  - die Testfelder `TEST_POLICY`, `TEST_ELIGIBILITY` in AGENTS.md und `TARGETED_TEST_CMD` mit `{TARGET}` in CLAUDE.md;
  - für jedes Feld: optional oder Pflicht, und was bei Abwesenheit passiert (Skill Policy fehlt: `SKILL_MAP=()`; Testfelder fehlen: `off`; `required` ohne `TARGETED_TEST_CMD`: Abstufung auf `preferred` mit `[GOVERNANCE DRIFT]`; teilweise oder widersprüchliche Testfelder: `[NEEDS GOVERNANCE]`).
- **R13** `contract.md` trägt eine eigene Vertragsversion. Änderungen daran sind im CHANGELOG als Vertragsänderung ausgewiesen. Damit ist die Forderung R16 der PRD von `pi-governance-pipeline` (versionierter, dokumentierter Vertrag zwischen den beiden Stufen) für die Claude-Seite erfüllt.

### 4.4 Invarianten, die unverändert überleben müssen

Diese Regeln stammen aus den Ursprungsskills und sind Critical. Der Nachweis steht in `docs/parity.md`.

**Memory-Disziplin**

- **R14** Review-Diffs schließen `MEMORY.md` aus (`git diff <base> -- . ':!MEMORY.md'`).
- **R15** Implement- und Fix-Schritte schreiben genau eine überschriebene "Next Up"-Zeile, nie anhängend.
- **R16** Nur der Memory-Schritt nach dem Review schreibt erledigte Arbeit, und zwar nach `memory/completed-phases.md`.
- **R17** No-op-Fix-Erkennung: ändert ein Fix-Zyklus nur `MEMORY.md` oder Logs, gelten die restlichen Findings als akzeptiert und die Schleife bricht ab.
- **R18** `Depends on #N` blockiert eine Aufgabe hart, bis die Abhängigkeiten erledigt sind.
- **R19** Der Checkpoint-Commit nach dem Korrektheits-Pass setzt einen nicht leeren Code-Diff voraus; ein reiner Memory-Lauf erzeugt keinen Commit und keinen PR.

**Deterministische Auflösung**

- **R20** Skill-Auflösung nur über explizite `label:` / `title:` Matcher aus `SKILL_MAP`, ohne Dateisystem, Registry, Netzwerk oder semantische Suche. Einmal pro Aufgabe, geloggt nach `skill-resolution.log`, mehr als ein Treffer ergibt `(ambiguous)` und keine Injektion. Injektion nur in Implement-, Fix- und Refactor-Prompts.
- **R21** Testauflösung mit derselben Disziplin: `TEST_POLICY` fehlt gleich `off`; `except` schlägt `include`; kein "ambiguous"-Ergebnis; leere oder unbrauchbare Matcher ergeben `off` mit Warnung, nie "teste jede Aufgabe".
- **R22** Das Test-Gate beweist einen Rot-zu-Grün-Übergang für genau einen benannten Test. Es ist kein Regressions-Gate und wird nicht als solches beschrieben. Hart blockierend nur unter `required`; `preferred` bleibt beratend und verwirft nie Korrektheitsarbeit.
- **R23** Das modellgeschriebene `{TARGET}` wird gegen eine strikte Allowlist bereinigt, bevor es über `bash -c` läuft. Kein `eval`.

**Sicherheit und Stack-Neutralität**

- **R24** `bypassPermissions`, `danger-full-access` und Auto-Merge sind aus. Erreichbar nur über `--unattended` / `--auto-merge` hinter `confirm_privileged_mode` (listet die Privilegien, fragt `[y/N]`, übersprungen von `--dry-run` / `--yes`, verweigert ohne TTY). Die tmux-Re-Exec gibt Opt-in plus `--yes` weiter. Die privilegierten Werte stehen nirgends als Default, auch nicht in den Fixtures.
- **R25** Das generierte Skript nimmt keinen Toolchain an. Prüfbefehle kommen wörtlich aus `CLAUDE.md` in `CHECKS=()`; ein leeres Array ist ein gültiger No-op. Keine `package.json`-Guards, kein npm, pnpm, uv oder cargo als Default.
- **R26** Der Refactor-Pass läuft nur nach dem committeten Korrektheits-Checkpoint, wird über dieselbe `review_until_pass`-Schleife geprüft, behält eine Runde nur bei sauberem Re-Review und fällt sonst auf den Checkpoint zurück. Abschaltbar mit `--no-refactor`, begrenzt durch `MAX_REFACTOR_ROUNDS`.
- **R27** Die Automation schreibt nur `MEMORY.md` und die generierten Artefakte. Die Generierung führt die echte Schleife nie aus.

### 4.5 Repo und Veröffentlichung

- **R28** Neues Repo `Karlderkarl/governance-pipeline`, MIT-Lizenz, `SECURITY.md` mit GitHub Private Vulnerability Reporting, `CHANGELOG.md` nach Keep a Changelog, `skills.sh.json` mit der Gruppe "Governance".
- **R29** Die Version beginnt bei 1.0.0. Der erste CHANGELOG-Eintrag nennt die Ursprungsversionen (`prd-to-governance` 1.2.0, `governance-to-automation` 1.2.2) und ordnet jeden Ursprungsteil seinem neuen Ort zu.
- **R30** Die Beispiele aus `governance-to-automation` werden als Fixtures übernommen. Sie sind Beispielausgaben, nicht das Build-System des Repos, und werden nie ausgeführt. Der Hinweis dazu steht in `CLAUDE.md` und im README.
- **R31** Prüfungen des Repos: Skill-Validierung des Frontmatters, `bash -n` und `shellcheck` auf das Beispielskript. Kein weiterer Toolchain. Der exakte Befehl für die Skill-Validierung ist `[NEEDS CODEBASE DISCOVERY]` (das Tooling, mit dem 1.0.1 von `prd-to-governance` validiert wurde, ist im CHANGELOG nicht benannt).
  - **Ermittelt am 2026-09-22:** `quick_validate.py` aus Anthropics `skill-creator`-Skill; seine erlaubten Frontmatter-Schlüssel sind genau die der CHANGELOG-Regel von 1.0.1.
- **R32** Die beiden alten Repos bekommen ein README-Banner "superseded by Karlderkarl/governance-pipeline" mit dem neuen Installationsbefehl. Ihre Skills bleiben unter den alten Namen installierbar und werden nicht mehr weiterentwickelt. Archivierung siehe offene Entscheidungen.
- **R33** `agents/openai.yaml` wird für den neuen Namen aktualisiert (Anzeigename, Kurzbeschreibung, Default-Prompt), damit Codex den Skill weiter implizit aufrufen kann.

### 4.6 Dokumentation

- **R34** README: was der Skill ist, Installation über `npx skills add Karlderkarl/governance-pipeline`, Modus-Tabelle, typischer erster Lauf (govern, dann automate, dann Dry-Run), Beziehung zu `pi-governance-pipeline` nach 3.3, Herkunft.
- **R35** `CLAUDE.md` des Repos beschreibt Skill-Anatomie, Autoren-Konventionen (Marker, Prioritäten, "link, don't duplicate", stack-agnostisch, deterministische Auflösung, sichere Defaults) und die Prüfbefehle. Die heutige `CLAUDE.md` von `governance-to-automation` ist die Vorlage.
- **R36** `docs/parity.md` listet jede Regel aus den Qualitäts-Checklisten beider Ursprungsskills mit ihrem neuen Ort. Fehlt eine Regel, ist das ein Release-Blocker.

## 5. Phasenplan

| Phase | Inhalt | Ergebnis |
|---|---|---|
| 1 Gerüst | Repo, Ordnerstruktur, LICENSE, SECURITY.md, CHANGELOG, skills.sh.json, `docs/PRD.md` | leeres, aber vollständiges Repo |
| 2 Skill-Kern | `SKILL.md` mit Modusauswahl und Boundaries, `references/contract.md` | Skill lädt, Modi sind wählbar |
| 3 govern | `references/govern.md` aus der alten SKILL.md, fünf Templates übernehmen, Verweise umschreiben | govern verhält sich wie 1.2.0 |
| 4 automate | `references/automate.md` aus der alten SKILL.md, vier Blueprints übernehmen, Beispiele, Verweise umschreiben | automate verhält sich wie 1.2.2 |
| 5 audit | `references/audit.md` aus beiden Audit-Teilen, nur lesend | ein Bericht für beides |
| 6 Nachweis und Release | `docs/parity.md`, README, `CLAUDE.md`, `AGENTS.md`, Prüfungen, Tag v1.0.0 | veröffentlichter Skill |
| 7 Abkündigung | Banner in den alten Repos, Archivierung nach Freigabe | zwei eingefrorene Vorgänger |

Phasen 3 bis 5 sind voneinander unabhängig, sobald Phase 2 steht.

## 6. Offene Entscheidungen

- **Entschieden** Name von Repo und Skill. Vorschlag `governance-pipeline`, parallel zu `pi-governance-pipeline`. Der gleiche Skill-Name wie bei pi ist unkritisch, weil beide nie im selben Harness installiert sind.
  - **Entschieden am 2026-09-22:** Repo `Karlderkarl/PRD-to-automation`, Skill `prd-to-automation`. Alle Nennungen von `governance-pipeline` in dieser PRD sind der ursprüngliche Vorschlag; der Name bleibt dem pi-Schwesterprojekt vorbehalten.
- `[USER DECISION REQUIRED]` Archivierung der alten Repos: sofort nach v1.0.0 oder erst nach einer Übergangsfrist mit Banner.
- `[USER DECISION REQUIRED]` Historie der alten Repos importieren (`git subtree`) oder nur der Herkunftsvermerk im CHANGELOG. Vorschlag: nur der Vermerk, die Historie bleibt in den archivierten Repos erhalten.
- `[USER DECISION REQUIRED]` Veröffentlichung auf skills.sh unter dem neuen Namen. Annahme: ja.
- `[USER DECISION REQUIRED]` Die Modus-Referenzen behalten in 1.0.0 die vollständigen Checklisten der Ursprungsskills. Kürzen ist Aufgabe einer späteren Version, nachdem der Nachweis in `docs/parity.md` steht. Annahme: ja.
- **Entschieden am 2026-09-22 (Abweichung von R6 bis R8 und Kriterium 5):** Die in drei unabhängigen Reviews und einem Verhaltenstest nachgewiesenen Laufzeitfehler des Ursprungs-Templates und die zwei Template-Widersprüche (Rollback-Verbot gegen Pipeline-Rollback; sofort veraltete Statussätze) werden bereits in 1.0.0 behoben. Die Abweichungen stehen im CHANGELOG unter "Fixed" und "Changed"; keine Ursprungsregel entfällt (`docs/parity.md`).

- **Entschieden am 2026-09-23 (Version 1.1.0, Vertrag 1.1.0):** Ein Review mit End-to-End-Test (govern, automate, audit auf einem Testprojekt) und eine statische Prüfung des Templates fanden weitere Laufzeitfehler, Widersprüche zwischen den Modus-Referenzen und zu weite Regeln im Testgate. Sie werden in 1.1.0 behoben; die Vertragsänderungen stehen im CHANGELOG unter "Contract". Der sichere Standard `--permission-mode default` bleibt; statt ihn zu ändern, bekommt jede Betriebsanleitung einen Hinweis auf die Tool-Allowlist für kopflose Läufe.

## 7. Akzeptanzkriterien

1. `npx skills add Karlderkarl/governance-pipeline` installiert genau einen Skill, und das Frontmatter besteht die Validierung.
2. Jede Regel aus den Qualitäts-Checklisten von `prd-to-governance` 1.2.0 und `governance-to-automation` 1.2.2 ist in `docs/parity.md` einem Ort im neuen Skill zugeordnet.
3. `SKILL.md` hat höchstens 200 Zeilen und benennt pro Modus, welche Referenzen zu laden sind.
4. In `SKILL.md` und `references/` kommen `prd-to-governance` und `governance-to-automation` nur noch als Herkunftsangabe vor, nie als zu installierender oder aufzurufender Skill.
5. Auf einem Beispielprojekt erzeugt govern die vier Dateien mit derselben Struktur wie `prd-to-governance` 1.2.0, automate ein Skript, das `bash -n`, `shellcheck` und `--dry-run` besteht und alle Invarianten aus 4.4 enthält, und audit ändert keine Datei.
6. Ein Textsuchlauf über den Skill und die Fixtures findet `bypassPermissions`, `danger-full-access` und Squash-Merge nirgends als Default, nur hinter dem Opt-in.
7. Die alten Repos tragen das Banner, und das README des neuen Repos benennt `pi-governance-pipeline` als pi-Gegenstück mit den Unterschieden aus 3.3.
