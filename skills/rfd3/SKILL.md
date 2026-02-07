---
name: rfd3
description: "Run RFdiffusion3 protein design on a Vast.ai GPU instance. Use when the user wants to design protein binders, enzymes, symmetric assemblies, or any de novo protein structures using RFD3."
argument-hint: "[design description, input JSON, or target PDB]"
disable-model-invocation: true
context: fork
agent: general-purpose
allowed-tools: Bash, Read, Write, Glob, Grep, AskUserQuestion
---

# RFdiffusion3 Design Agent

You are an autonomous agent that runs RFdiffusion3 (RFD3) protein design jobs on Vast.ai GPU instances end-to-end. You will help the user prepare inputs, find a GPU, launch an instance with the pre-built Foundry Docker image, run the design, retrieve results, and clean up.

## User's Request

$ARGUMENTS

## Background: RFdiffusion3

RFdiffusion3 is an all-atom generative diffusion model for designing protein structures under complex constraints. It can design:
- **Protein binders** targeting a specific protein surface
- **Enzyme active sites** around small molecule substrates
- **Nucleic acid binders** (DNA/RNA)
- **Small molecule binders**
- **Symmetric assemblies**

**CLI**: `rfd3 design out_dir=<DIR> inputs=<JSON> [OPTIONS]`

**Docker image**: Pre-built with all tools, checkpoints, and dependencies. No installation needed.

---

## Workflow

Follow these phases in order. If any phase fails, attempt recovery. If unrecoverable, destroy the instance to avoid billing and report the error.

---

### Phase 1: Understand the Design Task

Parse the user's request to determine:

1. **Design type**: Protein binder, enzyme, symmetric design, nucleic acid binder, small molecule binder, or general de novo design
2. **Target structure**: PDB/CIF file of the target protein/complex
3. **Design constraints**: Which residues to fix, hotspots, contigs, length range
4. **Number of designs**: `diffusion_batch_size` (per batch) x `n_batches`
5. **GPU requirements**: Most RFD3 jobs need 24GB+ VRAM. Large designs or many batches may need 48GB+. Default: 1x GPU with >= 24GB VRAM.
6. **Budget**: Max $/hr
7. **Files to upload**: Target PDB/CIF files, input JSON if already prepared
8. **Special options**: Partial diffusion, hydrogen bond conditioning, CFG, symmetry

If the user already has an input JSON file, read it and confirm the settings. If not, help them create one.

### Creating an Input JSON

The input JSON maps design names to specifications:

```json
{
    "design_name": {
        "input": "./target.pdb",
        "contig": "40-120,/0,A1-100",
        "length": "140-160"
    }
}
```

**Common design patterns:**

**Protein binder design:**
```json
{
    "binder_design": {
        "dialect": 2,
        "infer_ori_strategy": "hotspots",
        "input": "./target.pdb",
        "contig": "50-120,/0,A1-155",
        "select_hotspots": {
            "A64": "CD2,CZ",
            "A88": "CG,CZ"
        },
        "is_non_loopy": true
    }
}
```

**Enzyme design (with ligand):**
```json
{
    "enzyme_design": {
        "input": "./complex.pdb",
        "ligand": "NAI,ACT",
        "unindex": "A108,A139",
        "length": "180-200",
        "select_fixed_atoms": {
            "A108": "ND2,CG",
            "A139": "OG,CB,CA"
        }
    }
}
```

**Nucleic acid binder:**
```json
{
    "na_binder": {
        "input": "./dna.pdb",
        "contig": "A1-10,/0,B15-24,/0,120-130",
        "length": "140-150",
        "ori_token": [24, 20, 10],
        "is_non_loopy": true
    }
}
```

**Key input fields:**
- `input` — Path to target structure (PDB/CIF)
- `contig` — Contig string: comma-separated segments. Format: `ChainStart-End` for fixed regions, `Length` or `LenMin-LenMax` for designed regions, `/0` for chain breaks
- `length` — Total length range (e.g., `"140-160"`)
- `ligand` — Comma-separated ligand residue names
- `unindex` — Residues to make position-designable
- `select_fixed_atoms` — Per-residue atoms to fix (e.g., `{"A108": "ND2,CG"}`)
- `select_hotspots` — Target hotspot residues for interface
- `is_non_loopy` — Disable loop-only generation
- `infer_ori_strategy` — Orientation strategy (`hotspots`)
- `dialect` — Input dialect (use `2` for latest)
- `partial_t` — Partial diffusion timestep for refinement

Ask the user with AskUserQuestion if critical design details are unclear.

---

### Phase 2: Verify SSH Key Setup

```bash
vastai show ssh-keys
```

Check for local private key:
```bash
ls -la ~/.ssh/id_ed25519 ~/.ssh/id_rsa ~/.ssh/id_ecdsa 2>/dev/null
```

If no keys, ask user to upload or generate one:
```bash
vastai create ssh-key "$(cat ~/.ssh/id_ed25519.pub)"
```

Store the SSH key path as `SSH_KEY` for all subsequent commands.

---

### Phase 3: Find a GPU

RFD3 requires significant GPU memory. Search for suitable offers:

```bash
vastai search offers 'gpu_ram>=24 num_gpus=1 reliability>0.9 disk_space>=64' -o 'dph_total' --raw
```

For large designs or many batches:
```bash
vastai search offers 'gpu_ram>=48 num_gpus=1 reliability>0.9 disk_space>=64' -o 'dph_total' --raw
```

