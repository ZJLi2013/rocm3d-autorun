---
name: rocm-install-script-generator
version: 1.2.0
author: ZJLi2013
description: Generates bash install/run scripts for ML repos targeting ROCm GPU nodes, following the code-autorun docker_agent workflow (base image → install script → docker commit → run script). Use when asked to "generate install script", "dockerize ML repo", "create ROCm script", or "add to code-autorun".
allowed-tools: [Read, Write, Glob, Grep, Shell]
---

# ROCm Install Script Generator

Scripts go to `samples/auto_gen/`. Agent writes bash scripts run **inside** a pre-built ROCm container — no Dockerfile.

```
rocm/pytorch base image → docker run + install.sh → docker commit → docker run + run.sh
```

---

## Instructions

### 1. Gather Repo Information

Read `requirements.txt`, `setup.py`, `pyproject.toml`, `environment.yml`, and `README.md` (first 150 lines). Identify:

| Signal | What to look for |
|--------|-----------------|
| Python version | `python_requires` in setup.py / README badge; default `3.12` |
| Editable install | `pip install -e .` in README |
| C++ / CUDA extensions | `ext_modules`, `build_ext`, `.so` files, submodule `setup.py` |
| Native submodules | `.gitmodules`, `git submodule update` in README |
| ROCm-sensitive deps | `xformers`, `gsplat`, `flash-attn`, `triton`, `torch_geometric` |
| Pre-build patches | `git fetch … && git merge` before build |
| Long build | >10 min compile → split install from run |

---

### 2. Build the Install Script — Block by Block

Include only the blocks the repo actually needs.

---

#### Block A — Boilerplate (always)

```bash
#!/usr/bin/env bash
# <repo-name> — <repo-url>
# Prerequisite: /workspace = cloned repo root (bind-mounted by docker_agent)
set -e
cd /workspace
```

---

#### Block B — Conda env (always)

```bash
conda create -n <env_name> python=<3.10|3.11|3.12> -y
source /opt/conda/etc/profile.d/conda.sh && conda activate <env_name>
```

---

#### Block C — ROCm PyTorch (always)

```bash
pip uninstall -y torch torchvision torchaudio
pip install --pre torch==2.8.0 torchvision==0.23.0 torchaudio==2.8.0 \
  --index-url https://download.pytorch.org/whl/rocm6.4
```

---

#### Block D — ROCm-aware pre-built libs (if any trigger lib found in deps)

**Scan first.** Before writing any command, grep requirements.txt and README for these trigger names:

| Trigger lib | ROCm install method | Source build? |
|-------------|---------------------|---------------|
| `gsplat` | `pip install gsplat --index-url=https://pypi.amd.com/simple --extra-index-url https://pypi.org/simple` | ❌ Never |
| `xformers` | `pip install -U xformers==0.0.32.post2 --index-url https://download.pytorch.org/whl/rocm6.4` | ❌ Never |
| `flash-attn` | `pip install flash-attn --index-url=https://pypi.amd.com/simple --extra-index-url https://pypi.org/simple` | ❌ Never |
| `triton` | `pip install triton --index-url https://download.pytorch.org/whl/rocm6.4` | ❌ Never |
| `torch-geometric` / `torch_scatter` | PyG wheels `-f https://data.pyg.org/whl/torch-<VER>+<ROCM_TAG>.html` | ❌ Never |
| `diff-gaussian-rasterization` | source build in Block G + `PYTORCH_ROCM_ARCH` | ✅ Block G only |
| `simple-knn` | source build in Block G + `PYTORCH_ROCM_ARCH` | ✅ Block G only |
| `pytorch3d` | `pip install https://github.com/ZJLi2013/pytorch3d/releases/download/rocm6.4-py3.12/pytorch3d-0.7.9-cp312-cp312-linux_x86_64.whl` | ❌ Never (pre-built ROCm wheel) |

