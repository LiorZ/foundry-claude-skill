---
name: foundry
description: "Foundry protein design toolkit reference. Auto-invoked when the user discusses protein design, structure prediction, inverse folding, RFdiffusion3, RosettaFold3, ProteinMPNN, LigandMPNN, SolubleMPNN, Enhanced MPNN, or running Foundry tools on GPUs."
user-invocable: false
allowed-tools: Bash, Read, Write, Glob, Grep
---

# Foundry Protein Design Toolkit Reference

Foundry provides core protein design and prediction tools. All tools require GPU compute and are designed to run on Vast.ai instances using a pre-built Docker image that includes all tools, checkpoints, and dependencies.

## Tools Overview

| Tool | CLI Command | Purpose | GPU VRAM |
|------|-------------|---------|----------|
| **RFdiffusion3 (RFD3)** | `rfd3 design` | All-atom generative protein structure design | 24-48GB+ |
| **RosettaFold3 (RF3)** | `rf3 fold` | Biomolecular structure prediction | 24-48GB+ |
| **ProteinMPNN/LigandMPNN** | `mpnn` | Fixed-backbone inverse folding sequence design | 16-24GB |
| **Enhanced MPNN** | `mpnn` | Designability-optimized inverse folding (fine-tuned LigandMPNN) | 16-24GB |

## Docker Image

A pre-built Docker image based on `vastai/base-image:cuda-12.4.1-auto` is available with everything pre-installed:
- Python 3.12, PyTorch with CUDA 12.4
- Foundry with all model extras (`rc-foundry[all]`)
- All checkpoints pre-downloaded to `/root/.foundry/checkpoints`
- Enhanced MPNN weights included

When launching Vast.ai instances, use this image with:
```bash
vastai create instance <OFFER_ID> --image <FOUNDRY_DOCKER_IMAGE> --disk 64 --ssh
```

No `--onstart-cmd` is needed — the image is ready to use immediately. The working directory is `/workspace`.

## Available Checkpoints

All checkpoints are pre-installed in the Docker image at `/root/.foundry/checkpoints`:

| Name | File | Tool |
|------|------|------|
| `rfd3` | `rfd3_latest.ckpt` | RFdiffusion3 |
| `rf3` | `rf3_foundry_01_24_latest_remapped.ckpt` | RosettaFold3 (latest, recommended) |
| `rf3_preprint_921` | `rf3_foundry_09_21_preprint_remapped.ckpt` | RF3 (benchmark, 09/21 cutoff) |
| `rf3_preprint_124` | `rf3_foundry_01_24_preprint_remapped.ckpt` | RF3 (preprint) |
| `proteinmpnn` | `proteinmpnn_v_48_020.pt` | ProteinMPNN |
| `ligandmpnn` | `ligandmpnn_v_32_010_25.pt` | LigandMPNN |
| `solublempnn` | `solublempnn_v_48_020.pt` | SolubleMPNN |
| `enhanced_mpnn` | `enhanced_mpnn_step_80000.pt` | Enhanced MPNN |

Checkpoint env: `FOUNDRY_CHECKPOINT_DIRS=/root/.foundry/checkpoints`

---

## Tool 1: RFdiffusion3 (RFD3)

All-atom generative model for designing protein structures under complex constraints: protein binders, enzyme active sites, nucleic acid binders, small molecule binders, symmetric assemblies.

### CLI

```bash
rfd3 design out_dir=<OUTPUT_DIR> inputs=<INPUT_JSON> [OPTIONS]
```

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `out_dir` | required | Output directory |
| `inputs` | required | Input JSON/YAML file |
| `ckpt_path` | `rfd3` | Checkpoint (auto-resolved from registry) |
| `skip_existing` | `True` | Skip existing outputs |
| `diffusion_batch_size` | `8` | Designs per batch |
| `n_batches` | `1` | Number of batches |
| `dump_trajectories` | `False` | Save denoising trajectory (large files) |
| `prevalidate_inputs` | `False` | Validate inputs before loading model |
| `low_memory_mode` | `False` | Memory-efficient tokenization |

### Sampler Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `inference_sampler.num_timesteps` | `200` | Diffusion steps |
| `inference_sampler.step_scale` | `1.5` | Diversity vs designability tradeoff |
| `inference_sampler.noise_scale` | `1.003` | Noise scaling |
| `inference_sampler.use_classifier_free_guidance` | `False` | Enable CFG |
| `inference_sampler.cfg_scale` | `1.5` | CFG scale (if enabled) |
| `inference_sampler.kind` | `default` | `default` or `symmetry` |

### Input JSON Format

The input is a JSON file mapping example names to design specifications:

```json
{
    "design_name": {
        "input": "./path/to/target.pdb",
        "contig": "40-120,/0,A1-100",
        "length": "140-160",
        "ligand": "NAI,ACT",
        "unindex": "A108,A139",
        "select_fixed_atoms": {
            "A108": "ND2,CG",
            "A139": "OG,CB,CA"
        },
        "select_hotspots": {
            "E64": "CD2,CZ"
        },
        "is_non_loopy": true,
        "infer_ori_strategy": "hotspots",
        "dialect": 2
    }
}
```

