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
WS_URL=wss://api-crm.seudominio.com
EVOGO_PUBLIC_URL=https://whatsapp.seudominio.com
CORS_ORIGINS=https://crm.seudominio.com,https://api-crm.seudominio.com,https://whatsapp.seudominio.com
```

No DNS, crie os registros apontando para o servidor onde o painel esta rodando:

```text
crm.seudominio.com
api-crm.seudominio.com
whatsapp.seudominio.com
```

Se voce usar `whatsapp.meudominio.com`, ele deve apontar para o servico `evolution-go`, porta interna `4000`.

Resumo dos tres:

```text
crm.meudominio.com       -> evo-frontend:80
api-crm.meudominio.com   -> evo-gateway:3030
whatsapp.meudominio.com  -> evolution-go:4000
```

`WS_URL` deve usar o mesmo dominio do `BACKEND_URL`, mas com protocolo `wss://`.

Exemplo:

```env
BACKEND_URL=https://api-crm.meudominio.com
WS_URL=wss://api-crm.meudominio.com
```

Nao coloque `/cable` no final. O frontend ja adiciona esse caminho.

## Webhook do Evolution GO

`EVOGO_WEBHOOK_URL` e a URL que o Evolution GO chama quando recebe eventos do WhatsApp.

Nao deixe como padrao apenas a URL do EvoCRM, porque normalmente webhook precisa de um caminho especifico, nao so o dominio.

Use somente quando voce souber o endpoint receptor do EvoCRM. Exemplo de formato:

```env
EVOGO_WEBHOOK_URL=https://api-crm.seudominio.com/caminho-do-webhook
```

Se o endpoint correto ainda nao estiver confirmado, deixe vazio:

```env
EVOGO_WEBHOOK_URL=
```

Depois da primeira subida, se o EvoCRM tiver uma tela de integracao/canal WhatsApp que gera ou informa a URL de webhook, use essa URL completa aqui.

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

Para gerar a maioria dos segredos:

```bash
openssl rand -base64 48
```

A `ENCRYPTION_KEY` e uma excecao. O EvoCRM usa Fernet, que exige uma chave que
decodifique para exatamente 32 bytes em base64. Gere essa chave especifica com:

```bash
openssl rand -base64 32 | tr '+/' '-_'
```

Nao use `openssl rand -base64 48` para a `ENCRYPTION_KEY`, porque o tamanho fica
errado e o backend retorna erro 500 ao salvar configuracoes criptografadas.

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

## Troubleshooting

### Area de agentes retorna erro de CORS ou `Network Error`

Se o frontend estiver em um dominio diferente do exemplo, use o dominio exato
em `FRONTEND_URL` e `CORS_ORIGINS`.

Exemplo para painel em `app-crm.ninelabs.com.br`:

```env
FRONTEND_URL=https://app-crm.ninelabs.com.br
BACKEND_URL=https://api-crm.ninelabs.com.br
WS_URL=wss://api-crm.ninelabs.com.br
CORS_ORIGINS=https://app-crm.ninelabs.com.br,https://api-crm.ninelabs.com.br,https://whatsapp.ninelabs.com.br
```

Depois de alterar as variaveis no painel de deploy, recrie os servicos que
leem essa configuracao:

```bash
docker compose up -d --force-recreate evo-gateway evo-auth evo-crm evo-crm-sidekiq
```

Se o navegador mostrar CORS, mas o preflight retornar `502`, o problema nao e
so CORS. Confira se o dominio da API esta apontando para `evo-gateway` na porta
interna `3030`, e nao direto para `evo-crm` na porta `3000`.

Para a area de agentes, confira tambem se os servicos abaixo estao saudaveis,
porque `/api/v1/agents` depende do gateway e dos servicos de agentes:

```bash
docker compose ps evo-gateway evo-core evo-processor evo-bot-runtime
docker compose logs --tail=200 evo-gateway evo-core evo-processor evo-bot-runtime
```

Teste esperado:

```bash
curl -i -X OPTIONS 'https://api-crm.ninelabs.com.br/api/v1/agents?page=1&pageSize=24' \
  -H 'Origin: https://app-crm.ninelabs.com.br' \
  -H 'Access-Control-Request-Method: GET' \
  -H 'Access-Control-Request-Headers: authorization,content-type'
```

A resposta deve ter status 2xx/3xx e incluir
`Access-Control-Allow-Origin: https://app-crm.ninelabs.com.br`.

### `Testar agente` retorna `EvoAuth: Network error`

Esse erro vem do `evo-processor` ao tentar validar autenticacao no servico
`evo-auth`. Ele normalmente indica que o auth nao esta saudavel, nao esta na
mesma rede Docker, ou o processor subiu sem a URL interna correta do auth.

Recrie os servicos envolvidos para aplicar as URLs internas e a ordem de
subida:

```bash
docker compose up -d --force-recreate evo-auth evo-processor evo-bot-runtime evo-gateway
```

