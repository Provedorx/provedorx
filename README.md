# ProvedorX API — Releases

[![Latest release](https://img.shields.io/github/v/release/Provedorx/provedorx?label=API&sort=semver)](https://github.com/Provedorx/provedorx/releases/latest)
[![Platform](https://img.shields.io/badge/platform-linux%20x86__64%20%7C%20macOS%20arm64-2f81f7)](#assets)
[![Status](https://img.shields.io/badge/status-produ%C3%A7%C3%A3o-2ea043)](#)

Este repositório publica **apenas os binários de release** do ProvedorX. O
código-fonte permanece em um repositório privado.

O ProvedorX é um sistema de gestão para provedores de internet (ISP), com
cobrança, Pix, boleto, notas fiscais, gestão de contratos, rede/ONU, bot de
WhatsApp, observabilidade e atualização integrada.

## Componentes

| Binário | Função |
|---|---|
| `provedorx` | API GraphQL, autenticação, regras de negócio e HTTP |
| `provedorx-worker` | Workers de background (cobrança, eventos, status, backups, rede, fiscal, notificações) |
| `provedorx-bot` | Bot de WhatsApp (quando não está embutido na API) |

Os três binários são compilados da mesma tag e devem ser atualizados juntos.

## Assets

| Asset | Indicado para |
|---|---|
| `provedorx-bundle-linux-x86_64.tar.gz` | **Instalação/atualização no servidor Linux** |
| `provedorx-bundle-macos-arm64.tar.gz` | Instalação local/desenvolvimento no macOS |
| `provedorx-bundle-<plataforma>.tar.gz.sha256` | Verificação de integridade |
| `provedorx-<binário>-<plataforma>` | Instalação manual de um binário isolado |
| `latest.json` | Manifesto usado pelo atualizador automático |

Cada bundle contém:

```text
bin/provedorx
bin/provedorx-worker
bin/provedorx-bot
version.json
```

## Atualização recomendada

Use a própria interface do ProvedorX:

```text
Configurações → Atualizações → Verificar / Atualizar API
```

O sistema:

- consulta `latest.json` diretamente pelo GitHub (sem API REST e sem rate limit);
- compara versões com SemVer;
- baixa o bundle da plataforma correta;
- valida o SHA-256;
- faz backup da versão atual;
- instala API, worker e bot atomicamente;
- reinicia worker/bot e, por último, a API;
- permite rollback para versões anteriores.

## Instalação manual (Linux)

```bash
VERSION="vX.Y.Z"   # ou use latest/download para a última
BASE="https://github.com/Provedorx/provedorx/releases/latest/download"

curl -fL -o provedorx-bundle.tar.gz \
  "$BASE/provedorx-bundle-linux-x86_64.tar.gz"
curl -fL -o provedorx-bundle.tar.gz.sha256 \
  "$BASE/provedorx-bundle-linux-x86_64.tar.gz.sha256"

sha256sum -c provedorx-bundle.tar.gz.sha256

mkdir -p /tmp/provedorx-release
tar -xzf provedorx-bundle.tar.gz -C /tmp/provedorx-release

sudo install -m 0755 /tmp/provedorx-release/bin/provedorx       /opt/provedorx/bin/provedorx
sudo install -m 0755 /tmp/provedorx-release/bin/provedorx-worker /opt/provedorx/bin/provedorx-worker
sudo install -m 0755 /tmp/provedorx-release/bin/provedorx-bot    /opt/provedorx/bin/provedorx-bot

sudo systemctl restart 'provedorx-worker@*' provedorx-bot
sudo systemctl restart provedorx-api
```

> Ajuste os caminhos e os nomes das units conforme a instalação.

## Versões e tags

- Formato: `vMAJOR.MINOR.PATCH` (SemVer).
- A tag é a fonte de verdade da versão.
- O binário reporta a versão em `--version` e no `/health`.
- `latest.json` é regenerado em cada release com:
  - versão e tag;
  - commit;
  - data de build/publicação;
  - notas de release (commits e contribuidores).

## Requisitos (servidor)

- Linux x86_64 com systemd
- MariaDB/MySQL
- NATS
- Vault (segredos)
- MinIO (opcional, para arquivos/backups)
- Certificado TLS para HTTPS

## Links

- [Releases do frontend](https://github.com/Provedorx/provedorx-frontend/releases)
- [Última release da API](https://github.com/Provedorx/provedorx/releases/latest)

## Apoie o projeto

O ProvedorX é um projeto independente. O sistema possui uma tela de doação via
Pix em **Configurações → Atualizações → Apoie o Projeto**.

---

<sub>Este repositório não contém código-fonte. Assets são gerados automaticamente pelo GitHub Actions a partir de tags do repositório privado.</sub>
