# BYY Shared Docs — Claude Code Guide

The single source of truth for **documentation shared across all Best Year Yet teams** — Rails app, Analytics/Marketing, Front End (landing pages), and Legacy (PRO ColdFusion app). GitHub repo: `twminn/byy-shared-docs` (public). **Documentation only — nothing here is built or deployed.**

> Read automatically by Claude Code each session.

## What this repo is (and isn't)

- It's Markdown specs and guides, organized by domain:
  - `analytics/` — GTM/GA4, cross-domain tracking, Meta Pixel, deploy annotations, engagement tracking
  - `api/` — Rails API specs (e.g. landing-page lead capture), CORS, post-checkout flows
  - `ghl/` — GoHighLevel integration overview, implementation guide, API v2 migration
  - `legacy/` — notes/tasks for the BYY PRO ColdFusion team
- There is **no build, no test, no deploy** here. Don't look for or run package managers.

## This repo is a submodule elsewhere — edits propagate

This repo is included as a **read-only submodule** in several BYY projects (`BYY` Rails app, `BYY-Marketing`, `BYY-Landing` as `docs/shared-docs`, `BYY-Legacy` as `shared-docs`). Changes made *here* become the upstream those projects pull. So:

- Follow the contributing flow: branch → edit `.md` → PR → **request review from any other team the change affects** → merge.
- When a change affects another project's behavior (API params, GHL config, tracking), call it out so both teams update their env/config.

## Shared constants

- GTM container: `GTM-K9PVK9ZF` · GA4 Measurement ID: `G-6GPCZV3DHR`
- Lead capture endpoints: staging `https://byy-staging.bestyearyet.io/api/v1/landing_leads`, prod `https://bestyearyet.io/api/v1/landing_leads`

## AWS

All Best Year Yet projects use AWS account `byy` = `876524020257` (profile `byy`), **not** the M111/default account `918360870198`. This repo itself deploys nothing, but keep the rule in mind for any infra references in the docs.

Start at `README.md` for the action-item dashboard and per-team quick links.
