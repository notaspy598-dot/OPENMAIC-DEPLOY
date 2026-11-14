# OpenMAIC on the VPS (via Coolify)

The image is built on GitHub, so the VPS only pulls and runs it. OpenMAIC is
capped at 1.5 GB of RAM and its Postgres at 384 MB.

| File | What it is |
|---|---|
| `.github/workflows/build-openmaic.yml` | Builds a pinned OpenMAIC release and pushes it to `ghcr.io/<you>/openmaic` |
| `docker-compose.yml` | What Coolify deploys: the app and its own Postgres 16 |
| `openmaic.yml` | Models: Claude Sonnet with Haiku as fallback, your SearXNG for web search, and video/images/voice off |

## 1. Build the image on GitHub

1. Create a **private** GitHub repo, for example `openmaic-deploy`, and push this folder to it.
2. In the repo, go to **Actions → Build OpenMAIC image → Run workflow**. Keep `v1.4.2`, the latest stable release, or enter a newer tag.
3. Wait for it to go green. The first build takes roughly 15–30 minutes; later builds are faster because of the cache.

The image ends up at `ghcr.io/<your-github-username>/openmaic:v1.4.2`, and it is private.

## 2. Let the VPS pull the private image

Create a GitHub token: **Settings → Developer settings → Personal access tokens (classic)**. It needs only the `read:packages` scope. Then run this on the VPS:

```bash
docker login ghcr.io -u <your-github-username>
```

Paste the token when it asks for a password. Coolify uses the server's Docker login when it pulls images.

## 3. Generate the secrets

Run these on the VPS and keep the outputs. The Postgres password must be letters and digits only, which `-hex` gives you.

```bash
openssl rand -hex 24      # POSTGRES_PASSWORD
openssl rand -hex 24      # ACCESS_CODE
openssl rand -base64 32   # OPENMAIC_SECRET_KEY
```

## 4. Create the resource in Coolify

1. **New Resource → Docker Compose Empty**, then paste `docker-compose.yml` in.
2. Replace `CHANGE_ME` with your GitHub username, in lowercase.
3. **Environment Variables:** set these:
   - `POSTGRES_PASSWORD`, `ACCESS_CODE`, `OPENMAIC_SECRET_KEY`: from step 3.
   - `ANTHROPIC_API_KEY`: your Anthropic key.
   - `SEARXNG_URL`: your SearXNG's address. The simplest is its public `https://` domain. To use the internal address instead, turn on **Connect to Predefined Network** for both resources and use the SearXNG container's name with port `8080`. SearXNG's `settings.yml` must allow JSON: `search.formats: [html, json]`.
4. **Storages:** Coolify lists the `openmaic.yml` file mount. Paste the contents of `openmaic.yml` into it **before** the first deploy. If the file is missing, Docker creates a folder in its place and OpenMAIC won't start.
5. **Domain:** on the `openmaic` service, set `https://learn.<your-domain>:3000`. The `:3000` is the container port; Coolify adds HTTPS.
6. Deploy. Then open the domain and enter the `ACCESS_CODE`.

## 5. Check it isn't crowding the server

```bash
docker stats --no-stream | grep -Ei "openmaic|postgres-"
```

```bash
free -h
```

OpenMAIC should sit well under its 1.5 GB cap while idle. If it keeps hitting the cap and restarting, raise `mem_limit` a little, or free RAM from Supabase. Supabase's analytics, studio and imgproxy containers are the easiest to remove.

## Upgrading

OpenMAIC upgrades can be **one-way** (database migrations). First, back up the Postgres volume, or use Coolify's scheduled backup on the `postgres` service. Then:

1. Read the release notes for the new tag.
2. Run the workflow with that tag.
3. Change the tag in the compose file in Coolify, and redeploy.

## Turning features on later

In `openmaic.yml`:

- **Voice (AI teachers that speak):** add a TTS provider and set the `tts` slot.
- **Speech-to-text for uploaded videos:** add an ASR provider, such as `openai-whisper` with an OpenAI key, and set the `asr` slot.
- **Images / video:** these are the most expensive per lesson, so leave them off until you know your costs.
