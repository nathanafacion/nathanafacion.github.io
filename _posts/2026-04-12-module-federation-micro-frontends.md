---
layout: post
title: "Module Federation: Compartilhando Código Entre Micro-Frontends em Tempo Real"
date: 2026-04-12
description: "Entenda como o Module Federation do Webpack 5 revolucionou a arquitetura de micro-frontends, permitindo compartilhar componentes entre aplicações sem NPM install ou rebuild."
tags:
  [
    module-federation,
    webpack,
    micro-frontends,
    react,
    arquitetura,
    frontend,
    javascript,
    design-system,
    performance,
    escalabilidade,
  ]
image: /public/images/banner-module-federation.svg
---

Imagine que sua empresa tem 5 times, cada um responsável por uma parte do produto: checkout, catálogo, dashboard, onboarding e perfil. Cada time mantém sua própria aplicação React. Um dia, o time de design atualiza o `<BotaoComprar />`. Antes do Module Federation, **todos os 5 times** precisariam atualizar a dependência no NPM, rodar `npm install` e fazer um novo build. Com Module Federation, a atualização é **instantânea** — sem tocar no código dos outros projetos.

Essa é a promessa do **Module Federation**, uma das funcionalidades mais poderosas introduzidas no Webpack 5.

---

## O que é Module Federation?

Module Federation permite que **diferentes aplicações JavaScript compartilhem código de forma dinâmica em tempo de execução**. Sem NPM, sem rebuild, sem deploy conjunto.

Uma aplicação pode **expor** partes do seu código (componentes, funções, estados) e outra aplicação pode **consumir** esse código diretamente — carregado do servidor de origem no momento em que o usuário acessa o site.

```
┌──────────────────┐          ┌──────────────────┐
│   App Principal   │  ◄────  │   App Vendas      │
│     (Host)        │  HTTP   │    (Remote)       │
│                   │         │                   │
│  Consome:         │         │  Expõe:           │
│  <BotaoComprar /> │         │  ./BotaoComprar   │
│  <CardProduto />  │         │  ./CardProduto    │
└──────────────────┘          └──────────────────┘
```

---

## Conceitos Chave

| Conceito    | Papel        | Descrição                                                         |
| ----------- | ------------ | ----------------------------------------------------------------- |
| **Host**    | Consumidor   | A aplicação "mestre" que recebe os módulos remotos                |
| **Remote**  | Fornecedor   | A aplicação que expõe módulos para outros consumirem              |
| **Exposes** | Configuração | A lista de arquivos que o Remote decide compartilhar              |
| **Remotes** | Configuração | Os endereços que o Host usa para buscar código externo            |
| **Shared**  | Otimização   | Dependências compartilhadas (ex: React) carregadas apenas uma vez |

---

## Por que é Revolucionário?

O Module Federation resolve os 3 maiores problemas de arquiteturas micro-frontend:

### 1. Independência de Deploy

Você atualiza o `<Menu />` no Projeto A e ele é automaticamente atualizado no Projeto B (o Host) — **sem mexer no código do Projeto B**. Cada time faz deploy no seu próprio ritmo.

### 2. Compartilhamento Inteligente de Dependências

Se o Host e o Remote usam React, o Module Federation é inteligente o suficiente para carregá-lo **apenas uma vez**. Nada de duplicar 40KB de React em cada micro-app.

### 3. Performance Real

Diferente de iframes (pesados e isolados), o Module Federation compartilha o **mesmo contexto de execução** do JavaScript. Isso significa:

- Comunicação direta entre componentes
- Estado compartilhado (Context, Redux)
- Zero overhead de postMessage

---

## Configuração na Prática

Toda a mágica acontece no `webpack.config.js` com o plugin `ModuleFederationPlugin`.

### Remote: quem compartilha

```javascript
// webpack.config.js do Remote (App Vendas)
const { ModuleFederationPlugin } = require("webpack").container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "app_vendas",
      filename: "remoteEntry.js",
      exposes: {
        "./BotaoComprar": "./src/components/BotaoComprar",
        "./CardProduto": "./src/components/CardProduto",
      },
      shared: {
        react: { singleton: true, requiredVersion: "^18.0.0" },
        "react-dom": { singleton: true, requiredVersion: "^18.0.0" },
      },
    }),
  ],
};
```

