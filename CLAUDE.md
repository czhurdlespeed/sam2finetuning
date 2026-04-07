# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SAM2 Fine-tuning is a full-stack web application for fine-tuning Meta's Segment Anything Model 2 (SAM 2) on custom datasets. It consists of a Next.js frontend/backend and a Python-based ML training backend, connected via two git submodules.

## Repository Structure

- **sam2loranocodefinetuning/** — Next.js 16 web app (TypeScript, React 19). Handles UI, auth, job management, and streams training logs via SSE.
- **modalsam2/** — Fork of Meta's SAM 2 repo with training support. Python/PyTorch, uses Hydra configs for training runs.

Training jobs are submitted to Modal Labs (serverless GPU), checkpoints stored in Cloudflare R2, database is Neon PostgreSQL via Drizzle ORM, auth is Better-Auth (GitHub/Google OAuth).

## Common Commands

### Frontend (sam2loranocodefinetuning/)

```bash
pnpm install              # Install dependencies
pnpm dev                  # Dev server on localhost:3000
pnpm build                # Production build
pnpm lint                 # ESLint
pnpm db:generate          # Generate Drizzle migrations
pnpm db:migrate           # Run migrations
pnpm db:push              # Push schema to DB
pnpm db:studio            # Open Drizzle Studio
pnpm auth:generate        # Generate auth schema
pnpm create-user          # Create user via CLI
```

### Training Backend (modalsam2/sam2/)

```bash
pip install -e ".[dev]"   # Install with dev deps
cd checkpoints && ./download_ckpts.sh && cd ..  # Download model checkpoints

# Single-node training
python training/train.py \
  -c configs/sam2.1_training/sam2.1_hiera_b+_MOSE_finetune.yaml \
  --use-cluster 0 --num-gpus 8
```

## Architecture

### Data Flow

1. User authenticates (GitHub/Google OAuth) → admin must approve before training
2. User selects training params (LoRA rank, checkpoint size, dataset, epochs) in `Config.tsx`
3. `POST /api/train` submits job to Modal Labs, streams SSE logs back to `LogsDisplay.tsx`
4. Modal notifies `POST /api/jobs/complete` on finish; checkpoint uploaded to R2
5. User downloads checkpoint via `GET /api/download`

### Key Frontend Files

- `app/api/train/route.ts` — Training job submission, SSE streaming
- `app/api/jobs/route.ts` & `app/api/jobs/complete/route.ts` — Job CRUD and completion webhook
- `src/components/Config.tsx` — Main training configuration UI
- `src/components/TrainingControls.tsx` — Start/cancel training
- `src/components/LogsDisplay.tsx` — Real-time training log viewer
- `src/db/schema.ts` — Drizzle ORM schema (user, session, account, verification, training_job tables)
- `src/lib/auth.ts` — Better-Auth server config
- `src/constants/trainingConfig.ts` — UI dropdown options (ranks: 2/4/8/16/32; checkpoints: tiny/small/base+/large; datasets: MAZAK/irPOLYMER/visPOLYMER/TIG)

### Key Backend Files

- `training/train.py` — Training launch script (single/multi-node)
- `training/trainer.py` — Main training loop
- `training/model/sam2_train.py` — SAM2Train model class
- `training/dataset/` — Dataset loaders (VOS, SA-1B, SA-V, DAVIS)
- `training/loss_fns.py` — MultiStepMultiMasksAndIous loss
- `sam2/configs/` — Hydra configuration YAML files

### External Services

| Service | Purpose |
|---------|---------|
| Modal Labs | Serverless GPU training (`MODAL_TRAIN_URL`, `MODAL_KEY`, `MODAL_SECRET`) |
| Neon | Serverless PostgreSQL (`DATABASE_URL`) |
| Cloudflare R2 | Checkpoint storage (`CF_R2_*`, `AWS_*`) |
| Better-Auth | OAuth with GitHub/Google (`BETTER_AUTH_*`) |
| LiveKit | Voice agent integration (`LIVEKIT_*`) |
| Resend | Email notifications |
| Logfire | Observability |

## Tech Stack Summary

**Frontend**: Next.js 16, React 19, TypeScript 5, Tailwind CSS 4, Material UI 7, Drizzle ORM, PNPM
**Backend ML**: Python 3.10+, PyTorch 2.5+, Hydra 1.3, PEFT/LoRA, Modal Labs
**Infra**: Vercel (web), Modal (training), Neon (DB), Cloudflare R2 (storage)
