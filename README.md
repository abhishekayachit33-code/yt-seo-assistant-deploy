# yt-seo-assistant deploy

Manifests for the `yt-seo` namespace, synced by ArgoCD from `manifests/`
(`automated`, with `prune` and `selfHeal`).

**`kubectl apply` against `yt-seo` will not stick.** selfHeal reverts it and
prune deletes anything not in git. Change the manifests here and let ArgoCD
sync; to try something out first, deploy it to a scratch namespace instead.

## What runs

| Workload | Image | Port | Purpose |
|---|---|---|---|
| `yt-seo-web` | `yt-seo-web` | 3000 | Next.js frontend |
| `yt-seo-api` | `yt-seo-api` | 8000 | FastAPI backend |
| `yt-seo-assistant` | `yt-seo-assistant` | 8501 | Streamlit UI, being retired |
| `postgres` | `postgres:16` | 5432 | history + users, backed by a 5Gi PVC |

The Streamlit deployment is intentionally still here so the old and new UIs
can be compared against one database. Remove it, its Service, and its image
build once the React frontend is signed off.

## Routing

Everything is served from one host, `ytseo.local`:

- `/` and everything else → `yt-seo-web`
- `/api/v1/*` → `yt-seo-api`, with the `/api/v1` prefix stripped

The backend sits at `/api/v1`, **not** `/api`. Next.js serves NextAuth's own
handlers at `/api/auth/*`; routing all of `/api` to FastAPI swallows
`/api/auth/session` and breaks sign-in while every page still renders and
both pods stay healthy.

One origin also means the browser makes no cross-origin calls, so CORS never
enters the picture for normal use.

## Local access (minikube, docker driver)

The node IP is not routable from the host on the docker driver, so NodePort
addresses do not work from a browser. Use the ingress:

```sh
minikube addons enable ingress
sudo sh -c 'echo "127.0.0.1  ytseo.local" >> /etc/hosts'
minikube tunnel          # keep running; binds the controller on 127.0.0.1:80
```

Then open <http://ytseo.local>.

## Secrets

`yt-seo-secrets` is created out of band and is not in git. Required keys:

| Key | Used by |
|---|---|
| `DATABASE_URL` | api, streamlit |
| `POSTGRES_PASSWORD` | postgres |
| `YOUTUBE_API_KEY` | api, streamlit |
| `GEMINI_API_KEY`, `GEMINI_API_KEY_2` | api, streamlit |
| `DEEPSEEK_API_KEY` | api (optional; falls back to Gemini) |
| `HUGGINGFACE_API_KEY` | api (optional; thumbnail generation) |
| `SUPADATA_API_KEY` | api (optional; transcript fallback) |
| `JWT_SECRET` | api — signs auth tokens |
| `AUTH_SECRET` | web — NextAuth session encryption |

Add the two new ones to an existing cluster with:

```sh
kubectl create secret generic yt-seo-secrets -n yt-seo \
  --from-literal=JWT_SECRET="$(python3 -c 'import secrets;print(secrets.token_hex(32))')" \
  --from-literal=AUTH_SECRET="$(openssl rand -hex 32)" \
  --dry-run=client -o yaml | kubectl patch -n yt-seo --type merge -p "$(cat -)" secret yt-seo-secrets
```

`JWT_SECRET` and `AUTH_SECRET` are independent: the first signs the bearer
token the API issues, the second encrypts the NextAuth cookie. Rotating
either logs everyone out; rotating neither is required to deploy.

## Changing the public API address

`NEXT_PUBLIC_API_URL` is inlined into the client bundle when the web image is
**built**, not read at runtime. The value in `web-deployment.yaml` is there
for visibility only — editing it changes nothing the browser does. To point
the frontend somewhere else, rebuild the image with a new
`--build-arg NEXT_PUBLIC_API_URL=...` (see `PUBLIC_API_URL` in the app
repo's `Jenkinsfile`).
