# Foundry Claude Skills

Claude Code skills for running [Foundry](https://github.com/RosettaCommons/foundry) protein design tools on [Vast.ai](https://vast.ai) GPU instances.

Uses a pre-built Docker image (see `Dockerfile` in the Foundry repo) with all tools, checkpoints, and dependencies pre-installed — including Enhanced MPNN weights. Instances are ready to use immediately after launch.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| `/rfd3` | `rfd3 design` | Run RFdiffusion3 all-atom protein structure design |
| `/rf3` | `rf3 fold` | Run RosettaFold3 structure prediction |
| `/mpnn` | `mpnn` | Run ProteinMPNN/LigandMPNN/SolubleMPNN/Enhanced MPNN sequence design |
| `/run-foundry-job` | Multi-step | Run complete design pipelines (e.g., RFD3 + Enhanced MPNN + RF3) |

Each skill launches a Vast.ai GPU instance with the Foundry Docker image, uploads input files, runs the job, downloads results, and destroys the instance.

## Prerequisites

- **Claude Code** installed ([docs](https://docs.anthropic.com/en/docs/claude-code))
- **Vast.ai CLI** installed and authenticated:
  ```bash
  pip install vastai
  vastai set api-key <YOUR_API_KEY>
  ```
- **SSH keys** configured on Vast.ai:
  ```bash
  vastai show ssh-keys
  # If empty, upload your public key:
  vastai create ssh-key "$(cat ~/.ssh/id_ed25519.pub)"
  ```
- **Foundry Docker image** built and pushed to a registry (see `Dockerfile` in the Foundry repo)

## Installation

### Option 1: Install from a local path (for development)

Launch Claude Code with the `--plugin-dir` flag pointing to this repo:

```bash
claude --plugin-dir /path/to/foundry-claude-skill
```

This loads all skills for that session. Useful for testing and development.

### Option 2: Install via marketplace (recommended for regular use)

**Step 1 — Add this repo as a marketplace.**

From within Claude Code, run:

```
/plugin marketplace add /path/to/foundry-claude-skill
```

Or if hosted on GitHub:

```
/plugin marketplace add your-org/foundry-claude-skill
```

**Step 2 — Install the plugin from the marketplace.**

```
/plugin install foundry@foundry-skills
```

This permanently registers the skills. They will be available in all future Claude Code sessions.

**Step 3 — (Optional) Set project scope for team sharing.**

To make the plugin available for everyone working in a specific project:

```
/plugin install foundry@foundry-skills --scope project
```

This writes the configuration to `.claude/settings.json` which can be committed to your repo.

### Option 3: Manual configuration

Add the following to your Claude Code settings file (`~/.claude/settings.json` for user-wide, or `.claude/settings.json` in your project):

```json
{
  "extraKnownMarketplaces": {
    "foundry-skills": {
      "source": {
        "source": "directory",
        "path": "/path/to/foundry-claude-skill"
      }
    }
  },
  "enabledPlugins": {
    "foundry@foundry-skills": true
  }
}
```

### Verifying installation

After installation, you should be able to invoke skills directly:

```
/rfd3 Design binders for my target protein
```

If the skills are loaded correctly, Claude will launch a forked agent that handles the full Vast.ai workflow.

## Usage Examples

```
# Design protein binders targeting PD-L1
/rfd3 Design binders for PD-L1 using target.pdb with hotspots at A56, A115, A123

# Predict structure of a designed sequence
/rf3 Fold this sequence: MTSENPLLALREK...

# Design sequences for a backbone with Enhanced MPNN (best designability)
/mpnn Design 8 sequences for backbone.cif using Enhanced MPNN at temperature 0.1

# Design sequences with standard ProteinMPNN
/mpnn Design 10 sequences for backbone.cif using ProteinMPNN at temperature 0.1

# Run a full binder design pipeline
/run-foundry-job Design binders for target.pdb, then design sequences with Enhanced MPNN, then validate with RF3
```

## Skills Reference

### `/rfd3` — RFdiffusion3 Design
Designs protein structures under complex constraints. Supports protein binders, enzyme active sites, nucleic acid binders, small molecule binders, and symmetric assemblies.

### `/rf3` — RosettaFold3 Prediction
Predicts biomolecular structures for proteins, nucleic acids, small molecules, and their complexes. Supports templating, covalent modifications, and MSAs.

### `/mpnn` — Sequence Design
Fixed-backbone inverse folding with four model variants:
- **ProteinMPNN** — standard sequence design
- **LigandMPNN** — ligand-aware design (small molecules, DNA/RNA, ions)
- **SolubleMPNN** — solubility-optimized design for E. coli expression
- **Enhanced MPNN** — designability-optimized (recommended for de novo design). Fine-tuned LigandMPNN using ResiDPO that achieves ~2.7x better enzyme design success and ~2.3x better binder design success by optimizing directly for structural validation confidence rather than native sequence recovery.

### `/run-foundry-job` — Pipeline Runner
Runs multi-step design workflows on a single GPU instance. Common pipelines:
- Binder design: RFD3 → Enhanced MPNN → RF3
- Enzyme design: RFD3 → Enhanced MPNN → RF3
- Sequence optimization: MPNN → RF3

## Repository Structure

```
foundry-claude-skill/
├── .claude-plugin/
│   ├── plugin.json          # Plugin metadata
│   └── marketplace.json     # Marketplace listing
├── README.md
└── skills/
    ├── foundry/SKILL.md     # Reference docs (auto-invoked, not user-invocable)
    ├── rfd3/SKILL.md        # /rfd3 — RFdiffusion3 on Vast.ai
    ├── rf3/SKILL.md         # /rf3 — RosettaFold3 on Vast.ai
    ├── mpnn/SKILL.md        # /mpnn — MPNN variants on Vast.ai
    └── run-foundry-job/SKILL.md  # /run-foundry-job — Multi-step pipelines
```
