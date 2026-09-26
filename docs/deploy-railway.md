# Deploy no Railway

O AquaTrip já nasceu pronto para rodar atrás de um proxy/load balancer
(cookies `secure`, `trust proxy`, `/healthz` e `/readyz`, sessão
persistida no MySQL em vez de memória, migrations idempotentes). Este
guia cobre só a parte específica do Railway: variáveis, banco,
domínio e persistência de uploads.

O repositório já inclui `railway.json`, que diz ao Railway para:
- buildar com Nixpacks (detecta Node automaticamente pelo `package.json`/`package-lock.json`);
- antes de subir, rodar `npm run db:migrate` e só então `npm start`
  (as migrations gravam o que já foi aplicado em `_migrations`, então
  rodar de novo a cada deploy é seguro e rápido quando não há nada novo);
- usar `/healthz` como healthcheck de deploy.

## 1. Criar o projeto e o banco

1. No Railway, crie um projeto novo a partir deste repositório (GitHub) ou via `railway up` (CLI).
2. No mesmo projeto, clique em **+ New → Database → Add MySQL**.
3. Isso cria um serviço `MySQL` com as variáveis `MYSQLHOST`, `MYSQLUSER`,
   `MYSQLPASSWORD`, `MYSQLDATABASE`, `MYSQL_URL` (acesso público) e
   `MYSQL_PRIVATE_URL` (rede privada do projeto).

## 2. Variáveis de ambiente do serviço da aplicação

Abra o serviço do app → aba **Variables** e adicione:

| Variável | Valor | Observação |
|---|---|---|
| `NODE_ENV` | `production` | Ativa cookie `secure`, exige `TOTP_ENCRYPTION_KEY`, bloqueia o gateway de pagamento mock. |
| `DATABASE_URL` | `${{MySQL.MYSQL_PRIVATE_URL}}` | Reference Variable apontando pro serviço MySQL — use a URL **privada** (mais rápida e sem custo de tráfego público), não a `MYSQL_URL` pública. |
| `SESSION_SECRET` | gerar com `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"` | Obrigatória; trocar invalida sessões ativas. |
| `TOTP_ENCRYPTION_KEY` | gerar com `node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"` | Obrigatória em produção (cifra os segredos de 2FA). Guarde em lugar seguro — perdida, todo mundo reconfigura o 2FA. |
| `PUBLIC_BASE_URL` | `https://SEU-DOMINIO.up.railway.app` | Gere o domínio primeiro (passo 3), depois preencha esta variável e faça redeploy — ela é usada como `notification_url` de webhook de pagamento. |
| `PAYMENT_PROVIDER` | `mock` (padrão) ou `mercadopago` | **Em produção o provider `mock` é recusado em tempo de checkout.** Se quiser aceitar pagamento de verdade, use `mercadopago` + `MP_ACCESS_TOKEN`/`MP_PUBLIC_KEY`/`MP_WEBHOOK_SECRET`. Sem isso, o site sobe normalmente, mas o checkout falha ao ser chamado. |
| `ADMIN_NAME` / `ADMIN_EMAIL` / `ADMIN_PASSWORD` | à sua escolha | Só usadas por `npm run db:seed` (passo 5), rode uma vez e pode remover depois. |
| `UPLOAD_DIR` | `./uploads` (padrão) | Ver seção de Volume abaixo — sem volume, os arquivos somem a cada deploy. |

As demais variáveis de `.env.example` (LGPD, e-mail, backup, 2FA,
marketplace) têm valor padrão sensato e só precisam ser tocadas se o
comportamento correspondente for usado (SMTP real, comissão de
parceiro diferente, etc.).

> Todas as variáveis marcadas como obrigatórias acima fazem o processo
> falhar já na subida (`app.js`/`server.js` validam antes de abrir a
> porta) — se o deploy cair logo no início, comece checando os logs
> por `SESSION_SECRET`, `TOTP_ENCRYPTION_KEY` ou `DATABASE_URL`.

## 3. Domínio público

Em **Settings → Networking → Generate Domain** (ou conecte um domínio
próprio). Depois de gerar, volte e preencha `PUBLIC_BASE_URL` com essa
URL e faça um redeploy — os links de e-mail (verificação, redefinição
de senha) e o `notification_url` do Mercado Pago dependem dela.

## 4. Persistência dos uploads de imagem

`app/services/mediaService.js` grava as fotos enviadas (parceiros,
reviews) em disco local, fora de `app/public`, servidas por
`/media/:chave`. Disco de um serviço Railway é **efêmero**: sem um
Volume, cada deploy apaga tudo que foi enviado.

1. No serviço do app → **Settings → Volumes → New Volume**.
2. Mount path: `/app/uploads` (o Nixpacks builda a aplicação em `/app`,
   e `UPLOAD_DIR` por padrão resolve para `./uploads` a partir daí).
3. Redeploy.

Para múltiplas instâncias/réplicas, um Volume não é compartilhado
entre elas — nesse caso troque para um storage de objetos (S3, R2,
GCS) seguindo a observação já deixada em `mediaService.js`.

## 5. Primeiro deploy: rodar o seed do admin

As migrations já rodam sozinhas a cada deploy. O seed do usuário admin
é manual e roda só uma vez:

```bash
railway run npm run db:seed
```

(via CLI, com `ADMIN_NAME`/`ADMIN_EMAIL`/`ADMIN_PASSWORD` já definidos
nas variáveis do passo 2). Depois disso, é seguro remover essas três
variáveis do serviço.

## 6. Backups

`scripts/backup.sh`/`restore.sh` foram feitos para rodar via cron em
um host próprio, não sozinhos no Railway. Duas opções práticas:
- criar um serviço separado no mesmo projeto (Railway **Cron Job**,
  imagem a partir deste mesmo repo) agendado para rodar
  `npm run db:backup`, escrevendo em um Volume próprio; ou
- confiar nos backups automáticos do plugin MySQL do Railway (verifique
  o plano contratado) e tratar `scripts/backup.sh` como ferramenta de
  uso manual/local.

## Checklist final

- [ ] Plugin MySQL criado e `DATABASE_URL` referenciando `MYSQL_PRIVATE_URL`
- [ ] `NODE_ENV=production`
- [ ] `SESSION_SECRET` e `TOTP_ENCRYPTION_KEY` gerados e definidos
- [ ] Domínio gerado e `PUBLIC_BASE_URL` preenchida
- [ ] Volume montado em `/app/uploads`
- [ ] `PAYMENT_PROVIDER` decidido (`mock` só para demo; `mercadopago` com credenciais para checkout real)
- [ ] `npm run db:seed` rodado uma vez para criar o admin
- [ ] `/healthz` e `/readyz` respondendo `200` depois do deploy
