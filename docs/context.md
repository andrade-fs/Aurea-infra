# Contexto de Infraestructura - Aurea

## Overview

Este documento contiene la lógica específica de infraestructura para **Aurea-infra**. Para la visión general y arquitectura, consulta `PRD.md` en la raíz.

---

## 1. Stack de Infraestructura

### Servicios Docker

| Servicio | Imagen | Propósito | Puerto |
|----------|--------|-----------|--------|
| **signoz-query-service** | signoz/signoz:v0.110.1 | Dashboard de observabilidad | 8080 (interno) |
| **signoz-otel-collector** | signoz/signoz-otel-collector:v0.129.13 | Recolector de trazas | 4317 (gRPC), 4318 (HTTP) |
| **signoz-clickhouse** | clickhouse/clickhouse-server:25.5.6 | Base de datos de trazas | 8123 (HTTP), 9000 (nativo) |
| **signoz-zookeeper-1** | signoz/zookeeper:3.7.1 | Coordinación | 2181, 2888, 3888 |
| **aurea_postgres** | postgres:16-alpine | Base de datos principal | 5432 |
| **aurea_redis** | redis:7-alpine | Caché y sesiones | 6379 |

---

## 2. Docker Compose

### Redes

```yaml
networks:
  signoz-net:
    name: signoz-net
    external: false
```

### Puertos Expuestos (localhost vía SSH)

- PostgreSQL: `127.0.0.1:5432`
- Redis: `127.0.0.1:6379`
- ClickHouse HTTP: `127.0.0.1:8123`
- ClickHouse nativo: `127.0.0.1:9000`
- OTEL Collector gRPC: `127.0.0.1:4317`
- OTEL Collector HTTP: `127.0.0.1:4318`

---

## 3. Configuración de SigNoz

### OTEL Collector

**Archivo:** `otel-collector-config.yaml`

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:

exporters:
  clickhousetraces:
    datasource: tcp://${env:CLICKHOUSE_USER}:${env:CLICKHOUSE_PASSWORD}@${env:CLICKHOUSE_HOST}:${env:CLICKHOUSE_PORT}/?database=signoz_traces
  
  signozclickhousemetrics:
    dsn: tcp://${env:CLICKHOUSE_USER}:${env:CLICKHOUSE_PASSWORD}@${env:CLICKHOUSE_HOST}:${env:CLICKHOUSE_PORT}/?database=signoz_metrics
  
  clickhouselogsexporter:
    dsn: tcp://${env:CLICKHOUSE_USER}:${env:CLICKHOUSE_PASSWORD}@${env:CLICKHOUSE_HOST}:${env:CLICKHOUSE_PORT}/?database=signoz_logs

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [clickhousetraces]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [signozclickhousemetrics]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [clickhouselogsexporter]
```

### Variables de Entorno

```bash
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=aurea_secure_pass_2026
CLICKHOUSE_HOST=clickhouse
CLICKHOUSE_PORT=9000
```

---

## 4. Environment File (.env)

```bash
# PostgreSQL
POSTGRES_DB=aurea
POSTGRES_USER=aurea_user
POSTGRES_PASSWORD=aurea_secure_pass_2026

# Redis
REDIS_PASSWORD=aurea_secure_pass_2026

# ClickHouse
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=aurea_secure_pass_2026

# OTEL
OTEL_SERVICE_NAME=aurea-backend
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318/v1/traces
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.version=1.0.0
```

---

## 5. Integración con Backend

### Configuración del Backend

El backend usa OpenTelemetry para enviar trazas al OTEL Collector:

```python
# app/core/config.py
OTEL_EXPORTER_OTLP_ENDPOINT: str = "http://localhost:4318/v1/traces"

# app/core/bootstrap.py
exporter = OTLPSpanExporter(endpoint=otel_url)
provider.add_span_processor(SimpleSpanProcessor(exporter))
```

### Verificación de Funcionamiento

```bash
# Ver contenedores
docker ps --filter "name=signoz"

# Ver logs del collector
docker logs signoz-otel-collector --tail 10

# Probar endpoint
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"test"}}]}}]}'
```

---

## 6. Acceso a SigNoz UI

Según la configuración, SigNoz está diseñado para accederse vía **Coolify Domains** (no directamente por puerto).

### Para acceso local temporally

Agregar映射 de puerto al docker-compose:

```yaml
signoz:
  image: signoz/signoz:v0.110.1
  ports:
    - "3301:8080"
```

Luego acceder a: `http://localhost:3301`

---

## 7. Troubleshooting

### Error: Connection refused en 4318

```bash
# Verificar que el contenedor está corriendo
docker ps | grep otel

# Reiniciar el collector
docker restart signoz-otel-collector

# Ver logs
docker logs signoz-otel-collector
```

### Error: 404 al exportar trazas

Verificar que el endpoint incluya `/v1/traces`:
```bash
# .env del backend debe tener:
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318/v1/traces
```

---

## 8. Volúmenes

```yaml
volumes:
  postgres_data:
  clickhouse_data:
    name: signoz-clickhouse
  signoz_sqlite:
    name: signoz-sqlite
```

---

## 9. Health Checks

```yaml
# ClickHouse
healthcheck:
  test: ["CMD", "wget", "--spider", "-q", "--user=${CLICKHOUSE_USER}", "--password=${CLICKHOUSE_PASSWORD}", "localhost:8123/ping"]

# Wait for:
# - clickhouse: service_healthy
# - signoz: depends_on clickhouse
```