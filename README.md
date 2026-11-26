# fiapx-infra

Plataforma do **FIAP X — Sistema de Processamento de Vídeos**: as dependências de runtime, o cluster
local, a observabilidade e os scripts de criação de recursos.

Este repositório não contém código de aplicação. Os serviços vivem em
[`fiapx-services`](https://github.com/CleytonOngaratto/fiapx-services) e o front em
[`fiapx-web`](https://github.com/CleytonOngaratto/fiapx-web) — a divisão é por *toolchain e ciclo de
vida*: aqui tudo é **aplicado**, não construído.

## Stack de desenvolvimento

`dev/docker-compose.yaml` sobe as dependências que os serviços esperam:

| Serviço | Porta | Para que serve | Limite | Medido ocioso |
|---|---|---|---|---|
| **PostgreSQL** 17 | `5432` | estado autoritativo: usuários, vídeos, outbox, inbox | 256 MB | 29 MB |
| **RabbitMQ** 4 + management | `5672` · UI `15672` | fila de processamento, retry e DLQ | 512 MB | 98 MB |
| **Redis** 7 | `6379` | progresso do processamento (efêmero, *best-effort*) | 96 MB | 3 MB |
| **S3Mock** | `9090` | object storage compatível com S3: vídeos e zips | 384 MB | 186 MB |
| **Mailpit** | SMTP `1025` · UI `8025` | caixa de entrada local para a notificação de falha | 64 MB | 7 MB |

### Subir

```bash
cd dev
cp .env.example .env     # os padrões servem para desenvolvimento
docker compose up -d
docker compose ps
```

O serviço `s3-init` é one-shot: espera o S3 responder, cria os buckets `fiapx-videos` e `fiapx-zips`
pelo **AWS CLI** e sai com código 0. São os mesmos comandos que rodariam contra o S3 de verdade.

### Derrubar

```bash
docker compose down            # mantém os volumes
docker compose down -v         # apaga também os dados
```

## Por que S3Mock, e não MinIO

O object storage local mudou duas vezes em 2026-11-26, as duas por evento externo e as duas medidas:

1. **MinIO saiu.** As imagens da comunidade deixaram de ser baixáveis anonimamente — `minio/minio` e
   `minio/mc` devolvem `denied: requested access to the resource is denied` no Docker Hub, o quay.io
   exige autenticação na API de tags, e `bitnami/minio` não existe mais.
2. **LocalStack saiu.** A tag `:latest` (2026.11.0) virou a edição licenciada: sai com **código 55** e
   `License activation failed! No credentials were found` sem um `LOCALSTACK_AUTH_TOKEN`.

Avaliados como disponíveis: S3Mock, SeaweedFS, Garage, Zenko CloudServer. **Garage** era o preferido —
implementação S3 real, leve, com `PutBucketCors` documentado — mas a imagem oficial é *from scratch*,
**sem shell**, o que transforma a inicialização (layout, chave, bucket) numa sequência de `exec` com
sintaxe a descobrir. Ficou **avaliado e diferido**: é a troca natural se o orçamento de memória do
cluster apertar, e trocar custa um endpoint, porque o adapter fala S3 puro.

## Duas configurações que não são detalhe

### CORS: o dev é permissivo, o S3 real não é

O upload do vídeo é um `PUT` assinado feito **pelo navegador, direto no object storage** — os bytes
nunca passam pela JVM. Medido no S3Mock em 2026-11-26:

| Requisição | Resultado |
|---|---|
| `OPTIONS` com `Origin: http://localhost:5173` | **200** · `Access-Control-Allow-Origin: http://localhost:5173` · `Allow-Methods: PUT` |
| `PUT` com `Origin` | **200** · **ETag devolvido** · `Allow-Origin: *` · `Expose-Headers: *` |

⚠️ **E aqui está a lacuna de fidelidade, que é um risco real:** o S3Mock **não implementa
`put-bucket-cors`** (devolve `BucketAlreadyOwnedByYou`, e `get-bucket-cors` volta vazio) e libera
qualquer origem por padrão. **O S3 de verdade nega** sem uma regra de CORS no bucket. Ou seja: o
ambiente local **não é capaz de pegar** um CORS mal configurado. Na AWS a regra é explícita, via
`aws_s3_bucket_cors_configuration` no Terraform, com `ExposeHeaders: ["ETag"]` — sem o ETag exposto o
navegador não consegue confirmar o upload.

### RabbitMQ: watermark absoluto, e em arquivo

A imagem 4.x **recusa subir** com `RABBITMQ_VM_MEMORY_HIGH_WATERMARK` no ambiente: sai com
`error: deprecated environment variables detected`. O valor vai em `dev/rabbitmq/rabbitmq.conf`.

É absoluto de propósito: o padrão é *0,4 × o limite do container*, o que com 512 MB poria a marca em
~205 MB e ligaria *flow control* nos publishers no meio da demo. Conferir que pegou:

```bash
docker compose exec rabbitmq rabbitmq-diagnostics status | grep -i watermark
# "setting" tem de ser IGUAL a "computed to" — se o computed for ~0,4 do limite, ficou relativo
```

## Memória

A stack inteira, ociosa, mede **~323 MB** — contra ~1,3 GB de teto somado. Teto é para carga, não para
repouso. Em máquinas apertadas o gargalo medido **não** é o Docker: é o que roda no Windows ao lado
dele. Antes de subir o cluster local junto, feche o IDE e o navegador.

## Documentação da arquitetura

A documentação de arquitetura, os ADRs e o mapa de requisitos vivem em
[`fiapx-services`](https://github.com/CleytonOngaratto/fiapx-services), junto do código que eles descrevem.
