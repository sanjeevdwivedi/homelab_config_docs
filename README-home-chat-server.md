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
# Running the  search service
docker run --network=host -e HUGGINGFACE_HUB_TOKEN=<insert token here> -e QDRANT_HOST=localhost -p 6500:6500 search_service
```

the hugging face token in the above would be:
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 

---------- on the plex server
```bash
# Run the front end server
docker run -p 8000:8000  kokoro_server
```
```bash
# the server runs on the plex-garage.lan:8001
docker run -d -p 8001:8000 kokoro_server
```

then go to http://plex-garage.lan:8001/
