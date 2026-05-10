# Halligan CAPTCHA Solver

> Replication of **"Are CAPTCHAs Still Bot-hard?"** (USENIX Security 2025)  
> Original paper: [https://halligan.pages.dev](https://halligan.pages.dev) | Code: [code-philia/Halligan](https://github.com/code-philia/Halligan)

---

## What This Is

This repository documents our implementation and replication of **Halligan**, a generalised visual CAPTCHA solver built on a Vision Language Model (VLM). We replicated the benchmark from the USENIX Security 2025 paper, substituting the original GPT-4o backend with **Gemini 3.1 Flash-Lite** via Google's OpenAI-compatible API.

**Our results: 20 / 26 CAPTCHA types solved (76.9%) — single trial per type.**

---

## Changes From the Original

Two files were modified from the upstream repo:

**`halligan/agents/agent.py`** — swap in Gemini:
```python
# Original
def __init__(self, api_key: str, model: str = "gpt-4o-2024-11-20") -> None:
    self.client = openai.OpenAI(api_key=api_key, timeout=30)

# Ours
def __init__(self, api_key: str, model: str = "gemini-2.0-flash") -> None:
    self.client = openai.OpenAI(
        api_key=api_key,
        base_url="https://generativelanguage.googleapis.com/v1beta/openai/",
        timeout=30
    )
```

**`halligan/halligan/models/CLIP/setup.py`** — remove broken `pkg_resources` dependency:
```python
# Original uses pkg_resources which is unavailable in isolated build env
# Replace with a simple file read:
with open(os.path.join(os.path.dirname(__file__), "requirements.txt")) as f:
    requirements = [line.strip() for line in f if line.strip() and not line.startswith("#")]
```

---

## Requirements

- Linux (tested on Ubuntu 24, SSH access)
- Docker + Docker Compose
- [Pixi](https://pixi.sh) package manager
- Gemini API key (free tier works) — get one at [aistudio.google.com](https://aistudio.google.com)
- ~16 GB free disk space (Docker images + HuggingFace models)

---

## Setup & Run

### 1. Clone

```bash
git clone https://github.com/code-philia/Halligan.git
cd Halligan
```

### 2. Fix the Dockerfile (Debian Trixie compatibility)

Edit `benchmark/Dockerfile` — remove `libgl1-mesa-glx` (no longer exists in Trixie):

```dockerfile
# Remove this line:
libgl1-mesa-glx
# Keep:
libgl1
libglib2.0-0
```

### 3. Start the benchmark server

```bash
cd benchmark
docker compose up -d      # -d keeps it running after SSH disconnect
docker compose ps         # verify both containers are Up
```

You should see:
```
benchmark-browser-1   Up   0.0.0.0:5000->5000/tcp
benchmark-server-1    Up   0.0.0.0:3000->80/tcp
```

### 4. Install Pixi

```bash
curl -fsSL https://pixi.sh/install.sh | bash
source ~/.bashrc
```

### 5. Download models

```bash
cd Halligan/halligan
bash get_models.sh        # ~several GB from HuggingFace, takes a few minutes
```

### 6. Patch CLIP's setup.py

```bash
cat > halligan/models/CLIP/setup.py << 'EOF'
import os
from setuptools import setup, find_packages

with open(os.path.join(os.path.dirname(__file__), "requirements.txt")) as f:
    requirements = [line.strip() for line in f if line.strip() and not line.startswith("#")]

setup(
    name="clip",
    py_modules=["clip"],
    version="1.0",
    description="",
    author="OpenAI",
    packages=find_packages(exclude=["tests*"]),
    install_requires=requirements,
    include_package_data=True,
    extras_require={'dev': ['pytest']},
)
EOF
```

### 7. Install Python environment

```bash
pixi install
```

### 8. Configure environment

```bash
cp .env.example .env
nano .env
```

Set:
```env
OPENAI_API_KEY=your-gemini-api-key-here
BROWSER_URL=ws://localhost:5000
BENCHMARK_URL=http://benchmark
```

### 9. Patch agent.py for Gemini

```bash
nano halligan/agents/agent.py
```

Change `GPTAgent.__init__`:
```python
def __init__(self, api_key: str, model: str = "gemini-2.0-flash") -> None:
    self.model = model
    self.client = openai.OpenAI(
        api_key=api_key,
        base_url="https://generativelanguage.googleapis.com/v1beta/openai/",
        timeout=30
    )
```

### 10. Run

```bash
# Generate solver scripts for all 26 CAPTCHA types (~15 min)
pixi run env -u PYTHONPATH -u ROS_DISTRO -u AMENT_PREFIX_PATH -u AMENT_CURRENT_PREFIX python generate.py

# Execute and benchmark
pixi run env -u PYTHONPATH -u ROS_DISTRO -u AMENT_PREFIX_PATH -u AMENT_CURRENT_PREFIX python execute.py
```

> **Tip for SSH users:** Run inside `tmux` so the session survives disconnects:
> ```bash
> tmux new -s halligan
> # run commands inside tmux
> # Ctrl+B then D to detach, `tmux attach -t halligan` to reattach
> ```

---

## Results

Run date: **2026-05-08** | Duration: **~16 minutes**

| # | CAPTCHA Type | Provider | Paper (GPT-4o) | Ours (Gemini) |
|---|---|---|---|---|
| 1 | lemin | Lemin | 13% | ✗ |
| 2 | geetest/slide | GeeTest | 16% | **✓** |
| 3 | geetest/gobang | GeeTest | 92% | **✓** |
| 4 | geetest/icon | GeeTest | 46% | ✗ |
| 5 | geetest/iconcrush | GeeTest | 98% | **✓** |
| 6 | baidu | Baidu | 71% | **✓** |
| 7 | hcaptcha | hCaptcha | 82% | **✓** |
| 8 | botdetect | BotDetect | 84% | **✓** |
| 9 | arkose/multichoice/square_icon | Arkose Labs | 86% | **✓** |
| 10 | arkose/multichoice/galaxies | Arkose Labs | 100% | **✓** |
| 11 | arkose/multichoice/dice_pair | Arkose Labs | 43% | **✓** |
| 12 | arkose/multichoice/hand_number | Arkose Labs | 49% | ✗ |
| 13 | arkose/multichoice/card | Arkose Labs | 82% | **✓** |
| 14 | arkose/multichoice/counting | Arkose Labs | 54% | **✓** |
| 15 | arkose/multichoice/rotated | Arkose Labs | 95% | **✓** |
| 16 | arkose/paged/dice_match | Arkose Labs | 71% | ✗ (timeout) |
| 17 | arkose/paged/rockstack | Arkose Labs | 58% | ✗ |
| 18 | arkose/paged/numbermatch | Arkose Labs | 54% | **✓** |
| 19 | arkose/paged/orbit_match_game | Arkose Labs | 48% | **✓** |
| 20 | arkose/paged/3d_rollball_objects | Arkose Labs | 20% | **✓** |
| 21 | mtcaptcha | MTCaptcha | 66% | **✓** |
| 22 | recaptchav2 | Google | 68% | **✓** |
| 23 | tencent | Tencent | 23% | **✓** |
| 24 | yandex/text | Yandex | 96% | **✓** |
| 25 | yandex/kaleidoscope | Yandex | 47% | ✗ |
| 26 | amazon | Amazon WAF | 14% | **✓** |

**20 / 26 solved (76.9%)** on a single trial per type.

Paper's result: **60.7%** averaged over 100 trials per type with GPT-4o.

---

## Checking Results

```bash
# View all results
grep -E "Testing|Solved" halligan/agent-*.log

# Count solved
grep "Solved: True" halligan/agent-*.log | wc -l

# Save to file
grep -E "Testing|Solved" halligan/agent-*.log > results/summary.txt

# View notebook traces (open in Jupyter)
pixi run jupyter notebook --no-browser --port=8888 --ip=0.0.0.0
# Then open http://<server-ip>:8888 → results/execute/
```

---

## File Structure

```
Halligan/
├── benchmark/
│   ├── docker-compose.yml        # starts server + browser containers
│   └── Dockerfile                # patched: removed libgl1-mesa-glx
│
└── halligan/
    ├── generate.py               # Stage 1-3: generate solver scripts
    ├── execute.py                # run scripts, record pass/fail
    ├── basic_test.py             # sanity check
    ├── halligan/
    │   ├── agents/agent.py       # PATCHED: Gemini backend
    │   └── models/CLIP/setup.py  # PATCHED: removed pkg_resources
    ├── cache/                    # generated .py solver scripts
    ├── results/
    │   ├── generate/             # .ipynb traces from generate.py
    │   └── execute/              # .ipynb traces from execute.py
    └── agent-YYYY-MM-DD_*.log    # full execution log
```

---

## Stopping / Restarting

```bash
# Stop
cd benchmark && docker compose down

# Restart later
cd benchmark && docker compose up -d
cd ../halligan && pixi run ... python execute.py
```

No reinstallation needed after initial setup.

---

## Citation

```bibtex
@inproceedings{teoh2025captcha,
  title     = {Are CAPTCHAs Still Bot-hard? Generalized Visual CAPTCHA Solving
               with Agentic Vision Language Model},
  author    = {Teoh, Xiwen and Lin, Yun and Li, Siqi and Liu, Ruofan and
               Sollomoni, Avi and Harel, Yaniv and Dong, Jin Song},
  booktitle = {34th USENIX Security Symposium},
  year      = {2025},
  address   = {Seattle, WA, USA}
}
```
