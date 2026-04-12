---
layout: post
title: "Construindo uma Busca Inteligente de Carros com RAG, Neo4j e Next.js"
date: 2026-04-10
description: "Aprenda a construir do zero uma aplicação de busca por similaridade semântica usando RAG (Retrieval-Augmented Generation) com Neo4j, embeddings vetoriais e GPT-4o-mini. Tutorial completo com Next.js, TypeScript e Docker."
tags:
  [
    rag,
    neo4j,
    nextjs,
    ia,
    embeddings,
    typescript,
    openrouter,
    docker,
    llm,
    busca-vetorial,
  ]
image: /public/images/banner-rag-neo4j.svg
og_image: /public/images/banner-rag-neo4j.png
---

Imagine digitar **"carro vermelho dos anos 90"** e o sistema entender exatamente o que você quer — sem precisar preencher filtros de marca, ano, cor, ou combustível. Isso é **busca por similaridade semântica**, e neste post vou mostrar como construí essa aplicação do zero usando **RAG (Retrieval-Augmented Generation)**.

## O que é RAG e por que usar?

**RAG** é um padrão de arquitetura de IA que combina três etapas:

| Etapa          | Sigla | O que faz                                                          |
| -------------- | ----- | ------------------------------------------------------------------ |
| **Retrieval**  | R     | Busca vetorial — encontra os dados reais mais similares à pergunta |
| **Augmented**  | A     | Injeta os dados recuperados no contexto do LLM                     |
| **Generation** | G     | O LLM gera uma resposta natural baseada nos dados reais            |

A grande sacada: o LLM **não inventa dados**. Ele só interpreta e comunica o que o banco de dados já encontrou. Isso elimina alucinações e garante respostas baseadas em fatos.

## O pipeline completo

```
Usuário digita query ("SUV diesel potente")
        │
        ▼
[R] OpenRouter — text-embedding-3-small
        │  gera vetor de 1536 dimensões
        ▼
[R] Neo4j — busca vetorial (similaridade coseno)
        │  retorna top-6 carros do banco
        ▼
[A] Prompt aumentado com contexto dos carros
        │
        ▼
[G] OpenRouter — gpt-4o-mini
        │  resposta em linguagem natural
        ▼
Interface Next.js — resposta IA + cards dos carros
```

## Stack tecnológica

- **Next.js 16** (App Router, Server Components, React Compiler)
- **Neo4j 5.18** — banco de grafos com índice vetorial nativo
- **OpenRouter** — embeddings (`text-embedding-3-small`) + chat (`gpt-4o-mini`)
- **Docker** — container Neo4j com plugin APOC
- **TypeScript** — tipagem completa em todo o projeto
- **Tailwind CSS v4** — estilização

## Arquitetura do projeto

Usei **Screaming Architecture** — as pastas gritam o que a aplicação faz:

```
src/
├── app/
│   ├── page.tsx              # Server Component
│   └── api/
│       ├── seed/route.ts     # Popula Neo4j com embeddings
│       └── search/route.ts   # Pipeline RAG completo
│
├── services/                 # Camada de serviços (sem acoplamento)
│   ├── neo4j.ts              # Driver singleton + runQuery<T>
│   ├── embeddings.ts         # Geração de embeddings via OpenRouter
│   └── llm.ts                # Geração de resposta RAG
│
├── features/
│   └── cars/
│       ├── types.ts
│       └── components/
│           ├── CarCard/      # Card de resultado com barra de similaridade
│           └── CarSearch/    # Orquestrador principal (UI lean ~45 linhas)
│
└── lib/
    └── cars-data.ts          # Dataset de 60 carros (1990–2024)
```

Cada componente vive em sua própria pasta com `types.ts`, `constants.ts` e hooks separados (**folder-per-component**).

---

## Passo 1: O banco de dados — Neo4j com índice vetorial

Primeiro, subimos o Neo4j via Docker:

```yaml
# docker-compose.yml
services:
  neo4j:
    image: neo4j:5.18.0
    container_name: neo4j-cars
    ports:
      - "7474:7474" # Browser
      - "7687:7687" # Bolt
    environment:
      - NEO4J_AUTH=neo4j/password123
      - NEO4J_PLUGINS=["apoc"]
      - NEO4J_server_memory_heap_max__size=1G
    volumes:
      - neo4j_data:/data
```

Depois criamos o **índice vetorial** — o coração da busca por similaridade:

