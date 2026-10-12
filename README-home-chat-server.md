# Homelab AI Voice/RAG Stack — Runbook

Last updated: 2026-10-12

This documents the final container configuration for the self-hosted
voice/RAG stack across two servers, so the whole thing can be rebuilt
from scratch if a server is lost, a disk dies, or a container needs
to be recreated.

## Servers

| Server | Role |
|---|---|
| `sanjeev-debian-llm-vm` | GPU server — all backend services (DB, LLM, STT, TTS, vector DB, RAG search) |
| `plex-garage` | Frontend server — user-facing front end only |

---

## 1. Backend server: `sanjeev-debian-llm-vm`

### Service map

| Container | Image | Port(s) | GPU | Data location | Managed by |
|---|---|---|---|---|---|
| `pg-rdbms-llm` | `postgres` | 5432 | no | bind mount (see below) | systemd |
| `pgadmin` | `dpage/pgadmin4` | 5050 | no | bind mount `~/data/pgadmin` | systemd |
| `ollama` | `ollama/ollama` | 11434 | yes | named volume `ollama` | systemd |
| `qdrant` | `qdrant/qdrant:v1.14.0` **(pinned)** | 6333, 6334 | no | named volume `qdrant_data` | systemd |
| `whisper-stt` | `whisper-asr-webservice-gpu` (local build) | 9000 | yes | bind mount (cache) | systemd |
| `kokoro-tts` | `ghcr.io/remsky/kokoro-fastapi-gpu:v0.2.2` **(pinned)** | 8880 | yes | none (weights baked into image) | systemd |
| `search-service` | `search_service` (local build) | 6500 | no | none — stateless | **docker `--restart` flag, not systemd** |

### Why `search-service` is *not* under systemd

Every other container here is stable infrastructure. `search-service`
is under active development (rebuilt from source regularly). systemd's
`docker start -a <name>` assumes the container itself persists —
rebuilding the image and recreating the container (`stop` → `rm` →
`run`) breaks that assumption and causes the unit to fail/loop.

So `search-service` relies on Docker's own restart policy
(`--restart=unless-stopped`), set directly on every `docker run`. This
survives reboots and crashes exactly like systemd would, but doesn't
fight the dev workflow.

### Critical version pin — Qdrant client/server compatibility

**This caused a real outage — write it down.**

- Qdrant **server**: pinned to `qdrant/qdrant:v1.14.0` (never run
  `qdrant/qdrant` with no tag — it resolves to `latest` and will drift
  forward silently on every container recreation)
- Qdrant **client SDK** (`qdrant-client` in `search_service`'s
  `requirements.txt`): pinned to `==1.14.2`

These two must be bumped **together**, deliberately, when upgrading —
never left unpinned. An unpinned client picked up a newer SDK on
rebuild than the pinned server supported, causing "missing function"
errors against the server.

### Build/source locations

- `search_service` source: `~/code/ouroboros/kokoro_fastapi/search_service`
- `whisper-stt` — runs from a pre-built image `whisper-asr-webservice-gpu`

### Data locations (for backup / recovery)

| What | Where |
|---|---|
| Postgres data | `/var/lib/docker/volumes/b7485c2653fbfbc5a913f00820590bffe64d98261436d0139081f4da96088ee3/_data` (bind-mounted to `/var/lib/postgresql/data`) |
| pgAdmin config/backups | `/home/sanjeev/data/pgadmin` |
| Ollama models | named volume `ollama` → `/var/lib/docker/volumes/ollama/_data` |
| Qdrant vectors | named volume `qdrant_data` |
| Whisper model cache | `/home/sanjeev/code/whisper_asr_webservice/cache` |
| Kokoro TTS | none — weights are inside the pinned image `v0.2.2` |
| search-service | none — stateless, reads from Qdrant only |

Named volumes (`ollama`, `qdrant_data`) and bind mounts survive
`docker rm` of the container — only deleting the volume/folder itself
or the `docker-compose down -v` equivalent would lose this data.

### Recreate-from-scratch commands