**Key input fields:**
- `input`: Path to target structure (PDB/CIF)
- `contig`: Contig specification — defines which residues to keep/design and chain breaks
- `length`: Length range for designed protein (e.g., `"140-160"`)
- `ligand`: Comma-separated ligand residue names to include
- `unindex`: Residues to unindex (make designable in position)
- `select_fixed_atoms`: Per-residue atom selections to fix
- `select_hotspots`: Target hotspot residues for interface design
- `is_non_loopy`: Disable loop-only design mode
- `infer_ori_strategy`: How to determine orientation (`hotspots`)
- `dialect`: Input dialect version (use `2` for latest)
- `partial_t`: Partial diffusion timestep (for refinement)
- `ori_token`: Orientation token indices

### Output

- `{name}.cif` — Designed all-atom structures
- `{name}.json` — Full design metadata
- Trajectory files (if `dump_trajectories=True`)

---

## Tool 2: RosettaFold3 (RF3)

All-atom biomolecular structure prediction for proteins, nucleic acids, ligands, and complexes.

### CLI

```bash
rf3 fold inputs=<INPUT_FILE_OR_DIR> [OPTIONS]
```

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `inputs` | required | JSON, CIF, PDB file, list, or directory |
| `out_dir` | `./` | Output directory |
| `ckpt_path` | auto | Checkpoint path |
| `n_recycles` | `10` | Number of recycling iterations |
| `diffusion_batch_size` | `5` | Number of output structures |
| `num_steps` | `200` | Diffusion sampling steps (50 is faster, similar quality) |
| `early_stopping_plddt_threshold` | `0.5` | Skip low-confidence predictions |
| `seed` | `null` | Random seed |
| `dump_trajectories` | `False` | Save denoising trajectories |
| `skip_existing` | `False` | Skip existing predictions |
| `one_model_per_file` | `False` | Separate files per model |
| `annotate_b_factor_with_plddt` | `False` | pLDDT as B-factors |
| `template_noise_scale` | `1e-5` | Template noise |

### Structural Control

- `template_selection` — AtomSelection syntax for template regions (e.g., `"[A, B/*/1-42]"`)
- `ground_truth_conformer_selection` — Fix ligand conformations (e.g., `"[C, D]"`)
- `cyclic_chains` — List of chain IDs to cyclize

### Input JSON Format

```json
[
    {
        "name": "example_prediction",
        "components": [
            {
                "seq": "MTSENPLLALREK...",
                "chain_id": "A",
                "msa_path": "path/to/protein.a3m"
            },
            {
                "ccd_code": "MG"
            },
            {
                "smiles": "[nH]1cc[nH+]c1"
            },
            {
                "path": "path/to/ligand.sdf"
            }
        ],
        "template_selection": ["A"],
        "ground_truth_conformer_selection": ["C"]
    }
]
```

**Component types:**
- `seq` + optional `msa_path` — Protein/nucleic acid sequence (supports non-canonical: `(PTM)`)
- `ccd_code` — CCD compound code (e.g., `MG`, `NAG`)
- `smiles` — Small molecule SMILES string
- `path` — Structure file (CIF, PDB, SDF)

**AtomSelection syntax:** `CHAIN/RES_NAME/RES_ID/ATOM_NAME`
- `A` — all atoms in chain A
- `A/*/5-10` — residues 5-10 in chain A
- `B/*/1-42, B/*/49-63` — multiple regions (CDR framework)

### Output

- `{name}_metrics.csv` — Overall confidence metrics (pTM, pLDDT, ipTM)
- `{name}.score` — Granular per-atom metrics
- `{name}_model_0.cif.gz` ... `{name}_model_N.cif.gz` — Predicted structures

---

## Tool 3: ProteinMPNN / LigandMPNN / SolubleMPNN / Enhanced MPNN

Lightweight inverse-folding models for fixed-backbone protein sequence design.

### CLI

```bash
mpnn --model_type <MODEL_TYPE> --structure_path <STRUCTURE> [OPTIONS]
```

Or from JSON config:
```bash
mpnn --config_json config.json
```

### Model Variants

| Variant | model_type | Checkpoint | Use Case |
|---------|-----------|------------|----------|
| **ProteinMPNN** | `protein_mpnn` | `proteinmpnn_v_48_020.pt` | Standard protein sequence design |
| **LigandMPNN** | `ligand_mpnn` | `ligandmpnn_v_32_010_25.pt` | Design around small molecules, DNA, ions |
| **SolubleMPNN** | `protein_mpnn` | `solublempnn_v_48_020.pt` | Solubility-optimized design |
| **Enhanced MPNN** | `ligand_mpnn` | `enhanced_mpnn_step_80000.pt` | Designability-optimized (highest success rate) |

### Enhanced MPNN

