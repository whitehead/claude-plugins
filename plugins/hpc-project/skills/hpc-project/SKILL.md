---
name: hpc-project
description: This skill should be used when making a project "reproducible for Whitehead users", "HPC-ready", "shareable on the cluster", adding SLURM scripts, YAML configs, CLI entry points, or restructuring a codebase to follow a pattern of reproducible, well-documented tools that work for anyone on the Whitehead HPC.
version: 0.2.0
---

# HPC-Ready Project Pattern

Make a Python project reproducible, shareable, and easy to use for any Whitehead HPC user — not just the developer. Pattern: conda+uv install, YAML configs, CLI entry points, SLURM scripts, tests, and clear documentation.

## When to Use

- Restructuring an existing project to be shareable
- Adding SLURM scripts, configs, or CLI to an existing project
- Auditing a project for reproducibility gaps
- Setting up a new project that others will use

## The Pattern (What Makes a Project "HPC-Ready")

A project following this pattern has **8 pillars**:

### 1. One-Command Install (conda + uv + pyproject.toml)

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "PROJECT"
version = "0.1.0"
description = "One-line description"
readme = "README.md"
requires-python = ">=3.10"
license = { text = "MIT" }
dependencies = [
    # Pin major versions, not exact (users need flexibility)
    "numpy>=1.24.0",
    "pandas>=2.0.0",
    "pyyaml>=6.0",
    "typer>=0.9.0",
    "rich>=13.0.0",
]

[project.optional-dependencies]
# Separate heavy/GPU deps into extras so CPU-only users aren't blocked
gpu = [
    "torch==2.7.0",
]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.4.0",
]

[project.scripts]
PROJECT = "PROJECT.cli:app"

[tool.setuptools]
package-dir = {"" = "src"}
packages = ["PROJECT"]

[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "D"]
pydocstyle = { convention = "google" }

[tool.ruff.lint.per-file-ignores]
"tests/*.py" = ["D100", "D103"]
"*.ipynb" = ["D100", "D103"]
```

Install flow for users:
```bash
git clone https://github.com/YOUR_ORG/PROJECT.git
cd PROJECT
conda create -n PROJECT -c conda-forge python=3.11 uv pip -y
conda activate PROJECT
uv pip install -e "."          # or -e ".[gpu]" for GPU extras
```

### 2. CLI Entry Point (typer + rich)

Every project gets a CLI so users never need to write Python:

```python
# src/PROJECT/cli.py
import typer
from rich.console import Console

app = typer.Typer(help="PROJECT — one-line description")
console = Console()

@app.command()
def run(config: str = typer.Argument(..., help="Path to config YAML")):
    """Run the main pipeline."""
    from PROJECT.config import Config
    cfg = Config.from_yaml(config)
    # ... run pipeline
    console.print("[green]Done![/green]")

@app.command()
def version():
    """Show version and environment info."""
    from PROJECT import __version__
    console.print(f"PROJECT v{__version__}")

if __name__ == "__main__":
    app()
```

### 3. YAML Configuration (dataclass-backed)

No hardcoded paths. Everything configurable via YAML:

```python
# src/PROJECT/config.py
from dataclasses import dataclass, field, asdict
from pathlib import Path
import yaml

@dataclass
class Config:
    """Pipeline configuration."""
    input_dir: str = "."
    output_dir: str = "./output"
    file_pattern: str = "*.csv"
    gpu: bool = False
    # ... project-specific params

    @classmethod
    def from_yaml(cls, path: str) -> "Config":
        with open(path) as f:
            data = yaml.safe_load(f)
        return cls(**data)

    def to_yaml(self, path: str) -> None:
        with open(path, "w") as f:
            yaml.dump(asdict(self), f, default_flow_style=False)
```

Ship a default config in `data/` or `configs/`:
```yaml
# configs/example_config.yaml
input_dir: /path/to/your/data
output_dir: ./output
file_pattern: "*.csv"
gpu: true
```

### 4. SLURM Scripts (ready to sbatch)

Two standard scripts:

**`scripts/run.sh`** — Batch job:
```bash
#!/bin/bash
#SBATCH --job-name=PROJECT
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32gb
#SBATCH --time=04:00:00
#SBATCH --output=PROJECT-%j.out
# Uncomment and set for GPU jobs:
# #SBATCH --partition=YOUR_GPU_PARTITION
# #SBATCH --gres=gpu:1

# Usage: sbatch scripts/run.sh /path/to/config.yaml

set -e

if [ -z "$1" ]; then
    echo "Usage: sbatch scripts/run.sh /path/to/config.yaml"
    exit 1
fi

CONFIG_PATH="$(realpath "$1")"
CONFIG_DIR="$(dirname "$CONFIG_PATH")"
cd "$CONFIG_DIR"

source ~/.bashrc
conda activate PROJECT

