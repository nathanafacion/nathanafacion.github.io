---
layout: post
title: "Spec-Driven Development: Como o Spec Kit Transforma Especificações em Código com Agentes de IA"
date: 2026-06-06
description: "O Spec Kit do GitHub implementa a metodologia SDD para acabar com o vibe coding — transformando especificações ricas em implementações confiáveis com 30+ agentes de IA."
tags:
  [
    spec-kit,
    sdd,
    ia,
    agentes,
    desenvolvimento,
    github,
    copilot,
    especificacao,
    engenharia-de-software,
    workflow,
  ]
image: /public/images/banner-spec-kit-sdd.svg
---

Você já abriu um agente de IA, digitou "cria um app de tarefas com autenticação e banco de dados" e recebeu... algo. Um código que compila, talvez funcione parcialmente, mas que em dois dias de iteração ainda não reflete o que você tinha em mente. Isso tem um nome: **vibe coding** — e é o maior obstáculo para usar IAs de forma profissional em projetos reais. O **Spec Kit**, um toolkit open source do GitHub com mais de 109k stars, propõe uma inversão radical nessa lógica através da metodologia **Spec-Driven Development (SDD)**.

---

## O Problema do Vibe Coding

O vibe coding acontece quando você trata um agente de IA como oráculo: joga um prompt vago e espera que ele "entenda" o que você quer. O problema não é o modelo — é a ausência de contexto estruturado.

Sem especificação clara, o agente opera num vácuo de intenção. Cada iteração é uma tentativa de adivinhar o que você quer. Você corrige, ele regera, você corrige de novo. O código cresce sem coesão, critérios de aceitação nunca foram definidos e refatorações subsequentes desfazem o que o passo anterior fez.

O resultado típico:

- ✅ Código que compila
- ❌ Comportamento que diverge da intenção original
- ❌ Arquitetura sem fundamento técnico explícito
- ❌ Sem rastreabilidade entre decisão e implementação

---

## O que é Spec-Driven Development?

O SDD **inverte o script** do desenvolvimento tradicional. Por décadas, especificações eram andaimes descartáveis — construídos antes do "trabalho real" de codificar começar. O SDD muda isso: **as especificações tornam-se executáveis**, gerando diretamente implementações funcionais em vez de apenas guiá-las.

A filosofia central tem três pilares:

1. **Desenvolvimento orientado a intenção** — especificações definem o "o quê" e o "por quê" antes de qualquer decisão técnica sobre o "como"
2. **Refinamento em múltiplas etapas** — em vez de geração one-shot a partir de prompts, o projeto evolui por fases: requisitos → plano técnico → tasks → implementação
3. **Especificações ricas como artefatos de primeira classe** — não são descartadas após o código ser gerado; continuam sendo a fonte de verdade do projeto

### SDD vs Abordagem Tradicional

| Critério               | Abordagem Tradicional  | Spec-Driven Development                    |
| ---------------------- | ---------------------- | ------------------------------------------ |
| Ponto de partida       | Prompt → código direto | Requisitos → spec → plano → tasks → código |
| Critérios de aceitação | Implícitos ou ausentes | Definidos antes da implementação           |
| Stack tecnológica      | Decidida no prompt     | Definida na fase de planejamento           |
| Rastreabilidade        | Baixa                  | Alta — cada task conecta ao spec           |
| Manutenibilidade       | Depende do agente      | Sustentada pela documentação viva          |
| Paralelismo            | Difícil                | Nativo — tasks marcadas com `[P]`          |

---

## O Spec Kit

