# Dockerfile Notes

## Baseline build
- Image size: 350 MB
- Output of `docker run --rm xalid110/m7-03-cat-detection:v1`:
  ```
  ONNX model loaded OK: /home/app/model.onnx
    inputs:  1
    outputs: 1
  ```

## Stage 1 (builder) — why it exists

Stage 1 exists to do all the heavy lifting that must never end up in the shipped image: installing build-essential (gcc, make, the full compiler toolchain), downloading and extracting the ~200 MB ONNX Runtime tarball, compiling check_model.c against the ORT headers, and running the validation gate that rejects a missing or malformed model.onnx at build time. Without this stage, the runtime image would have to carry build-essential and the full ORT tarball — easily an extra 400–500 MB — plus there would be no clean separation between "things needed to compile" and "things needed to run," which is a security surface as well as a size problem.

## Stage 2 (runtime) — why it exists

Stage 2 exists to ship the absolute minimum needed to run the verifier: the Debian base layer, two apt packages (ca-certificates and libstdc++6), the compiled check_model binary, the libonnxruntime.so shared library, the model file, and a non-root user. Everything the builder needed — build-essential, curl, the ORT headers and tarball, the C source file — is discarded simply by starting a new `FROM`. The result is an image that can't be used to compile arbitrary code, has no package manager overhead, and lands at ~350 MB instead of the ~600 MB a single-stage build would produce (the model alone is 82 MB, which explains why we don't hit the ~250 MB guideline — that assumes a smaller model).

## Three architectural decisions in this Dockerfile

1. **Multi-stage split** — Without the builder/runtime split, every layer added in stage 1 (build-essential ~220 MB, ORT tarball ~200 MB before extraction, curl, headers) would be present in the final image. Removing the split would turn a ~350 MB image into a ~700 MB image and would ship a full C compiler into production — a significant supply-chain risk.

2. **`apt-get … && rm -rf /var/lib/apt/lists/*` pattern** — The apt cache (`/var/lib/apt/lists/`) is created during `apt-get update` and is not needed after install. Because Docker commits a new layer for every `RUN`, separating `update`, `install`, and `rm -rf` into separate RUN lines would freeze the cache directory into an immutable lower layer that can never be reclaimed. Combining them in a single `RUN` means the apt cache never exists in any committed layer, keeping each stage's base layer as small as possible.

3. **Non-root user (`useradd --uid 1001 app`)** — Without this, the container's process runs as uid 0 (root). If an attacker exploits a vulnerability in the ONNX Runtime C library or in check_model itself, root-in-container can map to host capabilities depending on the container runtime configuration. The non-root user narrows blast radius to what uid 1001 can reach inside the container's filesystem, and satisfies most container security policies (Kubernetes PodSecurityAdmission restricted profile, for example, rejects images that run as root).

## Final build (v2)

- Image size: 350 MB
- Labels:
  ```json
  {"maintainer":"Xalid110","model.framework":"ultralytics-yolo26","model.source":"m6-09-assessment","ort.version":"1.20.1"}
  ```
- Healthcheck spec:
  ```
  {[CMD-SHELL check_model /home/app/model.onnx || exit 1] 30s 10s 5s 0s 3}
  ```
  (interval=30s, timeout=10s, start-period=5s, retries=3)
- Base image pinned by digest: `debian:12-slim@sha256:0104b334637a5f19aa9c983a91b54c89887c0984081f2068983107a6f6c21eeb`