echo "================================================"
echo "PROJECT — $(date)"
echo "Host: $(hostname)"
echo "Config: ${CONFIG_PATH}"
echo "Python: $(which python)"
echo "================================================"

PROJECT run "${CONFIG_PATH}"

echo "Completed: $(date)"
```

**`scripts/jupyter_gpu.sh`** — Interactive notebook on GPU:
```bash
#!/bin/bash
#SBATCH --job-name=PROJECT_jupyter
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32gb
#SBATCH --time=04:00:00
#SBATCH --partition=YOUR_GPU_PARTITION
#SBATCH --gres=gpu:1
#SBATCH --output=PROJECT_jupyter-%j.out

# Usage: sbatch scripts/jupyter_gpu.sh

source ~/.bashrc
conda activate PROJECT
unset XDG_RUNTIME_DIR

NOTEBOOK_DIR="${SLURM_SUBMIT_DIR:-$(pwd)}"
jupyter-lab \
    --no-browser \
    --port-retries=0 \
    --ip=0.0.0.0 \
    --port=$(shuf -i 8900-10000 -n 1) \
    --notebook-dir="${NOTEBOOK_DIR}"
```

### 5. README for Non-Developers

The README is the user's entry point. It must answer: "How do I use this on the cluster?"

Required sections:
1. **What it does** (1-2 sentences)
2. **Getting Started** — clone, create env, install (copy-paste ready)
3. **Quick Start** — simplest possible usage example
4. **Configuration** — what each YAML field does
5. **SLURM Usage** — how to submit batch jobs
6. **Notebook Usage** — how to launch interactive sessions (if applicable)

Template:
```markdown
# PROJECT

One-line description of what this does.

## Getting Started

### 1. Set Up Your Environment (one time)

\`\`\`bash
git clone https://github.com/YOUR_ORG/PROJECT.git
cd PROJECT

conda create -n PROJECT -c conda-forge python=3.11 uv pip -y
conda activate PROJECT
uv pip install -e "."

# Optional: register Jupyter kernel
python -m ipykernel install --user --name PROJECT --display-name "PROJECT"
\`\`\`

### 2. Configure

Copy and edit the example config:
\`\`\`bash
cp configs/example_config.yaml my_config.yaml
# Edit my_config.yaml with your paths and parameters
\`\`\`

### 3. Run

\`\`\`bash
# Interactive
PROJECT run my_config.yaml

# On the cluster
sbatch scripts/run.sh my_config.yaml
\`\`\`

## Configuration Reference

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `input_dir` | str | `.` | Path to input data |
| `output_dir` | str | `./output` | Where to write results |
| ... | ... | ... | ... |
```

### 6. Tests (smoke tests + config validation)

Every project ships with tests that verify the install works and configs load. These are **not** exhaustive unit tests — they're smoke tests that catch broken installs, missing deps, and config regressions.

**`tests/conftest.py`**:
```python
import pytest
from pathlib import Path

@pytest.fixture
def project_root():
    """Return project root directory."""
    return Path(__file__).parent.parent

@pytest.fixture
def example_config(project_root):
    """Return path to example config."""
    return project_root / "configs" / "example_config.yaml"
```

**`tests/test_install.py`** — Catches broken installs immediately:
```python
"""Smoke tests: does the package install and import correctly?"""

def test_import():
    """Package imports without error."""
    import PROJECT

def test_version():
    """Version string is set."""
    from PROJECT import __version__
    assert __version__
    # Version follows semver pattern
    parts = __version__.split(".")
    assert len(parts) == 3
    assert all(p.isdigit() for p in parts)

def test_cli_entry_point():
    """CLI entry point is registered and callable."""
    import subprocess
    result = subprocess.run(["PROJECT", "--help"], capture_output=True, text=True)
    assert result.returncode == 0
    assert "Usage" in result.stdout or "usage" in result.stdout.lower()

def test_dependencies_importable():
    """All core dependencies are importable."""
    import yaml
    import typer
    import rich
    # Add project-specific imports here
```

**`tests/test_config.py`** — Catches config schema drift:
```python
"""Config loading and validation tests."""
from PROJECT.config import Config

def test_load_example_config(example_config):
    """Example config loads without error."""
    cfg = Config.from_yaml(str(example_config))
    assert cfg.input_dir is not None
    assert cfg.output_dir is not None

def test_config_roundtrip(tmp_path):
    """Config can be saved and reloaded identically."""
    cfg = Config()
    path = tmp_path / "test_config.yaml"
    cfg.to_yaml(str(path))
    cfg2 = Config.from_yaml(str(path))
    assert cfg == cfg2

def test_config_defaults():
    """Default config has sensible values."""
    cfg = Config()
    assert cfg.output_dir  # not empty
    assert cfg.file_pattern  # not empty
```

**`pyproject.toml` test config**:
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
```

**Running tests**:
```bash
# Quick smoke test (should pass on any fresh install)
pytest tests/test_install.py -v

