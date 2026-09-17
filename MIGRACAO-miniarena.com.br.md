# Mini Arena → miniarena.com.br  
## Documento único de migração (preparatório)

**Tipo:** material de referência para outra ferramenta / operador fazer o cutover.  
**Este documento não executa a migração.** Não altera o motor financeiro.  
**Data:** 2026-09-16  
**Domínio-alvo:** `https://miniarena.com.br`  
**Produto:** Mini Arena — VTT (D&D 5.5) + Mini App Telegram + loja (Mercado Pago) + coleção de dados 3D  
**Idioma do produto:** pt-BR  
**Origem:** código atual no sandbox Grok Build (pacote npm `miniarena`)

Trate **BLOQUEADORES**, **NÃO TOCAR** e o **checklist** como contrato. Não invente URL, SKU, preço nem identidade.

---

## Sumário

1. [O que você está migrando](#1-o-que-você-está-migrando)
2. [O que NÃO é este app](#2-o-que-não-é-este-app)
3. [Contrato: não tocar](#3-contrato-não-tocar)
4. [Mapa do produto (estado 2026-09-16)](#4-mapa-do-produto-estado-2026-09-16)
5. [Arquitetura as-is](#5-arquitetura-as-is)
6. [Por que a Hostinger compartilhada não serve](#6-por-que-a-hostinger-compartilhada-não-serve)
7. [Duas topologias possíveis](#7-duas-topologias-possíveis)
8. [Bloqueadores no código (fazer ANTES do DNS)](#8-bloqueadores-no-código-fazer-antes-do-dns)
9. [Variáveis de ambiente](#9-variáveis-de-ambiente)
10. [Banco de dados](#10-banco-de-dados)
11. [Como fazer — passo a passo](#11-como-fazer--passo-a-passo)
12. [Auth (Google nativo, sem broker Grok)](#12-auth-google-nativo-sem-broker-grok)
13. [Telegram Mini App](#13-telegram-mini-app)
14. [Mercado Pago](#14-mercado-pago)
15. [Assets, dados 3D e limites](#15-assets-dados-3d-e-limites)
16. [Headers, cookies, CSP](#16-headers-cookies-csp)
17. [Checklist de cutover](#17-checklist-de-cutover)
18. [Testes depois de no ar](#18-testes-depois-de-no-ar)
19. [Rollback](#19-rollback)
20. [Inventário de arquivos úteis](#20-inventário-de-arquivos-úteis)

---

## 1. O que você está migrando

O Mini Arena hoje corre em dois mundos:

| Superfície | URL atual | Papel |
|---|---|---|
| Mini App / mesa | `https://miniarena.grok.me` | UI, Telegram, cookies de sessão |
| Backend financeiro | `https://miniarena.vercel.app` | API, webhook Mercado Pago |
| Auth Google | `https://auth.grok.me` | broker Grok (não reutilizar no .com.br) |

**Destino:** um origin canônico

```
https://miniarena.com.br
```

servindo **app + API + webhooks**. `www.miniarena.com.br` deve responder **301** para o apex (sem `www`). Cookies `__Host-*` **não** funcionam em dois hosts ao mesmo tempo.

Resultado esperado para o jogador:

- Abrir `https://miniarena.com.br` no desktop/mobile → Hub Mini Arena
- Abrir o Mini App no Telegram → o mesmo origin
- Checkout Mercado Pago, webhook Telegram e login (quando houver) no mesmo domínio

---

## 2. O que NÃO é este app

- Não é WordPress, PHP, Laravel, HTML estático, nem SPA que se copia para `public_html`.
- Não roda em **Hospedagem de Sites** Hostinger (hPanel + Apache + PHP + MySQL).
- Não usa MySQL / MariaDB. O schema é **Postgres** (quoted camelCase, `timestamptz`, `on conflict`).
- Não é um `vite build` que gera só `dist/index.html`. O build gera **SSR Nitro** (hoje preset `vercel`).
- PGLite (Postgres em WASM) é **só preview/dev**. Produção sem `DATABASE_URL` = dados somem a cada restart.

Se alguém disser “sobe na pasta `public_html`”, está no caminho errado.

---

## 3. Contrato: não tocar

Não alterar lógica, preços, SKUs nem tabelas destes módulos. Login / idade / termos **não** escrevem no ledger.

| Arquivo | Papel |
|---|---|
| `src/lib/shop-finance.ts` | Ledger, crédito, estorno, períodos |
| `src/lib/shop-finance-access.ts` | Papéis financeiro / reconciliação / user |
| `src/lib/shop-pay.ts` | Checkout Pro / preapproval / notify URL |
| `src/lib/shop-reconcile.ts` | Relatório de divergência (**não** auto-heals) |
| `src/lib/shop-skus.ts` | SKUs cobrados (preço só no servidor) |
| `src/lib/shop-catalog.ts` | Catálogo servidor |
| `src/lib/shop-admin.ts` | Admin da loja |
| `src/routes/api/mercadopago.ts` | Webhook MP |
| `src/lib/auth/server.ts` | Better Auth (fork Hostinger: Google nativo, commit isolado) |
| `src/lib/auth/client.ts` | Cliente Better Auth |

**Exceção controlada:** no fork de deploy, `server.ts` / `client.ts` / `providers.ts` precisam de Google OAuth **nativo** no domínio novo. É um commit isolado. Não misturar com o ledger.

### Três identidades (não fundir neste cutover)

| ID | Onde | Para quê |
|---|---|---|
| `user.id` Better Auth | `"user"`, `account_onboarding` | login, idade, termos |
| `player_id` (6 dígitos) | `shop_wallets`, cookie `ma_store` | ouro / diamantes / PRO / PLUS |
| Telegram `user.id` | bot / chat | Mini App, webhook |

A conta Better Auth **não** vira `player_id` sozinha. Unificar isso é projeto à parte.

---

## 4. Mapa do produto (estado 2026-09-16)

Hub: `src/components/hub/Hub.tsx` (a landing só reexporta o Hub).

Menu inferior:

| Item | O que faz |
|---|---|
| Jogar | criar/entrar em sala (código 5 caracteres), mapas, fichas, NPCs. Sistema: `dnd-55-fanmade`. `MAX_ROOMS = 5` |
| Chat | amigos, DMs locais, perfil do amigo |
| Loja | ouro, diamantes, planos PRO/PLUS, itens de mesa |
| Coleção | hub com 4 cards: **COMUNS**, **EXÓTICOS**, **TEMPORADAS**, **PROMOCIONAIS** |
| Tutorial | textos mestre / jogador / mesa |
| Opções | sons, temas, **Perfil**, sair da mesa |

### 4.1 Entrada (login / idade / termos)

- Rota `/` **não exige login**. Telegram Mini App entra direto no Hub (`RequireReady`: se não há `user`, renderiza o Hub).
- `/login` redireciona para `/`. A tela `LoginScreen.tsx` existe no código mas a rota não a mostra.
- Se no futuro houver sessão Better Auth: **Conta → Idade (18+) → Termos/Privacidade → Hub**.
- Idade: só mês + ano. Versão `age-gate-v1`. Termos `terms-v1`, privacidade `privacy-v1`.
- **Não recolocar login obrigatório neste cutover** sem spec. O Telegram depende da entrada direta.

**Sair** em Opções sai da **mesa** (navega para `/`). Não chama `signOut()` da conta.

### 4.2 Mesa (`/table`)

- Grid 2D, tokens, HP, iniciativa, fichas 5.5.
- Dados 3D: `@3d-dice/dice-box` + `public/assets/dice-box/ammo/ammo.wasm`.
- Sync da sala: **HTTP poll** `pullRoom` / `pushRoom` → tabela `arena_rooms`. Sem WebSocket obrigatório. Não planejar TURN neste cutover.
- Mapas built-in: `/maps/patio.jpg`, `caverna.jpg`, `clareira.jpg`. Upload custom até 4 MB.
- nginx: `client_max_body_size 20m;`

### 4.3 Perfil (Opções → Perfil)

- Avatar **quadrado** 80×80 visual. Só biblioteca (5): Guerreiro, Maga, Anão, Ranger, Ladino. Sem upload.
- Molduras: `src/lib/frames.ts` + `/media/frames/*.webp`. Livres: sem moldura, bronze. Cadeado: prata (nv 20), ouro (nv 50), primeira aventura, rubi, olho do dragão.
- Cartões de visita 16:9: `src/lib/calling-cards.ts` + `/media/cards/*.webp` (nunca PNG/GIF originais no bundle).
- Nome único (`src/lib/player-name.ts`), MEU ID 6 dígitos, coroa → Loja planos.

### 4.4 Coleção (atualizado)

Hub da coleção: 4 cards verticais.

**COMUNS** — 12 hexágonos 001–012. 12 cores × 6 tipos (d4…d20) = 72 dados. Runtime: GLB partilhado + `manifest.json` em `/assets/dice-runtime/comum/`. Miniaturas WebP. Física da mesa = Dice Box (colliders). Não recriar malha a partir de `.diceforge` no browser.

**EXÓTICOS** — slot 001 **Olho de Dragão** (d20 teste authored).  
- Miniatura: `/assets/dice-photos/exotico/olho-de-dragao/d20.webp`  
- Inspeção 3D: o `d20.glb` **tal como exportado** (`SceneLoader.ImportMeshAsync`). O Mini Arena **não reconstrói** o dado.  
- Pacote completo esperado no ZIP de designer:

```
EXOTICO_olho_de_dragao.zip
├── dice.glb              ← inspeção (fonte da verdade visual)
├── d20.webp              ← thumbnail
├── textures/skin.png     ← PNG do olho (não caminho Windows)
├── source/*.diceforge    ← projeto original (arquivo, não runtime)
├── dice.json
└── manifest.json
```

Contrato visual authored: o olho é uma esfera **interna** (raio ~7 mm); números e casca (~11 mm) ficam à frente. Overlay com `clearDepthBefore` cola o olho por cima dos números — **não** usar isso no GLB de produção. Cor do olho através da resina exige `baseColor` da resina com G/B ≠ 0.

**TEMPORADAS / PROMOCIONAIS** — placeholders.

Persistência local da coleção: `mini-arena-dice-col`. Fragmentos de desmonte: 100 por dado.

### 4.5 Loja (dois mundos — não misturar)

1. **Itens de mesa** (mapas, dados, tokens) — gastam ouro **no cliente** (zustand, seed 120). Não é o ledger.
2. **Moeda cobrada** — Mercado Pago → webhook → `shop_wallets`.

Planos (`src/lib/account.ts`): free / pro R$ 12 / plus R$ 28.

Ouro (BRL): Punhado 500/4,90 · Saco 1500/12,90 · Bolsa 4000/24,90 · Arcas 10000/49,90 · Tesouro 25000/99,90.  
Diamantes: Fagulha 25/7,90 · Punhado 80/19,90 · Guilda 200/39,90 · Reino 500/79,90 · Imperial 1200/149,90.

### 4.6 Chat

Zustand `mini-arena-chat-v1`. Amigo seed **Adalberto**. DMs locais. Chat de voz **não** está implementado — não planejar servidor de voz neste cutover.

---

## 5. Arquitetura as-is

| Camada | Tecnologia | Nota |
|---|---|---|
| Runtime | Node.js **22.x** | `package.json` `engines` |
| UI | React 19 | |
| SSR / rotas | TanStack Start + Router | `src/router.tsx`, `src/routes/` |
| Bundler | Vite 8 + Nitro `3.0.260610-beta` | **preset `vercel` hoje** |
| CSS | Tailwind v4 | `src/styles.css` — identidade Imperial Void (hue ~294, dourado) |
| Auth | Better Auth `~1.6.30` | `/api/auth/*` |
| DB produção | `pg` | `DATABASE_URL` |
| DB preview | PGLite | sem `DATABASE_URL` |
| Pagamentos | Mercado Pago Checkout Pro + preapproval | `MERCADOPAGO_ENV=test` |
| Mini App | `telegram-web-app.js` | |
| Dados 3D | `@3d-dice/dice-box` + ammo.wasm + Babylon (inspeção authored) | `optimizeDeps.exclude` + `ssr.external` |
| Estado cliente | zustand persist | **não** é fonte da loja cobrada |

### Rotas HTTP

| Rota | Comportamento |
|---|---|
| `/` | Hub (sem login) |
| `/login` | `Navigate` → `/` |
| `/idade` | gate 18+ |
| `/legal` `/termos` `/privacidade` | termos (públicos) |
| `/table` | mesa |
| `/loja` `/loja/auth/continue` | loja / ticket |
| `/pagamento/$status` | retorno MP |
| `/api/auth/$` | Better Auth GET+POST |
| `/api/mercadopago` | GET health; POST HMAC |
| `/api/telegram` | secret token |
| `/api/wallet` | cookie `ma_store` |
| `/api/store/session` | sessão loja |
| `/api/purchases/$purchaseId/status` | status compra |

Não existe `/api/rtc`. Não criar neste cutover.

### Build atual

```
npm run build
  = scripts/with-app-env.mjs → vite build
  + scripts/fix-ssr-exports.mjs
  + scripts/migrate.mjs
```

`vite.config.ts` usa `nitro({ preset: "vercel", serverDir: "./server" })`.  
Output: `.vercel/output/functions/__server.func/`.  
`scripts/fix-ssr-exports.mjs` é **patch do preset Vercel**. Outro preset (Node no VPS) exige revalidar ou adaptar o patch.

Dev: `npm run dev` em `0.0.0.0:8080`. Não usar `vite preview` como produção.

### URLs hardcoded que QUEBRAM o domínio novo

| Arquivo | Problema |
|---|---|
| `src/lib/telegram.ts` | `PUBLISHED_APP_URL = "https://miniarena.grok.me"`. `miniAppUrl()` se o origin **não** contém `grok.me` **devolve grok.me**. No `.com.br` o Mini App aponta para o sítio antigo. **Bloqueador.** |
| `src/lib/telegram-bot.server.ts` | `MINI_APP_URL` grok.me; bot `Miniarenavtt_bot` |
| `src/lib/shop-pay.ts` | fallback notify = grok.me `/api/mercadopago` |
| `src/lib/auth/preview.ts` | issuer `auth.grok.me` — **proibido** em produção Hostinger |
| `.env.example` | ainda documenta grok.me / vercel.app |

`src/lib/app-urls.ts` já lê `MINIARENA_APP_URL` / `MINIARENA_API_URL` e recusa URL que não seja `https://`. Em produção **setar** as env.

---

## 6. Por que a Hostinger compartilhada não serve

Plano **Hospedagem de Sites** / `public_html`:

1. Sem processo Node persistente para SSR Nitro.
2. MySQL quebra o schema Postgres.
3. Webhooks MP/Telegram precisam de POST no mesmo app Node.
4. WASM + cookies `__Host-` exigem HTTPS na frente do Node.
5. O build não gera um site estático.

**O que a Hostinger pode ser neste projeto:**

- **Registo DNS do domínio** `miniarena.com.br` (sempre).
- **VPS (KVM)** se quiser hospedar o Node na própria Hostinger.
- **Não** hospedar o app no plano compartilhado.

Porte mínimo se for VPS: **2 vCPU / 4 GB RAM**, Ubuntu 22.04 ou 24.04, 40 GB+ disco. Melhor: buildar fora (CI) e copiar o artefato.

---

## 7. Duas topologias possíveis

### A — DNS na Hostinger, app na Vercel (menor risco)

O código **já** tem `vercel.json` e preset Nitro `vercel`.

```
jogador / Telegram
        │
        ▼
miniarena.com.br  (DNS Hostinger → CNAME/A da Vercel)
        │
        ▼
Vercel (Node 22 + Postgres Neon/Supabase)
```

**Quando escolher:** quer o domínio .com.br no ar rápido, sem administrar Linux.  
**O que fazer na Hostinger:** só DNS + SSL (ou SSL na Vercel).  
**O que NÃO fazer:** copiar ficheiros para `public_html`.

### B — Tudo no VPS Hostinger (um origin, você controla o servidor)

```
jogador / Telegram
        │
        ▼
nginx :443  (Let's Encrypt, X-Forwarded-Proto, client_max_body_size 20m)
        │
        ▼
Node 22  (Nitro preset node-server, PORT=3000, systemd)
        │
        ▼
Postgres 15+  (mesmo VPS ou managed: Neon/Supabase/Hostinger VPS)
```

**Quando escolher:** política de “tudo na Hostinger”, ou quer fugir da Vercel.  
**Obrigatório no código:** mudar preset Nitro de `vercel` para `node-server` (ou `node-middleware`) e revalidar `fix-ssr-exports.mjs`. Recusar PGLite se `NODE_ENV=production`.

**Recomendação deste documento:** se o domínio já está na Hostinger e o app já faz build Vercel, use **A** para o primeiro cutover. Use **B** só se houver motivo explícito (compliance, custo, ou recusa de Vercel).

Nas duas topologias o **origin canónico** é `https://miniarena.com.br`.

---

## 8. Bloqueadores no código (fazer ANTES do DNS)

Estes patches são do **fork de deploy**. Não misturar com o ledger.

1. **`miniAppUrl()`** — devolver `miniArenaAppUrl()` ou `window.location.origin`, nunca forçar grok.me.
2. **`telegram-bot.server.ts`** — `MINI_APP_URL` a partir de `MINIARENA_APP_URL`.
3. **`shop-pay.ts` notifyUrl** — só env `MERCADOPAGO_NOTIFICATION_URL` / `MINIARENA_API_URL`. Sem fallback grok.me.
4. **Auth** — Google nativo (`GOOGLE_CLIENT_ID` / `SECRET`). Remover `GROK_AUTH_*` e o fallback `grok_preview` do binário de produção.
5. **`trustedOrigins`** — incluir `https://miniarena.com.br`. Sem isso o POST de login devolve Invalid origin.
6. **`src/lib/db.ts`** — se `NODE_ENV=production` e não há `DATABASE_URL`, **morrer**. Hoje só recusa PGLite quando `VERCEL=1`.
7. **CSP** (`src/lib/http-security.ts`) — tirar `https://auth.grok.me` / `https://grok.com` se o fork não usar Grok; manter `telegram.org`, `accounts.google.com`, `'wasm-unsafe-eval'`.
8. **Preset Nitro** — Topologia A: manter `vercel`. Topologia B: `node-server` + systemd.
9. **Plugin PWA Grok** (`scripts/grok-pwa-plugin.mjs`) — injeta “Created with Grok” e `/__grok/*`. Decisão: remover no fork Hostinger.
10. **`safePublicOrigin`** — passar a aceitar `miniarena.com.br` via env (já entra se `MINIARENA_APP_URL` estiver certo).

Não “aproveitar” o cutover para unificar `player_id` com Better Auth, nem para chat de voz, nem para Steam/Android.

---

## 9. Variáveis de ambiente

Ficheiro no VPS: `/etc/miniarena.env` (`chmod 600`). Na Vercel: Project → Settings → Environment Variables, ambiente Production.

Nunca prefixar segredo com `VITE_` (isso vai para o browser). Sem barra no fim das URLs.

```
NODE_ENV=production
PORT=3000
NITRO_PORT=3000

DATABASE_URL=postgres://USER:PASS@HOST:5432/miniarena?sslmode=require

BETTER_AUTH_URL=https://miniarena.com.br
BETTER_AUTH_SECRET=   # openssl rand -hex 32 — estável; trocar invalida sessões

MINIARENA_APP_URL=https://miniarena.com.br
MINIARENA_API_URL=https://miniarena.com.br
WEB_RETURN_URL=https://miniarena.com.br
TELEGRAM_RETURN_URL=https://miniarena.com.br

# VAZIO até existir landing https real. Não inventar hostname.
MINIARENA_STORE_URL=

MERCADOPAGO_ENV=test
MERCADOPAGO_ACCESS_TOKEN=APP_USR-token-de-TESTE
MERCADOPAGO_WEBHOOK_SECRET=
MERCADOPAGO_NOTIFICATION_URL=https://miniarena.com.br/api/mercadopago

TELEGRAM_BOT_TOKEN=
TELEGRAM_WEBHOOK_SECRET=

# Google nativo (depois do commit isolado)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Admin loja (opcional)
# SHOP_ADMIN_PIN=
# SHOP_ADMIN_TELEGRAM_IDS=
```

**NÃO setar em produção Hostinger:**

- `VITE_AUTH_ENABLED=false`
- `VERCEL=1` (a menos que o host seja mesmo a Vercel)
- `GROK_AUTH_ISSUER` / `GROK_AUTH_CLIENT_ID` / `GROK_AUTH_CLIENT_SECRET`

`scripts/migrate.mjs` **sai 0 e não cria tabelas** se `DATABASE_URL` estiver vazio. Produção sem migrate = app sobe sem schema.

---

## 10. Banco de dados

Postgres **15+**. Não MySQL.

Migrações em `migrations/0001_*.sql` … `0011_account_onboarding.sql`, rastreadas em `_migrations`. Não reescrever ficheiro já aplicado — criar `0012_*.sql`. O glob **não desce** a `migrations/auth/`.

| Ficheiro | Conteúdo |
|---|---|
| `0001_auth.sql` | Better Auth: `"user"`, `"session"`, `"account"`, `"verification"` (camelCase **quoted**) |
| `0002_arena_rooms.sql` | salas da mesa |
| `0003_telegram_chat.sql` | bot / grupos |
| `0004_arena_write_secret.sql` | segredo de escrita |
| `0005_shop_cms.sql` | admin + catálogo |
| `0006_mp_orders.sql` | `shop_orders` |
| `0007_shop_ledger.sql` | purchases, wallets, subscriptions |
| `0008_shop_ledger_refunds.sql` | transações / webhook events |
| `0009_subscription_periods.sql` | períodos |
| `0010_store_sessions.sql` | tickets da loja |
| `0011_account_onboarding.sql` | idade + termos por `user_id` |

Pooler Supabase em modo **transaction** (porta 6543) pode quebrar o migrate multi-statement. Usar ligação **direct/session** (5432) para o migrate; pooler só no runtime se testado.

Opções de Postgres:

- Neon (o código já chama o backend de `"neon"` quando há `DATABASE_URL`)
- Supabase
- Postgres no próprio VPS (`apt install postgresql`)

---

## 11. Como fazer — passo a passo

### Fase 0 — Preparar o domínio na Hostinger (DNS)

Painel Hostinger → Domínios → `miniarena.com.br` → DNS.

**Topologia A (Vercel):**

1. Na Vercel: Add Domain `miniarena.com.br`.
2. Na Hostinger: registo que a Vercel pedir (A ou CNAME). Apex muitas vezes precisa de **A** para os IPs da Vercel; `www` CNAME para `cname.vercel-dns.com`.
3. `www` → 301 para apex (redirect da Vercel ou nginx).
4. Esperar propagação (minutos a algumas horas). `dig miniarena.com.br`.

**Topologia B (VPS):**

1. A `@` → IPv4 do VPS.
2. AAAA se houver IPv6.
3. `www` CNAME para `miniarena.com.br` ou A para o mesmo IP + 301 no nginx.

Não aponte o domínio para a pasta `public_html` do plano compartilhado.

### Fase 1 — Patch do código (fork)

Ordem:

1. Branch `deploy/miniarena-com-br`.
2. Patches da §8 (URLs, auth, fail-closed PGLite, CSP).
3. Topologia B: `nitro({ preset: "node-server" })` e testar `node .output/server/index.mjs`.
4. `npm test` + `npm run check:auth`.
5. **Não** mexer em `shop-finance.ts` / `shop-skus.ts` / webhook MP.

### Fase 2 — Postgres

1. Criar base `miniarena`.
2. Guardar `DATABASE_URL` com `sslmode=require` se o host exigir SSL.
3. Rodar `DATABASE_URL=... npm run db:migrate` **uma vez** contra a base real.
4. Confirmar tabelas: `"user"`, `arena_rooms`, `shop_wallets`, `account_onboarding`.

### Fase 3a — Deploy Vercel (topologia A)

```
vercel link
# env no dashboard (todas as da §9)
vercel --prod
# Add Domain miniarena.com.br
```

Build Command já está em `vercel.json`: `npm run build`.  
Node 22.x.

### Fase 3b — Deploy VPS (topologia B)

No Ubuntu, como root (resumo — adaptar utilizador `miniarena`):

```bash
# Node 22
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs nginx certbot python3-certbot-nginx

# App
adduser --system --group --home /opt/miniarena miniarena
# copiar o repositório ou o artefato de build para /opt/miniarena/app
cd /opt/miniarena/app
npm ci --omit=dev
# env
install -m 600 /path/to/miniarena.env /etc/miniarena.env
```

systemd `/etc/systemd/system/miniarena.service`:

```
[Unit]
Description=Mini Arena
After=network.target postgresql.service

[Service]
Type=simple
User=miniarena
WorkingDirectory=/opt/miniarena/app
EnvironmentFile=/etc/miniarena.env
ExecStart=/usr/bin/node /opt/miniarena/app/.output/server/index.mjs
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

nginx (esqueleto):

```
server {
  listen 80;
  server_name miniarena.com.br www.miniarena.com.br;
  return 301 https://miniarena.com.br$request_uri;
}
server {
  listen 443 ssl http2;
  server_name miniarena.com.br;
  # ssl_certificate preenchido pelo certbot

  client_max_body_size 20m;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
server {
  listen 443 ssl http2;
  server_name www.miniarena.com.br;
  return 301 https://miniarena.com.br$request_uri;
}
```

```
certbot --nginx -d miniarena.com.br -d www.miniarena.com.br
systemctl enable --now miniarena
```

**Obrigatório:** `X-Forwarded-Proto` = `https`. Sem isso cookies `Secure` / HSTS / Better Auth falham atrás do nginx.

### Fase 4 — Integrações (Google, Telegram, MP)

Ver §12–14. Fazer **depois** do HTTPS estável no apex.

### Fase 5 — Cutover Telegram

1. BotFather → Mini App URL = `https://miniarena.com.br`
2. `setWebhook` para `https://miniarena.com.br/api/telegram` com o mesmo `TELEGRAM_WEBHOOK_SECRET`
3. Abrir o Mini App num telemóvel real (não só o preview)

### Fase 6 — Mercado Pago

1. Painel MP → webhook `https://miniarena.com.br/api/mercadopago`
2. Manter `MERCADOPAGO_ENV=test` no primeiro dia
3. Uma compra teste de SKU barato (`g-punhado`)
4. Só então produção

---

## 12. Auth (Google nativo, sem broker Grok)

Hoje o Google na UI chama o broker `auth.grok.me`. Isso **não** funciona no .com.br.

No Google Cloud Console:

1. APIs e serviços → Credenciais → ID do cliente OAuth (Web).
2. Origens JavaScript autorizadas: `https://miniarena.com.br`
3. URIs de redirecionamento:  
   `https://miniarena.com.br/api/auth/callback/google`  
   (confirmar o path exacto do Better Auth no commit isolado; hoje o broker usa `/api/auth/oauth2/callback/grok-google`.)
4. Ecrã de consentimento: app em teste com o e-mail do operador; depois verificação se for público.

No código: plugin Google do Better Auth, **não** `genericOAuth` para Grok. `BETTER_AUTH_URL=https://miniarena.com.br`. Secret estável.

E-mail/senha já está ligado (`emailAndPasswordEnabled = true`) e **não** depende do broker — continua a funcionar se o origin estiver em `trustedOrigins`.

Não há envio de e-mail de verificação/reset neste codebase. A UI “esqueci a senha” não manda e-mail. Não é bloqueador do cutover, mas documente.

Cookie de sessão:

```
__Host-grok-auth.session_token
```

Requisitos `__Host-` (o browser recusa se faltar um): HTTPS, `Path=/`, **sem** `Domain`, `Secure`. O prefixo `grok-auth` no nome pode ficar; renomear invalida sessões.

---

## 13. Telegram Mini App

- Bot actual: `Miniarenavtt_bot` (`src/lib/telegram-bot.server.ts`).
- Token: `TELEGRAM_BOT_TOKEN`. Webhook POST `/api/telegram` exige header `x-telegram-bot-api-secret-token` = `TELEGRAM_WEBHOOK_SECRET`.
- Script no `<head>`: `https://telegram.org/js/telegram-web-app.js`.
- CSP já permite `web.telegram.org`.
- Entrada **sem login** no Hub. Não “corrigir” isso.
- Selects/checkboxes nativos, alvos 44px, `env(safe-area-inset-*)`.

Comando webhook (exemplo, depois do HTTPS):

```
curl -F "url=https://miniarena.com.br/api/telegram" \
     -F "secret_token=SEU_TELEGRAM_WEBHOOK_SECRET" \
     https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook
```

BotFather → Bot settings → Menu button / Mini App → URL `https://miniarena.com.br`.

---

## 14. Mercado Pago

Não inferir ambiente pelo prefixo `APP_USR`. Usar `mpConfiguredEnv()` (`src/lib/shop-mp-env.ts`) e a env `MERCADOPAGO_ENV`.

Webhook: HMAC `x-signature` + `x-request-id`. GET de health **não** vaza token. POST inválido responde `"ok"` ou `"unauthorized"` — não vazar stack.

`MINIARENA_STORE_URL` vazio ⇒ botões +Ouro/+Diamantes mostram loja indisponível. **Não inventar** hostname.

Retorno após Checkout: `/pagamento/processando|sucesso|pendente|falhou?oid=...`

Manter o motor financeiro intacto. Só apontar URLs.

---

## 15. Assets, dados 3D e limites

Servir como estáticos (CDN ou nginx `alias` / o próprio Node):

| Path | Uso |
|---|---|
| `/assets/dice-box/ammo/ammo.wasm` | física Dice Box — **obrigatório** |
| `/assets/dice-box/themes/` | temas clássicos da mesa |
| `/assets/dice-runtime/comum/` | GLB + colliders COMUNS |
| `/assets/dice-photos/exotico/olho-de-dragao/` | webp + **d20.glb authored** (vários MB) |
| `/media/cards/` `/media/frames/` | cache imutável |
| `/avatars/` `/maps/` `/sounds/` | perfil e mesa |

WASM: CSP precisa de `'wasm-unsafe-eval'`. Já está.

Inspeção authored usa Babylon (`DieMini3D.tsx`) e `ImportMeshAsync`. GLBs grandes (~2,7 MB) — não comprimir com um compressor que parta glTF.

nginx: `client_max_body_size 20m` (mapas 4 MB + futuros uploads).

---

## 16. Headers, cookies, CSP

`src/lib/http-security.ts` + `server/middleware/security-headers.ts`.

HSTS só quando o middleware vê HTTPS. nginx **tem** de mandar `X-Forwarded-Proto`.

CSP actual inclui `https://grok.com` e `https://auth.grok.me`. No fork Hostinger, remover se não houver Grok. Manter:

- `script-src`: `'self' 'unsafe-inline' 'wasm-unsafe-eval' https://telegram.org`
- `connect-src`: `'self' https://accounts.google.com` (+ `wss:` se no futuro houver)
- `frame-src` / `form-action`: `accounts.google.com` / `www.mercadopago.com.br`
- `frame-ancestors`: `'self' https://web.telegram.org https://*.telegram.org`

Cookie da loja: `ma_store` (não é `__Host-`). Ticket 2 min, sessão 30 min, `player_id` 6 dígitos.

---

## 17. Checklist de cutover

Fazer **nesta ordem**. Não apontar o DNS antes dos patches 1–6.

- [ ] Branch de deploy; patches §8
- [ ] `npm test` verde
- [ ] Postgres criado; `db:migrate` contra a base real
- [ ] Env §9 preenchidas (sem grok.me)
- [ ] HTTPS no apex (`https://miniarena.com.br` abre)
- [ ] `www` → 301 apex
- [ ] GET `/api/mercadopago` → `{ok:true}`
- [ ] Hub em `/` sem login
- [ ] Mesa: criar sala, rolar dado 3D, ammo.wasm 200
- [ ] Coleção → COMUNS abre hexes; EXÓTICOS 001 carrega GLB
- [ ] Google OAuth nativo (se o login voltar a estar visível)
- [ ] BotFather Mini App URL actualizada
- [ ] Webhook Telegram `getWebhookInfo` aponta para .com.br
- [ ] Webhook MP aponta para .com.br; compra teste em `MERCADOPAGO_ENV=test`
- [ ] Headers: `X-Forwarded-Proto` / cookie Secure
- [ ] grok.me deixa de receber tráfego de produção (ou redirect 301 temporário)

---

## 18. Testes depois de no ar

| Teste | Esperado |
|---|---|
| Desktop Chrome `https://miniarena.com.br` | Hub, identidade dourada/roxa |
| Mobile Chrome | Hub, safe-area, toques 44px |
| Telegram Mini App | Hub directo, sem ecrã de login |
| `/table` + rolar d20 | física + som |
| Coleção → EXÓTICOS → Olho de Dragão | miniatura webp + inspeção GLB (olho **dentro** da casca, não por cima dos números) |
| Compra teste `g-punhado` | webhook HMAC, wallet creditada, **ledger intacto** |
| `www.miniarena.com.br` | 301 apex |
| `http://` | 301 https |

---

## 19. Rollback

1. DNS: voltar A/CNAME para o host anterior (Vercel/grok.me) — TTL baixo (300s) no cutover ajuda.
2. BotFather: Mini App URL antiga.
3. Webhook MP: URL antiga.
4. **Não** correr migrate “para trás”. Migrações só avançam (`0012_…`).
5. `BETTER_AUTH_SECRET` antigo se já tiverem sessões no host antigo.

---

## 20. Inventário de arquivos úteis

### Código de produto (não reescrever no cutover)

```
src/routes/                 rotas TanStack
src/components/hub/         Hub, Loja, Telegram
src/components/table/       mesa VTT
src/components/collection/  Coleção + DieMini3D (GLB authored)
src/components/auth/        idade / termos / login (rota login desligada)
src/lib/shop-*.ts           motor financeiro  ← NÃO TOCAR
src/lib/app-urls.ts         URLs públicas via env
src/lib/telegram.ts         BLOQUEADOR grok.me
src/lib/db.ts               PGLite vs Postgres
src/lib/http-security.ts    CSP / cookies
migrations/0001–0011        schema
public/assets/              dados, ammo.wasm, fotos
public/media/               cartões e molduras WebP
```

### Build / ops

```
package.json                Node 22, npm run build
vite.config.ts              Nitro preset vercel
vercel.json                 install + build
scripts/migrate.mjs         aplica SQL
scripts/fix-ssr-exports.mjs patch Vercel
scripts/with-app-env.mjs    injeta env no Vite
.env.example                desactualizado (ainda grok.me)
```

### Este documento

```
docs/MIGRACAO-miniarena.com.br.md
public/downloads/MIGRACAO-miniarena.com.br.md
```

---

## Apêndice — o que dizer à outra ferramenta

Use este bloco como prompt de execução:

```
Migrar o Mini Arena para https://miniarena.com.br seguindo docs/MIGRACAO-miniarena.com.br.md.

NÃO alterar src/lib/shop-finance.ts, shop-skus.ts, shop-pay.ts (excepto URLs via env),
shop-reconcile.ts, shop-catalog.ts, shop-admin.ts, src/routes/api/mercadopago.ts.

NÃO recolocar login obrigatório em /.
NÃO apontar o domínio para public_html / PHP / MySQL.
NÃO reconstruir dados 3D no browser; GLB authored é só ImportMeshAsync.
NÃO inventar MINIARENA_STORE_URL.

Fazer: patches da §8, Postgres + migrate, HTTPS no apex, env da §9,
webhook Telegram e MP, BotFather Mini App URL, smoke da §18.
```

Fim do documento.
