# Desafio DevSecOps — Gerenciador de Tarefas

## Sobre o Projeto

Este repositório faz parte do desafio prático do módulo de DevSecOps da ADA Tech.

O projeto consiste em uma aplicação web de gerenciamento de tarefas, disponibilizada inicialmente com vulnerabilidades de segurança propositalmente inseridas no código e com uma pipeline de CI/CD incompleta.

O objetivo do desafio foi implementar os controles de segurança na pipeline, identificar as vulnerabilidades existentes, corrigi-las e realizar o deploy da aplicação somente após a aprovação dos gates de segurança.

## Objetivos

O projeto teve como objetivos:

- Implementar uma pipeline DevSecOps utilizando GitHub Actions;
- Implementar Secrets Scanning com Gitleaks;
- Implementar SAST com Semgrep;
- Implementar SCA com Grype;
- Fazer a pipeline interromper sua execução quando um problema de segurança fosse encontrado;
- Corrigir as vulnerabilidades identificadas;
- Atualizar as dependências do projeto;
- Publicar a aplicação utilizando GitHub Pages;
- Documentar o funcionamento da pipeline e o processo de correção das vulnerabilidades.

## Tecnologias e Ferramentas

| Tecnologia/Ferramenta | Função |
|---|---|
| Git | Controle de versão |
| GitHub | Hospedagem do repositório e gerenciamento do código |
| GitHub Actions | Automação da pipeline CI/CD |
| Gitleaks | Secrets Scanning |
| Semgrep | SAST (Static Application Security Testing) |
| Grype | SCA (Software Composition Analysis) |
| GitHub Pages | Deploy da aplicação |

---

## Como a Pipeline Funciona

A pipeline é executada automaticamente a cada `push` na branch `main`.

O fluxo implementado é:

```text
Checkout do Código

        ↓

      Build

        ↓

    Gitleaks

        ↓

     Semgrep

        ↓

      Grype

        ↓

  GitHub Pages

        ↓

    Produção
```

Cada ferramenta de segurança funciona como um **gate** da pipeline.

Caso uma etapa de segurança apresente uma condição de falha, a execução da pipeline é interrompida e as etapas seguintes não são executadas.

Esse comportamento demonstra o conceito de **break the build**, no qual problemas de segurança impedem que o código avance para as etapas seguintes até que sejam corrigidos.

---

# 1. Secrets Scanning — Gitleaks

O primeiro controle de segurança implementado foi o **Gitleaks**, utilizado para identificar possíveis segredos expostos no código-fonte, como chaves de API, senhas e tokens.

## Implementação

O step foi adicionado ao arquivo `.github/workflows/pipeline.yml`:

```yaml
- name: 🔑 Secrets Scanning
  uses: gitleaks/gitleaks-action@v2
```

## Detecção

Após a implementação do Gitleaks, a pipeline foi executada novamente.

O scanner identificou informações sensíveis presentes no arquivo:

```text
src/script.js
```

Essas informações faziam parte das vulnerabilidades propositalmente inseridas no projeto para o desafio.

## Correção

As informações sensíveis foram removidas do arquivo `src/script.js`.

Após a correção, foi realizado um novo commit e a pipeline foi executada novamente.

## Resultado

Na nova execução, o Gitleaks não encontrou mais segredos expostos e a etapa foi aprovada.

O processo demonstrou o funcionamento do gate de segurança: a pipeline foi interrompida enquanto existiam informações sensíveis no código e voltou a avançar após a correção.

---

# 2. SAST — Semgrep

O segundo controle implementado foi o **Semgrep**, utilizado para realizar análise estática do código-fonte, conhecida como SAST (Static Application Security Testing).

O objetivo foi identificar padrões de código que poderiam representar vulnerabilidades de segurança.

## Implementação

O step foi adicionado ao arquivo `.github/workflows/pipeline.yml`:

```yaml
- name: 🔍 SAST - Semgrep
  run: |
    pip install semgrep
    semgrep scan --config auto --error src/
```

A opção `--error` faz com que a etapa seja considerada como falha quando forem encontrados problemas de acordo com as regras utilizadas.

## Problemas Identificados

Durante a execução sobre o código original, o Semgrep identificou dois problemas principais:

- Uso da função `eval()`;
- Uso inseguro de `innerHTML`.

Esses problemas faziam parte das vulnerabilidades propositalmente inseridas no projeto.

## Correção do `eval()`

O código utilizava `eval()` para executar dinamicamente uma instrução.

A função foi removida e substituída por uma operação direta, eliminando a necessidade de executar código dinamicamente.

A correção passou a utilizar:

```javascript
console.log("Tarefa adicionada: " + input.value);
```

## Correção do `innerHTML`

O código também utilizava `innerHTML` para inserir o conteúdo informado pelo usuário.

Para evitar a inserção direta de conteúdo HTML, o código foi alterado para criar um elemento e inserir o texto utilizando `innerText`:

```javascript
const li = document.createElement('li');
li.innerText = input.value;
output.appendChild(li);
```

## Resultado

Após as correções, a pipeline foi executada novamente.

O Semgrep deixou de identificar os problemas encontrados anteriormente e a etapa foi aprovada.

Esse processo demonstrou o funcionamento do SAST como um gate de segurança integrado ao processo de CI/CD.

---

# 3. SCA — Grype

O terceiro controle implementado foi o **Grype**, utilizado para realizar SCA (Software Composition Analysis), verificando componentes e dependências do projeto em busca de vulnerabilidades conhecidas.

## Implementação

O step foi configurado no arquivo `.github/workflows/pipeline.yml`:

```yaml
- name: 📦 SCA - Grype
  run: |
    npm install --package-lock-only
    curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh \
      | sh -s -- -b /usr/local/bin
    grype dir:./ --fail-on medium
```

O comando `--fail-on medium` configura o Grype para retornar falha quando forem encontradas vulnerabilidades classificadas a partir do nível definido.

## Atualização das Dependências

O arquivo `package.json` originalmente utilizava versões antigas das dependências do projeto.

As versões foram atualizadas conforme as versões indicadas no roteiro do projeto:

```json
{
  "dependencies": {
    "lodash": "4.18.1",
    "express": "4.22.3",
    "axios": "1.20.0"
  }
}
```

## Resultado

Após a atualização das dependências e a configuração do Grype, a etapa de SCA passou a fazer parte do fluxo de segurança da pipeline.

Com isso, o projeto passou a possuir três controles distintos de segurança antes do deploy:

- Gitleaks para Secrets Scanning;
- Semgrep para SAST;
- Grype para SCA.

---

# 4. Deploy — GitHub Pages

Após a aprovação das etapas de segurança, a pipeline realiza o deploy da aplicação utilizando o **GitHub Pages**.

## Configuração

O workflow utiliza as actions do GitHub para configurar o ambiente, preparar os arquivos da aplicação e realizar a publicação:

```yaml
- name: 🌐 Configurar GitHub Pages
  uses: actions/configure-pages@v4

- name: 📤 Upload para Produção
  uses: actions/upload-pages-artifact@v3
  with:
    path: './src'

- name: 🚀 Deploy em Produção
  id: deployment
  uses: actions/deploy-pages@v4
```

O diretório `src/` contém os arquivos da aplicação que são disponibilizados pelo GitHub Pages.

## Fluxo de Segurança até a Produção

```text
Código

  ↓

Checkout

  ↓

Build

  ↓

Gitleaks

  ↓

Semgrep

  ↓

Grype

  ↓

Deploy

  ↓

GitHub Pages

  ↓

Produção
```

O deploy ocorre após a execução das etapas anteriores da pipeline.

Dessa forma, os controles de segurança são executados antes da publicação da aplicação.

---

# 5. Problemas Identificados e Correções

Durante a implementação da pipeline, os controles de segurança foram executados sobre o código original do projeto.

As falhas encontradas foram utilizadas para demonstrar o funcionamento dos gates de segurança.

## Gitleaks

**Problema:** informações sensíveis estavam presentes no arquivo `src/script.js`.

**Correção:** remoção das informações sensíveis do código-fonte.

**Resultado:** o Gitleaks passou após a correção.

## Semgrep — `eval()`

**Problema:** utilização da função `eval()` para executar código dinamicamente.

**Correção:** substituição da execução dinâmica por uma chamada direta ao `console.log()`.

**Resultado:** o problema deixou de ser identificado pelo Semgrep.

## Semgrep — `innerHTML`

**Problema:** utilização de `innerHTML` para inserir diretamente conteúdo fornecido pelo usuário.

**Correção:** substituição por criação de elemento e utilização de `innerText`.

**Resultado:** o problema deixou de ser identificado pelo Semgrep.

## Grype

**Problema:** o projeto possuía dependências com versões antigas.

**Correção:** atualização das dependências indicadas no roteiro do projeto e integração do Grype à pipeline.

**Resultado:** o SCA passou a fazer parte do fluxo de segurança antes do deploy.

---

# 6. Resultado Final

Ao final da implementação, a pipeline passou a seguir o seguinte fluxo:

```text
┌──────────────────────┐
│     Código-fonte     │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│       Checkout       │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│        Build         │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│      Gitleaks        │
│  Secrets Scanning    │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│       Semgrep        │
│        SAST          │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│        Grype         │
│         SCA          │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│    GitHub Pages      │
│       Deploy         │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│      Produção        │
└──────────────────────┘
```

## Gates de Segurança

| Gate | Tipo | Objetivo |
|---|---|---|
| Gitleaks | Secrets Scanning | Identificar possíveis segredos expostos |
| Semgrep | SAST | Identificar problemas no código-fonte |
| Grype | SCA | Identificar vulnerabilidades em componentes e dependências |

Os gates são executados antes do deploy.

Quando uma etapa apresenta uma condição de falha, a execução da pipeline é interrompida.

## Situação Final

Ao final do projeto:

- A pipeline de CI/CD foi implementada;
- O Secrets Scanning foi implementado com Gitleaks;
- O SAST foi implementado com Semgrep;
- O SCA foi implementado com Grype;
- Os problemas identificados pelo Gitleaks e Semgrep foram corrigidos;
- As dependências do projeto foram atualizadas;
- A pipeline passou a executar os controles de segurança antes do deploy;
- A aplicação foi publicada utilizando GitHub Pages;
- A aplicação ficou disponível em ambiente de produção.

---

# 7. URL de Produção

A aplicação está disponível em:

**https://emersonsreis.github.io/projeto-devsecop-desafio/**

---

# 8. Conclusão

O projeto demonstrou, na prática, a aplicação dos princípios de DevSecOps ao integrar controles de segurança diretamente ao processo de CI/CD.

A utilização de Gitleaks, Semgrep e Grype permitiu incorporar verificações de segurança ao fluxo de desenvolvimento, fazendo com que problemas identificados pelos gates pudessem interromper a execução da pipeline antes do avanço para as etapas seguintes.

O processo também demonstrou a importância de identificar problemas, realizar as correções necessárias e executar novamente a pipeline para validar as alterações antes da publicação da aplicação.

Dessa forma, a segurança passou a fazer parte do fluxo de desenvolvimento e entrega da aplicação, em vez de ser tratada somente como uma etapa posterior ao desenvolvimento.