# Full test suite
pytest tests/ -v
```

### 7. Versioning & Dependency Management

#### Version Bumping

Use **single-source versioning** — the version lives in ONE place and is read everywhere:

**`src/PROJECT/__init__.py`**:
```python
"""PROJECT — one-line description."""

__version__ = "0.1.0"
```

**`pyproject.toml`** reads it dynamically OR is kept in sync manually. For simplicity, keep the version in both `__init__.py` and `pyproject.toml` and bump both together.

#### When to Bump

| Change | Bump | Example |
|--------|------|---------|
| Bug fix, docs, minor tweak | PATCH | 0.1.0 -> 0.1.1 |
| New feature, new CLI command, new config field | MINOR | 0.1.0 -> 0.2.0 |
| Breaking change (config schema change, removed feature) | MAJOR | 0.1.0 -> 1.0.0 |

**Bump procedure** (update both locations):
```bash
# In __init__.py: change __version__ = "X.Y.Z"
# In pyproject.toml: change version = "X.Y.Z"
# Then:
pytest tests/test_install.py::test_version -v  # verify version parses
git commit -m "bump: vX.Y.Z — reason for bump"
git tag vX.Y.Z
```

#### Dependency Freshness

**Pinning strategy**: Use `>=` floor pins for most deps (users need flexibility), exact `==` pins only for deps where version matters for correctness (torch, cellpose, etc.).

**When restructuring or auditing**, check for stale floors:
```bash
# Show what's currently installed vs what's pinned
uv pip list | grep -E "numpy|pandas|scipy"
```

If a floor pin is >2 major versions behind, bump it.

**Adding new dependencies**: Always add to `pyproject.toml`, never bare `pip install`. Pin floor to the version you tested against:
```bash
# Find current version
python -c "import newdep; print(newdep.__version__)"
# Add to pyproject.toml: "newdep>=X.Y.0"
# Reinstall: uv pip install -e "."
# Run tests: pytest tests/test_install.py -v
```

### 8. CLAUDE.md for AI Collaboration

Keep the CLAUDE.md focused on what an AI assistant needs:
- Project overview, structure, key design decisions
- Dev setup command (conda activate + install)
- CLI commands
- Where configs live and how they work
- How to run tests

## Audit Checklist

When applying this pattern to an existing project, check each pillar:

```
[ ] pyproject.toml with extras for heavy deps (GPU, optional features)
[ ] CLI entry point via typer (no "run this script with python")
[ ] YAML config with dataclass backing (no hardcoded paths)
[ ] SLURM scripts: batch job + jupyter (with GPU partition commented/configurable)
[ ] README: clone-to-running in <5 minutes for any cluster user
[ ] Smoke tests: test_install.py (import, version, CLI, deps) + test_config.py (load, roundtrip, defaults)
[ ] Versioning: __version__ in __init__.py, matches pyproject.toml, semver bumps
[ ] CLAUDE.md: structure, setup, CLI commands, design decisions, how to run tests
[ ] Example config shipped in configs/ or data/
[ ] .gitignore covers Python, Jupyter, .env, .DS_Store
[ ] Optional deps separated into extras (gpu, cellpose3, etc.)
[ ] conda activate pattern uses `source ~/.bashrc` in SLURM scripts
[ ] Dependency floors not >2 major versions behind current
[ ] pytest configured in pyproject.toml [tool.pytest.ini_options]
```

## Instructions

### If restructuring an existing project:

1. **Read** the current project structure, pyproject.toml, any scripts, and CLAUDE.md
2. **Audit** against the checklist above — identify which pillars are missing or incomplete
3. **Present** the audit to the user: "Here's what's already good, here's what needs work"
4. **Ask** the user which pillars to implement (or do all)
5. **Implement** each pillar, adapting the templates above to the project's specific needs
6. **Add tests**: Create `tests/test_install.py` and `tests/test_config.py` adapted to the project
7. **Version check**: Ensure `__version__` in `__init__.py` matches `pyproject.toml`, bump dep floors if stale
8. **Verify**: Run `uv pip install -e ".[dev]"` then `pytest tests/ -v` — everything must pass

### If starting fresh:

Scaffold the project with the standard conda+uv+pyproject.toml pattern, then apply this skill to add the remaining pillars (CLI, config, SLURM scripts, README).

## Key Principles

- **Copy-paste ready**: Every command in the README should work if copied verbatim
- **No tribal knowledge**: A new lab member should be able to use this without asking anyone
- **Configs over code**: Users configure YAML, they don't edit Python
- **GPU as optional extra**: CPU-only install must work; GPU is an opt-in extra
- **SLURM-first**: The primary execution path is `sbatch`, not `python script.py`
- **Relative paths in configs**: Configs use relative paths; SLURM scripts `cd` to config dir so they resolve correctly
