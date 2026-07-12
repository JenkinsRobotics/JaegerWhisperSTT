<h1 align="center">JaegerWhisperSTT</h1>

<p align="center">
  <em>The stt-slot engine module for the Jaeger ecosystem — two-pass Whisper transcription with VAD and wake word, pins JaegerOS only.</em>
</p>

<p align="center">
  <a href="https://github.com/JenkinsRobotics/JaegerWhisperSTT/releases"><img src="https://img.shields.io/badge/version-0.9.0--dev-2EA44F?style=for-the-badge" alt="Version"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-2EA44F?style=for-the-badge" alt="License"></a>
  <img src="https://img.shields.io/badge/python-3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python 3.11+">
</p>

---

## What it is

JaegerWhisperSTT is an **engine module** — the `stt` slot of the Jaeger
ecosystem. The module IS the engine: this package owns
`AudioSessionNode`, the pywhispercpp-backed Whisper engine (two
algorithmic modes), its own settings-catalog config slice, and its
`module.yaml` manifest (module/slot/version/consumes/produces/tools/
factory) — the seam
[`discover_modules()`](https://github.com/JenkinsRobotics/JaegerOS/blob/main/jaeger_os/core/modules.py)
reads to bind a slot to this module at boot.

It pins [JaegerOS](https://github.com/JenkinsRobotics/JaegerOS) **only**
— never [JaegerAI](https://github.com/JenkinsRobotics/JaegerAI) — so a
robot body can listen without the AI product installed at all. Two real
consumers today: the JaegerAI product and JP01's non-AI console.

- **Two-pass — "dual whisper"** — a fast `base.en` model gates an
  accurate `medium.en` model: the fast pass makes wake-word matching
  responsive, the accurate pass commits the final transcript. A
  `continuous` mode is also registered for the non-gated case.
- **VAD-segmented** — voice-activity detection drives when a pass runs,
  not a fixed window.
  Wake-word gating (`require_wake_word`) is opt-in per instance.
- **whisper.cpp backend** — via `pywhispercpp`, the same native engine
  that accelerates on Apple Silicon (Metal) without any Python-side
  GPU code.
- **Bus contract** — `/sense/transcript` + `/sense/user_speech_start`
  out, the `listen` tool, no inbound topics. Declared once, in
  `module.yaml` — nowhere else.

## Install

```bash
git clone https://github.com/JenkinsRobotics/JaegerWhisperSTT.git
cd JaegerWhisperSTT
pip install -e .
```

Pins `jaeger-os` (framework substrate: transport, `nodes.base`,
`core.audio`, `core.voice`, `core.instance.setting_meta`) plus
Whisper's own third-party libraries (`pywhispercpp`, `webrtcvad-wheels`,
`sounddevice`, `numpy`) — see `requirements.txt`. While staging
pre-release, the `jaeger-os` dependency is a `file://` path reference to
a sibling `JaegerOS` clone; a real version-range pin replaces it once
`jaeger-os` has published releases.

## Quick start

Prove the module contract — manifest parses, `AudioSessionNode` builds
correctly-wired on an injected bus, the bus contract round-trips —
without touching microphone hardware or loading the real Whisper model
weights:

```bash
pytest jaeger_whisper_stt/nodes/whisper_stt/tests
# or run it directly:
python -m jaeger_whisper_stt.nodes.whisper_stt.tests.test_module_contract
```

Bind the `stt` slot into a running JaegerOS instance — `discover_modules()`
finds this module automatically once it's installed (it registers itself
under the `jaeger_os.module_roots` entry-point group; JaegerOS never
imports or names this package directly):

```python
from jaeger_os.core.modules import discover_modules
modules = discover_modules()
modules["stt"]  # -> this module, factory jaeger_whisper_stt.nodes.whisper_stt:make_audio_session_node
```

Pick the STT method at the voice loop's startup: `--stt-mode two_pass`
(default) or `--stt-mode continuous`.

## Architecture

JaegerWhisperSTT is an **engine module** — the third tier in the Jaeger
ecosystem's four-tier map, pinning JaegerOS and consumed by JaegerAI or
any other JaegerOS project that needs the `stt` slot filled:

```
JaegerOS      ← the framework this repo pins. Never forked, never edited.

JaegerAI      ← the Mind — one of two real consumers of this module.
                Installs it as an optional extra (.[whisper_stt]).

Modules       ← YOU ARE HERE. stt slot. Pins JaegerOS ONLY — never
                JaegerAI — so a robot body can listen standalone.

Projects      ← JP01's non-AI console — the other real consumer.
```

See
[`JAEGER_ECOSYSTEM.md`](https://github.com/JenkinsRobotics/JaegerOS/blob/main/dev/docs/vision/JAEGER_ECOSYSTEM.md)
for the whole-ecosystem picture (module inventory, the connection rule)
and
[`THREE_TIER_STRUCTURE.md`](https://github.com/JenkinsRobotics/JaegerOS/blob/main/dev/docs/vision/THREE_TIER_STRUCTURE.md)
for the tier-map reasoning — both canonical in JaegerOS, linked here
rather than duplicated.

## Ecosystem

| Repo | Tier | What |
|---|---|---|
| [JaegerOS](https://github.com/JenkinsRobotics/JaegerOS) | Framework | Bus, node, modules/slots, supervisor, safety, contract, capability layer. This repo pins it, only it. |
| [JaegerAI](https://github.com/JenkinsRobotics/JaegerAI) | Mind (product) | Installs this module as an optional extra for voice. |
| [JaegerKokoroTTS](https://github.com/JenkinsRobotics/JaegerKokoroTTS) | Engine module (`tts` slot) | The speaking sibling — same discipline, own repo. |
| **JaegerWhisperSTT** | Engine module (`stt` slot) | This repo. |
| JP01 | Project (Body) | Consumes this module directly for its non-AI console. |

## Development

```bash
pytest jaeger_whisper_stt/nodes/whisper_stt/tests   # module-contract + engine-mode smoke (6 tests)
```

Note: a real, previously-untestable architectural coupling exists —
`AudioSessionNode`'s factory calls `ensure_tts_node()` for AEC
reference-buffer sharing, so a from-scratch standalone smoke needs the
factory path exercised carefully (the module-contract test constructs
`AudioSessionNode` directly for exactly this reason — see its
docstring). No doc in this repo describes behavior the code doesn't
implement yet (mark it `(planned)` instead) — see JaegerOS's
`CONVENTIONS.md` for the full ecosystem ruleset this module follows.

---

## License

[Apache-2.0](LICENSE) © Jenkins Robotics
