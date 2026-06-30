# Boltz 2.2.1 Cached Docker Runner

This repository contains a small Boltz 2.2.1 Docker workflow for running predictions without downloading the Boltz cache at runtime. The Docker image bakes the cache into `/opt/boltz-cache`, and `run.sh` mounts the current folder as `/workspace`.

## Repository Contents

- `run.sh` - editable Docker run command for `NusA_open.yaml`.
- `Dockerfile` - builds a Boltz 2.2.1 image with the local `.boltz/` cache baked in.
- `requirements.txt` - Python package pin for Boltz.
- `REQUIREMENTS.md` - host, GPU, disk, and cache requirements.
- `DOCKER_HUB.md` - copy-ready Docker Hub overview text.
- `NusA_open.yaml`, `NusA_close.yaml`, `affinity.yaml` - example Boltz input files.
- `boltz_results_NusA_open/`, `boltz_results_NusA_close/` - example output folders.

## Build The Image

The build needs a local `.boltz/` folder with the downloaded Boltz2 components:

```text
.boltz/
  boltz2_aff.ckpt
  boltz2_conf.ckpt
  mols.tar
  mols/
```

Build:

```bash
docker build -t boltz:221 .
```

The `.boltz/` folder is ignored by Git, but it is intentionally included in the Docker build context so the image can be self-contained.

## Run A Prediction

Edit `run.sh` directly if you want a different input file or sampling settings, then run:

```bash
bash run.sh
```

The script uses this cache path:

```bash
--cache /opt/boltz-cache
```

That path is inside the image, so the first prediction should not download the Boltz model or molecule cache.

## CLI Options

`run.sh` intentionally keeps the command explicit so users can edit it directly.

Docker options:

- `docker run` - starts a new container from the image.
- `--rm` - removes the container after the run finishes.
- `-it` - runs interactively with a terminal attached.
- `--gpus all` - exposes all available NVIDIA GPUs to the container.
- `--shm-size=8g` - gives the container an 8 GB shared-memory segment.
- `-v "${PWD}:/workspace"` - mounts the current folder into the container.
- `-w /workspace` - runs commands from the mounted workspace.
- `--entrypoint /bin/bash` - starts Bash instead of the image default entrypoint.
- `boltz:221` - Docker image tag used for prediction.
- `-lc '...'` - asks Bash to run the quoted Boltz command.

Boltz options:

- `predict NusA_open.yaml` - runs structure prediction for the selected YAML input.
- `--use_msa_server` - asks Boltz to generate missing protein MSAs through the remote MSA server.
- `--no_kernels` - disables optional optimized kernels; useful when kernel compatibility is uncertain.
- `--cache /opt/boltz-cache` - uses the cache baked into the Docker image.
- `--diffusion_samples 20` - generates 20 structure samples.
- `--recycling_steps 10` - runs 10 recycling iterations.
- `--max_parallel_samples 1` - keeps samples serial to reduce VRAM pressure.
- `--step_scale 1.0` - sets the diffusion step scale.
- `--use_potentials` - enables physical/contact guidance potentials.
- `--output_format pdb` - writes prediction structures as PDB files.

Useful additional Boltz options:

- `--accelerator cpu` - runs on CPU instead of GPU; slower but avoids VRAM limits.
- `--accelerator gpu` - runs on CUDA GPU.
- `--devices 1` - uses one CPU/GPU device.
- `--sampling_steps N` - controls diffusion sampling steps; lower values are useful for smoke tests.
- `--num_workers 0` - disables extra dataloader worker processes.
- `--out_dir PATH` - writes outputs to a selected directory.
- `--seed N` - makes sampling reproducible for comparable tests.
- `--override` - overwrites existing prediction outputs.

## GPU Guidance

Boltz2 can fill small laptop GPUs. On an 8 GB GPU, keep:

```bash
--max_parallel_samples 1
```

If a run fails with CUDA out-of-memory, lower:

```bash
--diffusion_samples
--recycling_steps
```

For a very conservative first test, edit `run.sh` to:

```bash
--diffusion_samples 1
--recycling_steps 3
--max_parallel_samples 1
```

## Laptop Smoke Benchmark

A tiny local smoke test was run on this machine without writing outputs to the repository:

- Input: 40 amino acids, single protein chain, `msa: empty`.
- Settings: `diffusion_samples=1`, `recycling_steps=1`, `sampling_steps=5`, `max_parallel_samples=1`, `num_workers=0`, `--no_kernels`.
- CPU result: `506.502 s`.
- GPU result: `50.981 s`.
- GPU speedup: about `9.9x`.

This test confirms that the laptop GPU can run a very small Boltz2 job, but it does not prove that the full 495-residue NusA examples will fit in 8 GB VRAM.