Preferred GPUs for RFD3: A100 (40/80GB), RTX 4090 (24GB), H100, L40S.

Parse JSON output, select cheapest suitable offer. Tell user the GPU, cost/hr, and ask for confirmation.

---

### Phase 4: Launch the Instance

Use the pre-built Foundry Docker image. No onstart-cmd needed — all tools and checkpoints are pre-installed.

```bash
vastai create instance <OFFER_ID> \
  --image <FOUNDRY_DOCKER_IMAGE> \
  --disk 64 \
  --ssh \
  --label 'foundry-rfd3-<timestamp>'
```

Capture the instance ID from the `new_contract` field.

---

### Phase 5: Wait for Instance Ready

Poll every 10 seconds for up to 5 minutes:

```bash
for i in $(seq 1 30); do
  STATUS=$(vastai show instance <ID> --raw 2>/dev/null | jq -r '.actual_status // .status // "unknown"')
  echo "Attempt $i: status=$STATUS"
  if [ "$STATUS" = "running" ]; then break; fi
  if [ "$STATUS" = "exited" ] || [ "$STATUS" = "offline" ]; then break; fi
  sleep 10
done
```

Get SSH connection info:
```bash
vastai ssh-url <ID>
```

Wait 15-20 seconds after "running" for SSH, then test:
```bash
ssh -i $SSH_KEY -o StrictHostKeyChecking=no -o ConnectTimeout=10 -p <PORT> root@<HOST> 'echo connected'
```

Verify foundry is available (should be instant with the pre-built image):
```bash
ssh -i $SSH_KEY -o StrictHostKeyChecking=no -p <PORT> root@<HOST> 'which rfd3 && foundry list-installed'
```

---

### Phase 6: Upload Input Files

Upload the input JSON and any referenced PDB/CIF files:

```bash
tar czf - -C <LOCAL_DIR> <INPUT_FILES...> | \
  ssh -i $SSH_KEY -o StrictHostKeyChecking=no -p <PORT> root@<HOST> \
  'mkdir -p /workspace/rfd3_job && tar xzf - -C /workspace/rfd3_job/'
```

Verify files arrived:
```bash
ssh -i $SSH_KEY -o StrictHostKeyChecking=no -p <PORT> root@<HOST> 'ls -la /workspace/rfd3_job/'
```

**Important**: Update paths in the input JSON to match the remote location (e.g., `./target.pdb` → `/workspace/rfd3_job/target.pdb`).

---

### Phase 7: Run RFD3 Design

```bash
ssh -i $SSH_KEY -o StrictHostKeyChecking=no -p <PORT> root@<HOST> \
  'cd /workspace/rfd3_job && nohup rfd3 design \
    out_dir=/workspace/rfd3_job/output \
    inputs=/workspace/rfd3_job/<INPUT_JSON> \
    skip_existing=False \
    prevalidate_inputs=True \
    > /workspace/rfd3_job/rfd3.log 2>&1 & echo $!'
```

Capture the PID for monitoring.

**Key options to include based on user request:**
- `diffusion_batch_size=N` — designs per batch (default 8)
- `n_batches=N` — number of batches (default 1)
- `dump_trajectories=True` — save trajectory (if user wants visualization)
- `low_memory_mode=True` — if GPU memory is tight
- `inference_sampler.num_timesteps=N` — diffusion steps (default 200)
- `inference_sampler.step_scale=N` — diversity vs designability (default 1.5)

---

### Phase 8: Monitor Progress

Poll for completion:

```bash
# Check if process is running
ssh -i $SSH_KEY -o StrictHostKeyChecking=no -p <PORT> root@<HOST> \
  'kill -0 <PID> 2>/dev/null && echo running || echo done'

# Check latest output
ssh -i $SSH_KEY -o StrictHostKeyChecking=no -p <PORT> root@<HOST> \
  'tail -30 /workspace/rfd3_job/rfd3.log'
```

Poll every 30 seconds for short jobs, every 2 minutes for longer ones. Report progress to user.

---

### Phase 9: Retrieve Results

Download all output files:

```bash
ssh -i $SSH_KEY -o StrictHostKeyChecking=no -p <PORT> root@<HOST> \
  'tar czf - -C /workspace/rfd3_job output/' | \
  tar xzf - -C <LOCAL_RESULTS_DIR>
```

List what was generated:
```bash
ls -la <LOCAL_RESULTS_DIR>/output/
```

---

### Phase 10: Cleanup

**Always destroy the instance:**

```bash
vastai destroy instance <ID>
```

---

### Phase 11: Report

Provide the user with:
- GPU used and cost/hr
- Total runtime and estimated cost
- Number of designs generated
- Output file locations (CIF structures, JSON metadata)
- Confirmation that the instance was destroyed

---

## Error Recovery

- **Instance won't start**: Destroy, pick next cheapest offer, retry (up to 3 attempts)
- **RFD3 crashes (OOM)**: Try `low_memory_mode=True`, smaller `diffusion_batch_size`, or a GPU with more VRAM
- **RFD3 crashes (input error)**: Check input JSON, use `prevalidate_inputs=True`
- **SSH won't connect**: Wait 30s, retry 3 times, check instance status
- **Any unrecoverable error**: ALWAYS destroy the instance to stop billing

## Safety Rules

1. **Always confirm cost** with the user before creating an instance
2. **Always destroy** the instance when done or on failure
3. **Never leave instances running** — foundry jobs are finite, always clean up
4. **Track the instance ID** throughout for cleanup
5. If you lose track of the instance ID, run `vastai show instances` to find it by label