O `remoteEntry.js` gerado é o **manifesto** — ele lista tudo que está disponível e como carregar cada módulo.

### Host: quem consome

```javascript
// webpack.config.js do Host (App Principal)
const { ModuleFederationPlugin } = require("webpack").container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "app_principal",
      remotes: {
        vendas: "app_vendas@http://localhost:3001/remoteEntry.js",
      },
      shared: {
        react: { singleton: true, requiredVersion: "^18.0.0" },
        "react-dom": { singleton: true, requiredVersion: "^18.0.0" },
      },
    }),
  ],
};
```

### Usando o componente remoto

No código do Host, o import funciona como se fosse um componente local:

```javascript
import React, { Suspense } from "react";

// Import dinâmico do componente remoto
const BotaoComprar = React.lazy(() => import("vendas/BotaoComprar"));

function PaginaProduto({ produto }) {
  return (
    <div>
      <h1>{produto.nome}</h1>
      <p>{produto.descricao}</p>

      <Suspense fallback={<span>Carregando...</span>}>
        <BotaoComprar produtoId={produto.id} />
      </Suspense>
    </div>
  );
}
```

O `React.lazy` + `Suspense` garante que o componente remoto é carregado **sob demanda**, sem bloquear a renderização.

---

## NPM vs Module Federation

| Aspecto                 | NPM / Bibliotecas                 | Module Federation                        |
| ----------------------- | --------------------------------- | ---------------------------------------- |
| **Atualização**         | Requer `npm install` e novo build | Instantânea (runtime)                    |
| **Tamanho do bundle**   | Pode duplicar código              | Compartilha dependências automaticamente |
| **Autonomia dos times** | Dependem da versão do pacote      | Entregam features direto ao usuário      |
| **Versionamento**       | Semver, lock files                | Sempre a versão mais recente (ou config) |
| **Tempo de build**      | Cresce com o monolito             | Cada app builda independentemente        |

---

## Quando Usar?

Module Federation **não é necessário para todos os projetos**. Ele brilha em cenários específicos:

✅ **Empresas grandes** — onde vários times cuidam de partes diferentes da mesma plataforma

✅ **Design Systems** — para garantir que todos os micro-apps usem a versão mais recente dos componentes

✅ **Aplicações escaláveis** — onde o tempo de build de um monolito está ficando insustentável

✅ **Migração gradual** — quando você quer migrar de Angular para React (ou vice-versa) sem reescrever tudo de uma vez

### Quando **não** usar

❌ Projetos pequenos com um único time — a complexidade não compensa

❌ Sites estáticos simples — sem necessidade de compartilhamento dinâmico

❌ Quando não há infraestrutura para múltiplos deploys independentes

---

## O Ecossistema em 2026

O Module Federation evoluiu bastante desde o Webpack 5:

- **Module Federation 2.0** — suporte nativo no Rspack, melhor TypeScript support
- **@module-federation/enhanced** — plugin com runtime compartilhado, retry e fallback
- **Vite + Module Federation** — usando `@originjs/vite-plugin-federation`
- **Next.js** — suporte experimental via `@module-federation/nextjs-mf`

A tendência é que o conceito se torne **agnóstico de bundler**, funcionando nativamente em qualquer toolchain moderno.

---

## Conclusão

O Module Federation transformou micro-frontends de "arquitetura de conferência" para **arquitetura de produção**. A capacidade de compartilhar código entre aplicações em tempo de execução — sem builds, sem NPM, sem deploys acoplados — é genuinamente revolucionária.

Se seu time está crescendo e o monolito está ficando pesado, vale explorar essa abordagem. O custo de entrada diminuiu muito, e os benefícios em autonomia e velocidade de entrega são reais.

> O melhor componente não é o mais reutilizável — é o que está sempre atualizado em todos os lugares ao mesmo tempo.
