# cateim/stirling-pdf

Build automático **não-oficial** do [Stirling-PDF](https://github.com/Stirling-Tools/Stirling-PDF)
no **flavor `core` (100% MIT)** — todas as 50+ ferramentas de PDF, **sem o limite de 5 usuários**
da edição proprietária.

## ⚠️ Não tem login embutido

O flavor `core` **não inclui** o módulo `app/proprietary` — e é justamente lá que mora todo o
sistema de **login / SSO / gestão de usuários** (que é comercial no Stirling). Portanto esta imagem
**roda em modo aberto, sem autenticação própria**. Proteja o acesso por uma **camada externa**:

- **Cloudflare Access** (Zero Trust) — recomendado, dá login via Google/OTP no edge;
- Authelia / Authentik na frente;
- reverse proxy com Basic Auth;
- ou mantenha só na LAN.

## Como funciona

O workflow [`.github/workflows/build.yml`](.github/workflows/build.yml) roda **semanalmente**
(+ manual via *Run workflow*):

1. Descobre a última release do Stirling-PDF (GitHub API).
2. **Pula** se aquela versão já existe no Docker Hub (economiza build).
3. Faz `checkout` do upstream **na tag da release** e builda o **Dockerfile oficial deles**
   (`docker/embedded/Dockerfile`) passando `--build-arg STIRLING_FLAVOR=core`.
4. Publica em `cateim/stirling-pdf:<versão>` e `cateim/stirling-pdf:latest` (**arm64**).

Não há Dockerfile próprio aqui: reutilizamos o build oficial, então base image, dependências e
etapas de compilação acompanham o upstream automaticamente — só trocamos o *flavor* para `core`.

## Licença

O flavor `core` é distribuído sob **MIT** (© Stirling PDF Inc.). Esta imagem **não contém** o módulo
`app/proprietary` (que tem licença comercial). Nada é modificado, removido de avisos ou burlado —
apenas selecionamos o *flavor* MIT que o próprio projeto oferece via `STIRLING_FLAVOR`. Os avisos de
copyright e licença do upstream são preservados.

## Uso

```yaml
services:
  stirling-pdf:
    image: cateim/stirling-pdf:latest
    container_name: stirling-pdf
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - ./data/config:/configs
      - ./data/tessdata:/usr/share/tessdata   # idiomas de OCR (eng, por, osd)
      - ./data/logs:/logs
```

> Coloque **sempre** atrás de uma camada de autenticação externa (Cloudflare Access), pois não há login.

## Setup do CI (uma vez)

1. Crie o repositório **público** no GitHub (o runner arm64 gratuito `ubuntu-24.04-arm` exige repo
   público; em repo privado, use QEMU em `ubuntu-latest` — bem mais lento — ou um self-hosted arm64).
2. Em **Settings → Secrets and variables → Actions**, adicione:
   - `DOCKERHUB_USERNAME` — seu usuário do Docker Hub (`cateim`).
   - `DOCKERHUB_TOKEN` — um *Access Token* do Docker Hub (Account Settings → Security).
3. Dispare o primeiro build em **Actions → Docker Publish → Run workflow**.