> **pytorch3d note:** Use the pre-built ROCm 6.4 / Python 3.12 wheel from [ZJLi2013/pytorch3d](https://github.com/ZJLi2013/pytorch3d/releases/tag/rocm6.4-py3.12) — no source build. Add `pytorch3d` to `EXCLUDE_PKGS` so Block E doesn't re-install it.

Install each matched lib **before** Block E. Block E's `EXCLUDE_PKGS` / `pat` already covers all Block D packages — no manual extension needed.

```bash
# Include only lines for libs actually present in the repo's deps:
pip install -U xformers==0.0.32.post2 --index-url https://download.pytorch.org/whl/rocm6.4
pip install gsplat --index-url=https://pypi.amd.com/simple --extra-index-url https://pypi.org/simple
pip install flash-attn --index-url=https://pypi.amd.com/simple --extra-index-url https://pypi.org/simple
pip install triton --index-url https://download.pytorch.org/whl/rocm6.4
pip install torch_geometric
pip install pyg_lib torch_scatter torch_sparse torch_cluster torch_spline_conv \
  -f https://data.pyg.org/whl/torch-<TORCH_VER>+<ROCM_TAG>.html
pip install https://github.com/ZJLi2013/pytorch3d/releases/download/rocm6.4-py3.12/pytorch3d-0.7.9-cp312-cp312-linux_x86_64.whl
```

---

#### Block E — Strip torch pins + bulk install

Apply each sub-step independently based on what the repo provides; both can coexist.

```bash
# ── E1: requirements.txt (if present) ────────────────────────────────────────
# Cover ALL packages from Blocks C + D — add only the ones present in Block D
EXCLUDE_PKGS="torch|torchvision|torchaudio|xformers|gsplat|flash.attn|triton|torch.geometric|pyg.lib|torch.scatter|torch.sparse|torch.cluster|torch.spline.conv|pytorch3d"
# Pass 1: strip package-name form  (e.g. "gsplat>=1.0", "torch==2.6")
grep -vEi "^(${EXCLUDE_PKGS})([<>=!~;[:space:]]|$)" requirements.txt \
  > requirements.tmp && mv requirements.tmp requirements.txt
# Pass 2: strip git+URL form  (e.g. "git+https://github.com/nerfstudio-project/gsplat.git")
# Same packages can appear as VCS URLs — they trigger source builds that fail in build isolation.
GIT_EXCLUDE="gsplat|pytorch3d|xformers|flash.attn|flash-attn|triton"
grep -vEi "git\+https?://[^[:space:]]*(${GIT_EXCLUDE})" requirements.txt \
  > requirements.tmp && mv requirements.tmp requirements.txt
pip install -r requirements.txt
# ❌ Skip: requirements_optional/extra/dev.txt   ❌ pip install -e ".[all/full/extras]"
# ✅ environment.yml: conda env update -n <env> --file environment.yml --prune

# ── E2: pyproject.toml (if any Block C/D package is pinned in dependencies) ──
# Handles both PEP 621 form: "gsplat @ git+https://..." / "gsplat>=1.0"
# and Poetry form:           gsplat = {git = "https://..."}
python3 -c "
import re, pathlib; p = pathlib.Path('pyproject.toml')
pkg = r'torch|torchvision|torchaudio|xformers|gsplat|flash\.attn|flash-attn|triton|torch\.geometric|pyg\.lib|torch\.scatter|torch\.sparse|torch\.cluster|torch\.spline\.conv|pytorch3d'
# PEP 621: quoted dep string starting with package name
pat1 = re.compile(r'[ \t]*\"(' + pkg + r')[^\"]*\",?[^\n]*\n', re.I)
# Poetry: <pkg> = {git = \"...\"}
pat2 = re.compile(r'[ \t]*(' + pkg + r')\s*=\s*\{[^\}]*git[^\}]*\}[^\n]*\n', re.I)
txt = p.read_text(); txt = pat1.sub('', txt); txt = pat2.sub('', txt); p.write_text(txt)
print('pyproject.toml: Block C/D package pins + git entries removed')
"
```

> **E1 trigger:** `requirements.txt` exists.  
> **E2 trigger:** `pyproject.toml` has any Block C/D package in `[project.dependencies]` — applies regardless of whether E1 also runs.  
> **Both:** always exclude the full set; trim `EXCLUDE_PKGS` / `pat` to only packages actually installed in Blocks C+D for this repo.

---

#### Block F — Editable install (if setup.py / pyproject.toml present)

```bash
pip install -e .    # core deps only; never use pip install -e ".[all]" / ".[full]" / ".[extras]"
```

---

#### Block G — C++ / CUDA extension build (if repo has native extensions)

```bash
export PYTORCH_ROCM_ARCH="gfx942"
git config --global --add safe.directory /workspace
# git config --global --add safe.directory /workspace/<subrepo>   # repeat per submodule

# Apply ROCm patch if needed (repo-specific — check README / known issues):
# cd <subrepo> && git fetch origin <pr-ref>:<branch> && git merge <branch> --no-edit && cd /workspace

# Build each extension — adapt path and tool to the repo's actual structure:
cd <extension_dir> && python setup.py build_ext --inplace && cd /workspace
# or: pip install -e submodules/<ext>
# or: cmake -B build && cmake --build build
```

See `docs/skills/rocm-install-experience/` for repo-specific build notes (croco, gsplat submodules, etc.).

---

#### Block H — Smoke test (recommended)

```bash
# Print torch version and CUDA status — do NOT assert cuda.is_available() here.
# The install container may not have GPU devices mounted; import success is sufficient.
python -c "import torch; print('torch', torch.__version__, '| cuda:', torch.cuda.is_available())"
```

---

### 3. Run Script + docker_agent Invocation

**Split into `_run.sh`** when build is long or sample needs runtime checkpoints.

```bash
#!/usr/bin/env bash
# <repo-name> run
set -e
cd /workspace
source /opt/conda/etc/profile.d/conda.sh && conda activate <env_name>
# wget -q <ckpt_url> -O <path>   OR   huggingface-cli download <model_id> --local-dir <path>
python3 <sample_script>.py <args>
```

**docker_agent command** (always output this after generating scripts):

```bash
cd /path/to/code-autorun && export PYTHONPATH=./src
python -m docker_agent \
  --repo_url https://github.com/<owner>/<repo>.git \
  --base-image rocm/pytorch:rocm6.4.3_ubuntu24.04_py3.12_pytorch_release_2.6.0 \
  --install-script samples/auto_gen/<repo>_install.sh \
  [--run-script samples/auto_gen/<repo>_run.sh] \   # omit if install-only
  --run-timeout 3600
```

Scripts: `samples/auto_gen/<repo>_install.sh` (always) · `samples/auto_gen/<repo>_run.sh` (two-phase only).
Hand-written references live in `samples/manually_scripts/` — do not modify.

---

## Decision Guide

| Condition | Block(s) |
|-----------|---------|
| Always | A + B + C |
| `requirements.txt` exists | E1 |
| `pyproject.toml` has `"torch` in `[project.dependencies]` | E2 (independent of E1) |
| `setup.py` / `pyproject.toml` exists | F |
| Any Block-D trigger lib in deps | D (matched lines only) + extend EXCLUDE_PKGS |
| `ext_modules` / `build_ext` / `.gitmodules` | G |
| Long build or runtime checkpoints | H + `_run.sh` |
| `requirements_optional.txt` / `.[all]` present | **Skip** — core only |
| `environment.yml` instead of requirements.txt | Block E conda variant |

---

## Self-Check Checklist

After writing the script, verify each item before finalizing.

**Block A–C (always)**
- [ ] `set -e` at top
- [ ] `conda activate` before any `pip`
- [ ] `pip uninstall -y torch torchvision torchaudio` before ROCm wheel
- [ ] Use `pip install` (NOT `pip3 install`) for torch — `pip3` may resolve to base conda's pip after `conda activate`, installing into the wrong env
- [ ] No `import pkg_resources` check anywhere — `pkg_resources.__version__` doesn't exist; use `import torch` (Block H) as the only smoke test

**Block D (pre-built ROCm libs)**
- [ ] Scanned requirements.txt + README for all 8 trigger libs?
- [ ] Each matched lib uses ROCm-specific index (AMD / pytorch ROCm) — NOT bare `pip install` or source build?
- [ ] Each Block-D lib added to `EXCLUDE_PKGS`?

**Block E**
- [ ] E1 (requirements.txt): `EXCLUDE_PKGS` pattern covers torch/torchvision/torchaudio + all Block D libs actually installed; optional/dev txt skipped
- [ ] E2 (pyproject.toml): `pat` covers same full set; applied if any Block C/D package appears in `[project.dependencies]`; runs before Block F regardless of E1

**Block G (extensions)**
- [ ] `PYTORCH_ROCM_ARCH` exported before any `build_ext` or extension install
- [ ] `git config safe.directory` added for workspace + each submodule
- [ ] Build paths match the repo's actual directory structure

**Final**
- [ ] Smoke test present
- [ ] Script saved to `samples/auto_gen/`
- [ ] docker_agent invocation provided