O [Spec Kit](https://github.com/github/spec-kit) é a implementação oficial de referência do SDD, criada e mantida pelo GitHub. Com mais de 109k stars, ele fornece um conjunto de prompts estruturados (chamados de **slash commands**) que guiam o agente de IA por cada fase do processo.

O Spec Kit é **independente de tecnologia** — funciona com qualquer linguagem, framework ou plataforma — e suporta mais de **30 agentes de IA**, incluindo GitHub Copilot, Claude Code, Gemini CLI, Cursor, Codex CLI, e muitos outros.

### Instalação

```bash
# Instala o Specify CLI via uv
uv tool install specify-cli

# Inicializa um novo projeto com integração ao GitHub Copilot
specify init meu-projeto --integration copilot
```

---

## O Fluxo Completo em 7 Etapas

### Etapa 1 — Constitution: os princípios do projeto

```
/speckit.constitution
```

Antes de qualquer linha de código ou requisito, você estabelece os **princípios e guardrails** do projeto. Isso cria o arquivo `.specify/memory/constitution.md` — uma memória persistente que o agente consulta em todas as etapas subsequentes.

A constitution responde perguntas como: qual é o estilo de código preferido? Quais padrões de segurança devem ser seguidos? Quais decisões de arquitetura são inegociáveis?

### Etapa 2 — Specify: os requisitos

```
/speckit.specify
```

Aqui você descreve **o que precisa ser construído e por quê**. O ponto-chave: **não mencione stack tecnológica ainda**. Foque em user stories, comportamentos esperados e critérios de aceitação. O arquivo gerado é `specs/001-meu-projeto/spec.md`.

Exemplo de fragmento de spec:

```markdown
## User Stories

- Como usuário autenticado, quero criar tarefas com título, descrição e data de vencimento
- Como usuário, quero filtrar tarefas por status (pendente, em progresso, concluída)
- Como administrador, quero visualizar métricas de produtividade por usuário

## Critérios de Aceitação

- Tarefas devem persistir entre sessões
- Criação de tarefa deve responder em menos de 200ms
- Filtros devem ser aplicados sem recarregar a página
```

### Etapa 3 — Plan: o plano técnico

```
/speckit.plan
```

Com os requisitos definidos, agora sim você define a **stack, a arquitetura e as decisões técnicas**. O agente analisa o spec e propõe um plano técnico coerente com as restrições estabelecidas na constitution. Gera `specs/001-meu-projeto/plan.md`.

### Etapa 4 — Tasks: o breakdown de trabalho

```
/speckit.tasks
```

O agente converte o plano técnico numa lista de **tasks atômicas e acionáveis**, com dependências explícitas e marcação de paralelismo. O arquivo `specs/001-meu-projeto/tasks.md` terá uma estrutura como:

```markdown
## Tasks

- [ ] [1] Configurar projeto Next.js com TypeScript
- [ ] [2] [P] Implementar schema do banco de dados (Prisma)
- [ ] [2] [P] Configurar autenticação NextAuth.js
- [ ] [3] Criar endpoints de API para tarefas
- [ ] [4] [P] Implementar componente TaskList
- [ ] [4] [P] Implementar componente TaskForm
- [ ] [5] Integrar frontend com API
- [ ] [6] Escrever testes de integração
```

O `[P]` indica tasks que podem ser executadas **em paralelo** — algo que agentes de IA como o GitHub Copilot Agent aproveitam diretamente para acelerar a implementação.

### Etapa 5 — Implement: a execução

```
/speckit.implement
```

Com toda a estrutura definida, o agente executa as tasks sequencialmente (respeitando dependências e paralelismo). A diferença crucial aqui: o agente não está "tentando adivinhar" — ele tem um plano técnico, critérios de aceitação e o contexto da constitution para guiar cada decisão.

### Etapas Opcionais

**Clarificação** — antes de especificar, identifica ambiguidades:

```
/speckit.clarify
```

**Análise de consistência** — verifica se spec, plan e tasks estão alinhados:

```
/speckit.analyze
```

**Integração com GitHub Issues** — converte tasks em issues rastreáveis:

```
/speckit.taskstoissues
```

**Checklists de qualidade** — gera listas de verificação específicas:

```
/speckit.checklist
```

---

## Estrutura de Arquivos Gerada

Após executar o fluxo completo, seu projeto terá:

```
.
├── .specify/
│   ├── memory/
│   │   └── constitution.md      # Princípios e guardrails
│   ├── scripts/
│   └── templates/
│       ├── plan-template.md
│       ├── spec-template.md
│       └── tasks-template.md
└── specs/
    └── 001-minha-feature/
        ├── contracts/
        │   ├── api-spec.json    # Contrato da API
        │   └── signalr-spec.md
        ├── data-model.md        # Modelagem de dados
        ├── plan.md              # Plano técnico
        ├── quickstart.md        # Guia de início rápido
        ├── research.md          # Pesquisa e contexto
        └── spec.md              # Requisitos e user stories
```

Esses artefatos não são descartáveis. Eles continuam sendo a **fonte de verdade** do projeto — úteis para onboarding, auditorias, code reviews e futuras iterações.

---

## Fases de Desenvolvimento Suportadas

O Spec Kit não é apenas para projetos novos. Ele suporta três fases distintas:

| Fase                                   | Cenário           | Foco                                       |
| -------------------------------------- | ----------------- | ------------------------------------------ |
| **0-to-1 (Greenfield)**                | Projeto do zero   | Spec completa, liberdade total de stack    |
| **Creative Exploration**               | Prototipagem      | Implementações paralelas, múltiplas stacks |
| **Iterative Enhancement (Brownfield)** | Sistema existente | Modernização incremental, compatibilidade  |

Para projetos brownfield, a constitution é especialmente poderosa — você codifica as restrições do sistema legado, padrões existentes e decisões que não podem ser revertidas.

---

## Extensions e Presets

O Spec Kit é extensível. Você pode customizar o workflow para o contexto da sua organização:

- **Extensions**: adicionam novas capacidades — integração com Jira, processos de code review customizados, V-Model para sistemas críticos
- **Presets**: customizam workflows existentes — templates de spec da sua empresa, terminologia do domínio, padrões organizacionais

A hierarquia de prioridade garante que customizações locais sempre prevaleçam:

```
1. Project-Local Overrides  (.specify/templates/overrides/)
2. Presets                  (.specify/presets/templates/)
3. Extensions               (.specify/extensions/templates/)
4. Spec Kit Core            (.specify/templates/)
```

---

## Por Que Isso Importa para Desenvolvedores React

Se você trabalha com React e usa ferramentas como GitHub Copilot no dia a dia, o SDD muda a qualidade das entregas de forma concreta.

Em vez de pedir "cria um hook de autenticação", você define primeiro o comportamento esperado, os edge cases, os critérios de aceitação — e só então o agente implementa. O resultado é um hook que:

- 🧠 Reflete a intenção real, não a interpretação do modelo
- ✅ Tem critérios de aceitação verificáveis
- 📦 Encaixa na arquitetura do projeto (definida no plan)
- 🔄 Pode ser iterado com contexto preservado

O SDD é especialmente valioso em times — onde múltiplas pessoas e agentes precisam trabalhar a partir da mesma fonte de verdade.

---

## Conclusão

O Spec Kit representa uma mudança de paradigma na forma como usamos agentes de IA para desenvolvimento. Em vez de tratar a IA como um oráculo de código, o SDD a posiciona como um executor de especificações — muito mais confiável, auditável e colaborativo.

Com a evolução dos modelos e a proliferação de agentes no workflow de desenvolvimento, a habilidade de **escrever boas especificações** vai se tornar tão fundamental quanto a habilidade de escrever bom código. Provavelmente mais.

Se você ainda não experimentou o Spec Kit, o repositório oficial está em [github.com/github/spec-kit](https://github.com/github/spec-kit). Comece com um projeto pequeno — uma feature nova, um módulo isolado — e observe a diferença entre código gerado por prompt e código gerado por spec.

O vibe coding tem prazo de validade. O SDD é o próximo passo.
