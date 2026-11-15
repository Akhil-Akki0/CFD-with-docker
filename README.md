# CFD Client Website — RIMZ OpenFOAM CFD Platform

A React/TypeScript and Node.js/Express workbench for preparing, queuing, and executing native OpenFOAM v2606 Computational Fluid Dynamics (CFD) cases.

## Accuracy contract

- The backend executes native `blockMesh`, `snappyHexMesh`, `checkMesh`, `simpleFoam`, `chtMultiRegionSimpleFoam`, `decomposePar`, `mpirun`, `reconstructPar`, and `foamToVTK` only.
- A validated STL payload is required. No placeholder geometry or synthetic mesh is created.
- `/healthz` reports that the web service is alive. `/readyz` reports whether the core OpenFOAM pipeline is actually available; it returns `503` and lists missing binaries if it is not.
- CFD metrics are returned only after native meshing, solver execution, VTK extraction, and force-coefficient parsing succeed.
- Failed, aborted, or incomplete runs return `metrics: null`; the UI shows unavailable values instead of estimates.
- All temporary cases execute inside isolated `/tmp/of_cases/<uuid>/` directories with automatic 24-hour disk cleanup.

## Recommended local setup: Docker Compose

The Compose runtime installs the official OpenCFD `openfoam2606-default` package, pins the v2606 bashrc, and checks the required binaries during image build. This is the easiest way to run the web app and its CFD solver together without relying on OpenFOAM being installed on the host.

```bash
docker compose up --build
```

Open `http://localhost:3000`. The solver-ready endpoint is `http://localhost:3000/readyz`; a ready response should include an empty `missingBinaries` list and HTTP 200.

## Run the Node development server directly on Linux/WSL

`npm run dev` starts the web/backend process but does not install OpenFOAM. Install the matching OpenCFD package first, then provide its environment to the backend:

```bash
curl -fsSL https://dl.openfoam.com/add-debian-repo.sh | sudo bash
sudo apt-get update
sudo apt-get install openfoam2606-default
export OPENFOAM_BASHRC=/usr/lib/openfoam/openfoam2606/etc/bashrc
source "$OPENFOAM_BASHRC"
npm ci
npm run dev
```

On Windows, use Ubuntu under WSL2 for native execution. Alternatively, use Docker Desktop with Linux containers and Docker Compose.

A `200` from `/healthz` alone does **not** prove the solver is installed. `/readyz` checks `blockMesh`, `snappyHexMesh`, `checkMesh`, `foamToVTK`, and `simpleFoam`; if one is missing, it returns `503` and names the missing executable. Parallel mode additionally needs `decomposePar`, `mpirun`, and `reconstructPar`.

## Backend architecture

- **Single backend entry point:** `server.ts` (Express + Vite dev middleware / compiled production `dist/server.cjs`)
- **Backend modules (`/backend`):** `openfoamRunner.js` handles isolated native execution and streaming; `openfoamDiagnostics.js` translates solver errors and parses mesh health; `validation.js` validates payloads and STL data; `auth.js` handles authentication; `logger.js` writes redacted structured logs.

## Development checks

```bash
npm ci
npm run lint
npm run build
```

The official OpenCFD v2606 release provides Ubuntu packages; the Docker runtime uses the OpenCFD Debian/Ubuntu repository and installs `openfoam2606-default` rather than depending on a floating `latest` image tag. See the [official release](https://www.openfoam.com/news/main-news/openfoam-v2606) and [Debian/Ubuntu installation instructions](https://gitlab.com/openfoam/core/openfoam/-/wikis/precompiled/debian).