```bash
# Postgres
docker run -d --name pg-rdbms-llm \
  -e POSTGRES_PASSWORD=<password> \
  -v /var/lib/docker/volumes/b7485c2653fbfbc5a913f00820590bffe64d98261436d0139081f4da96088ee3/_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres

# pgAdmin
docker run -d --name pgadmin \
  -p 5050:80 \
  -e PGADMIN_DEFAULT_EMAIL=<email> \
  -e PGADMIN_DEFAULT_PASSWORD=<password> \
  -v /home/sanjeev/data/pgadmin:/var/lib/pgadmin \
  dpage/pgadmin4

# Ollama (models survive via the 'ollama' named volume)
docker run -d --gpus=all --name ollama \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama

# Qdrant — PINNED VERSION, do not use :latest
docker run -d --name qdrant \
  -p 6333:6333 -p 6334:6334 \
  -v qdrant_data:/qdrant/storage \
  qdrant/qdrant:v1.14.0

# Whisper STT
docker run -d --gpus all --name whisper-stt \
  -p 9000:9000 \
  -v /home/sanjeev/code/whisper_asr_webservice/cache:/root/.cache/ \
  -e ASR_MODEL=large-v3-turbo -e ASR_ENGINE=whisperx \
  whisper-asr-webservice-gpu

# Kokoro TTS — pinned image tag
docker run -d --gpus all --name kokoro-tts \
  -p 8880:8880 \
  ghcr.io/remsky/kokoro-fastapi-gpu:v0.2.2

# search-service — rebuilt from source regularly; qdrant-client pinned
# to 1.14.2 in requirements.txt to match the server above
cd ~/code/ouroboros/kokoro_fastapi/search_service
docker build -t search_service .
docker stop search-service 2>/dev/null; docker rm search-service 2>/dev/null
docker run -d --name search-service --restart=unless-stopped \
  --network=host \
  -e QDRANT_HOST=localhost -p 6500:6500 \
  search_service
```

> No HuggingFace token is needed for `search-service` — confirmed
> unnecessary and removed from the run command.

After creating any of the systemd-managed containers above for the
first time, disable Docker's own restart handling so systemd is the
sole owner:

```bash
docker update --restart=no pg-rdbms-llm pgadmin ollama qdrant whisper-stt kokoro-tts
```

### systemd units (`/etc/systemd/system/`)

Each systemd-managed container gets a unit of this shape — shown
once as a template, then the six concrete files.

```ini
[Unit]
Description=Docker container <name>
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target
# (dependency line added only for pgadmin, which needs postgres first)

[Service]
Restart=always
RestartSec=5
ExecStart=/usr/bin/docker start -a <name>
ExecStop=/usr/bin/docker stop -t 10 <name>

[Install]
WantedBy=multi-user.target
```

Concrete units created:

- `/etc/systemd/system/docker-pg-rdbms-llm.service`
- `/etc/systemd/system/docker-pgadmin.service`
  (adds `Requires=docker-pg-rdbms-llm.service` / `After=docker-pg-rdbms-llm.service`
  so pgAdmin starts after Postgres)
- `/etc/systemd/system/docker-ollama.service`
- `/etc/systemd/system/docker-qdrant.service`
- `/etc/systemd/system/docker-whisper-stt.service`
- `/etc/systemd/system/docker-kokoro-tts.service`

`search-service` intentionally has **no** systemd unit (see above).

Enable/start all six:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now docker-pg-rdbms-llm.service
sudo systemctl enable --now docker-pgadmin.service
sudo systemctl enable --now docker-ollama.service
sudo systemctl enable --now docker-qdrant.service
sudo systemctl enable --now docker-whisper-stt.service
sudo systemctl enable --now docker-kokoro-tts.service
```

### Health check after boot/recovery

```bash
sudo systemctl status docker-pg-rdbms-llm docker-pgadmin docker-ollama \
  docker-qdrant docker-whisper-stt docker-kokoro-tts --no-pager
docker ps
docker exec ollama ollama list        # confirm models survived
curl http://localhost:6333            # confirm Qdrant version == 1.14.0
```

---

## 2. Frontend server: `plex-garage`

### Service map

| Container | Image | Port | Managed by |
|---|---|---|---|
| `kokoro-frontend` | `kokoro-server` (local build) | 8001 → 8000 | docker `--restart` flag (dev container, no systemd) |

No persistent data — stateless front end.

### Build location

Dockerfile: `/home/sanjeev/docker_servers/ouroboros`

### Dev rebuild loop (repeatable, no lookups needed)

```bash
cd /home/sanjeev/docker_servers/ouroboros
docker build -t kokoro-server .
docker stop kokoro-frontend && docker rm kokoro-frontend
docker run -d --name kokoro-frontend --restart=unless-stopped \
  -p 8001:8000 \
  kokoro-server
