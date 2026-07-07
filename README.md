# cateim/stirling-pdf

[![Docker Pulls](https://img.shields.io/docker/pulls/cateim/stirling-pdf?logo=docker)](https://hub.docker.com/r/cateim/stirling-pdf)
[![Image](https://img.shields.io/docker/v/cateim/stirling-pdf/latest?logo=docker&label=latest)](https://hub.docker.com/r/cateim/stirling-pdf/tags)
[![Build](https://github.com/CaTeIM/stirling-pdf/actions/workflows/build.yml/badge.svg)](https://github.com/CaTeIM/stirling-pdf/actions/workflows/build.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](#licença)

Build automático **não-oficial** do [Stirling-PDF](https://github.com/Stirling-Tools/Stirling-PDF)
no **flavor `core` (100% MIT)** — todas as 50+ ferramentas de PDF, **sem o limite de 5 usuários**
da edição proprietária. Publicado no Docker Hub como
[`cateim/stirling-pdf`](https://hub.docker.com/r/cateim/stirling-pdf).

> Este repositório **não tem código de aplicação** — o app é do Stirling-PDF. Aqui só existe a
> automação de CI que builda a imagem MIT e a publica.

## ⚠️ Não tem login embutido

O flavor `core` **não inclui** o módulo `app/proprietary` — e é justamente lá que mora todo o
sistema de **login / SSO / gestão de usuários** (comercial no Stirling). Portanto esta imagem
**roda em modo aberto, sem autenticação própria**. Proteja o acesso por uma **camada externa**:

- **Cloudflare Access** (Zero Trust) — recomendado, dá login via Google/OTP no edge;
- Authelia / Authentik na frente;
- reverse proxy com Basic Auth;
- ou mantenha só na LAN.

## Deploy

O arquivo [`docker-compose.yml`](docker-compose.yml) já vem pronto para o alvo de produção
(porta `8082` só em loopback, dados em `/srv/pdf`, limites de RAM/CPU, healthcheck e hardening),
com **todas as variáveis de ambiente úteis comentadas** para referência. Ajuste conforme seu host.

### Preparar o host (antes do 1º deploy)

O container roda como **UID/GID 1000** (inicia como root e dropa privilégio via `PUID/PGID`), então
os diretórios de dados no host precisam pertencer a `1000:1000`:

```bash
sudo mkdir -p /srv/pdf/{config,logs,tessdata,customFiles,pipeline,fonts}
sudo chown -R 1000:1000 /srv/pdf
```

**OCR (Tesseract).** O container popula `eng` e `osd` em `/srv/pdf/tessdata` no 1º boot. Para
reconhecer **português**, baixe o idioma do repositório oficial:

```bash
sudo curl -fSL -o /srv/pdf/tessdata/por.traineddata \
  https://github.com/tesseract-ocr/tessdata_fast/raw/main/por.traineddata
sudo chown 1000:1000 /srv/pdf/tessdata/por.traineddata
```

> Se após o 1º boot a pasta `tessdata` vier vazia, baixe também `eng.traineddata` e
> `osd.traineddata` do mesmo modo (o mount pode sombrear os idiomas embutidos em alguns setups).

### Portainer (automático, via Git)

O jeito recomendado — o Portainer clona este repo e mantém a stack atualizada sozinho:

1. **Stacks → Add stack → Repository**.
2. **Repository URL:** `https://github.com/CaTeIM/stirling-pdf`
3. **Compose path:** `docker-compose.yml`
4. Ligue **Automatic updates** (polling a cada X min _ou_ webhook) e marque **Re-pull image**,
   para o container acompanhar tanto mudanças no compose quanto novas versões do `:latest`.

### `docker compose` manual

```bash
git clone https://github.com/CaTeIM/stirling-pdf
cd stirling-pdf
docker compose up -d
```

> Em qualquer caso: coloque **sempre** atrás de autenticação externa (Cloudflare Access) — a
> imagem `core` não tem login.

## Arquitetura

Imagem **multi-arch**: **`linux/amd64` + `linux/arm64`**, publicada como um único manifesto
(`:latest` e `:<versão>`). O build é **nativo por arquitetura** — arm64 no runner `ubuntu-24.04-arm`
e amd64 no `ubuntu-latest`, ambos gratuitos em repo público —, evitando a lentidão do QEMU. Cada
arch é enviado por _digest_ e um job final junta tudo com `docker buildx imagetools create`.

O alvo de produção é o **Orange Pi 5** (arm64); o amd64 permite rodar/testar a mesma imagem em x86.

## Como funciona o build

O workflow [`.github/workflows/build.yml`](.github/workflows/build.yml) roda **todo domingo
04:00 UTC** (+ manual via _Run workflow_, + push no `master` que toque o workflow), em **3 jobs**:

1. **prepare** (sem secrets) — descobre a última release (GitHub API), valida a tag e decide se
   builda (pula se a versão já existe no Docker Hub, salvo disparo manual).
2. **build** (matrix `amd64` + `arm64`) — faz `checkout` do upstream **na tag** e builda o
   **Dockerfile oficial deles** (`docker/embedded/Dockerfile`) com `--build-arg STIRLING_FLAVOR=core`;
   cada arch é enviado **por digest**.
3. **merge** — junta os digests no manifesto multi-arch `:<versão>` + `:latest`.

Não há Dockerfile próprio aqui: reutilizamos o build oficial, então base image, dependências e
etapas de compilação acompanham o upstream automaticamente — só trocamos o _flavor_ para `core`.

### Segurança do pipeline

- Actions **fixadas em SHA de commit**, atualizadas pelo [Dependabot](.github/dependabot.yml).
- `permissions: contents: read`, `persist-credentials: false`, gatilho `push` restrito ao `master`.
- **Recomendado:** use um _Access Token_ do Docker Hub com escopo **só do repositório
  `cateim/stirling-pdf`, Read & Write** (sem Delete/Admin) como `DOCKERHUB_TOKEN`.
- **Fronteira de confiança:** o pipeline builda código do upstream a cada release; se o
  Stirling-Tools for comprometido, a imagem também será. Os passos `RUN` do Dockerfile rodam em
  sandbox do BuildKit (sem acesso ao token do runner), mas o risco de imagem envenenada é aceito.

## Licença

O flavor `core` é distribuído sob **MIT** (© Stirling PDF Inc.). Esta imagem **não contém** o
módulo `app/proprietary` (licença comercial). Nada é modificado, removido de avisos ou burlado —
apenas selecionamos o _flavor_ MIT que o próprio projeto oferece via `STIRLING_FLAVOR`. Os avisos
de copyright e licença do upstream são preservados.

## Setup do CI (uma vez)

1. Repositório **público** no GitHub (o runner arm64 gratuito `ubuntu-24.04-arm` exige repo
   público; em repo privado, use QEMU em `ubuntu-latest` — bem mais lento — ou self-hosted arm64).
2. Em **Settings → Secrets and variables → Actions**, adicione:
   - `DOCKERHUB_USERNAME` — seu usuário do Docker Hub (`cateim`).
   - `DOCKERHUB_TOKEN` — um _Access Token_ do Docker Hub (Account Settings → Security).
3. Dispare o primeiro build em **Actions → Docker Publish → Run workflow** (ou deixe o cron rodar).