```typescript
// API Route: /api/seed
await runQuery(`
  CREATE VECTOR INDEX car_embeddings IF NOT EXISTS
  FOR (c:Car) ON (c.embedding)
  OPTIONS {
    indexConfig: {
      \`vector.dimensions\`: 1536,
      \`vector.similarity_function\`: 'cosine'
    }
  }
`);
```

O Neo4j 5 tem **suporte nativo a busca vetorial** — não precisa de extensão extra. Configuramos 1536 dimensões (igual ao modelo de embedding) e similaridade coseno.

## Passo 2: O dataset — 60 carros com descrições para embedding

Cada carro é transformado em uma **descrição textual rica** que será convertida em vetor:

```typescript
export function buildCarDescription(car: Car): string {
  return (
    `${car.ano} ${car.marca} ${car.modelo} — ${car.tipo}, cor ${car.cor}. ` +
    `Motor ${car.motor} ${car.combustivel}, câmbio ${car.cambio}, ${car.potencia} cv. ` +
    `Carro ${car.tipo.toLowerCase()} ${
      car.ano <= 1995
        ? "clássico e vintage"
        : car.ano <= 2015
          ? "moderno"
          : "recente e atual"
    } ideal para ${
      car.tipo === "SUV"
        ? "família e aventura off-road"
        : car.tipo === "esportivo"
          ? "performance e dirigibilidade esportiva"
          : car.tipo === "pickup"
            ? "trabalho, campo e carga"
            : "uso urbano e dia a dia"
    }.`
  );
}
```

Essa função é crucial. A qualidade do embedding depende diretamente da qualidade do texto. Incluímos:

- **Atributos factuais**: marca, modelo, ano, motor, potência
- **Contexto semântico**: "clássico e vintage", "ideal para família e aventura off-road"
- **Sinônimos implícitos**: alguém que busca "carro para campo" vai bater com "aventura off-road"

O dataset cobre de **Gol 1990** a **Tesla Model 3 2022** e **BYD Dolphin 2024**, com variedade de tipos, combustíveis e faixas de preço.

## Passo 3: Gerando embeddings em lote

Na hora de popular o banco, geramos embeddings em **lotes de 20** para não sobrecarregar a API:

```typescript
// services/embeddings.ts
export async function generateEmbeddingsBatch(
  texts: string[],
): Promise<number[][]> {
  const response = await fetch(`${OPENROUTER_BASE_URL}/embeddings`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: texts,
    }),
  });

  const data = await response.json();
  // Garante que a ordem corresponda ao input
  const sorted = data.data.sort((a, b) => a.index - b.index);
  return sorted.map((item) => item.embedding);
}
```

Cada texto vira um **vetor de 1536 números decimais**. A mágica dos embeddings é que textos com significado semelhante ficam próximos no espaço vetorial.

## Passo 4: A busca vetorial no Neo4j

Quando o usuário faz uma busca, transformamos a query em embedding e usamos o índice vetorial do Neo4j:

```typescript
// API Route: /api/search
export async function POST(req: NextRequest) {
  const { query, topK = 5 } = await req.json();

  // 1. Gerar embedding da busca
  const queryEmbedding = await generateEmbedding(query);

  // 2. Busca vetorial por similaridade
  const results = await runQuery(
    `
    CALL db.index.vector.queryNodes('car_embeddings', $topK, $queryEmbedding)
    YIELD node, score
    RETURN node, score
    ORDER BY score DESC
  `,
    { topK, queryEmbedding },
  );

  // 3. Geração RAG (falha silenciosa — cards aparecem mesmo sem LLM)
  let aiResponse = null;
  try {
    aiResponse = await generateRagResponse(query, cars);
  } catch {
    /* os cards funcionam independente */
  }

  return NextResponse.json({ results: cars, aiResponse });
}
```

Repare na **decisão arquitetural**: se o LLM falhar, os cards dos carros ainda aparecem. O **Retrieval não depende do Generation**.

## Passo 5: A resposta do LLM — geração contextualizada

O LLM recebe um prompt com os carros recuperados e gera uma análise:

```typescript
// services/llm.ts
const context = cars
  .map(
    (c, i) =>
      `${i + 1}. ${c.marca} ${c.modelo} ${c.ano} — ${c.tipo} ${c.cor}, ` +
      `${c.combustivel}, motor ${c.motor} (${c.potencia}cv), ` +
      `R$ ${c.preco.toLocaleString("pt-BR")} | similaridade: ${Math.round(c.score * 100)}%`,
  )
  .join("\n");

