# cloud-bsd — Projektplan

**Stand:** 2026-10-25 · **Status:** Phase 0

## 1. Ziel

Ein Werkzeug, das in einer Diskussion zwischen **zwei Personen** gesprochene Sprache live transkribiert,
**Scheinargumente und Ablenkungsmanöver** erkennt und sie sichtbar macht:

- **Live-Modus** für die Diskutierenden: Hinweis-Overlay über dem Video, Ziel < 2 s nach Satzende.
- **Zuschauer-Modus** (später): um einige Sekunden verzögerter Stream, in dem das Scheinargument
  *genau in dem Moment* markiert wird, in dem es ausgesprochen wird (Wort-Zeitstempel, wie Cascadas Karaoke-Replay).

Ausgangslage: Cascada läuft rein offline/lokal (RTX 3070 Ti). cloud-bsd soll schrittweise online-fähig werden.

## 2. Erkennungsziele

Taxonomie nach [Quarks: Scheinargumente](https://www.quarks.de/gesellschaft/scheinargumente/).
Entscheidend für die Architektur ist, **wie viel Kontext** ein Typ zur Erkennung braucht:

| # | Typ | Kurzdefinition | Benötigter Kontext | Erwartete Schwierigkeit |
|---|-----|----------------|--------------------|------------------------|
| 1 | Ad hominem | Angriff auf die Person statt auf das Argument | Einzelsatz | niedrig |
| 2 | Ad populum | „Alle/viele glauben das“ | Einzelsatz | niedrig |
| 3 | Ad verecundiam | Fachfremde Autorität als Beleg | Einzelsatz + Weltwissen | mittel |
| 4 | Falsches Dilemma | Nur zwei Optionen werden zugelassen | Einzelsatz | mittel |
| 5 | Dammbruch (Slippery Slope) | Übertriebene Kausalkette zum Horrorszenario | 1–3 Sätze | mittel |
| 6 | Post hoc ergo propter hoc | Zeitfolge = Ursache | 1–3 Sätze | mittel |
| 7 | Ad ignorantiam | „Nicht widerlegt, also wahr“ | 1–3 Sätze | mittel |
| 8 | Tu quoque | „Du machst es doch selbst“ | Vorwurf des Gegenübers | hoch |
| 9 | Strohmann | Verzerrte Wiedergabe der Gegenposition | **Was das Gegenüber tatsächlich sagte** | hoch |
| 10 | Red Herring | Ablenkung vom eigentlichen Thema | **Thema der Diskussion** | hoch |

Konsequenz: Der Klassifikator bekommt nie nur den letzten Satz, sondern einen **Zustand** aus
Diskussionsthema + letzten Wortwechseln beider Seiten + aktuellem Satz.

## 3. Zielarchitektur (Endausbau)

```
 Browser A ──┐                         ┌── Browser B
  (Video,    │   WebRTC (LiveKit)      │   (Video,
   Overlay)  └──────────┬──────────────┘    Overlay)
                        │ Audio-Track pro Teilnehmer
                        ▼
               ┌──────────────────┐
               │  BSD-Agent       │  (LiveKit Agent = unsichtbarer Teilnehmer)
               │  ├ ASR pro Track │  Nemotron 3.5 Streaming (oder Parakeet v3)
               │  ├ Transcript    │  Sätze + Wort-Zeitstempel, Sprecher A/B
               │  ├ Detektor S1   │  Jev: 10 parallele Fragen → Wahrscheinlichkeiten
               │  └ Detektor S2   │  Claude: bestätigt + formuliert Erklärung
               └────────┬─────────┘
                        │ LiveKit Data Channel (JSON-Events)
                        ▼
                 Overlay in beiden Browsern
```

**Zweistufige Erkennung („System 1 / System 2“):**

1. **Jev (System 1)** bewertet jeden fertigen Satz im Kontext. Pro Typ eine Ja/Nein-Frage („Noul“)
   bzw. eine Choice-Frage über alle Typen. Liefert kalibrierte Wahrscheinlichkeiten, sehr schnell.
2. **Schwelle:** Unter *p_low* → verwerfen. Über *p_high* → sofort vorläufig anzeigen.
   Dazwischen → nur an S2 weiterreichen.
3. **Claude (System 2)** prüft Kandidaten, bestätigt/verwirft und liefert eine Ein-Satz-Begründung
   für die Anzeige. Bei Verwerfen wird der vorläufige Hinweis zurückgezogen.

Fehlalarme sind teurer als verpasste Treffer — ein falsches „STROHMANN!“ zerstört die Glaubwürdigkeit.
Schwellen werden deshalb auf **Präzision** optimiert.

## 4. Was aus Cascada übernommen wird

| Cascada | Verwendung in cloud-bsd |
|---|---|
| `server/model_manager.py` (Parakeet-Inferenz-Queue) | ASR-Backend „parakeet“, Phase 1–2 |
| `server/transcription_tracker.py`, `sentence_segmenter.py` | Satzbildung für Parakeet; entfällt mit Nemotron weitgehend |
| `server/event_bus.py`, `sse_handler.py` | Event-Verteilung (Phase 2), später Data Channel |
| `server/ai_connector/claude_detection.py` | Vorlage für Detektor S2 (Gesprächs-Kontext, Graceful Degradation) |
| `bridge/` (WASAPI-Capture) | Lokaler Prototyp mit zwei Mikrofonen (Phase 2) |
| Dashboard: Transkript-Spalten, Karaoke-Replay | Lokales UI (Phase 2), Zuschauer-Modus (Phase 5) |

Code wird **kopiert und angepasst**, nicht als Abhängigkeit eingebunden — Cascada bleibt unverändert.

**Lehren aus Cascada, die von Anfang an gelten:**

- `.env`, `venv*`, `__pycache__`, Logs und Aufnahmen sind in `.gitignore` und werden nie committet.
- Tests prüfen Verhalten, nicht JS-Quelltext per String-Suche.
- Iterationen werden wie in Cascada in `docs/iteration-log.md` dokumentiert.

## 5. Phasen

### Phase 0 — Setup & Machbarkeit der Erkennung (die eigentliche Risiko-Phase)

Ohne brauchbare Erkennung ist alles andere wertlos. Daher zuerst, rein textbasiert, ohne Audio.

- [ ] Jev-Zugang klären (API-Key, Preis, **Deutsch-Unterstützung**, Datenstandort, Aufbewahrung)
- [ ] Testdatensatz `data/eval/`: 60–100 deutsche Dialog-Ausschnitte (Thema + 2–6 Wortwechsel + Zielsatz),
      davon ≥ 40 % **ohne** Scheinargument; Quellen: selbst geschrieben, abgetippte Talkshow-Passagen
- [ ] Label-Schema (JSON): Typ(en), Sprecher, Begründung
- [ ] Eval-Skript: Jev vs. Claude Haiku vs. Kombination — Precision/Recall pro Typ, Latenz p50/p95, Kosten
- [ ] Prompt-/Fragen-Design für Jev (Noul pro Typ vs. eine Choice-Frage)

Datenformat eines Testfalls (`data/eval/*.jsonl`, eine Zeile pro Fall; Vorschlag, beim Umsetzen anpassbar):

```json
{
  "id": "klima-007",
  "topic": "Soll Deutschland ein Tempolimit auf Autobahnen einführen?",
  "context": [
    {"speaker": "A", "text": "Ein Tempolimit würde den CO2-Ausstoß messbar senken."},
    {"speaker": "B", "text": "Das bringt laut Umweltbundesamt nur wenige Millionen Tonnen."}
  ],
  "target": {"speaker": "A", "text": "Sie fahren doch selbst einen SUV, Sie haben gar kein Recht, da mitzureden."},
  "labels": ["tu_quoque", "ad_hominem"],
  "rationale": "Angriff auf das Verhalten der Person statt auf die Zahl des UBA."
}
```

`labels` ist leer (`[]`) für Fälle ohne Scheinargument. Typ-Bezeichner: `ad_hominem`, `ad_populum`,
`ad_verecundiam`, `false_dilemma`, `slippery_slope`, `post_hoc`, `ad_ignorantiam`, `tu_quoque`,
`strawman`, `red_herring`.

**Exit-Kriterium:** Für mindestens 5 der 10 Typen Precision ≥ 0,8 bei Recall ≥ 0,5; Jev-Latenz p95 < 300 ms.
Sonst: Taxonomie auf erkennbare Typen reduzieren oder Ansatz überdenken.

### Phase 1 — ASR-Vergleich Parakeet v3 vs. Nemotron 3.5

- [ ] Separate venv mit NeMo ≥ 26.06 (venv312 von Cascada bleibt unangetastet)
- [ ] Vergleichsskript über Cascadas 26 WAV-Aufnahmen (~34 min) + 2–3 manuell transkribierte Referenzclips
- [ ] Messen: WER (mit Referenz), Finalisierungslatenz je `att_context_size` (160/560/1120 ms),
      **Wort-Zeitstempel verfügbar?**, VRAM, Echtzeitfaktor auf der 3070 Ti

**Exit-Kriterium:** Entscheidung für ein ASR-Backend. Nemotron bevorzugt, wenn deutsche WER
akzeptabel und Wort-Zeitstempel vorhanden.

### Phase 2 — Lokaler Prototyp

- [ ] Projektgerüst: `server/` (FastAPI), `detector/`, `asr/`, `ui/`
- [ ] ASR-Backend aus Phase 1, zwei Streams (zwei Mikrofone oder Mikro + Loopback)
- [ ] Detektor-Pipeline S1 → Schwelle → S2 als Event-Bus-Abonnent
- [ ] UI: zwei Transkript-Spalten, Hinweise inline am Satz, Typ + Begründung + Konfidenz
- [ ] Diskussionsthema als Eingabe vor Start (nötig für Red Herring)
- [ ] Logging jeder Erkennung (Satz, Kontext, p-Werte, S2-Urteil) für spätere Auswertung

**Exit-Kriterium:** 10-minütige Live-Diskussion stabil; Hinweis erscheint < 2 s nach Satzende.

### Phase 3 — Online: Videokonferenz

- [ ] LiveKit-Server (zunächst LiveKit Cloud Free Tier oder selbst gehostet)
- [ ] Web-Client: zwei Teilnehmer, Video + Overlay (HTML über dem Video, nicht ins Bild gebrannt)
- [ ] BSD-Agent als LiveKit-Agent: abonniert die Audio-Tracks, sendet Erkennungen per Data Channel
- [ ] GPU bleibt lokal (3070 Ti), erreichbar per Cloudflare Tunnel oder Tailscale → keine Cloud-Kosten
- [ ] Einwilligungs-Dialog vor Beitritt (siehe Abschnitt 8)

**Exit-Kriterium:** Zwei Personen an verschiedenen Orten führen eine Diskussion mit Live-Overlay.

### Phase 4 — Cloud-Betrieb

- [ ] Container-Image für den BSD-Agent (CUDA-Basis-Image, Modell beim Build eingebacken)
- [ ] Google Compute Engine, **L4 (G2)** in EU-Region (z. B. europe-west4), Start/Stopp per Skript
- [ ] Alternative evaluieren: Cloud Run mit L4 (Scale-to-Zero; Modell-Ladezeit & Abrechnung prüfen)
- [ ] Monitoring: Latenz pro Stufe, GPU-Auslastung, Kosten pro Sitzung

**Exit-Kriterium:** Sitzung startet aus dem Browser ohne lokalen Rechner; Kosten pro Stunde bekannt.

### Phase 5 — Zuschauer-Modus

- [ ] Server-seitiges Aufzeichnen/Weiterleiten des Konferenz-Streams mit fester Verzögerung (5–10 s)
- [ ] Markierung im Takt der Sprache über Wort-Zeitstempel (Karaoke-Prinzip aus Cascada)
- [ ] Optional: Einbrennen ins Video (z. B. LiveKit Egress + Overlay-Template) für Weitergabe/Streaming

## 6. Latenzbudget (Live-Modus, Ziel < 2 s ab Satzende)

| Stufe | Parakeet (Cascada-Ansatz) | Nemotron 3.5 Streaming |
|---|---|---|
| Audio → Server (WebRTC) | ~100 ms | ~100 ms |
| Satz finalisiert | **2–5 s** (gleitendes Fenster) | ~0,1–0,6 s (je Chunk-Größe) |
| Jev (S1) | < 300 ms (Ziel, zu messen) | < 300 ms |
| Anzeige vorläufig | **≈ 2,5–5,5 s** | **≈ 0,5–1 s** |
| Claude (S2) bestätigt | + 1–2 s | + 1–2 s |

Die Spracherkennung ist der Engpass — daher Phase 1 vor Phase 2.

## 7. Kosten (grobe Richtwerte, US-Preise, vor Phase 4 für EU verifizieren)

| Posten | Annahme | Kosten |
|---|---|---|
| GPU-VM L4 on-demand | ~$0,70/h, 40 h/Monat | ~$30/Monat (+ Disk) |
| GPU-VM L4 Spot | ~$0,38/h | ~50 % günstiger, kann abgeschaltet werden |
| L4 24/7 | | ~$500/Monat — vermeiden |
| Jev | unbekannt | in Phase 0 klären |
| Claude Haiku (S2) | nur Kandidaten, kurze Prompts | gering |
| LiveKit | Free Tier / selbst gehostet | ~$0 im POC |

Kostenhebel: Phasen 0–3 auf eigener GPU; VM nur bei Bedarf; Nemotron statt Fenster-Neuberechnung
(deutlich mehr Streams pro GPU).

## 8. Recht & Datenschutz

- **§ 201 StGB (Vertraulichkeit des Wortes):** Aufnahme/Auswertung nicht-öffentlich gesprochener Worte
  nur mit **Einwilligung aller Beteiligten** → Pflicht-Dialog vor Beitritt, Einwilligung protokollieren.
- **DSGVO:** Sprache und Transkript sind personenbezogen. Bei Cloud-APIs (Jev, Claude, ggf. gehostete ASR)
  Auftragsverarbeitung, Datenstandort und Aufbewahrung klären. Eigener ASR-Server in EU-Region vereinfacht das.
- Aufnahmen und Transkripte standardmäßig **nicht** speichern; Speichern nur opt-in.
- Hinweise sind Einschätzungen einer KI, kein Urteil — so auch im UI formulieren.

## 9. Risiken

| Risiko | Auswirkung | Gegenmaßnahme |
|---|---|---|
| Erkennung zu ungenau (v. a. Strohmann, Red Herring) | Produkt unglaubwürdig | Phase 0 zuerst; Typen reduzieren; hohe Schwellen |
| Jev kann kein Deutsch / nicht verfügbar | S1 fällt weg | Fallback: nur Claude Haiku (langsamer), oder kleines lokales Klassifikationsmodell |
| Nemotron ohne Wort-Zeitstempel | Zuschauer-Modus schwierig | Parakeet für Zeitstempel nachlaufen lassen (Phase 5 ist nicht latenzkritisch) |
| NeMo-/CUDA-Abhängigkeiten (bekannt aus Cascada) | Zeitverlust | Separate venv, später Container mit festem CUDA-Image |
| Spot-VM wird mitten in Sitzung beendet | Abbruch | Für Live-Sitzungen on-demand |

## 10. Offene Fragen

1. Für wen sind die Hinweise primär — die Diskutierenden, Zuschauer oder beide?
2. Gibt es Jev-Zugang? Preis, Deutsch, Datenstandort?
3. Sprache: nur Deutsch, oder auch Englisch?
4. GitHub-Repo öffentlich oder privat?