Enhanced MPNN is a fine-tuned LigandMPNN trained with **ResiDPO** (Residue-level Designability Preference Optimization). Instead of optimizing for native sequence recovery like standard MPNN, it directly optimizes for **designability** — whether designed sequences fold into the target structure with high confidence (as measured by AlphaFold2 pLDDT scores).

**Performance improvements over standard LigandMPNN:**
- **Enzyme design**: ~2.7x higher success rate (17.6% vs 6.6%)
- **Binder design**: ~2.3x higher success rate (16.1% vs 7.1%)
- Makes previously "undesignable" backbones designable — doubles the fraction of viable backbone scaffolds

**How it works:**
- ResiDPO decouples preference learning from KL regularization at the residue level
- Residue-level Preference Learning (RPL) improves positions where pLDDT is low
- Residue-level Constraint Learning (RCL) preserves knowledge at positions already working well
- Trained on 19k PDB structures with AF2 pLDDT as the reward signal

**When to use Enhanced MPNN (recommended for most de novo design):**
- Binder or enzyme design pipelines (RFD3 → MPNN → RF3 validation)
- Any scenario where standard MPNN yields low designability/validation rates
- Maximizing the fraction of designs that pass structure validation

**When to prefer standard MPNN/LigandMPNN:**
- When sequence recovery (similarity to native) matters more than designability
- Benchmarking against published results using original models

**Usage** — same CLI as LigandMPNN, just swap the checkpoint:
```bash
mpnn --model_type ligand_mpnn \
  --checkpoint_path /root/.foundry/checkpoints/enhanced_mpnn_step_80000.pt \
  --structure_path backbone.cif \
  --temperature 0.1 \
  --number_of_batches 8
```

### Key Parameters

**Global:**
- `--model_type` — `protein_mpnn` or `ligand_mpnn`
- `--checkpoint_path` — Path to model weights
- `--is_legacy_weights` — Set `True` for original repository weights (not needed for Enhanced MPNN)
- `--out_directory` — Output directory
- `--write_fasta` — Write FASTA output (default: True)
- `--write_structures` — Write designed structures (default: True)

**Per-input:**
- `--structure_path` — Input structure (CIF/PDB)
- `--batch_size` — Sequences per batch (default: 1)
- `--number_of_batches` — Number of batches (default: 1)
- `--temperature` — Sampling temperature, controls diversity (default: 0.1)
- `--seed` — Random seed
- `--designed_chains` — Chains to redesign
- `--fixed_chains` — Chains to keep fixed
- `--designed_residues` — Specific residues to design
- `--fixed_residues` — Specific residues to fix

**Advanced:**
- `--omit` — Amino acids to exclude (default: `["UNK"]`)
- `--bias` — Per-residue logit bias
- `--structure_noise` — Noise level (default: 0.0)
- `--symmetry_residues` — Residues for symmetric design
- `--homo_oligomer_chains` — Homo-oligomer chains

### JSON Config Format

```json
{
    "checkpoint_path": "enhanced_mpnn_step_80000.pt",
    "model_type": "ligand_mpnn",
    "out_directory": "./outputs/",
    "inputs": [
        {
            "structure_path": "complex.pdb",
            "name": "example",
            "seed": 42,
            "batch_size": 1,
            "number_of_batches": 5,
            "temperature": 0.1,
            "fixed_chains": ["A"],
            "designed_chains": ["B"]
        }
    ]
}
```

### Output

- `{name}_sequences_*.fasta` — Designed sequences
- `{name}_*.cif` — Designed structures (if `write_structures=True`)

---

## Common Design Workflows

### Protein Binder Design Pipeline
1. **RFD3**: Generate binder backbones targeting a protein → outputs CIF structures
2. **Enhanced MPNN**: Design sequences for the generated backbones (~2.3x better success) → outputs FASTA sequences
3. **RF3**: Validate designed sequences fold correctly → outputs confidence metrics

### Enzyme Design Pipeline
1. **RFD3**: Design enzyme scaffolds around a ligand/substrate
2. **Enhanced MPNN**: Design ligand-aware sequences (~2.7x better designability) → outputs FASTA sequences
3. **RF3**: Validate predicted structure matches design

### Sequence Optimization
1. Start with an existing structure (PDB/CIF)
2. **ProteinMPNN/SolubleMPNN**: Redesign sequences for stability/solubility
3. **RF3**: Predict structure of redesigned sequences

---

## Vast.ai GPU Recommendations

| Tool | Min VRAM | Recommended GPU | Notes |
|------|----------|-----------------|-------|
| RFD3 | 24 GB | A100 40GB, RTX 4090 | Large designs need 48GB+ |
| RF3 | 24 GB | A100 40GB, RTX 4090 | Multi-chain complexes need more |
| MPNN (all variants) | 8 GB | RTX 4090, RTX 3090 | Very lightweight |

Use the pre-built Foundry Docker image — no setup commands needed. Instance is ready to run immediately after launch.
