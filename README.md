# GTZAN YOLOv8n-cls Genre Classifier API — Render deployment

Same API as the Hugging Face version (`GET /`, `POST /predict`), rebuilt
to fit Render's free-tier 512MB RAM limit:
- Matplotlib dropped as a dependency (the magma colormap is a baked-in
  256-entry lookup table instead of a live `matplotlib.cm` import).
- CPU-only torch wheel (`--extra-index-url .../whl/cpu`) instead of the
  default PyPI torch, which otherwise pulls in unused CUDA libraries.

**Honest caveat:** even trimmed, `torch` + `ultralytics` + `librosa`
still add up to a real amount of RAM. This should fit in 512MB, but if
the Render logs show the service getting killed (look for "Out of
memory" or the process restarting right after a request), that's the
free tier's ceiling — see the fallback note at the bottom.

## Deploy steps

1. Go to https://render.com and sign up (no credit card required for the free tier).
2. Push this folder to a new GitHub repo (Render deploys from a Git repo, not a direct file upload).
3. In the Render dashboard: **New > Web Service**, connect the repo.
4. Render should auto-detect the `Dockerfile`. If asked, set:
   - **Environment**: Docker
   - **Instance Type**: Free
5. Deploy. First build takes several minutes (installing torch/ultralytics from scratch).
6. Once live, your API base URL will look like:
   `https://<your-service-name>.onrender.com`

## Endpoints

Same as before — `GET /` for a health check, `POST /predict` with a
multipart `file` field (WAV, MP3, M4A, OGG, or FLAC).

## Cold starts

Render's free tier spins the service down after 15 minutes of no
traffic. The next request after that triggers a cold start — expect
20–40 seconds before the first response, same idea as Hugging Face's
sleep behavior, just a shorter idle window (15 min vs 48 hr).

## If it runs out of memory

If the free instance can't hold torch + ultralytics + librosa in
512MB reliably, the two fallbacks, in order of effort:
1. **Export the model to ONNX** and swap `ultralytics`'s `YOLO(...)`
   for `onnxruntime` inference instead — onnxruntime's CPU footprint is
   substantially smaller than full PyTorch. This needs a small rewrite
   of the prediction code but keeps everything else the same. Ask me
   and I'll build that version.
2. **Move to Google Cloud Run's free tier** instead, which allows
   configuring more RAM per instance while still scaling to zero
   (no cost when idle) — better fit for this dependency stack, at the
   cost of needing a card on file (won't be charged within free limits).