const systemPrompt =
  "Você é um consultor especialista em veículos automotores brasileiro. " +
  "Analise os resultados e responda de forma natural e prestativa. " +
  "Destaque o melhor match e aponte as melhores opções. " +
  "Limite-se a 3-4 frases.";
```

O LLM não tem acesso ao banco de dados — ele só vê os carros que o Neo4j já filtrou. Isso é o **"Augmented"** do RAG.

## Passo 6: A camada de serviços — desacoplamento

O driver do Neo4j usa o padrão **singleton** para não criar conexões a cada request:

```typescript
// services/neo4j.ts
let driver: Driver | null = null;

export function getDriver(): Driver {
  if (!driver) {
    driver = neo4j.driver(uri, neo4j.auth.basic(user, password));
  }
  return driver;
}

export async function runQuery<T>(cypher: string, params = {}): Promise<T[]> {
  const session = getDriver().session();
  try {
    const result = await session.run(cypher, params);
    return result.records.map((r) => r.toObject() as T);
  } finally {
    await session.close();
  }
}
```

A sessão é **sempre fechada no finally** — mesmo se der erro. Isso evita vazamento de conexão.

## Passo 7: O frontend — Server Components + hooks

A `page.tsx` é um **Server Component** — ela consulta o Neo4j no servidor para saber se o banco já está populado:

```typescript
// app/page.tsx
export default async function HomePage() {
  const isSeeded = await getIsSeeded(); // roda no servidor
  return <CarSearch initialSeeded={isSeeded} />;
}
```

O componente `CarSearch` é **lean**: apenas ~45 linhas de JSX. Toda a lógica está no hook `useCarSearch`:

```typescript
// useCarSearch.ts (resumo)
export function useCarSearch(initialSeeded: boolean) {
  const [query, setQuery] = useState("");
  const [status, setStatus] = useState<SearchStatus>("idle");
  const [results, setResults] = useState<CarResult[]>([]);
  const [aiResponse, setAiResponse] = useState<string | null>(null);

  async function handleSearch(e?: React.FormEvent) {
    e?.preventDefault();
    setStatus("loading");
    const res = await fetch("/api/search", {
      method: "POST",
      body: JSON.stringify({ query, topK: 6 }),
    });
    const data = await res.json();
    setResults(data.results);
    setAiResponse(data.aiResponse);
    setStatus("done");
  }

  return { query, setQuery, status, results, aiResponse, handleSearch };
}
```

## Por que busca vetorial é melhor que filtros tradicionais?

| Busca tradicional                          | Busca por similaridade                  |
| ------------------------------------------ | --------------------------------------- |
| Precisa de palavras exatas                 | Entende sinônimos e contexto            |
| `"vermelho"` não encontra `"cor vermelha"` | Encontra carros semanticamente próximos |
| Filtros rígidos por campo                  | Busca em qualquer dimensão semântica    |
| Não entende intenção                       | `"bom para família"` → SUV espaçoso     |

## Como rodar o projeto

```bash
# 1. Subir Neo4j
docker-compose up -d

# 2. Configurar .env.local
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password123
OPENROUTER_API_KEY=sua-chave-aqui

# 3. Instalar e rodar
yarn install
yarn dev

# 4. Na interface, clique em "Popular Banco" (gera embeddings, ~2-3 min)
```

## Decisões de arquitetura que importam

1. **Screaming Architecture** — pastas organizadas por domínio (`features/cars`), não por tipo de arquivo
2. **Folder-per-component** — cada componente em pasta própria com `types.ts`, `constants.ts` e hooks
3. **Falha silenciosa no LLM** — se o gpt-4o-mini cair, os cards ainda aparecem
4. **Server Components** — a verificação de seed roda no servidor, sem loading state no cliente
5. **Singleton do driver** — evita criar conexões desnecessárias com o Neo4j

## Conclusão

Com **~500 linhas de código** conseguimos montar um pipeline RAG completo:

- **Embedding** de dados em 1536 dimensões
- **Busca vetorial** nativa no Neo4j com similaridade coseno
- **Geração contextualizada** com GPT-4o-mini
- **Interface moderna** com Next.js 16 e Tailwind

O código completo está no [GitHub](https://github.com/nathanafacion/IA_APLICADA). Fique à vontade para clonar, experimentar e adaptar para seu próprio dataset!

---

_Tem dúvidas ou quer ver outro projeto explicado? Me encontra no [GitHub](https://github.com/nathanafacion)._
