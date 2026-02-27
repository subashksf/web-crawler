# Local Runtime Stack (W-006)

This project uses Docker Compose for local infrastructure:
- Queue: RabbitMQ
- Database: PostgreSQL
- Cache: Redis

## Prerequisites
- Docker Desktop (or Docker Engine + Compose plugin)

## Start (one command)
```bash
docker compose up -d
```

## Check service health
```bash
docker compose ps
```

Expected: `queue`, `db`, and `cache` should report `healthy`.

If you see `Cannot connect to the Docker daemon`, start Docker Desktop (or Docker Engine) first, then rerun:
```bash
docker compose up -d
docker compose ps
```

## Stop
```bash
docker compose down
```

## Stop and remove local data volumes
```bash
docker compose down -v
```

## Optional: environment overrides
You can override these values in a local `.env` file:
- `RABBITMQ_DEFAULT_USER`
- `RABBITMQ_DEFAULT_PASS`
- `POSTGRES_DB`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
