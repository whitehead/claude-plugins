---
name: fry-python-tool
description: Find and use existing GPU-powered computational biology tools for the Whitehead fry cluster, or scaffold a new one. Single-purpose Python tools that produce outputs (embeddings, segmentations, predictions) for downstream analysis.
version: 0.4.0
---

# Fry Python Tools

Single-purpose, GPU-powered Python tools for computational biology on the Whitehead fry cluster. Each tool wraps one model or library, takes a YAML config, and produces an output (embeddings, segmentations, predictions) that you use downstream in your analysis. Install with conda+uv, run with `sbatch`.

## Available Tools

Install and run any of these on fry right now:

| Tool | What it produces | Wraps | Repo |
|------|-----------------|-------|------|
| **goudacell** | Cell segmentation masks | Cellpose | [cheeseman-lab/goudacell](https://github.com/cheeseman-lab/goudacell) |
| **emmentalembed** | Protein embeddings | ESM | [cheeseman-lab/emmentalembed](https://github.com/cheeseman-lab/emmentalembed) |

### Quick install (any tool)

```bash
git clone https://github.com/cheeseman-lab/TOOL.git
cd TOOL
conda create -n TOOL -c conda-forge python=3.11 uv pip -y
conda activate TOOL
uv pip install -e ".[gpu]"
```

Then configure and run:
```bash
cp configs/example_config.yaml my_config.yaml
# Edit my_config.yaml with your paths
sbatch scripts/run.sh my_config.yaml
```

## Building a New Tool

When you need GPU compute for a new task (e.g., structure prediction, variant effect scoring, image classification), scaffold a new tool following the same pattern.

### When to use this

- You have a Python script or notebook that runs a model on GPU and produces an output file
- You want others on fry to use it without understanding the Python
- The tool does ONE thing: input data + config in, output files out

### Project structure

```
TOOL/
├── src/TOOL/
│   ├── __init__.py          # __version__ = "0.1.0"
│   ├── cli.py               # typer CLI entry point
│   └── config.py            # dataclass-backed YAML config
├── configs/
│   └── example_config.yaml  # ship a working example
├── scripts/
│   ├── run.sh               # sbatch GPU job script
│   └── jupyter_gpu.sh       # interactive GPU notebook
├── tests/
│   ├── conftest.py
│   ├── test_install.py      # smoke tests
│   └── test_config.py       # config validation
├── pyproject.toml
├── README.md
└── CLAUDE.md
```

### pyproject.toml

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "TOOL"
version = "0.1.0"
description = "One-line: what output does this produce?"
readme = "README.md"
requires-python = ">=3.10"
license = { text = "MIT" }
dependencies = [
    "pyyaml>=6.0",
    "typer>=0.9.0",
    "rich>=13.0.0",
]

[project.optional-dependencies]
gpu = [
    "torch==2.7.0",
    # The model/library this tool wraps
]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.4.0",
]

[project.scripts]
TOOL = "TOOL.cli:app"

[tool.setuptools]
package-dir = {"" = "src"}
packages = ["TOOL"]

[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "D"]
pydocstyle = { convention = "google" }

[tool.ruff.lint.per-file-ignores]
"tests/*.py" = ["D100", "D103"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Key decisions:
- **GPU deps in `[gpu]` extra** — CPU-only install must work for testing/dev
- **`>=` floor pins** for most deps, exact `==` pins only for torch or model-specific libs
- **`src/` layout** — prevents accidental imports from the repo root

### CLI (typer + rich)

```python
# src/TOOL/cli.py
import typer
from rich.console import Console

app = typer.Typer(help="TOOL — one-line description")
console = Console()

@app.command()
def run(config: str = typer.Argument(..., help="Path to config YAML")):
    """Run the tool."""
    from TOOL.config import Config
    cfg = Config.from_yaml(config)
    # ... load model, process input, write output
    console.print("[green]Done![/green]")

@app.command()
def version():
    """Show version and environment info."""
    from TOOL import __version__
    console.print(f"TOOL v{__version__}")

if __name__ == "__main__":
    app()
```

### YAML Config (dataclass-backed)

```python
# src/TOOL/config.py
from dataclasses import dataclass, asdict
import yaml

@dataclass
class Config:
    """Tool configuration."""
    input_dir: str = "."
    output_dir: str = "./output"
    gpu: bool = False
    # ... tool-specific params (model name, batch size, etc.)

    @classmethod
    def from_yaml(cls, path: str) -> "Config":
        with open(path) as f:
            data = yaml.safe_load(f)
        return cls(**data)

    def to_yaml(self, path: str) -> None:
        with open(path, "w") as f:
            yaml.dump(asdict(self), f, default_flow_style=False)
```

### SLURM GPU Scripts

**`scripts/run.sh`** — GPU batch job:
```bash
#!/bin/bash
#SBATCH --job-name=TOOL
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32gb
#SBATCH --time=04:00:00
#SBATCH --partition=YOUR_GPU_PARTITION
#SBATCH --gres=gpu:1
#SBATCH --output=TOOL-%j.out

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
conda activate TOOL

echo "================================================"
echo "TOOL — $(date)"
echo "Host: $(hostname)"
echo "Config: ${CONFIG_PATH}"
echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader 2>/dev/null || echo 'none')"
echo "================================================"

TOOL run "${CONFIG_PATH}"

echo "Completed: $(date)"
```

**`scripts/jupyter_gpu.sh`** — Interactive GPU notebook:
```bash
#!/bin/bash
#SBATCH --job-name=TOOL_jupyter
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32gb
#SBATCH --time=04:00:00
#SBATCH --partition=YOUR_GPU_PARTITION
#SBATCH --gres=gpu:1
#SBATCH --output=TOOL_jupyter-%j.out

source ~/.bashrc
conda activate TOOL
unset XDG_RUNTIME_DIR

NOTEBOOK_DIR="${SLURM_SUBMIT_DIR:-$(pwd)}"
jupyter-lab \
    --no-browser \
    --port-retries=0 \
    --ip=0.0.0.0 \
    --port=$(shuf -i 8900-10000 -n 1) \
    --notebook-dir="${NOTEBOOK_DIR}"
```

### Smoke Tests

**`tests/test_install.py`**:
```python
"""Smoke tests: does the package install and import correctly?"""

def test_import():
    import TOOL

def test_version():
    from TOOL import __version__
    parts = __version__.split(".")
    assert len(parts) == 3

def test_cli_entry_point():
    import subprocess
    result = subprocess.run(["TOOL", "--help"], capture_output=True, text=True)
    assert result.returncode == 0

def test_dependencies_importable():
    import yaml
    import typer
    import rich
```

**`tests/test_config.py`**:
```python
from TOOL.config import Config

def test_load_example_config(example_config):
    cfg = Config.from_yaml(str(example_config))
    assert cfg.input_dir is not None

def test_config_roundtrip(tmp_path):
    cfg = Config()
    path = tmp_path / "test_config.yaml"
    cfg.to_yaml(str(path))
    cfg2 = Config.from_yaml(str(path))
    assert cfg == cfg2
```

### README

Required sections:
1. **What output it produces** (1-2 sentences)
2. **Getting Started** — clone, conda create, install
3. **Quick Start** — simplest `sbatch` example
4. **Configuration** — YAML field reference table
5. **GPU Partitions** — which fry partitions to use

## Instructions

### If the user wants to use an existing tool:

1. Check the **Available Tools** table above
2. Provide the install + run commands for their specific tool
3. Help them write a config YAML for their data

### If the user wants to build a new tool:

1. Scaffold the full structure above
2. Replace `TOOL` with the actual tool name everywhere
3. Add the model/library-specific logic to `cli.py` and `config.py`
4. Fill in the GPU partition placeholder in SLURM scripts
5. Write a concrete `example_config.yaml`
6. Verify: install and run `pytest tests/ -v`

### If restructuring an existing script into a tool:

1. **Read** the current script/notebook
2. **Identify** what model it wraps, what input it takes, what output it produces
3. **Scaffold** the tool structure and move the core logic into `cli.py`
4. **Verify**: `uv pip install -e ".[dev]"` then `pytest tests/ -v`

## Key Principles

- **One tool, one output**: Each tool does one GPU compute task and produces files for downstream use
- **Clone-to-running in 5 minutes**: Any fry user can install and run without help
- **Configs over code**: Users edit YAML, never Python
- **GPU as optional extra**: CPU-only install works for dev/testing; `[gpu]` adds torch + model deps
- **SLURM-first**: Primary execution path is `sbatch scripts/run.sh config.yaml`