Depois valide a comunicacao interna a partir do processor:

```bash
docker compose exec evo-processor sh -lc 'python - <<PY
import socket
socket.create_connection(("evo-auth", 3001), timeout=5).close()
print("evo-auth:3001 reachable")
PY'
```

Se esse teste falhar, confira primeiro o estado e os logs do auth:

```bash
docker compose ps evo-auth evo-processor
docker compose logs --tail=200 evo-auth evo-processor
```

### Agente nao responde e `evo-bot-runtime` mostra `pipeline.auth.unauthorized`

Esse log indica que o `evo-bot-runtime` recebeu uma chamada em `/events`, mas
rejeitou a autenticacao interna. Confira se `BOT_RUNTIME_SECRET` esta definido
no ambiente de deploy e se o mesmo valor esta sendo passado para `evo-crm`,
`evo-crm-sidekiq`, `evo_crm_init` e `evo-bot-runtime`.

Depois de alterar a variavel ou atualizar o compose, recrie os servicos que
participam do fluxo do agente:

```bash
docker compose up -d --force-recreate evo_crm_init evo-crm evo-crm-sidekiq evo-bot-runtime
```

### Cloudflare beacon bloqueado pela CSP

O erro para `https://static.cloudflareinsights.com/beacon.min.js` acontece
porque a CSP que vem embutida na imagem do `evo-frontend` (definida no nginx em
`/etc/nginx/conf.d/default.conf`) nao libera o dominio do Cloudflare Web
Analytics na diretiva `script-src`.

Esse erro do beacon nao impede o carregamento dos agentes. Ele so indica que a
CSP bloqueou o script de analytics da Cloudflare.

Esse template ja corrige isso de forma nativa. O comando do servico
`evo-frontend` ajusta a CSP do nginx antes de subir, adicionando os dominios do
Cloudflare e do Google Analytics ao `script-src`:

```text
https://www.google-analytics.com
https://static.cloudflareinsights.com
```

O ajuste e idempotente: so e aplicado quando os dominios ainda nao estao
presentes na CSP. Para aplicar a correcao em uma instalacao existente, recrie o
frontend:

```bash
docker compose up -d --force-recreate evo-frontend
```

Se preferir nao expor analytics, a alternativa continua sendo desativar o
Cloudflare Web Analytics para esse dominio.

### Salvar config (ex.: API key da OpenAI) retorna 500 com `Fernet::Secret::InvalidSecret`

Se ao salvar uma configuracao em `POST /api/v1/admin/app_configs/...` o backend
retornar 500 e o log do `evo-crm` mostrar:

```text
Fernet::Secret::InvalidSecret - Secret must be 32 bytes, instead got 33
app/models/installation_config.rb:...:in 'encrypt_sensitive_value'
```

a `ENCRYPTION_KEY` esta com tamanho invalido. O Fernet exige uma chave que
decodifique para exatamente 32 bytes em base64. Quem gerou a chave com
`openssl rand -base64 48` (ou similar) cai nesse erro.

Gere uma chave valida:

```bash
openssl rand -base64 32 | tr '+/' '-_'
```

Atualize `ENCRYPTION_KEY` no ambiente de deploy e recrie os servicos que leem
essa variavel:

```bash
docker compose up -d --force-recreate \
  evo-auth evo-crm evo-crm-sidekiq evo-core evo-processor evo-bot-runtime
```

Troque a chave antes de cadastrar dados sensiveis. Se ja existirem valores
criptografados com outra `ENCRYPTION_KEY`, eles deixarao de ser descriptografados
apos a troca.

### `GET /api/v1/admin/app_configs/...` ou `/api/v1/dashboard_apps` retorna 500

Esse erro normalmente indica que o EvoCRM subiu, mas as migracoes ou seeds do
servico `evo_crm_init` nao prepararam todos os registros de configuracao do
dashboard.

Primeiro rode novamente o job de inicializacao do CRM e recrie os processos que
dependem dele:

```bash
docker compose up -d --force-recreate evo_crm_init evo-crm evo-crm-sidekiq
```

Depois confira o log do job:

```bash
docker compose logs --tail=200 evo_crm_init
```

Se aparecer erro de banco, migration ou seed, corrija esse erro antes de
reiniciar o frontend. Nao use `docker compose down -v` em producao com dados
reais, porque isso apaga os volumes.

### `relation "installation_configs" does not exist`

Esse erro indica que o banco do EvoCRM foi iniciado sem todas as tabelas.

Primeiro tente recriar os jobs de inicializacao:

```bash
docker compose up -d --force-recreate evo_auth_init evo_crm_init evo-crm evo-crm-sidekiq
```

Se for uma instalacao nova e sem dados importantes, o caminho mais limpo e recriar os volumes:

```bash
docker compose down -v
docker compose up -d
```

Nao use `down -v` em instalacao com dados reais, porque isso apaga banco, Redis e dados persistentes.
