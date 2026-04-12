---
layout: post
title: "Core Web Vitals em 2026: Seu Site é Rápido ou Apenas Parece Rápido?"
date: 2026-04-12
description: "Entenda as 3 métricas que o Google usa para medir a experiência real do usuário — LCP, CLS e INP — e como otimizá-las para melhorar ranqueamento e conversão do seu site."
tags:
  [core-web-vitals, performance, seo, lcp, cls, inp, web, ux, google, frontend]
image: /public/images/banner-core-web-vitals.svg
---

Se você trabalha com SEO, UX ou desenvolvimento front-end, já sabe: **o Google não perdoa sites lentos**. Mas velocidade pura não é o único fator que importa. Os **Core Web Vitals (CWV)** vão além — eles medem a **experiência real do usuário**, e em 2026 continuam sendo um dos sinais de ranqueamento mais importantes.

A diferença entre um site que "parece rápido" e um que **é** rápido está em três métricas. Vamos entender cada uma delas, por que importam, e como otimizá-las na prática.

---

## As 3 Métricas que Definem a Experiência

| Métrica | Nome Completo             | O que Mede                  | Meta    |
| ------- | ------------------------- | --------------------------- | ------- |
| **LCP** | Largest Contentful Paint  | Performance de carregamento | < 2,5s  |
| **CLS** | Cumulative Layout Shift   | Estabilidade visual         | < 0,1   |
| **INP** | Interaction to Next Paint | Interatividade              | < 200ms |

Essas três métricas juntas respondem a uma pergunta fundamental: **"O usuário conseguiu usar o site sem frustração?"**

---

## 1. LCP — Largest Contentful Paint

O LCP mede quanto tempo leva para o **maior elemento visível** da viewport aparecer — geralmente um hero banner, imagem principal ou bloco de texto grande.

```
Navegação iniciada
      │
      ▼
  [HTML]  →  [CSS]  →  [Fonts]  →  [Imagem Hero]
                                         │
                                         ▼
                                    ✅ LCP = 1.8s
```

### 🎯 Meta: abaixo de 2,5 segundos

### Técnicas de otimização

- **Otimize imagens**: use formatos modernos como **WebP** ou **AVIF**. Um JPEG de 500KB pode cair para 80KB em AVIF
- **Priorize o above-the-fold**: o conteúdo "acima da dobra" deve carregar primeiro
- **Use `fetchpriority="high"`** na imagem do hero:

```html
<img
  src="/hero-banner.webp"
  alt="Banner principal"
  width="1200"
  height="630"
  fetchpriority="high"
/>
```

- **Preconnect** a origens externas de fontes ou CDNs:

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://cdn.exemplo.com" crossorigin />
```

- **Evite render-blocking**: CSS e JS no `<head>` que bloqueiam renderização são os maiores vilões do LCP

---

## 2. CLS — Cumulative Layout Shift

Sabe quando você vai clicar em um botão e o conteúdo **"pula"**, fazendo você clicar em algo errado? Isso é um CLS ruim. Essa métrica mede a **instabilidade visual** da página — quanto os elementos se movem durante o carregamento.

### 🎯 Meta: menor que 0,1

### Os principais causadores

- ❌ Imagens sem dimensões definidas
- ❌ Anúncios ou embeds que injetam conteúdo dinamicamente
- ❌ Fontes web que causam FOUT (Flash of Unstyled Text)
- ❌ Conteúdo inserido acima do viewport via JavaScript

### Como corrigir

**Sempre defina `width` e `height` em imagens e vídeos:**

```html
<!-- ❌ Ruim — causa layout shift -->
<img src="/foto.webp" alt="Produto" />

<!-- ✅ Bom — o navegador reserva o espaço -->
<img src="/foto.webp" alt="Produto" width="800" height="600" />
```

**Use `aspect-ratio` no CSS para containers dinâmicos:**

```css
.video-container {
  aspect-ratio: 16 / 9;
  width: 100%;
  background: #1a1a2e;
}
```

**Reserve espaço para anúncios com `min-height`:**

```css
.ad-slot {
  min-height: 250px;
  background: var(--surface);
}
```

---

## 3. INP — Interaction to Next Paint

O INP é a métrica mais recente, tendo **substituído o FID (First Input Delay)** em março de 2024. Enquanto o FID media apenas a primeira interação, o INP avalia **todas as interações** ao longo da vida da página — cliques, toques, teclados — e reporta o pior caso (p75).

### 🎯 Meta: abaixo de 200 milissegundos

### O que causa um INP ruim?

O principal vilão é **JavaScript pesado que bloqueia a main thread**. Quando o navegador está executando um script longo, ele não consegue responder ao clique do usuário.

```
  Clique do usuário
        │
        ▼
  [Main Thread ocupada — executando JS...]   ← INP alto 😤
        │
        ▼  (500ms depois)
  Resposta visual
```

### Técnicas de otimização

**Quebre tarefas longas com `scheduler.yield()`:**

```javascript
async function processarLista(items) {
  for (const item of items) {
    processarItem(item);

    // Devolve controle ao browser a cada iteração
    await scheduler.yield();
  }
}
```

**Use `requestIdleCallback` para trabalho não-urgente:**

```javascript
requestIdleCallback(() => {
  // Analytics, pré-carregamento, logs
  enviarAnalytics(dados);
});
```

**Faça code splitting agressivo** — carregue apenas o JS necessário para a página atual:

```javascript
// Next.js / React — import dinâmico
const ModalPesado = dynamic(() => import("./ModalPesado"), {
  loading: () => <Skeleton />,
});
```

---

## Como Medir na Prática

### Dados de campo (usuários reais)

- **Google Search Console** → relatório Core Web Vitals
- **Chrome UX Report (CrUX)** → dados agregados de usuários Chrome
- **`web-vitals`** library no seu código:

```javascript
import { onLCP, onCLS, onINP } from "web-vitals";

onLCP(console.log);
onCLS(console.log);
onINP(console.log);
```

### Dados de laboratório (simulação)

- **Lighthouse** (DevTools → Lighthouse tab)
- **PageSpeed Insights** — [pagespeed.web.dev](https://pagespeed.web.dev)
- **WebPageTest** — [webpagetest.org](https://www.webpagetest.org)

> ⚠️ **Importante**: dados de laboratório são úteis para debug, mas o Google usa **dados de campo** para ranqueamento. Seu Lighthouse pode dar 100, mas se seus usuários reais (CrUX) falham, o ranqueamento sofre.

---

## Checklist Rápido

- [ ] Imagens em WebP/AVIF com `width` e `height` definidos
- [ ] `fetchpriority="high"` no LCP element
- [ ] `<link rel="preconnect">` para origens externas
- [ ] CSS crítico inline, resto assíncrono
- [ ] Fontes com `font-display: swap` e preload
- [ ] Espaço reservado para ads e embeds (`min-height`)
- [ ] Bundle JS com code splitting por rota
- [ ] Nenhuma long task > 50ms na main thread
- [ ] `web-vitals` coletando dados reais em produção
- [ ] Relatório CWV do Search Console sem URLs vermelhas

---

## Por que Focar Nisso Agora?

Os números falam por si:

> Sites que passam nos Core Web Vitals têm **24% menos probabilidade de abandono de página**.

Não é apenas sobre agradar o Google — é sobre **não perder dinheiro por uma experiência ruim**. Cada milissegundo a mais no LCP, cada shift inesperado no layout, cada clique que demora para responder é um usuário a menos na sua conversão.

Em 2026, performance não é diferencial. **É o mínimo.**
