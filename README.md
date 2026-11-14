# S.E.C.U.R.E. Privacy parameter server

Coordinates collaborative training: requests local training from the participant server for each collaborator and aggregates the returned weights.

## Configuration

Read from environment variables (or a local `.env` file, which is never copied into the image).

| Variable | Value |
|---|---|
| `MONGODB_URI` | MongoDB connection string |
| `TAPIS_BASE_URL` | `https://icicleai.tapis.io` |
| `SANDBOX_SERVER_URL` | Participant server URL |
| `HOST`, `PORT`, `DEBUG` | Optional; default `0.0.0.0`, `8000`, `false`. Never enable `DEBUG` on a pod. |

`GET /health` returns `{"status": "ok", "build_sha": ...}` for the deployment check.

## Run locally

```bash
uv sync
uv run python app.py
```

Dependencies are locked with uv: after changing `pyproject.toml`, run `uv lock` and commit `uv.lock`.

## Deployment

Deployed to the `secureprivacyparam` Tapis pod (image `ghcr.io/icicle-ai/secureprivacyparam`) by `.github/workflows/deploy.yaml`, using the shared
[ICICLE CI/CD templates](https://github.com/ICICLE-ai/cicd-templates/wiki/Quick-start) (`v2.3.0`).
Bump `service-version` in `icicle-service.yaml` and push to `main` to release.
CI checks the rollout through `GET /health`.
