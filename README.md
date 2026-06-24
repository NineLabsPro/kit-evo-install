# Kit Evo

Template de instalacao Docker Compose para subir EvoCRM Community e Evolution GO juntos.

O formato principal e o `compose.yaml`. Ele pode ser usado direto em uma VPS com Docker Compose ou colado em paineis como Dokploy, Coolify e Easypanel.

## Servicos publicados

Configure estes dominios antes do deploy em producao:

| Dominio | Variavel | Servico | Porta interna |
| --- | --- | --- | --- |
| Painel web do EvoCRM | `FRONTEND_URL` | `evo-frontend` | `80` |
| API/Gateway do EvoCRM | `BACKEND_URL` | `evo-gateway` | `3030` |
| API Evolution GO/WhatsApp | `EVOGO_PUBLIC_URL` | `evolution-go` | `4000` |

Exemplo:

```env
FRONTEND_URL=https://crm.seudominio.com
BACKEND_URL=https://api-crm.seudominio.com
EVOGO_PUBLIC_URL=https://whatsapp.seudominio.com
CORS_ORIGINS=https://crm.seudominio.com,https://api-crm.seudominio.com,https://whatsapp.seudominio.com
```

No DNS, crie os registros apontando para o servidor onde o painel esta rodando:

```text
crm.seudominio.com
api-crm.seudominio.com
whatsapp.seudominio.com
```

## Instalacao em VPS

```bash
cp env.example .env
docker compose up -d
```

Antes de subir, altere todos os segredos no `.env`, principalmente:

```text
POSTGRES_PASSWORD
REDIS_PASSWORD
SECRET_KEY_BASE
JWT_SECRET_KEY
DOORKEEPER_JWT_SECRET_KEY
EVOAI_CRM_API_TOKEN
ENCRYPTION_KEY
BOT_RUNTIME_SECRET
EVOGO_GLOBAL_API_KEY
```

Para gerar segredos:

```bash
openssl rand -base64 48
```

## Dokploy

Crie um projeto Docker Compose.

Use o conteudo de `compose.yaml`.

Na aba de variaveis de ambiente, cole o conteudo de `env.example` e altere:

```text
FRONTEND_URL
BACKEND_URL
EVOGO_PUBLIC_URL
CORS_ORIGINS
todos os segredos
```

Configure os dominios no Dokploy assim:

```text
crm.seudominio.com       -> evo-frontend:80
api-crm.seudominio.com   -> evo-gateway:3030
whatsapp.seudominio.com  -> evolution-go:4000
```

Use volumes nomeados. Eles estao definidos no final do `compose.yaml` e sao os mais adequados para backup do banco, Redis e dados persistentes.

## Coolify

Crie um novo recurso usando Docker Compose.

Use o `compose.yaml` como arquivo principal e cadastre as variaveis do `env.example` na UI.

Configure os dominios:

```text
crm.seudominio.com       -> evo-frontend, porta 80
api-crm.seudominio.com   -> evo-gateway, porta 3030
whatsapp.seudominio.com  -> evolution-go, porta 4000
```

Se o painel pedir a porta exposta de cada app, use a porta interna da tabela acima.

## Easypanel

Crie um app baseado em Docker Compose.

Cole o `compose.yaml` e configure as variaveis do `env.example`.

Configure tres dominios:

```text
crm.seudominio.com       -> evo-frontend:80
api-crm.seudominio.com   -> evo-gateway:3030
whatsapp.seudominio.com  -> evolution-go:4000
```

## Observacoes importantes

O Docker Hub deve ser usado para publicar imagens. Este reposititorio deve ser usado como template de instalacao.

Nao publique `.env` real. Publique apenas `env.example`.

Evite usar `latest` em producao se houver tags estaveis disponiveis. Troque `EVOCRM_TAG` e `EVOGO_TAG` para versoes fixas quando possivel.

O primeiro deploy pode demorar, porque os servicos `*_init` preparam banco, seeds e migracoes.
