# Recherche-Notizen

Stand: **2026-10-25**. Vor Entscheidungen auf Aktualität prüfen — Preise und Modell-Listen ändern sich.

## 1. Jev (TypeSafe AI) — System-1-Klassifikator

Quelle: [LangChain Blog: Building a harness with Jev](https://www.langchain.com/blog/building-a-harness-with-jev)

- „System One“-Modell, **kein LLM, generiert keinen Text**. Bewertet einen *state* und beantwortet
  typisierte Fragen mit Wahrscheinlichkeiten. Trainiert mit „RLCD“ (Reinforcement Learning for Calibrated Decisions).
- Fragetypen:
  - **Choice** — Auswahl aus Optionen, Wahrscheinlichkeit pro Option + Konfidenz
  - **Score** — Einstufung auf geordneter Skala
  - **Noul** — Ja/Nein, Wahrscheinlichkeit dass eine Aussage zutrifft
- Mehrere Fragen pro Request werden **parallel** ausgewertet → 10 Scheinargument-Typen kosten kaum mehr Latenz als einer.
- Aufruf laut Artikel: Request mit `model: "jev-latest"`, `state`, `questions`.
  LangChain-Paket `langchain-typesafe`, Klasse `TypeSafeClassifier`, Env-Var `TYPESAFE_API_KEY`.
  **Exakte Request-/Response-Struktur nicht verifiziert** — offizielle TypeSafe-Doku prüfen.
- Herstellerangabe: bis zu 200× schneller, 400× günstiger als vergleichbare LLMs bei Klassifikation.
  **Keine absoluten Latenzwerte, keine unabhängigen Benchmarks.**
- **Unbekannt:** Deutsch-Unterstützung, Preis, Datenstandort/Aufbewahrung, ob lokal betreibbar.

## 2. Taxonomie Scheinargumente

Quelle: [Quarks: Scheinargumente](https://www.quarks.de/gesellschaft/scheinargumente/) —
10 Typen mit Definition und Beispiel. Übersicht inkl. Kontextbedarf: `docs/plan.md`, Abschnitt 2.

## 3. Spracherkennung (ASR)

### Parakeet TDT 0.6B v3 (in Cascada im Einsatz)

[Model Card](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3) — Release 2025-08-14, **weiterhin das neueste
multilinguale Parakeet** (Stand Okt. 2026). 25 europäische Sprachen, CC-BY-4.0.
Deutsch WER: 5,04 % (FLEURS), 4,84 % (CoVoST) — ohne Interpunktion/Großschreibung gemessen.
Wort- und Segment-Zeitstempel: ja. Offline-Modell; Cascada simuliert Streaming über ein Sliding Window
(→ Satz-Finalisierung erst nach mehreren Sekunden).

### Nemotron 3.5 ASR Streaming 0.6B (Kandidat)

[Model Card](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b) — Release 2026-06-04.

- Cache-aware FastConformer (24 Layer) + RNN-T, ~600 M Parameter. Jeder Frame wird nur einmal verarbeitet.
- Chunk-Größen / Latenz: 80, 160, 320, 560, 1120 ms (`att_context_size`).
- 40 Sprach-Locales; **Deutsch (de-DE) „transcription-ready“**. Sprache fest oder `auto`.
- Deutsch WER (FLEURS, Sprache vorgegeben): 9,81 % (80 ms) · 9,21 (160) · 8,83 (320) · 8,42 (560) · 8,31 (1120).
  Nicht direkt vergleichbar mit Parakeets 5,04 % (unterschiedliche Normalisierung) — selbst messen (Phase 1).
- Interpunktion/Großschreibung: ja. **Wort-Zeitstempel: in der Model Card nicht erwähnt** → testen.
- Lizenz OpenMDW-1.1 (kommerziell nutzbar). Laufzeit: NeMo 26.06 (GitHub main) oder transformers ≥ 5.13.
- Durchsatz laut NVIDIA (H100): 6–17× mehr parallele Streams als Parakeet RNNT 1.1B.

### Weitere NVIDIA-Modelle (nicht relevant)

parakeet-unified-en-0.6b (2026-04, nur EN), nemotron-speech-streaming-en-0.6b (2025-12, nur EN),
multitalker-parakeet-streaming-0.6b-v1 (Sprechertrennung, EN), parakeet_realtime_eou_120m-v1 (Satzende, EN),
canary-1b-v2 (2025-08, kein Nachfolger).

## 4. Gehostete ASR-Angebote

| Anbieter | Modell | Streaming | Latenz | Preis |
|---|---|---|---|---|
| [Baseten](https://www.baseten.co/blog/introducing-nvidia-nemotron-35-asr-streaming/) | Nemotron 3.5 (NIM) | WebSocket (Riva Realtime API) | ~100 ms erstes Token, ~100–140 ms Finalisierung (eigener Benchmark, H100, bis 100 Streams) | nicht angegeben |
| [DeepInfra](https://deepinfra.com/nvidia/Nemotron-3.5-ASR-Streaming-Multilingual-0.6b) | Nemotron 3.5 Multilingual | offenbar nur Datei-Upload | – | $0,0002/min Audio, „Zero retention“, Region „Global“ |
| [build.nvidia.com](https://build.nvidia.com/nvidia/parakeet-tdt-0_6b/deploy) | Parakeet TDT v3 | nur als selbst zu betreibender NIM-Container | – | – |

Self-Hosting Nemotron als NIM-Container: [NVIDIA NIM Doku](https://docs.nvidia.com/nim/speech/latest/asr/deploy-asr-models/nemotron-asr-streaming.html).

## 5. Google Cloud GPUs

Richtwerte von Aggregatoren, meist US-Regionen —
offizielle Preise: [Google Accelerator-optimized VM Pricing](https://cloud.google.com/products/compute/pricing/accelerator-optimized).

| GPU (Maschinentyp) | VRAM | On-demand | Spot | Bewertung |
|---|---|---|---|---|
| T4 (n1 + T4) | 16 GB | ~$0,37/h | ~$0,18–0,21/h | reicht für 1–2 Sitzungen |
| **L4 (G2)** | 24 GB | ~$0,70/h (europe-west4 ~$0,74) | ~$0,38/h | **Empfehlung** |
| RTX PRO 6000 (G4) | 96 GB | ~$4,50/h (Maschine) | ~$1,79/h | überdimensioniert |
| H100 | 80 GB | ab ~$11/h | – | nur für viele Streams |

Bedarf: Parakeet 0.6B belegt in Cascada ~2,7 GB VRAM für zwei Streams.

**Cloud Run mit GPU** ([Doku](https://docs.cloud.google.com/run/docs/configuring/services/gpu)): L4 u. a. in
europe-west1/-west4; Instanzstart ~5 s (+ Modell-Ladezeit); Scale-to-Zero möglich, Doku zur GPU-Abrechnung
im Leerlauf widersprüchlich → vor Nutzung prüfen. GPU-Worker-Pools skalieren nicht automatisch.

Kostenhebel: eigene GPU per Tunnel (Cloudflare/Tailscale) für POC · VM nur bei Bedarf starten ·
Spot nur für Tests · Nemotron statt Fenster-Neuberechnung.

## 6. Videokonferenz

**LiveKit** (Open-Source-WebRTC-SFU) mit Agents-Framework: ein Agent tritt als Teilnehmer bei und
erhält die Audio-Tracks **pro Teilnehmer getrennt** → Sprecherzuordnung ohne Diarisierung.
Ergebnisse per Data Channel an die Clients, Anzeige als HTML-Overlay.
LiveKit hat Nemotron 3.5 bereits demonstriert: [LiveKit Blog](https://livekit.com/blog/nemotron-3.5-asr-multilingual-teleprompter).
