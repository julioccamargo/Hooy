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
```
4.2. Dockerfile Seguro para um Serviço Frontend (/apps/<pasta-do-servico>/Dockerfile)
Este exemplo usa um build multi-stage para criar uma imagem Nginx mínima e segura para uma aplicação frontend (e.g., Vue.js), incluindo melhorias de segurança e confiabilidade.

```
# --- Estágio 1: Ambiente de Build ---
# Usa uma imagem Node.js completa para instalar dependências e construir a aplicação.
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

# --- Estágio 2: Ambiente de Produção ---
# Começa a partir de uma imagem Nginx limpa e leve.
FROM nginx:stable-alpine

# Adiciona o curl para as verificações de saúde (health checks).
RUN apk add --no-cache curl

# Copia apenas os arquivos da aplicação compilada do estágio 'builder'.
COPY --from=builder /app/dist /usr/share/nginx/html

# Copia o arquivo de configuração do Nginx.
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Define a posse dos arquivos para o usuário não-root 'nginx' para maior segurança.
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid && \
    chown -R nginx:nginx /var/cache/nginx

# Muda para o usuário não-root.
USER nginx

# Expõe a porta HTTP padrão.
EXPOSE 80

# Define uma verificação de saúde para permitir que a plataforma monitore o estado do serviço.
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

# Comando para iniciar o servidor Nginx.
CMD ["nginx", "-g", "daemon off;"]
```

5. Principais Desafios e Soluções
A implementação deste pipeline envolveu a superação de vários desafios do mundo real:

Inconsistência de Ambientes: Resolvido com a implementação de Docker e o versionamento do package-lock.json para garantir builds 100% reproduzíveis nos ambientes local, de homologação e de produção.

Falhas na Instalação de Dependências (EINTEGRITY): Superados problemas persistentes de rede/cache no ambiente de CI através da implementação de tratamentos de erro robustos, fallbacks de registro e estratégias de limpeza de cache.

Permissões Complexas do GitHub: Desenhadas soluções para políticas de segurança a nível de organização (Third-party access, Workflow permissions) que estavam bloqueando os tokens do CI, exigindo um mergulho profundo no modelo de segurança do GitHub.

Integração com a Plataforma de Deploy: Depurados problemas de integração validando sistematicamente a acessibilidade da imagem, tokens de autenticação e configurações de porta do serviço, identificando ao final a necessidade de uma tag :latest específica exigida pela plataforma.

6. Resultado do Projeto
O projeto foi um sucesso completo. A plataforma "Hooy - Você autêntico" agora se beneficia de um processo de CI/CD profissional e totalmente automatizado. Isso empoderou a equipe de desenvolvimento ao eliminar tarefas de deploy manuais, reduzir drasticamente o erro humano e acelerar o ciclo de feedback, permitindo a entrega de valor aos usuários finais de forma mais rápida e confiável.
