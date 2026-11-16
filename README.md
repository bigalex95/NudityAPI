# NudityAPI

A small FastAPI service that wraps the pre-trained [NudeNet](https://github.com/notAI-tech/NudeNet) models behind a REST API: classify an image as safe/unsafe, or return the image with detected regions drawn on it. It also serves as a deployment sample (Docker, GitHub Actions build/push/deploy). The models are NudeNet's; nothing is trained here.

## Endpoints

| Method | Path | Input | Output |
| --- | --- | --- | --- |
| POST | `/classify/image` | multipart `file` (jpg/jpeg/png) | JSON from NudeNet's `NudeClassifier` (safe/unsafe scores) |
| POST | `/classify/url/{url}` | URL of an image in the path | same JSON |
| POST | `/detect/image` | multipart `file` (jpg/jpeg/png) | PNG with bounding boxes (`NudeDetector`, mode `fast`) |
| POST | `/detect/url/{url}` | URL of an image in the path | PNG with bounding boxes |
| GET | `/` | | redirects to the interactive docs at `/docs` |

Images are resized to fit 256 px before inference. Which endpoints a deployment exposes depends on the entry module:

- `app/main.py`: all four endpoints.
- `app/classifier.py`: classification only (this is what the `Dockerfile` and `Dockerfile.prod` run: `uvicorn classifier:app`).
- `app/detector.py`: detection only.
- `heroku/`: the same code with package-relative imports, used by the `Procfile` (`heroku.main:app`).

## Run locally

Requires Python 3.8 (see `runtime.txt`; dependencies are pinned to 2021 versions). The NudeNet checkpoint is downloaded on first start.

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cd app
uvicorn main:app --port 8000
```

Then open http://localhost:8000/docs, or:

```bash
curl -X POST -F "file=@photo.jpg" http://localhost:8000/classify/image
curl -X POST -F "file=@photo.jpg" http://localhost:8000/detect/image -o boxes.png
curl -X POST "http://localhost:8000/classify/url/https://example.com/photo.jpg"
```

## Docker

```bash
docker build -t nudity .
docker run -p 80:80 nudity            # serves classifier:app on port 80
# or: docker-compose up --build        # uses Dockerfile.prod, mounts ./app
```

The README examples above were written from the code and were not executed in this cleanup (the pinned 2021 dependencies were not installed).

## CI/CD

- `.github/workflows/docker-image.yml`: on push/PR to `main`, builds the `Dockerfile`, pushes it to Docker Hub, then pulls and restarts the container over SSH on two hosts. Needs repository secrets (`DOCKERHUB_*`, `SSH_*`).
- `.github/workflows/main.yml`: deploys to Heroku on push to `main` (needs `HEROKU_API_KEY`).

There are no automated tests.

## Security note

`/classify/url/*` and `/detect/url/*` make the server fetch any URL a client supplies, with no allow-list, timeout or size limit. That is a server-side request forgery (SSRF) and resource-exhaustion risk. Expose these endpoints only behind authentication or on a trusted network, or remove them. The CORS middleware is commented out, and the deploy workflows connect as `root` over SSH. Dependencies are old (2021) and have known vulnerabilities; update them before any public deployment.

## Known issues

- Invalid input returns `{"ERROR", "..."}` in code, which is a Python set (serialized as a JSON array), not an object with an error field.
- `app/` and `heroku/` duplicate each other except for imports.
- `app/utils/img_proccessing.py` has a misspelled name.

## Credits and license

Models and Python package: [NudeNet](https://github.com/notAI-tech/NudeNet) by notAI.tech. Web framework: FastAPI. This repository is under GPL-3.0, see [LICENSE](LICENSE).
