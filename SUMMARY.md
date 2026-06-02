# Docker Entrypoint Changes

This change keeps the self-hosted Docker image reusable across deployments while preserving the existing read-only runtime hardening.

## Why Runtime Substitution Exists

The Docker image is built with placeholder values for public environment variables, such as `__NEXT_PUBLIC_SUPABASE_URL__`. At container startup, `docker-entrypoint.sh` replaces those placeholders with values from `.env` so the same image can run against different Supabase projects and public app URLs.

## What Was Missing

The previous substitution covered common built assets under `/app/.next`, but Next.js standalone also stores runtime routing and header configuration in JSON manifest files. In this app, the Content-Security-Policy header came from `/app/.next/routes-manifest.json`, so the browser received a CSP containing `__NEXT_PUBLIC_SUPABASE_URL__` instead of the real Supabase URL.

That could block browser-side Supabase requests even though the container had the correct runtime environment variables.

## What Changed

- Placeholder substitution now includes `.json` files under `/app/.next`, covering Next.js route and header manifests.
- The standalone `server.js` is copied to `/app/.next/cache/gnubok-server.js` before substitution because `/app/server.js` lives on a read-only filesystem at runtime.
- The copied server script is patched to keep `/app` as the runtime root, so Next.js still finds `.next` and Node still resolves `/app/node_modules` correctly.

## Why This Approach

This avoids weakening the container by making the image filesystem writable. Only the runtime tmpfs/cache area is modified, while the shipped bundle remains immutable after startup. It also keeps the existing generic-image deployment model instead of requiring a rebuild for every environment.
