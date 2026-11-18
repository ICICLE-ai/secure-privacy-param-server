# S.E.C.U.R.E. Privacy parameter server

Coordinates collaborative training: requests local training from the participant server for each collaborator and aggregates the returned weights. It is one of the four services of S.E.C.U.R.E. Privacy, Secure Exchange, Compliance, Understanding & Risk Evaluation: a domain-agnostic collaborative machine learning platform where researchers analyze data and train models together with privacy-preserving features.

**Tags:** CI4AI, Software

### License

[![License](https://img.shields.io/badge/License-BSD_3--Clause-yellow.svg)](https://opensource.org/licenses/BSD-3-Clause)

Distributed under the BSD 3-Clause License. See [`LICENSE`](LICENSE).

## References

S.E.C.U.R.E. Privacy is split into four services, each in its own repository:

- [S.E.C.U.R.E. Privacy Frontend](https://github.com/ICICLE-ai/secure-privacy-frontend)
- [S.E.C.U.R.E. Privacy App Server](https://github.com/ICICLE-ai/secure-privacy-app-server)
- [S.E.C.U.R.E. Privacy Participant Server](https://github.com/ICICLE-ai/secure-privacy-participant-server)
- [S.E.C.U.R.E. Privacy Param Server](https://github.com/ICICLE-ai/secure-privacy-param-server) (this repository)

Related resources:

- [ICICLE CI/CD templates](https://github.com/ICICLE-ai/cicd-templates/wiki), which build and deploy each service
- [Tapis](https://tapis.readthedocs.io/), which provides login (OAuth2) and the pods the services run in
- [Adversarial Robustness Toolbox](https://adversarial-robustness-toolbox.readthedocs.io/), used for membership-inference risk analysis

Key terms:

- **Differential privacy**: adding calibrated random noise to data so that no single record can be singled out.
- **Geo-indistinguishability**: differential privacy for locations; a point is moved by a random distance so nearby places look alike.
- **k-anonymity**: every combination of identifying attributes is shared by at least *k* records.
- **Collaborative (federated) learning**: each participant trains on their own data, and only model weights are combined.
- **Membership inference attack**: guessing from a model's outputs whether a record was in its training data.

## Acknowledgements

This research was also supported in part by the National Science Foundation (NSF) under award 2112533, and by the United States Department of Agriculture (USDA) under grant number NR233A750004G019.

*National Science Foundation (NSF) funded AI institute for Intelligent Cyberinfrastructure with Computational Learning in the Environment (ICICLE) (OAC 2112606)*

## Issue reporting

Report bugs and request features in [GitHub Issues](https://github.com/ICICLE-ai/secure-privacy-param-server/issues).

---

# Tutorials

## Run the param server locally

This walkthrough starts the param server on your machine against a local MongoDB and checks that it responds.

**Prerequisites:** Python 3.12, [uv](https://docs.astral.sh/uv/) and Docker.

1. Start a local MongoDB:

   ```bash
   docker run -d --name mongo -p 27017:27017 mongo:9.0
   ```

2. Create a `.env` file in the repository root:

   ```bash
   MONGODB_URI=mongodb://localhost:27017/secure_privacy
   TAPIS_BASE_URL=https://icicleai.tapis.io
   SANDBOX_SERVER_URL=http://localhost:8001   # a running participant server, used for training
   ```

3. Install the dependencies and start the server:

   ```bash
   uv sync
   uv run python app.py
   ```

   Dependencies are locked with uv: after changing `pyproject.toml`, run `uv lock` and commit `uv.lock`.

4. Check that it is up:

   ```bash
   curl http://localhost:8000/health   # {"build_sha": "unknown", "status": "ok", ...}
   ```

---

# How-To Guides

## Configure the param server

Read from environment variables (or a local `.env` file, which is never copied into the image).

| Variable | Value |
|---|---|
| `MONGODB_URI` | MongoDB connection string |
| `TAPIS_BASE_URL` | `https://icicleai.tapis.io` |
| `SANDBOX_SERVER_URL` | Participant server URL |
| `HOST`, `PORT`, `DEBUG` | Optional; default `0.0.0.0`, `8000`, `false`. Never enable `DEBUG` on a pod. |

`GET /health` returns `{"status": "ok", "build_sha": ...}` for the deployment check.

## Release a new version

Deployed to the `secureprivacyparam` Tapis pod (image `ghcr.io/icicle-ai/secureprivacyparam`) by `.github/workflows/deploy.yaml`, using the shared
[ICICLE CI/CD templates](https://github.com/ICICLE-ai/cicd-templates/wiki/Quick-start) (`v2.3.0`).
Bump `service-version` in `icicle-service.yaml` and push to `main` to release.
CI checks the rollout through `GET /health`.

---

# Explanation

## How the services fit together

This repository is the **param server**.

```
browser ──> frontend (secureprivacy, Nginx)
              ├─ /         static React build
              └─ /api/...  ─> app server (secureprivacyapp) ── dataset routes ──> participant server (secureprivacyparticipant)
                                     └── /api/start_training ──> param server (secureprivacyparam) ──> participant server
```

The browser only calls relative `/api/...` URLs. The frontend's Nginx forwards them to the app
server, which is the only backend the browser reaches. All three backends share one MongoDB-compatible
database, and each runs in its own Tapis pod, deployed from its own repository.

## Collaborative training

`/api/start_training` receives the collaborators, dataset metadata and hyperparameters for a model. For each
collaborator it calls the participant server's `/api/trainLocalModel` in parallel, so each participant's
model is trained on their own data. It then averages the returned weights layer by layer (federated
averaging) and saves the result on the model record in MongoDB, marking it `Done`. Raw data never leaves
the participant server; only model weights reach this server.