```

Image is always tagged `kokoro-server`; container is always named
`kokoro-frontend`. The restart flag survives reboots/crashes without
systemd — same rationale as `search-service` above.

---

## 3. Quick recovery checklist

1. **GPU server reboot/crash:** nothing to do — systemd auto-starts
   the six stable services in order; `search-service` restarts on its
   own via its Docker restart policy.
2. **GPU server rebuilt from scratch (new disk/OS):**
   - Recreate the `ollama` and `qdrant_data` named volumes' underlying
     data from backup if lost (if not lost, `docker run` with the same
     volume names reattaches automatically).
   - Re-run all `docker run` commands above in order.
   - Re-create the six systemd units and `enable --now` each.
   - Rebuild `search-service` from its source checkout.
3. **Qdrant acting up / "missing function" errors:** check version
   pinning first —
   ```bash
   curl http://localhost:6333                     # server version
   docker exec search-service pip show qdrant-client   # client version
   ```
   Server should read `1.14.0`, client should read `1.14.2`. If either
   drifted, pin it back (server: recreate container with
   `qdrant/qdrant:v1.14.0`; client: fix `requirements.txt` and rebuild).
4. **Frontend server crash/reboot:** Docker's own restart policy brings
   `kokoro-frontend` back automatically — no systemd involved.

---

## 4. Not yet done

- Centralized monitoring/alerting (Uptime Kuma) and Home Assistant
  remote-restart integration — planned next, not yet implemented.



### this stuff below is older and for manual use only. The above is the latest using systemd and docker auto start.


Process to start the voice chat server.

For docker compose:

```bash
# Backups are stored in ~/data/pgadmin.
docker run --name pgadmin -p 5050:80 -e PGADMIN_DEFAULT_EMAIL=user@domain.com -e PGADMIN_DEFAULT_PASSWORD=llmpassword -v /home/sanjeev/data/pgadmin:/var/lib/pgadmin  dpage/pgadmin4
```

```bash
# Postgres command
docker run \
  --name pg-rdbms-llm \
  -e POSTGRES_PASSWORD=randompassword \
  -v /var/lib/docker/volumes/b7485c2653fbfbc5a913f00820590bffe64d98261436d0139081f4da96088ee3/_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  -d postgres
```

```bash
# Running ollama
docker run -d --gpus=all -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

```bash
# running the speech container:
docker run  --gpus all -p 9000:9000  -v /home/sanjeev/code/whisper_asr_webservice/cache:/root/.cache/   -e ASR_MODEL=large-v3-turbo   -e ASR_ENGINE=whisperx whisper-asr-webservice-gpu
```

```bash
# running the kokoro container
docker run --gpus all -p 8880:8880 ghcr.io/remsky/kokoro-fastapi-gpu:v0.2.2
```

```bash
# Run Qdrant with the right volumes
docker run -p 6333:6333 -p 6334:6334 \
  -v qdrant_data:/qdrant/storage \
  qdrant/qdrant
```

```bash
This is the new way to build search_service
docker build -t search_service .
docker stop search-service && docker rm search-service
docker run -d --name search-service --restart=unless-stopped \
  --network=host -e QDRANT_HOST=localhost -p 6500:6500 \
  search_service


# This is obsolete - Running the  search service
docker run --network=host -e QDRANT_HOST=localhost -p 6500:6500 search_service
```

obsolete. the hugging face token is not needed at all. the hugging face token in the above would be:
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 

-e HUGGINGFACE_HUB_TOKEN=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 

---------- on the plex server
```bash
docker build -t kokoro-server .
docker stop kokoro-frontend && docker rm kokoro-frontend
docker run -d --name kokoro-frontend --restart=unless-stopped \
  -p 8001:8000 \
  kokoro-server
```

then go to http://plex-garage.lan:8001/
