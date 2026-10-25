# cloud-bsd — Bullshit-Detektor

Erkennt in einer Diskussion zwischen zwei Menschen **in Echtzeit Scheinargumente und Ablenkungsmanöver**
(Strohmann, Red Herring, Ad hominem, …) und blendet sie ein — zunächst lokal, später als Videokonferenz-Tool.

Technische Basis: das Projekt [Cascada](../Cascada) (Dual-Stream-Transkription mit NVIDIA Parakeet,
Event Bus, SSE-Dashboard, Claude-Anbindung).

- **Plan & Phasen:** [docs/plan.md](docs/plan.md)
- **Status:** Phase 0 — Projekt-Setup & Machbarkeit

## Setup

```bash
cp .env.example .env   # Keys eintragen, .env wird nie committet
```
