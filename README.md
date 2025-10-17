# Pipeline de CI/CD para a Aplicação "Hooy - Você autêntico"

![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)

## 1. Visão Geral do Projeto

Este projeto documenta a implementação bem-sucedida de um pipeline de CI/CD completo e de ponta a ponta para a plataforma **"Hooy - Você autêntico"**, uma aplicação complexa construída em uma arquitetura monorepo. O objetivo principal foi substituir um processo de deploy manual e propenso a erros por um sistema totalmente automatizado e sem intervenção humana, permitindo que a equipe de desenvolvimento entregue novas funcionalidades aos ambientes de homologação e produção com alta velocidade e confiança.

O pipeline automatiza todo o ciclo de vida para múltiplos serviços independentes (frontends e backends) contidos em um único repositório, garantindo consistência, confiabilidade e segurança.

---

## 2. Arquitetura e Estratégia de Deploy

O pipeline foi projetado com uma estratégia de ambiente duplo, garantindo que o código seja testado em um ambiente de homologação antes de ser promovido para produção.

**Workflow do Ambiente de Homologação:**
* **Gatilho:** `push` direto na branch `homologacao`.
* **Ação:** Constrói, testa e implanta automaticamente o serviço específico que foi alterado no servidor de homologação.
* **Propósito:** Validação rápida e garantia de qualidade para novas funcionalidades.

**Workflow do Ambiente de Produção:**
* **Gatilho:** Um **Pull Request** é revisado, aprovado e **mesclado (merged)** na branch `master`.
* **Ação:** Constrói, testa e implanta automaticamente a versão estável do serviço no servidor de produção.
* **Propósito:** Lançamentos seguros, controlados e auditáveis para os usuários finais.

Essa arquitetura garante que a branch `master` sempre represente um estado estável e pronto para produção.

---

## 3. Conceitos Fundamentais Implementados

Este projeto é uma aplicação prática de princípios fundamentais de DevOps:

-   **CI/CD (Integração Contínua/Implantação Contínua):** Automação da integração de mudanças de código e sua subsequente implantação em ambientes ativos.
-   **Infraestrutura como Código (IaC):** O próprio pipeline é definido como código (arquivos `.yml`), tornando-o versionável, repetível e transparente.
-   **Dockerização:** Cada serviço é containerizado, garantindo que ele execute de forma idêntica em qualquer máquina e abstraindo a infraestrutura subjacente.
-   **Builds Docker Multi-Stage:** Criação de imagens de produção otimizadas, pequenas e seguras, separando o ambiente de construção do ambiente final de execução.
-   **Gerenciamento de Segredos:** Gerenciamento seguro de dados sensíveis, como webhooks de deploy e tokens de acesso, usando os segredos criptografados do GitHub.

---

## 4. Exemplo de Implementação Genérica

Abaixo estão exemplos anonimizados e comentados dos principais arquivos de configuração que alimentam esta automação.

### 4.1. Workflow de Produção do GitHub Actions (`.github/workflows/deploy-prod-service.yml`)

Este workflow implanta um serviço em produção. Ele é acionado apenas por um merge bem-sucedido na branch `master`.

```yaml
# Nome do Workflow
name: CI/CD - Deploy <Nome do Serviço> para Produção

# Gatilho: Roda apenas quando um Pull Request para a branch 'master' é fechado e mesclado.
on:
  pull_request:
    types: [closed]
    branches:
      - master
  workflow_dispatch: {} # Permite acionamento manual

jobs:
  build-and-deploy:
    # Esta condição garante que o job só execute se o PR foi de fato mesclado.
    if: github.event.pull_request.merged == true
    name: Build & Deploy da Imagem de Produção (<Nome do Serviço>)
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout do Código
        uses: actions/checkout@v4

      - name: Login no GitHub Container Registry (GHCR)
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GH_PAT }}

      - name: Build e Push da Imagem Docker
        uses: docker/build-push-action@v5
        with:
          context: ./apps/<pasta-do-servico>
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/hooy/<nome-do-servico>:${{ github.sha }}
            ghcr.io/${{ github.repository_owner }}/hooy/<nome-do-servico>:latest

      - name: Acionar Deploy de Produção
        run: curl -X POST "${{ secrets.EASYPANEL_WEBHOOK_URL_PROD_<NOME_DO_SERVICO> }}" --fail
