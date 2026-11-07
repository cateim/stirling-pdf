# Changelog

Todas as mudanças notáveis do cateim/stirling-pdf são documentadas neste arquivo.

O formato é baseado em [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/).
As versões seguem o esquema **CalVer `YYYY.M.R`** (ano, mês sem zero à esquerda, revisão do mês), **não** o Versionamento Semântico: `2026.7.1` não diz nada sobre compatibilidade, só quando a mudança foi publicada.

> Este changelog foi iniciado em 2026-11-06. A primeira versão resume o repositório como estava nessa data, reconstruída a partir do histórico do git.
>
> Ele registra só a automação de build e deploy deste repositório. O aplicativo é o [Stirling-PDF](https://github.com/Stirling-Tools/Stirling-PDF), do Stirling-Tools, e as mudanças dele ficam nas [releases do upstream](https://github.com/Stirling-Tools/Stirling-PDF/releases). As tags `:<versão>` da imagem no Docker Hub seguem a versão do upstream, não o CalVer deste arquivo.

## [Unreleased]

## [2026.7.1] - 2026-07-07

### ✨ Added

- **Imagem [`cateim/stirling-pdf`](https://hub.docker.com/r/cateim/stirling-pdf) no Docker Hub**: build automático e não oficial do Stirling-PDF no flavor `core`, que o próprio upstream oferece em `STIRLING_FLAVOR` e que sai 100% MIT (© Stirling PDF Inc.), sem o módulo `app/proprietary` e, por isso, sem o limite de 5 usuários da edição proprietária. Nada do upstream é modificado, e os avisos de copyright e licença ficam como estão. Publicada como `:<versão>`, a da release do upstream, e `:latest`.
- **Workflow `Docker Publish`** ([`.github/workflows/build.yml`](./.github/workflows/build.yml)): roda todo domingo às 04:00 UTC, por disparo manual em **Run workflow** e a cada push no `master` que toque o workflow. Descobre a última release do upstream pela API do GitHub e pula o build quando aquela versão já existe no Docker Hub, salvo no disparo manual.
- **Build com o Dockerfile oficial do upstream**: o workflow faz checkout do Stirling-PDF na tag da release e builda o `docker/embedded/Dockerfile` deles com `--build-arg STIRLING_FLAVOR=core`. Como não há Dockerfile próprio, imagem base, dependências e etapas de compilação acompanham o upstream sem mudança aqui.
- **Imagem multi-arch `linux/amd64` + `linux/arm64`**: cada arquitetura é buildada nativamente (`ubuntu-latest` e `ubuntu-24.04-arm`, sem QEMU), enviada por digest e unida num único manifesto pelo job `merge` (`docker buildx imagetools create`). O alvo de produção é um Orange Pi 5 (arm64); o amd64 permite rodar e testar a mesma imagem em x86.
- **[`docker-compose.yml`](./docker-compose.yml) pronto para produção**: porta `8082` só em loopback, dados em bind mounts sob `/srv/pdf`, limites de RAM, CPU e PIDs, healthcheck em `/api/v1/info/status`, rotação de log, `no-new-privileges`, upload limitado a 100 MB, interface em pt-BR e telemetria desligada. As demais variáveis compatíveis com o `core` vêm comentadas como referência, junto com um hardening avançado (`cap_drop`, `read_only`) para testar antes de ligar e um serviço `cloudflared` opcional (perfil `tunnel`).
- **Guia de deploy no [README](./README.md)**: preparação do host (pastas com dono `1000:1000`, porque o container começa como root e cai para UID/GID 1000), idioma português do OCR (`por.traineddata` do Tesseract), deploy pelo Portainer (stack via Git, com atualização automática e **Re-pull image**) ou `docker compose up -d`, e a configuração única do CI (secrets `DOCKERHUB_USERNAME` e `DOCKERHUB_TOKEN`, repositório público para os runners gratuitos).
- **Aviso de que a imagem `core` não tem login**: login, SSO e gestão de usuários moram no módulo proprietário, que este build não inclui. A imagem roda sem autenticação própria e precisa de uma camada externa na frente (Cloudflare Access, Authelia, Authentik ou reverse proxy com Basic Auth), ou de ficar só na LAN.

### 🔐 Security

- **Pipeline endurecido**: actions fixadas em SHA de commit e atualizadas pelo Dependabot ([`.github/dependabot.yml`](./.github/dependabot.yml), um PR agrupado por semana), `permissions: contents: read`, `persist-credentials: false`, push restrito ao `master` e descoberta de versão num job sem secrets. A tag do upstream é validada antes do uso, contra injeção no `$GITHUB_OUTPUT` e ref malicioso no checkout.
- **Fronteira de confiança documentada**: o pipeline builda código do upstream a cada release, então um upstream comprometido gera uma imagem comprometida, risco aceito. Os passos `RUN` do Dockerfile rodam no sandbox do BuildKit, sem acesso ao token do Docker Hub, e o README recomenda um Access Token com escopo só deste repositório (Read & Write, sem Delete nem Admin).

[Unreleased]: https://github.com/cateim/stirling-pdf/compare/1a3807b...HEAD
[2026.7.1]: https://github.com/cateim/stirling-pdf/tree/1a3807b
