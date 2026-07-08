# Videoflix

A Netflix-style video-streaming app: register and sign in, then watch videos
delivered as adaptive **HLS** streams (480p / 720p / 1080p). Uploaded videos are
transcoded in the background with FFmpeg.

This is an **umbrella repository**: it ties the two repositories that make up the
project together as **git submodules** and adds a single Docker Compose file to
run the whole stack locally with one command.

## Repository layout

| Path        | What it is | Repository (submodule) |
| ----------- | ---------- | ---------------------- |
| `backend/`  | Django REST API — auth (JWT in HttpOnly cookies), HLS transcoding, streaming | [`tranqn/videoflix-backend`](https://github.com/tranqn/videoflix-backend) |
| `frontend/` | Vanilla-JS client | **fork** of the Developer Akademie [`project.Videoflix`](https://github.com/Developer-Akademie-Backendkurs/project.Videoflix), forked to [`tranqn/project.Videoflix`](https://github.com/tranqn/project.Videoflix) |

The frontend is a fork so upstream changes can still be pulled in (`git fetch
upstream`) while the customised client lives on the fork's `main`.

## Getting the code

`backend/` and `frontend/` are git submodules, so clone with
`--recurse-submodules` to pull everything in one go:

```bash
git clone --recurse-submodules https://github.com/tranqn/videoflix.git
cd videoflix
```

Already cloned without it? Pull the submodules in afterwards:

```bash
git submodule update --init --recursive
```

To later update a submodule to its latest commit: `git submodule update --remote backend`.

## Quickstart (local, one command)

Prerequisites: [Docker](https://docs.docker.com/get-docker/) with Compose v2.20+.

```bash
# 1. Create the backend environment file and fill in the values
cp backend/.env.template backend/.env
#    Set at least: SECRET_KEY, DB_NAME / DB_USER / DB_PASSWORD and the EMAIL_* SMTP
#    credentials. The defaults already target the frontend on 127.0.0.1:5500.

# 2. Build and start the whole stack (Postgres, Redis, API + RQ worker, frontend)
docker compose up --build
```

Then open **<http://127.0.0.1:5500>**.

> Use `127.0.0.1`, **not** `localhost`. The frontend calls the API at
> `http://127.0.0.1:8000`, and the HttpOnly auth cookies are only sent back when
> both sides use the same host.

| Service | URL |
| ------- | --- |
| Frontend (open this) | <http://127.0.0.1:5500> |
| API | <http://127.0.0.1:8000/api/> |
| Django admin | <http://127.0.0.1:8000/admin/> |
| Background-jobs dashboard | <http://127.0.0.1:8000/django-rq/> |

The backend entrypoint runs the migrations and creates the admin superuser
automatically from the `DJANGO_SUPERUSER_*` values in `backend/.env`.

### Adding a video

Open the admin, add a **Video** (title, description, category) with a source
file. On save, a background job builds a thumbnail and transcodes the video to
HLS in all three resolutions; its status flips to **Ready** when finished and it
appears in the frontend. Watch progress on the RQ dashboard.

## How the Compose setup works

`docker-compose.yml` in this folder **includes the backend's own
`backend/docker-compose.yml` unchanged** (Postgres, Redis, Django API and the RQ
worker) and adds one small service: a Caddy static server that serves
`frontend/` on port **5500** — the origin the API's CORS and cookie settings
already expect. So you don't have to run the frontend separately (no Live Server).

For **production** (single origin, automatic HTTPS via Caddy) use the backend's
dedicated stack instead:

```bash
cd backend && docker compose -f compose.prod.yml up -d --build
```

## Running the tests

```bash
docker compose exec web python manage.py test
```

The suite uses in-memory SQLite, a local-memory cache and synchronous jobs, so
it needs no external services.

## More documentation

- [`backend/README.md`](backend/README.md) — full API reference, endpoint table and details
- [`frontend/readme.md`](frontend/readme.md) — frontend notes
