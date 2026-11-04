# ProvedorX API — Releases

[![Latest release](https://img.shields.io/github/v/release/Provedorx/provedorx?label=API&sort=semver)](https://github.com/Provedorx/provedorx/releases/latest)
[![Platform](https://img.shields.io/badge/platform-linux%20x86__64%20%7C%20macOS-2f81f7)](#assets)
[![Status](https://img.shields.io/badge/status-produ%C3%A7%C3%A3o-2ea043)](#)

Este repositório publica **apenas os binários e artefatos de release** do
ProvedorX. O código-fonte permanece em um repositório privado.

O ProvedorX é um sistema de gestão para provedores de internet (ISP), com
cobrança, Pix, boleto, notas fiscais, contratos, rede/ONU, bot de WhatsApp,
observabilidade e atualização integrada.

## Assets

| Asset | Indicado para |
|---|---|
| `provedorx-bundle-linux-x86_64.tar.gz` | Servidor Linux x86_64 |
| `provedorx-bundle-macos-arm64.tar.gz` | macOS Apple Silicon |
| `provedorx-bundle-<plataforma>.tar.gz.sha256` | Verificação SHA-256 |
| `latest.json` | Manifesto usado pelo atualizador |
| `provedorx-<binário>-<plataforma>` | Binário isolado (instalação manual) |

> Para Linux arm64, gere o bundle localmente com
> `LINUX_PLATFORM=linux/arm64 ./build.sh linux` (o release oficial publica
> Linux x86_64 e macOS arm64).

Cada bundle contém:

```text
bin/provedorx
bin/provedorx-worker
bin/provedorx-bot
deploy/bootstrap.sh
deploy/env/*.env.example
deploy/sql/01-schema.sql
deploy/sql/02-seed.sql
deploy/sql/03-bootstrap.sql.tpl
deploy/radius/templates/
deploy/systemd/
deploy/docker-compose.release.yml
deploy/docker/Dockerfile.runtime
deploy/vault.hcl
version.json
```

Nenhum `.env` com segredos é distribuído.

---

## Primeira instalação — systemd (Debian/Ubuntu)

Pré-requisitos: Debian 12/Ubuntu 22.04+, acesso root, internet e as portas
`8000`/`8001` liberadas (ou um proxy na frente).

```bash
BASE="https://github.com/Provedorx/provedorx/releases/latest/download"

curl -fL -O "$BASE/provedorx-bundle-linux-x86_64.tar.gz"
curl -fL -O "$BASE/provedorx-bundle-linux-x86_64.tar.gz.sha256"
sha256sum -c provedorx-bundle-linux-x86_64.tar.gz.sha256

sudo mkdir -p /opt/provedorx-release
sudo tar -xzf provedorx-bundle-linux-x86_64.tar.gz -C /opt/provedorx-release
cd /opt/provedorx-release

sudo ./deploy/bootstrap.sh --mode systemd
```

O bootstrap:

1. instala MariaDB, Vault, MinIO, NATS, FreeRADIUS e dependências;
2. gera senha do banco, JWT, encryption keys, tokens do bot, credenciais do
   MinIO e senha do admin;
3. cria banco/usuário, aplica schema + seed + settings/gateways/admin;
4. inicializa o Vault com **2 chaves de unseal** (`/etc/provedorx/vault-keys.json`);
5. grava o segredo do MinIO no Vault;
6. configura o FreeRADIUS com as templates de produção e gera usuário/senha
   exclusivos do Radius e o segredo dos NAS;
7. gera `/etc/provedorx/{common,api,worker,bot}.env`;
8. instala units systemd, polkit e certificado TLS self-signed;
9. inicia API, bot, workers, NATS e FreeRADIUS.

No final ele mostra o e-mail/senha do admin, o caminho das chaves do Vault e
as credenciais do Radius (usuário/senha do banco e segredo dos NAS). **Guarde
esses dados.**

### Frontend (systemd)

O frontend é um release separado:

```bash
curl -fL -O https://github.com/Provedorx/provedorx-frontend/releases/latest/download/provedorx-frontend.tar.gz
sudo mkdir -p /opt/frontend
sudo tar -xzf provedorx-frontend.tar.gz -C /opt/frontend --strip-components=1
```

A API serve os arquivos de `/opt/frontend` em `/` e `/assets`.

---

## Primeira instalação — Docker Compose

Pré-requisitos: Docker Engine + Compose v2.24+.

```bash
BASE="https://github.com/Provedorx/provedorx/releases/latest/download"
curl -fL -O "$BASE/provedorx-bundle-linux-x86_64.tar.gz"
curl -fL -O "$BASE/provedorx-bundle-linux-x86_64.tar.gz.sha256"
sha256sum -c provedorx-bundle-linux-x86_64.tar.gz.sha256

mkdir -p /opt/provedorx-release
tar -xzf provedorx-bundle-linux-x86_64.tar.gz -C /opt/provedorx-release
cd /opt/provedorx-release

./deploy/bootstrap.sh --mode docker
```

O compose sobe MariaDB, NATS, Vault, MinIO, API, workers e bot. Para subir
apenas a infraestrutura e inicializar banco/Vault/MinIO:

```bash
./deploy/bootstrap.sh --mode docker --infra-only
```

### Frontend (Docker)

Extraia o release do front em `frontend/` na raiz do bundle. O compose monta
esse diretório em `/opt/frontend` no container da API:

```bash
curl -fL -O https://github.com/Provedorx/provedorx-frontend/releases/latest/download/provedorx-frontend.tar.gz
mkdir -p frontend
tar -xzf provedorx-frontend.tar.gz -C frontend --strip-components=1
```

---

## Já tenho instalação — systemd

> Faça backup do banco e de `/etc/provedorx` antes. **Nunca** rode
> `bootstrap.sh --reset-db` em produção.

1. Baixe e valide o bundle novo.
2. Pare os serviços:

```bash
sudo systemctl stop provedorx-api provedorx-bot 'provedorx-worker@*'
```

3. Backup:

```bash
sudo cp -a /opt/provedorx/bin /opt/provedorx/bin.bak-$(date +%F)
sudo cp -a /etc/provedorx /etc/provedorx.bak-$(date +%F)
```

4. Instale os binários novos:

```bash
sudo install -m 0755 bin/provedorx        /opt/provedorx/bin/provedorx
sudo install -m 0755 bin/provedorx-worker /opt/provedorx/bin/provedorx-worker
sudo install -m 0755 bin/provedorx-bot    /opt/provedorx/bin/provedorx-bot
```

5. Compare `deploy/env/*.env.example` com `/etc/provedorx/*.env` e adicione
   as chaves novas manualmente (`GITHUB_API_REPO`, `GITHUB_FRONT_REPO`,
   `SYSTEMD_*`, etc.).
6. Aplique somente **migrations pendentes**. Não reaplique
   `01-schema.sql`/`02-seed.sql` em banco existente.
7. Suba os serviços:

```bash
sudo systemctl start provedorx-nats
sudo systemctl start provedorx-api
sudo systemctl start provedorx-bot
sudo systemctl start 'provedorx-worker@*'
```

8. Confira `curl -s http://127.0.0.1:8001/health`.

Para (re)configurar o FreeRADIUS em uma instalação existente, rode novamente o
bootstrap com `--with-radius`; ele reaproveita os segredos, detecta o schema
existente e só regrava as configurações do Radius. Se o banco for externo,
use `--skip-db` e garanta o usuário do Radius no MariaDB.

---

## Já tenho instalação — containers no Proxmox/Docker

O release novo entrega binários + `deploy/docker/Dockerfile.runtime`. Para quem
já tem um container da API rodando:

1. **Backup** do banco (`mariadb-dump`) e dos volumes de uploads/front.
2. Baixe e valide o bundle novo.
3. Gere a imagem nova:

```bash
cd /opt/provedorx-release
docker build -f deploy/docker/Dockerfile.runtime -t provedorx:<versao> .
```

4. Recrie **somente a API** com os mesmos envs, rede, portas e volumes do
   container atual. Com Compose:

```bash
docker compose up -d --no-deps api
```

Com `docker run`, inspecione o container atual e repita os mesmos argumentos,
trocando apenas a imagem:

```bash
docker inspect <container_api> --format '{{json .Mounts}}'
docker inspect <container_api> --format '{{json .Config.Env}}'
```

### Cenário Proxmox: API e front em containers separados

Se o banco, Vault, MinIO e NATS já estão em outras máquinas/containers, use o
compose de instalação existente (ele **não** cria infraestrutura nem aplica
schema/seed):

```bash
cd /opt/provedorx-release
mkdir -p deploy/env
cp deploy/env/*.env.example deploy/env/
```

Edite `deploy/env/common.env` com os valores atuais:

```dotenv
DATABASE_URL=mysql://usuario:senha@10.60.1.133:3306/isp?charset=utf8mb4
VAULT_ADDR=http://10.60.1.132:8200
VAULT_TOKEN=<token atual do Vault>
NATS_URL=nats://nats:4222   # NATS incluído no compose externo
ENCRYPTION_KEY=<mesma chave da instalação atual>
SECRETES_KEY=<mesma chave da instalação atual>
```

Edite `deploy/env/api.env`:

```dotenv
JWT_SECRET=<mesmo segredo da instalação atual>
FRONTEND_DIR=/opt/frontend
GITHUB_API_REPO=Provedorx/provedorx
GITHUB_FRONT_REPO=Provedorx/provedorx-frontend
# URL pública da API para o /config.js do front; vazio = mesma origem
FRONTEND_API_URL=
```

> **Muito importante:** mantenha os mesmos `JWT_SECRET`, `ENCRYPTION_KEY` e
> `SECRETES_KEY` da instalação atual. Trocar `SECRETES_KEY` impede
> descriptografar segredos já gravados no banco.

Suba apenas API, workers e bot:

```bash
docker compose -f deploy/docker-compose.external.yml up -d --build
```

O front continua no container dele; substitua os arquivos pelo release novo
(ver README do frontend). Se quiser que a API sirva o front, monte o diretório
compartilhado em `/opt/frontend`.

O Radius pode continuar na máquina/container atual. O release inclui as
templates em `deploy/radius/templates`; ajuste host/banco/usuário/senha e o
segredo dos NAS conforme os valores gerados pelo bootstrap.

Se o Radius estiver em outra máquina (ex.: `10.60.1.134`), crie o usuário no
MariaDB liberando o IP dele:

```sql
CREATE USER 'radius'@'10.60.1.134' IDENTIFIED BY '<senha gerada>';
GRANT SELECT, INSERT, UPDATE, DELETE ON isp.* TO 'radius'@'10.60.1.134';
FLUSH PRIVILEGES;
```

Depois rode no host do Radius:

```bash
sudo ./deploy/bootstrap.sh --radius-only \
  --db-host 10.60.1.133 --db-name isp \
  --radius-db-user radius --radius-db-password '<senha>' \
  --radius-net 10.60.1.0/24
```

Ele instala apenas o FreeRADIUS e grava a configuração; não instala API, banco,
Vault, MinIO ou NATS.

O compose externo sobe NATS, API, bot e um container por worker (events,
billing, status, backups, maintenance, notifications, network, fiscal). Ele
força `API_BACKGROUND_WORKERS=none` para os loops não rodarem duplicados na
API. Se quiser manter os loops dentro da API durante a transição, remova os
serviços `worker-*` e use `API_BACKGROUND_WORKERS=legacy`.

Se já tem um NATS externo, defina `NATS_URL=nats://seu-host:4222` no
`deploy/.env` (variável do Compose) e remova o serviço `nats` do arquivo.

5. **Não** rode `./deploy/bootstrap.sh --mode docker` no mesmo host com
   containers/volumes existentes. Ele cria uma stack e volumes novos; use o
   bootstrap apenas para instalação limpa ou em outra máquina.
6. Ajuste o banco:

```sql
UPDATE provider_settings
SET github_api_repo = 'Provedorx/provedorx',
    github_front_repo = 'Provedorx/provedorx-frontend',
    front_dir = '/opt/frontend'
WHERE id = 1;
```

7. Atualize o container/diretório do frontend com o release novo.
8. Confira `/health` e o rodapé da interface.

> Em Docker, a atualização automática de **binário da API** pela interface não
> é o caminho recomendado: recrie o container com a imagem nova. A atualização
> do **frontend** funciona se `/opt/frontend` for um volume gravável e, se o
> front estiver em outro container, compartilhado entre os dois.

### Banco e migrations

- Instalação nova: o bootstrap aplica schema + seed + bootstrap.
- Instalação existente: **não** reaplique schema/seed; aplique apenas as
  migrations pendentes do repositório privado, sempre após backup.
- Se o release não trouxer migrations novas, não há nada a aplicar além dos
  binários/envs.

---

## Build de validação (sem publicar)

Antes de criar a tag, rode o workflow manualmente:

```text
Actions → Release ProvedorX → Run workflow
```

Informe uma versão de teste (ex.: `0.2.1`). Ele compila Linux x86_64 e macOS,
mas **não publica release** e **não roda testes nem banco de dados**. A tag
`vX.Y.Z` continua sendo o gatilho da publicação real.

## Atualização pela interface

```text
Configurações → Atualizações → Verificar / Atualizar API
```

O sistema consulta `latest.json` no GitHub sem usar a API REST, compara SemVer,
baixa o bundle da plataforma, valida o SHA-256, faz backup, instala API/worker/
bot atomicamente, reinicia worker/bot e agenda o restart da API. O rollback
usa `/opt/provedorx/bin/.backups/<versao>`.

## Instalação manual (somente binários)

```bash
BASE="https://github.com/Provedorx/provedorx/releases/latest/download"
curl -fL -O "$BASE/provedorx-bundle-linux-x86_64.tar.gz"
curl -fL -O "$BASE/provedorx-bundle-linux-x86_64.tar.gz.sha256"
sha256sum -c provedorx-bundle-linux-x86_64.tar.gz.sha256

mkdir -p /tmp/provedorx-release
tar -xzf provedorx-bundle-linux-x86_64.tar.gz -C /tmp/provedorx-release

sudo install -m 0755 /tmp/provedorx-release/bin/provedorx        /opt/provedorx/bin/provedorx
sudo install -m 0755 /tmp/provedorx-release/bin/provedorx-worker /opt/provedorx/bin/provedorx-worker
sudo install -m 0755 /tmp/provedorx-release/bin/provedorx-bot    /opt/provedorx/bin/provedorx-bot

sudo systemctl restart 'provedorx-worker@*' provedorx-bot
sudo systemctl restart provedorx-api
```

## Requisitos

- Linux x86_64/arm64 com systemd, ou Docker Compose
- MariaDB/MySQL (com as tabelas `rad*` e `nas`)
- NATS
- Vault (segredos)
- MinIO (arquivos/backups)
- Certificado TLS para HTTPS

## Links

- [Releases do frontend](https://github.com/Provedorx/provedorx-frontend/releases)
- [Última release da API](https://github.com/Provedorx/provedorx/releases/latest)

## Apoie o projeto

O ProvedorX é um projeto independente. Há uma tela de doação via Pix em
**Configurações → Atualizações → Apoie o Projeto**.

---

<sub>Este repositório não contém código-fonte. Assets são gerados automaticamente pelo GitHub Actions a partir de tags do repositório privado.</sub>
