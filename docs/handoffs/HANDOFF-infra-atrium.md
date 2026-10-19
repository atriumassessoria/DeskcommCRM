# HANDOFF — Infraestrutura própria da Giacomolli Atrium

> Leia este arquivo ANTES de qualquer mudança neste fork. Ele documenta o deploy
> real em produção, que diverge do `docs/SETUP.md` padrão do projeto em alguns
> pontos. Autor: Claude (Sonnet 5), sessão de 2026-10-19, a pedido do Norberto
> Giacomolli (dono).

## O que é isto

Fork próprio (`atriumassessoria/DeskcommCRM`) de `melgarafael/DeskcommCRM`,
rodando em produção em `https://crm.giacomolliatrium.com.br`, na mesma VPS que
já hospeda outro produto da Atrium ("Gio", um ERP agro em Next.js/SQLite,
repo separado). Objetivo: virar o motor de CRM/pipeline comercial da
Giacomolli Atrium — vendas, leads, propostas — com IA e WhatsApp.

## Decisão de arquitetura — por que só UM canal de WhatsApp aqui

O Gio (outro produto) já tem um número oficial da Meta Cloud API atendendo
clientes agro de verdade, em produção, com IA própria. Decisão tomada com o
Norberto: **não duplicar isso aqui**. Este Deskcomm conecta o número
**pessoal** do Norberto via **WAHA (QR code)** — é o canal dele responder
manualmente leads/oportunidades comerciais, não um segundo robô paralelo ao
Gio. Se alguma automação de IA for configurada neste canal, ela deve **exigir
aprovação humana antes de enviar** (o Norberto pediu isso explicitamente) —
confirme se existe esse modo antes de ativar qualquer resposta automática
daqui.

Não configure nem peça credenciais do canal oficial da Meta aqui — não é o
escopo deste deploy.

## Infraestrutura (fatos, não inventar de novo)

- **VPS**: `2.25.162.237` (Hostinger KVM, Ubuntu, 1 vCPU, ~3.8GB RAM, compartilhada
  com Gio/CargoDesk/eParana/n8n). Deploy em `/opt/deskcomm`, systemd+Caddy
  (`/etc/caddy/Caddyfile`) como reverse proxy — **não é o Caddy embutido no
  `docker-compose.prod.yml`** deste repo, que fica desligado via
  `docker-compose.override.yml` local na VPS.
- **Banco**: Supabase CLOUD (não self-hosted), projeto `crmgiacomolliatrium`
  (ref `qatetytbetdmbaikcwlv`, região Virgínia do Norte — não São Paulo, foi
  o que saiu no provisionamento). Postgres 17. Conexão direta por IPv6
  funciona da VPS (`db.qatetytbetdmbaikcwlv.supabase.co:5432`), confirmado.
- **Redis/Upstash**: self-hosted via `srh` (serverless-redis-http) + `redis`
  do próprio `docker-compose.prod.yml` — **nenhuma conta Upstash real**,
  `UPSTASH_REDIS_REST_URL=http://srh:80` aponta pro container interno.
- **WAHA**: self-hosted via `docker-compose.prod.yml`, sem número pareado
  ainda na hora deste handoff — ocioso até o onboarding em
  `/onboarding/connect-whatsapp` ser concluído pelo Norberto.
- **IA**: `OPENAI_API_KEY`/`ANTHROPIC_API_KEY` reaproveitadas do Gio (mesma
  conta/billing da Atrium nos dois produtos).
- **Marca**: `.env` tem `APP_NAME=Giacomolli Atrium`,
  `APP_LOGO_URL=https://app.giacomolliatrium.com.br/logo-atrium-dourada.svg`
  (hospedada no Gio, reaproveitada), `APP_ACCENT_HEX=#b4873a` — usa o
  white-label nativo do produto (`lib/env.ts`), não crie um mecanismo novo.
- Nenhuma conta Meta/WhatsApp Business, nenhuma conta Resend/Sentry
  configurada ainda — todas opcionais, deixadas de fora por enquanto.

## ⚠️ A armadilha que já caímos — schema × versão da imagem

O deploy inicial usou `supabase/baseline.sql` da branch `main` **do fork
desatualizado** que clonamos primeiro (`thalena-lima/deskcomm`, bem atrás do
upstream). O container `:stable` (imagem pré-buildada, tag fixa) esperava um
schema mais novo — resultado: `Could not find the table/column/function` em
cascata, menu vazio, autenticação quebrada.

**Correção aplicada**: identificamos a revisão exata da imagem rodando via
`docker inspect ghcr.io/melgarafael/deskcommcrm:stable --format
'{{json .Config.Labels}}'` → `org.opencontainers.image.revision`, buscamos
esse commit exato no repo **upstream** (`melgarafael/DeskcommCRM`, não um
fork desatualizado) e reaplicamos `baseline.sql` de lá. Schema foi de 127
para 193 tabelas.

**Regra pra qualquer deploy futuro, inclusive deste fork**: a versão do
`supabase/baseline.sql`/migrations aplicada no banco e a versão do CÓDIGO
rodando (seja imagem `:stable` ou build próprio) **têm que ser exatamente a
mesma revisão**. Antes de aplicar schema novo ou trocar a imagem, confirme
que os dois lados vão coincidir.

## ⚠️ Decisão em aberto — como builds deste fork chegam em produção

Hoje o `docker-compose.prod.yml` puxa a imagem pronta
`ghcr.io/melgarafael/deskcommcrm:stable` (do repo **original**, não deste
fork) — não builda a partir do código daqui. Ou seja: **mudanças de código
feitas neste fork não vão pra produção sozinhas**.

Existe `docker-compose.build.yml` (build local a partir do `Dockerfile`) como
opção documentada no próprio `docker-compose.prod.yml`, mas a VPS de produção
só tem **1 vCPU** — buildar um Next.js ali é lento e disputa CPU com o Gio/
CargoDesk que já rodam na mesma máquina.

**Isto precisa ser decidido antes do primeiro deploy de código customizado**:
build na própria VPS (simples, mas lento/arriscado pros outros serviços),
build local (Docker Desktop do Norberto) + push pra um registry que a VPS
puxe, ou CI (GitHub Actions no fork) publicando imagem própria. Não escolha
sozinho sem alinhar com o Norberto ou com a sessão Claude Code da pasta
"Atrium Consultoria" (é quem cuida da infra/VPS).

## Estado no momento deste handoff (2026-10-19)

- Primeiro usuário criado: `norberto@giacomolliatrium.com.br`, já promovido a
  `platform_admins` (bootstrap manual via SQL, `scope='full'`,
  `mfa_required=true` — MFA ainda precisa ser ativado pela tela por ele).
  `/admin/sistema` (ativação de módulos da instalação) ainda não foi
  visitado/concluído.
- Onboarding do tenant (`/onboarding/*`) em andamento: organização
  "Giacomolli Assessoria Empresarial" criada, treino do agente de IA em
  configuração, funil "Atendimento" sendo montado, conexão WhatsApp (WAHA)
  ainda não pareada.
- `platform_admins`/`modulos_instalados` ficam vazios até esses passos
  manuais serem concluídos — se algo "não aparecer no menu", confira essas
  tabelas antes de assumir bug de código.

## Convites úteis deste repo (já existiam, reforçando)

As skills `.claude/skills/DeskcommCRM` e `.claude/skills/sistema-vivo` deste
repo têm a doutrina de código (multi-tenancy/RLS, tripla de migration,
invariantes de arquitetura) — use sempre, não é opcional, conforme a própria
descrição delas já instrui.
