# KG-Transporter Alert — Jak 2

<p align="center">
  <img src="https://img.shields.io/badge/OpenGOAL-Mod-blue.svg" alt="OpenGOAL Mod">
  <img src="https://img.shields.io/badge/Game-Jak%202-orange.svg" alt="Game">
  <img src="https://img.shields.io/badge/AI--assisted-Modding-purple.svg" alt="AI Assisted">
</p>

---

> [!NOTE]
> This mod moved from the `jak2/features/transport-ag/alert` branch of [whozghiar/jak-project](https://github.com/whozghiar/jak-project) to this repository. Earlier releases stay installable from the launcher catalog.

## 📖 Overview
While Haven City is on **alert (level ≥ 1)**, a **Crimson Guard Troop Transport** (`transport-ag`, the retail drop-ship) descends near the player roughly **once per minute**, deploys a squad of Crimson Guards, and departs. It is a scripted reinforcement actor tied to the city alert system.

- **Target Game:** Jak 2
- **Repository:** [`whozghiar/jak2-mod-transport-ag-alert`](https://github.com/whozghiar/jak2-mod-transport-ag-alert)

## ✨ Key Features
- **Feature:** Scripted troop drop-ship spawns 10–18 m from the player during city alerts (level ≥ 1).
- **Feature:** Realistic opening rear-hatch sequence with sound effects and squad deployment of Crimson Guards on the ground.
- **Feature:** Strict 60-second cooldown ensuring at most one transport per minute.
- **Feature:** `.fr3` merc geometry injection into `lwidea/b/c.fr3` for seamless Haven City free-roam rendering.

## 🚀 Step-by-Step Guide to Run the Mod

### 1. Select the Active Game
Make sure your environment is targeting Jak 2:
```bash
task set-game-jak2
```

### 2. Binary Compilation
- **Status:** Required (Layer 1 & Layer 2 — Decompiler & Runtime)
- **Details:** Compiles the runtime, compiler, and decompiler required for asset extraction:
```bash
task build-release-game
task build-release-decomp
```

### 3. Asset Extraction
- **Status:** Custom extraction required (Layer 2)
- **Details:** Re-run extraction to process custom assets and modified decompiler configuration (`transport-ag` injected into `lwide*.fr3`):
```bash
task extract
```

### 4. Launch the Game
Run the game natively:
```bash
task boot-game
```
*(Or iterate fast via the OpenGOAL REPL using `task repl`, then hot-reload with `(mi)` and `(r)`).*

### 5. Enable the Mod (OFF by default)
This mod ships **disabled** — a fresh install plays Haven City exactly like stock
Jak 2 (no alert drop-ship, retail story transports untouched). Open the in-game
Mods menu with **L3 + SELECT** (works in retail boot, no debug mode required):

```
Mods ▸ transport-ag-alert ▸ Enable
```

The choice persists across level reloads. Turn it off to restore vanilla traffic.

## 🎥 Demonstration Video
[![Demonstration Video](https://img.youtube.com/vi/yF5ZNcgOR10/maxresdefault.jpg)](https://youtu.be/yF5ZNcgOR10)

▶️ **[Watch the demonstration video on YouTube](https://youtu.be/yF5ZNcgOR10)**

## 📖 Technical Documentation
For the complete technical breakdown, architecture, and developer notes, refer to:
- 📄 [`docs/modding/current_mod/transport_alert_readme.md`](docs/modding/current_mod/transport_alert_readme.md)

---
*(AI-assisted)*
