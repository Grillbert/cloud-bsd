# CLAUDE.md — Briefing für Coding-Agents

Lies dies zuerst, dann [docs/plan.md](docs/plan.md) (Phasen, Architektur, Exit-Kriterien)
und [docs/research.md](docs/research.md) (recherchierte Fakten zu Modellen, APIs, Kosten — mit Quellen).

## Worum es geht

**cloud-bsd** („Bullshit-Detektor“) erkennt in einer Diskussion zwischen **zwei Personen** live
Scheinargumente (Strohmann, Red Herring, Ad hominem, … — 10 Typen nach Quarks) und zeigt sie an.
Sprache der Diskussionen und der UI: **Deutsch**. Code, Kommentare, Commit-Messages: Englisch.

Zweistufige Erkennung: **Jev** (TypeSafe AI, schneller Klassifikator, kein LLM) als System 1,
**Claude** als System 2 zur Bestätigung und Begründung. Fehlalarme sind schlimmer als verpasste Treffer.

## Aktueller Stand

- Repo enthält nur Planung. **Es gibt noch keinen Code.**
- Nächster Schritt: **Phase 0** — Machbarkeit der Erkennung, rein textbasiert (kein Audio, keine GPU nötig).
  Details und Datenformat in `docs/plan.md`, Abschnitt Phase 0.
- Nicht vorgreifen: keine Videokonferenz-, Cloud- oder Audio-Arbeit, bevor Phase 0 ihr Exit-Kriterium erfüllt.

## Schwesterprojekt Cascada (Code-Quelle)

`D:\Projects\Cascada` — lokales Call-Cockpit mit Dual-Stream-Transkription (funktioniert).
Code wird von dort **kopiert und angepasst**, nie importiert; Cascada selbst nicht verändern.

| Datei in Cascada | Nutzen für cloud-bsd |
|---|---|
| `server/model_manager.py` | Parakeet laden + GPU-Inferenz-Queue |
| `server/transcription_tracker.py` | Satz-Finalisierung (15-s-Sliding-Window, Wort-Zeitstempel). Komplex & fehleranfällig — mit Nemotron (Phase 1) evtl. überflüssig |
| `server/event_bus.py`, `server/sse_handler.py` | Pub/Sub + Server-Sent Events ans Browser-UI |
| `server/ai_connector/claude_detection.py` | Vorlage für Detektor S2 (Gesprächskontext, Timeouts, Graceful Degradation) |
| `bridge/capture_stream.py` | WASAPI-Mikrofon/Loopback-Aufnahme (Windows) |
| `dashboard/js/components/transcript-col.js`, `audio-player.js` | Transkript-Spalten, Karaoke-Replay per Wort-Zeitstempel |
| `docs/iteration-log.md` | Historie der Bugs/Fixes im Transkriptions-Pipeline — vor Änderungen an Satzbildung lesen |
| `recordings/*/agent.wav`, `customer.wav` | 26 WAVs (16 kHz mono, ~34 min) für den ASR-Vergleich in Phase 1 |

## Umgebung (Windows 11, RTX 3070 Ti 8 GB)

- Shell: PowerShell 5.1 und Git Bash verfügbar. `gh` ist **nicht** installiert.
- Cascada-Server: Python 3.12 (`venv312`), NeMo 2.6.2, `torch 2.7.1+cu118`. Cascada-Bridge: Python 3.14 (`venv`).
- **CUDA-Falle:** CUDA 11.8 und 13.1 sind beide installiert. Für cu118-PyTorch müssen `CUDA_PATH` und
  `CUDA_HOME` auf v11.8 zeigen (siehe Cascadas `start_server.bat`); NeMos TDT-Decoder braucht `nvrtc` zur Laufzeit.
- NeMos `model.transcribe()` liefert `Hypothesis`-Objekte → Text über `.text`.
- Für Nemotron 3.5 (braucht NeMo ≥ 26.06 bzw. transformers ≥ 5.13) eine **eigene venv** anlegen —
  Cascadas `venv312` niemals verändern.

## Regeln

- **Secrets:** Keys nur in `.env` (git-ignored), Vorlage `.env.example`. Nie committen — in Cascada ist
  genau das passiert und der Key musste widerrufen werden.
- Nicht ins Repo: venvs, `__pycache__`, Logs, Audio, Modelle (siehe `.gitignore`).
- Tests prüfen Verhalten, nicht Quelltext per String-Suche.
- Änderungen nach jeder abgeschlossenen Iteration in `docs/iteration-log.md` festhalten
  (Datum, Motivation, geänderte Dateien) — Format wie in Cascada.
- Unbekannte API-Details (insb. Jev) nicht raten: Doku prüfen oder nachfragen; Annahmen kennzeichnen.
- Datenschutz: Aufnahmen/Transkripte echter Personen standardmäßig nicht speichern (siehe Plan, Abschnitt 8).

## Offene Entscheidungen (beim Nutzer erfragen, nicht selbst festlegen)

Siehe `docs/plan.md`, Abschnitt 10 — u. a. Zielgruppe der Hinweise, Jev-Zugang, nur Deutsch oder auch Englisch.
