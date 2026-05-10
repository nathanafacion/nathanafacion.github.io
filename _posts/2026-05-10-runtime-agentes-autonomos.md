---
layout: post
title: "Por Dentro do Runtime: Como um Agente Autônomo Realmente Funciona"
date: 2026-05-10
description: "Uma análise técnica completa de um runtime de agentes autônomos: 9 contratos YAML, 7 módulos Python, circuit breaker, telemetria estruturada, e todos os padrões de engenharia de software que fazem um agente ser um produto auditável."
tags:
  [
    agentes,
    python,
    llm,
    engenharia-de-software,
    arquitetura,
    openai,
    telemetria,
    circuit-breaker,
    spec-driven,
    runtime,
    ia,
    contratos,
  ]
image: /public/images/banner-runtime-agentes.svg
---

Você define nove arquivos Markdown com blocos YAML. O agente **roda**. Mas o que acontece entre o momento em que você digita `python main.py rodar` e o `trace.json` aparecer na pasta?

Este post abre o motor.

O runtime não sabe nada sobre monitoramento, incidentes ou latência. Ele sabe ler contratos e executar. Cada campo YAML declarado tem uma linha Python que o lê. Cada decisão de segurança declarada tem um guardrail que a aplica.

---

## A filosofia: spec-driven development

Antes de entrar no código, o princípio de design mais importante:

> **Markdown é documentação. YAML é máquina. O bloco YAML dentro do `.md` é o que vai para o runtime.**

O padrão usado aqui tem nome: **spec-driven development**. A especificação — escrita em formato legível por humanos — dirige o comportamento. O código não tem opinião sobre o domínio. Ele é um intérprete genérico de contratos.

Isso tem consequências profundas:

- Você pode **criar um agente novo sem escrever Python** — só contratos
- O mesmo runtime executa um agente de monitoramento, um de análise de documentos, um de suporte ao cliente
- As regras de negócio ficam nos contratos, não enterradas no código
- Auditoria é simples: o comportamento está declarado, não inferido

A estrutura é:

```
aula04-runtime/
├── monitor-agent/        ← 9 contratos do agente
│   ├── agent.md
│   ├── rules.md
│   ├── skills.md
│   ├── hooks.md
│   ├── memory.md
│   └── contracts/
│       ├── loop.md
│       ├── planner.md
│       ├── executor.md
│       └── toolbox.md
└── runtime/              ← 7 módulos Python genéricos
    ├── main.py
    ├── contratos.py
    ├── ciclo.py
    ├── planejador.py
    ├── ferramentas.py
    ├── executor.py
    ├── telemetria.py
    └── validador.py
```

Nove contratos descrevem **o quê**. Sete módulos implementam **o como**.

<figure style="margin: 2rem 0; text-align: center;">
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 320" style="max-width:820px;width:100%;">
    <rect width="820" height="320" rx="12" fill="#0d1b2a"/>
    <!-- Contratos -->
    <rect x="20" y="20" width="360" height="280" rx="10" fill="#0a1520" stroke="#00ff88" stroke-width="1.5"/>
    <text x="200" y="50" text-anchor="middle" font-family="monospace" font-size="13" fill="#00ff88" font-weight="bold">9 CONTRATOS (Markdown + YAML)</text>
    <text x="40" y="80" font-family="monospace" font-size="11" fill="#4a9a6a">📄 agent.md       — identidade</text>
    <text x="40" y="100" font-family="monospace" font-size="11" fill="#4a9a6a">📄 rules.md        — limites e políticas</text>
    <text x="40" y="120" font-family="monospace" font-size="11" fill="#4a9a6a">📄 skills.md       — interfaces das ferramentas</text>
    <text x="40" y="140" font-family="monospace" font-size="11" fill="#4a9a6a">📄 hooks.md        — pontos de observação</text>
    <text x="40" y="160" font-family="monospace" font-size="11" fill="#4a9a6a">📄 memory.md       — o que guardar</text>
    <text x="40" y="185" font-family="monospace" font-size="10" fill="#336655">  contracts/</text>
    <text x="40" y="205" font-family="monospace" font-size="11" fill="#4a9a6a">📄 loop.md         — ciclo e condições de parada</text>
    <text x="40" y="225" font-family="monospace" font-size="11" fill="#4a9a6a">📄 planner.md      — formato de resposta da LLM</text>
    <text x="40" y="245" font-family="monospace" font-size="11" fill="#4a9a6a">📄 executor.md     — retry e validação</text>
    <text x="40" y="265" font-family="monospace" font-size="11" fill="#4a9a6a">📄 toolbox.md      — registro de ferramentas</text>
    <text x="200" y="295" text-anchor="middle" font-family="monospace" font-size="10" fill="#336655">definem O QUÊ</text>
    <!-- Seta -->
    <line x1="385" y1="160" x2="440" y2="160" stroke="#00ccff" stroke-width="2"/>
    <polygon points="438,154 450,160 438,166" fill="#00ccff"/>
    <!-- Módulos -->
    <rect x="455" y="20" width="345" height="280" rx="10" fill="#0a1520" stroke="#00ccff" stroke-width="1.5"/>
    <text x="628" y="50" text-anchor="middle" font-family="monospace" font-size="13" fill="#00ccff" font-weight="bold">7 MÓDULOS PYTHON (Runtime)</text>
    <text x="475" y="80" font-family="monospace" font-size="11" fill="#3a7a9a">🐍 contratos.py    — carrega os 9 .md</text>
    <text x="475" y="100" font-family="monospace" font-size="11" fill="#3a7a9a">🐍 ciclo.py        — loop principal</text>
    <text x="475" y="120" font-family="monospace" font-size="11" fill="#3a7a9a">🐍 planejador.py   — percepção e LLM</text>
    <text x="475" y="140" font-family="monospace" font-size="11" fill="#3a7a9a">🐍 ferramentas.py  — gera implementações</text>
    <text x="475" y="160" font-family="monospace" font-size="11" fill="#3a7a9a">🐍 executor.py     — executa, avalia, hooks</text>
    <text x="475" y="180" font-family="monospace" font-size="11" fill="#3a7a9a">🐍 telemetria.py   — trace e métricas</text>
    <text x="475" y="200" font-family="monospace" font-size="11" fill="#3a7a9a">🐍 validador.py    — consistência</text>
    <text x="475" y="230" font-family="monospace" font-size="11" fill="#3a7a9a">🐍 main.py         — CLI (rodar/validar/replay)</text>
    <text x="628" y="295" text-anchor="middle" font-family="monospace" font-size="10" fill="#2a5a7a">implementam O COMO</text>
  </svg>
  <figcaption style="font-size:0.85rem;color:#888;margin-top:0.5rem;">Os 9 contratos definem o comportamento. Os 7 módulos Python interpretam e executam.</figcaption>
</figure>

---

## O monitor-agent: o agente de exemplo

Antes de entrar nos módulos Python, vale entender o agente que estamos executando. O `monitor-agent` é um agente de monitoramento e diagnóstico de incidentes de produção. Ele não tem lógica de domínio em Python — toda a sua identidade, regras e capacidades estão nos contratos.

### agent.md — identidade

```yaml
nome: monitor-agent
descricao: agente de monitoramento e diagnostico de incidentes de producao
tipo: task_based

objetivo: resolver_incidente

contrato_saida:
  formato: json
  campos_obrigatorios:
    - diagnostico
    - evidencias
    - recomendacao
    - severidade
```

O campo `tipo` define o modo de operação: `task_based` (executa e entrega), `interactive` (pergunta antes de agir), `goal_oriented` (decompõe objetivos) ou `autonomous` (reage a eventos). O `contrato_saida` define o que o agente **promete entregar** — e o runtime pode usar isso para verificar se a promessa foi cumprida.

### rules.md — as leis do agente

```yaml
ferramentas_obrigatorias:
  - relatorio_incidente

limites:
  max_etapas: 10
  sem_progresso: 3
  limite_tempo_segundos: 120
  chamadas_ferramenta:
    consultar_metricas: 3
    buscar_logs: 3
    historico_deploys: 2
    relatorio_incidente: 1
    total: 9

acoes_sensiveis:
  - rollback_deploy

politicas:
  - parar se nao houver progresso apos 3 tentativas consecutivas
  - relatorio_incidente e obrigatorio antes de finalizar
  - relatorio_incidente so pode ser chamado apos coletar evidencias
  - os argumentos evidencia e recomendacao do relatorio_incidente devem conter dados reais coletados
  - rollback requer confirmacao humana
```

Este contrato é o **guardião do sistema**. Cada linha tem uma função:

| Campo                                  | O que protege                                                              |
| -------------------------------------- | -------------------------------------------------------------------------- |
| `ferramentas_obrigatorias`             | Impede que o agente finalize sem chamar `relatorio_incidente`              |
| `limites.max_etapas: 10`               | Trava de segurança contra loop infinito                                    |
| `limites.sem_progresso: 3`             | Detecta estagnação — mesma ferramenta 3 vezes seguidas                     |
| `limites.limite_tempo_segundos: 120`   | Wall-clock timeout de 2 minutos                                            |
| `limites.chamadas_ferramenta.total: 9` | Orçamento total de chamadas                                                |
| `acoes_sensiveis`                      | Pausa e pede confirmação humana antes de executar `rollback_deploy`        |
| `politicas`                            | Injetadas como texto no prompt da LLM — guiam a decisão sem alterar código |

### skills.md — as ferramentas (interface, não implementação)

```yaml
habilidades:
  - nome: consultar_metricas
    descricao: consulta metricas de latencia, throughput e taxa de erro do servico
    entrada:
      nome_servico: string
      janela_tempo_minutos: int
    saida:
      latencia_p99_ms: float
      vazao_rps: int
      taxa_erro: float
      status: string

  - nome: buscar_logs
    descricao: busca logs estruturados do servico em uma janela de tempo
    entrada:
      nome_servico: string
      janela_tempo_minutos: int
      nivel_minimo: string
    saida:
      eventos: list
      contagem_total: int

  - nome: historico_deploys
    descricao: consulta historico de deploys recentes do servico
    entrada:
      nome_servico: string
      janela_tempo_horas: int
    saida:
      deploys: list
      contagem_total: int

  - nome: relatorio_incidente
    descricao: abre incidente formal com evidencias e recomendacao de acao
    entrada:
      nome_servico: string
      severidade: string
      evidencia: object
      recomendacao: object
    saida:
      id_incidente: string
      status: string
```

**Ponto crítico**: `skills.md` define a **interface** — entradas e saídas, tipos de dados. Não há implementação aqui. Quem implementa é o `ferramentas.py` em tempo de execução, usando a LLM para gerar dados realistas. Em produção, você trocaria a chamada à LLM por uma chamada real à API do Grafana, Datadog, PagerDuty — mas o **contrato não mudaria**.

### hooks.md — observabilidade declarativa

```yaml
ganchos:
  antes_da_etapa: log
  apos_etapa: log
  antes_da_acao: log
  apos_acao: log
  em_erro: alerta
```

Cinco pontos de observação. Cinco linhas YAML. O runtime lê isso e chama a função de gancho nos momentos corretos do ciclo. `em_erro: alerta` faz o log aparecer destacado no terminal quando uma ferramenta falha.

### memory.md — o que lembrar e o que esquecer

```yaml
memoria_curta:
  guardar:
    - resultado_de_ferramenta
    - decisao_do_planejador
    - evidencia_coletada
    - erro_encontrado
  descartar:
    - prompt_sistema_completo
    - argumentos_mock_internos
    - dados_de_entrada_repetidos
  max_registros: 20

resumo_final:
  max_linhas: 5
  campos:
    - objetivo
    - etapas_executadas
    - ferramentas_chamadas
    - resultado_final
    - proximos_passos
```

Esse contrato governa o que fica no `estado["historico"]`. Com `max_registros: 20`, registros mais antigos são comprimidos para não inflar o contexto da LLM. O `resumo_final` define os campos que o `gerar_resumo_final()` vai produzir ao final da execução.

### contracts/planner.md — o contrato com a LLM

```yaml
formato_saida:
  proxima_acao: CHAMAR_FERRAMENTA | FINALIZAR | PERGUNTAR_USUARIO
  nome_ferramenta: opcional
  argumentos_ferramenta: opcional
  criterio_sucesso: obrigatorio
  pergunta: opcional (obrigatorio se PERGUNTAR_USUARIO)

regras:
  - sempre definir proxima acao
  - nunca retornar texto livre
  - coletar evidencias de metricas, logs e deploys antes de diagnosticar
  - analisar as evidencias coletadas para identificar a causa raiz
  - so usar FINALIZAR apos registrar o incidente com diagnostico e recomendacao
```

Este contrato não é para o runtime — é para a **LLM**. O `planejador.py` lê essas regras e as injeta no system prompt. A LLM é obrigada a responder exatamente nesse formato JSON. Nenhuma resposta em texto livre é aceita.

<figure style="margin: 2rem 0; text-align: center;">
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 780 200" style="max-width:780px;width:100%;">
    <rect width="780" height="200" rx="10" fill="#0d1b2a"/>
    <!-- agent.md -->
    <rect x="15" y="20" width="130" height="160" rx="8" fill="#0a1f35" stroke="#00ff88" stroke-width="1.5"/>
    <text x="80" y="45" text-anchor="middle" font-family="monospace" font-size="10" fill="#00ff88" font-weight="bold">agent.md</text>
    <text x="80" y="65" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">nome</text>
    <text x="80" y="80" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">tipo</text>
    <text x="80" y="95" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">objetivo</text>
    <text x="80" y="110" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">contrato_saida</text>
    <!-- rules.md -->
    <rect x="160" y="20" width="130" height="160" rx="8" fill="#1a0a10" stroke="#ff4466" stroke-width="1.5"/>
    <text x="225" y="45" text-anchor="middle" font-family="monospace" font-size="10" fill="#ff4466" font-weight="bold">rules.md</text>
    <text x="225" y="65" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">max_etapas</text>
    <text x="225" y="80" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">sem_progresso</text>
    <text x="225" y="95" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">limite_tempo</text>
    <text x="225" y="110" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">obrigatorias</text>
    <text x="225" y="125" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">acoes_sensiveis</text>
    <text x="225" y="140" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">politicas</text>
    <!-- skills.md -->
    <rect x="305" y="20" width="130" height="160" rx="8" fill="#0a1520" stroke="#9b4fff" stroke-width="1.5"/>
    <text x="370" y="45" text-anchor="middle" font-family="monospace" font-size="10" fill="#9b4fff" font-weight="bold">skills.md</text>
    <text x="370" y="65" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">nome da tool</text>
    <text x="370" y="80" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">descrição</text>
    <text x="370" y="95" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">entrada: {tipos}</text>
    <text x="370" y="110" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">saida: {tipos}</text>
    <!-- hooks.md -->
    <rect x="450" y="20" width="130" height="160" rx="8" fill="#0a1520" stroke="#ffaa00" stroke-width="1.5"/>
    <text x="515" y="45" text-anchor="middle" font-family="monospace" font-size="10" fill="#ffaa00" font-weight="bold">hooks.md</text>
    <text x="515" y="65" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">antes_da_etapa</text>
    <text x="515" y="80" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">apos_etapa</text>
    <text x="515" y="95" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">antes_da_acao</text>
    <text x="515" y="110" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">apos_acao</text>
    <text x="515" y="125" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">em_erro: alerta</text>
    <!-- memory.md -->
    <rect x="595" y="20" width="170" height="160" rx="8" fill="#0a1520" stroke="#00ccff" stroke-width="1.5"/>
    <text x="680" y="45" text-anchor="middle" font-family="monospace" font-size="10" fill="#00ccff" font-weight="bold">memory.md</text>
    <text x="680" y="65" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">guardar: [resultados]</text>
    <text x="680" y="80" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">descartar: [prompts]</text>
    <text x="680" y="95" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">max_registros: 20</text>
    <text x="680" y="110" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">resumo_final campos</text>
    <!-- Labels -->
    <text x="80" y="185" text-anchor="middle" font-family="monospace" font-size="8" fill="#336644">identidade</text>
    <text x="225" y="185" text-anchor="middle" font-family="monospace" font-size="8" fill="#882233">guardrails</text>
    <text x="370" y="185" text-anchor="middle" font-family="monospace" font-size="8" fill="#553388">interfaces</text>
    <text x="515" y="185" text-anchor="middle" font-family="monospace" font-size="8" fill="#886600">observabilidade</text>
    <text x="680" y="185" text-anchor="middle" font-family="monospace" font-size="8" fill="#2a5a7a">memória</text>
  </svg>
  <figcaption style="font-size:0.85rem;color:#888;margin-top:0.5rem;">Os 5 contratos raiz do agente — cada um com uma responsabilidade distinta.</figcaption>
</figure>

---

## contratos.py — a camada de carregamento

````python
def carregar_yaml_do_md(caminho_arquivo: Path) -> dict:
    """Extrai o primeiro bloco YAML de um arquivo .md."""
    if not caminho_arquivo.exists():
        return {}
    texto = caminho_arquivo.read_text(encoding="utf-8")
    correspondencia = re.search(r"```yaml\n(.*?)```", texto, re.DOTALL)
    if not correspondencia:
        return {}
    return yaml.safe_load(correspondencia.group(1)) or {}
````

`carregar_yaml_do_md` é o ponto de junção entre documentação e máquina. Uma expressão regular extrai o primeiro bloco ` ```yaml ` do arquivo Markdown e faz parse com `yaml.safe_load`. O resultado é um dicionário Python — sem nenhum conhecimento de domínio.

```python
def carregar_contratos(caminho_agente: Path) -> dict:
    pasta_contratos = caminho_agente / "contracts"
    return {
        "agente":          carregar_yaml_do_md(caminho_agente / "agent.md"),
        "ciclo":           carregar_yaml_do_md(pasta_contratos / "loop.md"),
        "planejador":      carregar_yaml_do_md(pasta_contratos / "planner.md"),
        "caixa_ferramentas": carregar_yaml_do_md(pasta_contratos / "toolbox.md"),
        "executor":        carregar_yaml_do_md(pasta_contratos / "executor.md"),
        "regras":          carregar_yaml_do_md(caminho_agente / "rules.md"),
        "ganchos":         carregar_yaml_do_md(caminho_agente / "hooks.md"),
        "habilidades":     carregar_yaml_do_md(caminho_agente / "skills.md"),
        "memoria":         carregar_yaml_do_md(caminho_agente / "memory.md"),
    }
```

Nove chamadas, nove contratos, um dicionário com nove chaves. A partir daqui, todo o runtime opera sobre esse `contratos: dict`. Nenhum módulo abre arquivos diretamente — todos recebem os contratos já carregados.

### criar_estado — a folha em branco

```python
def criar_estado(contratos, texto_entrada, modo=None, evento=None):
    regras = contratos.get("regras", {})
    ciclo = contratos.get("ciclo", {})
    agente = contratos.get("agente", {})
    config_chamadas = regras.get("limites", {}).get("chamadas_ferramenta", {})

    if isinstance(config_chamadas, dict):
        max_chamadas_ferramenta = config_chamadas.get("total", 10)
        limites_por_ferramenta = {
            k: v for k, v in config_chamadas.items() if k != "total"
        }
    else:
        max_chamadas_ferramenta = config_chamadas
        limites_por_ferramenta = {}

    tipo_agente = modo or agente.get("tipo", "task_based")

    return {
        "objetivo": ciclo.get("objetivo", "desconhecido"),
        "entrada": texto_entrada,
        "tipo_agente": tipo_agente,
        "evento": evento,
        "etapa": 0,
        "chamadas_ferramenta": 0,
        "chamadas_por_ferramenta": {},
        "max_etapas": regras.get("limites", {}).get("max_etapas", 10),
        "max_chamadas_ferramenta": max_chamadas_ferramenta,
        "limites_por_ferramenta": limites_por_ferramenta,
        "sem_progresso": regras.get("limites", {}).get("sem_progresso", 3),
        "limite_tempo_segundos": regras.get("limites", {}).get("limite_tempo_segundos", 120),
        "max_tokens": regras.get("limites", {}).get("max_tokens", 50000),
        "tokens_consumidos": {"prompt": 0, "completion": 0, "total": 0},
        "acoes_sensiveis": regras.get("acoes_sensiveis", []),
        "historico": [],
        "concluido": False,
        "resultado": "",
        "etapas_sem_progresso": 0,
        "ultima_ferramenta": None,
    }
```

O estado é o **objeto vivo** da execução. Ele nasce aqui, vazio, com todos os limites lidos dos contratos. Cresce a cada iteração do ciclo com novo histórico, contadores incrementados, tokens acumulados. Morre quando o ciclo encerra — e é serializado no `trace.json`.

Observe que `tipo_agente = modo or agente.get("tipo", "task_based")`: o parâmetro CLI sobrescreve o contrato. Você pode forçar `--modo interactive` mesmo que `agent.md` declare `task_based`. Essa precedência é uma decisão de UX explícita.

---

## ciclo.py — o coração do runtime

O `ciclo.py` é o módulo mais longo e mais importante. Ele não faz nada por conta própria — orquestra todos os outros. A função `rodar` é o loop principal:

```
perceber → planejar → circuit breaker → agir → avaliar
```

Repetido até que o estado diga `concluido = True` ou algum limite seja atingido.

<figure style="margin: 2rem 0; text-align: center;">
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 140" style="max-width:760px;width:100%;">
    <rect width="760" height="140" rx="10" fill="#0d1b2a"/>
    <!-- Boxes das fases -->
    <rect x="20" y="40" width="120" height="60" rx="8" fill="#0a1f35" stroke="#00ff88" stroke-width="2"/>
    <text x="80" y="66" text-anchor="middle" font-family="monospace" font-size="12" fill="#00ff88" font-weight="bold">PERCEBER</text>
    <text x="80" y="84" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">perceber(estado)</text>
    <!-- Seta -->
    <line x1="140" y1="70" x2="165" y2="70" stroke="#00ccff" stroke-width="2"/>
    <polygon points="163,64 175,70 163,76" fill="#00ccff"/>
    <rect x="175" y="40" width="120" height="60" rx="8" fill="#0a1f35" stroke="#00ccff" stroke-width="2"/>
    <text x="235" y="66" text-anchor="middle" font-family="monospace" font-size="12" fill="#00ccff" font-weight="bold">PLANEJAR</text>
    <text x="235" y="84" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">chamar_llm()</text>
    <!-- Seta -->
    <line x1="295" y1="70" x2="320" y2="70" stroke="#ff4466" stroke-width="2"/>
    <polygon points="318,64 330,70 318,76" fill="#ff4466"/>
    <!-- Circuit Breaker -->
    <rect x="330" y="30" width="140" height="80" rx="8" fill="#1a0a15" stroke="#ff4466" stroke-width="2" stroke-dasharray="6 3"/>
    <text x="400" y="58" text-anchor="middle" font-family="monospace" font-size="11" fill="#ff4466" font-weight="bold">⚡ CIRCUIT</text>
    <text x="400" y="74" text-anchor="middle" font-family="monospace" font-size="11" fill="#ff4466" font-weight="bold">BREAKER</text>
    <text x="400" y="96" text-anchor="middle" font-family="monospace" font-size="9" fill="#882233">valida resposta LLM</text>
    <!-- Seta -->
    <line x1="470" y1="70" x2="495" y2="70" stroke="#9b4fff" stroke-width="2"/>
    <polygon points="493,64 505,70 493,76" fill="#9b4fff"/>
    <rect x="505" y="40" width="100" height="60" rx="8" fill="#0a1520" stroke="#9b4fff" stroke-width="2"/>
    <text x="555" y="66" text-anchor="middle" font-family="monospace" font-size="12" fill="#9b4fff" font-weight="bold">AGIR</text>
    <text x="555" y="84" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">executar()</text>
    <!-- Seta -->
    <line x1="605" y1="70" x2="630" y2="70" stroke="#00ff88" stroke-width="2"/>
    <polygon points="628,64 640,70 628,76" fill="#00ff88"/>
    <rect x="640" y="40" width="100" height="60" rx="8" fill="#0a1f35" stroke="#00ff88" stroke-width="2"/>
    <text x="690" y="66" text-anchor="middle" font-family="monospace" font-size="12" fill="#00ff88" font-weight="bold">AVALIAR</text>
    <text x="690" y="84" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">avaliar()</text>
    <!-- Seta de retorno -->
    <path d="M 690 100 Q 690 125 400 125 Q 110 125 80 100" fill="none" stroke="#00ff88" stroke-width="1.5" stroke-dasharray="5 3"/>
    <polygon points="82,102 74,98 84,94" fill="#00ff88"/>
    <text x="400" y="120" text-anchor="middle" font-family="monospace" font-size="9" fill="#336644">próxima iteração</text>
  </svg>
  <figcaption style="font-size:0.85rem;color:#888;margin-top:0.5rem;">O ciclo principal de uma iteração: 4 fases + circuit breaker entre planejar e agir.</figcaption>
</figure>

### Os guardrails de segurança

Antes de cada iteração, o ciclo verifica quatro condições de parada:

**1. Limite de tempo:**

```python
def verificar_tempo(estado, inicio):
    limite = estado.get("limite_tempo_segundos", 120)
    return (time.time() - inicio) >= limite
```

**2. Limite de tokens:**

```python
def verificar_limite_tokens(estado):
    return estado["tokens_consumidos"]["total"] >= estado.get("max_tokens", 50000)
```

**3. Estagnação (sem progresso):**

```python
def verificar_sem_progresso(estado, nome_ferramenta):
    if nome_ferramenta == estado.get("ultima_ferramenta"):
        estado["etapas_sem_progresso"] += 1
    else:
        estado["etapas_sem_progresso"] = 0
    estado["ultima_ferramenta"] = nome_ferramenta
    return estado["etapas_sem_progresso"] >= estado.get("sem_progresso", 3)
```

Se a LLM ficar chamando `consultar_metricas` repetidamente sem mudar de ferramenta, três vezes seguidas, o ciclo encerra. Isso detecta o caso onde a LLM entrou em loop.

**4. Limite de chamadas por ferramenta:**

```python
chamadas_desta_ferramenta = estado["chamadas_por_ferramenta"].get(nome_ferramenta, 0)
limite_desta_ferramenta = estado["limites_por_ferramenta"].get(nome_ferramenta)
if limite_desta_ferramenta and chamadas_desta_ferramenta >= limite_desta_ferramenta:
    # encerra
```

O `rules.md` define `consultar_metricas: 3` — o ciclo conta e bloqueia na terceira chamada.

### Ferramentas obrigatórias antes do FINALIZAR

```python
if plano.get("proxima_acao") == "FINALIZAR":
    obrigatorias = contratos.get("regras", {}).get("ferramentas_obrigatorias", [])
    faltantes = [
        nome for nome in obrigatorias
        if nome not in estado["chamadas_por_ferramenta"]
    ]
    if faltantes:
        # força chamada da ferramenta obrigatória antes de finalizar
        plano = {
            "proxima_acao": "CHAMAR_FERRAMENTA",
            "nome_ferramenta": faltantes[0],
            "argumentos_ferramenta": montar_argumentos_mock(habilidade_faltante, estado["historico"]),
            "criterio_sucesso": f"{faltantes[0]} obrigatorio antes de finalizar",
        }
```

A LLM pode querer finalizar sem ter chamado `relatorio_incidente`. O runtime não permite. Intercepta a ação `FINALIZAR`, verifica quais ferramentas obrigatórias ainda não foram usadas, e substitui o plano por uma chamada forçada à ferramenta pendente.

### Human-in-the-loop

```python
if nome_ferramenta in estado.get("acoes_sensiveis", []):
    tel.registrar("confirmacao_humana", {"ferramenta": nome_ferramenta})
    if not pedir_confirmacao_humana(nome_ferramenta):
        estado["concluido"] = True
        estado["resultado"] = f"encerrado por negacao humana ({nome_ferramenta})"
        break
```

`rollback_deploy` está em `acoes_sensiveis`. Antes de executar, o runtime pausa e imprime no terminal:

```
[CONFIRMACAO HUMANA] A ferramenta 'rollback_deploy' requer autorizacao.
Autorizar execucao de 'rollback_deploy'? (s/n):
```

Se `EOFError` (sem terminal interativo), nega por segurança. Essa é a diferença entre automação irresponsável e automação confiável.

### O circuit breaker (validar_resposta_llm)

```python
_ACOES_VALIDAS = {"CHAMAR_FERRAMENTA", "FINALIZAR", "PERGUNTAR_USUARIO"}

def validar_resposta_llm(plano, ferramentas_disponiveis):
    problemas = []
    if not isinstance(plano, dict):
        return ["resposta da LLM nao e um dicionario valido"]

    proxima_acao = plano.get("proxima_acao")
    if not proxima_acao:
        problemas.append("campo 'proxima_acao' ausente na resposta da LLM")
    elif proxima_acao not in _ACOES_VALIDAS:
        problemas.append(f"proxima_acao '{proxima_acao}' invalida")

    if proxima_acao == "CHAMAR_FERRAMENTA":
        nome = plano.get("nome_ferramenta")
        if not nome:
            problemas.append("CHAMAR_FERRAMENTA sem 'nome_ferramenta'")
        elif nome not in ferramentas_disponiveis:
            problemas.append(f"ferramenta '{nome}' nao existe")

    if proxima_acao == "PERGUNTAR_USUARIO":
        if not plano.get("pergunta"):
            problemas.append("PERGUNTAR_USUARIO sem campo 'pergunta'")

    return problemas
```

O nome do padrão aqui é **Circuit Breaker** — um padrão de tolerância a falhas que "abre o circuito" quando detecta falhas consecutivas. Aqui o circuito protege contra respostas malformadas da LLM.

Mas o runtime não desiste na primeira falha: tenta auto-correção:

```python
if problemas_llm:
    # auto-correção 1: ação inválida mas ferramenta existe
    nome_no_plano = plano.get("nome_ferramenta") or plano.get("proxima_acao")
    if (any("invalida" in p for p in problemas_llm)
            and nome_no_plano in nomes_ferramentas_disponiveis):
        plano["proxima_acao"] = "CHAMAR_FERRAMENTA"
        plano["nome_ferramenta"] = nome_no_plano

    # auto-correção 2: ferramenta não existe → usar próxima não utilizada
    elif any("nao existe" in p for p in problemas_llm):
        ferramenta_fallback = next(
            (h["nome"] for h in habilidades
             if h["nome"] in nomes_ferramentas_disponiveis
             and h["nome"] not in estado["chamadas_por_ferramenta"]),
            None,
        )
        if ferramenta_fallback:
            plano = {
                "proxima_acao": "CHAMAR_FERRAMENTA",
                "nome_ferramenta": ferramenta_fallback,
                ...
            }
        else:
            # sem recovery → encerra
            estado["concluido"] = True
            break
```

Essa lógica é **graceful degradation**: a LLM errou o nome da ferramenta, o runtime substitui pelo próximo disponível não utilizado e continua. Só encerra se não há como se recuperar.

### O painel de KPIs

```python
def exibir_kpis(estado, tel, inicio, contratos):
    # barra visual de tokens
    pct_tokens = tokens_total / max_tokens
    blocos_cheios = int(pct_tokens * 10)
    barra = "█" * blocos_cheios + "░" * (10 - blocos_cheios)

    # status de cada ferramenta (✓ usada, ! obrigatória pendente, ○ disponível)
    for nome in nomes_ferramentas:
        if nome in estado["chamadas_por_ferramenta"]:
            partes_ferramentas.append(f"✓{nome}")
        elif nome in obrigatorias:
            partes_ferramentas.append(f"!{nome}")
        else:
            partes_ferramentas.append(f"○{nome}")

    print(f"  ┌─ KPIs ________________________________┐")
    print(f"  │ Progresso:  {etapa}/{max_etapas} etapas    {chamadas}/{max_chamadas} chamadas")
    print(f"  │ Tokens:     {tokens_total}/{max_tokens} ({pct_str})  {barra}")
    print(f"  │ Ferramentas: {texto_ferramentas}")
    print(f"  │ Qualidade:  {ok}/{ok+parcial+falha} ok   {parcial} parcial   {falha} falha")
    print(f"  │ Alertas:    {cb} circuit_breaker   {pv} payload_invalido")
    print(f"  │ Latencia:   {texto_lat}")
    print(f"  └_______________________________________┘")
```

Este painel é exibido ao final de **cada iteração**. O operador vê em tempo real: quantas etapas restam, consumo de tokens com barra visual, quais ferramentas já foram usadas (com destaque para as obrigatórias pendentes), qualidade das avaliações, e latência das fases `planejar` e `agir`.

---

## planejador.py — percepção e decisão

Dois problemas fundamentais em qualquer agente: **o que a LLM precisa saber** e **como ela deve responder**. O `planejador.py` resolve os dois.

### perceber — montando o contexto

```python
def perceber(estado):
    partes = [f"Alerta: {estado['entrada']}"]
    partes.append(f"Modo: {estado.get('tipo_agente', 'task_based')}")

    if estado.get("evento"):
        partes.append(f"Evento trigger: {estado['evento']}")

    for registro in estado["historico"]:
        etapa = registro["etapa"]
        ferramenta_usada = registro.get("plano", {}).get("nome_ferramenta", "nenhuma")
        if registro.get("resultado_acao"):
            partes.append(
                f"Etapa {etapa} [{ferramenta_usada}]: "
                f"{json.dumps(registro['resultado_acao'], ensure_ascii=False)}"
            )

    ferramentas_usadas = list(estado["chamadas_por_ferramenta"].keys())
    if ferramentas_usadas:
        partes.append(f"Ferramentas ja utilizadas: {', '.join(ferramentas_usadas)}")

    partes.append(f"Etapas realizadas: {estado['etapa']}/{estado['max_etapas']}")

    if estado.get("etapas_sem_progresso", 0) > 0:
        partes.append(f"ATENCAO: {estado['etapas_sem_progresso']} etapas sem progresso detectadas")

    return "\n".join(partes)
```

A percepção é uma **string de contexto** que cresce a cada iteração. Na primeira etapa, é apenas o alerta. Na terceira, já contém os resultados das duas ferramentas chamadas anteriormente. A LLM não tem memória própria entre chamadas — é a percepção que carrega esse contexto.

> Se o contexto é ruim, a decisão é ruim. Percepção define qualidade.

Note o alerta de estagnação: se `etapas_sem_progresso > 0`, a percepção inclui `"ATENCAO: N etapas sem progresso detectadas"`. Isso dá à LLM a oportunidade de mudar de estratégia antes que o circuit breaker atue.

### construir_prompt_sistema — o system prompt dinâmico

```python
def construir_prompt_sistema(contratos):
    agente = contratos.get("agente", {})
    habilidades = contratos.get("habilidades", {}).get("habilidades", [])

    bloco_ferramentas = ""
    for habilidade in habilidades:
        nome = habilidade.get("nome", "")
        descricao = habilidade.get("descricao", "")
        entradas = habilidade.get("entrada", {})
        saidas = habilidade.get("saida", {})
        texto_entradas = ", ".join(f"{n}: {t}" for n, t in entradas.items())
        texto_saidas = ", ".join(f"{n}: {t}" for n, t in saidas.items())
        bloco_ferramentas += f"- {nome}: {descricao}\n  entrada: {{{texto_entradas}}}\n"

    regras_planejador = contratos.get("planejador", {}).get("regras", [])
    politicas = contratos.get("regras", {}).get("politicas", [])

    return f"""Voce e o planejador de um agente autonomo.
Agente: {nome_agente} - {descricao_agente}
Objetivo: {objetivo}

Ferramentas disponiveis:
{bloco_ferramentas}
Formato de resposta (APENAS JSON valido):
{{
  "proxima_acao": "CHAMAR_FERRAMENTA" ou "FINALIZAR" ou "PERGUNTAR_USUARIO",
  "nome_ferramenta": "...",
  "argumentos_ferramenta": {{}},
  "criterio_sucesso": "...",
}}

IMPORTANTE - Regras do planejador:
{texto_regras}

IMPORTANTE - Politicas do agente:
{texto_politicas}
"""
```

O system prompt é **100% gerado a partir dos contratos**. O código não hardcoda nenhum domínio. As ferramentas, regras, políticas, tipo de agente — tudo vem dos arquivos Markdown. Mude o agente, mude o prompt automaticamente.

### chamar_llm vs planejador_mock

```python
def chamar_llm(percepcao, contratos, historico=None):
    chave_api = os.environ.get("OPENAI_API_KEY")

    if not chave_api:
        return planejador_mock(percepcao, contratos, historico or []), _TOKENS_ZERO.copy()

    from openai import OpenAI
    cliente = OpenAI(api_key=chave_api)
    resposta = cliente.chat.completions.create(
        model="gpt-4o-mini",
        response_format={"type": "json_object"},
        messages=[
            {"role": "system", "content": construir_prompt_sistema(contratos)},
            {"role": "user", "content": percepcao},
        ],
    )
    ...
```

O import do `openai` é **lazy** — só acontece quando há API key. Sem `OPENAI_API_KEY`, o `planejador_mock` entra em cena: percorre as habilidades em ordem e simula decisões. Isso permite desenvolver, testar e depurar sem custo de tokens.

```python
def planejador_mock(percepcao, contratos, historico=None):
    # modo interactive: simula pergunta na primeira etapa
    if tipo_agente == "interactive" and not historico:
        return {
            "proxima_acao": "PERGUNTAR_USUARIO",
            "pergunta": "Qual servico esta com problema?",
        }

    # percorre ferramentas em ordem, usa a primeira não chamada
    for nome in nomes_ferramentas:
        if nome not in percepcao:
            return {
                "proxima_acao": "CHAMAR_FERRAMENTA",
                "nome_ferramenta": nome,
                "argumentos_ferramenta": montar_argumentos_mock(habilidade, historico),
                "criterio_sucesso": f"{nome} executado com sucesso",
            }

    # todas chamadas → finalizar com resumo das evidências
    return {"proxima_acao": "FINALIZAR", "criterio_sucesso": f"Diagnostico: {resumo}"}
```

O mock é **determinístico**: sempre executa todas as ferramentas na ordem definida em `skills.md`, depois finaliza. Isso torna os testes reproduzíveis.

---

## ferramentas.py — geração automática de implementações

Este módulo resolve o problema mais elegante do framework: **como implementar ferramentas sem escrever a implementação**.

### construir_ferramenta — o factory pattern

```python
def construir_ferramenta(habilidade):
    nome = habilidade.get("nome", "")
    descricao = habilidade.get("descricao", "")
    campos_saida = habilidade.get("saida", {})

    texto_saida = "\n".join(
        f"  - {campo}: {tipo}" for campo, tipo in campos_saida.items()
    )

    prompt_sistema = f"""Voce e uma ferramenta chamada '{nome}'.
Funcao: {descricao}

Voce DEVE retornar APENAS JSON valido com exatamente estes campos:
{texto_saida}

Regras:
- Gere dados realistas e coerentes com os argumentos recebidos
- NUNCA use placeholders como 'mock', 'exemplo', 'teste'
- Responda em portugues"""

    def funcao(argumentos):
        prompt_usuario = f"Argumentos recebidos:\n{json.dumps(argumentos, indent=2)}"
        dados_llm, uso_tokens = _chamar_llm_ferramenta(
            prompt_sistema, prompt_usuario, campos_saida
        )

        if dados_llm:
            dados_llm["_entrada"] = argumentos
            return {"sucesso": True, "dados": dados_llm, "_tokens": uso_tokens}

        # fallback: sem API key
        dados = {
            nome_campo: _gerar_valor_fallback(tipo_campo, nome_campo)
            for nome_campo, tipo_campo in campos_saida.items()
        }
        return {"sucesso": True, "dados": dados, "_tokens": _TOKENS_ZERO.copy()}

    return funcao
```

`construir_ferramenta` recebe um dicionário de contrato e retorna uma **função Python** — um closure. O prompt de sistema é capturado no closure e reutilizado a cada chamada. Essa é a aplicação do padrão **Factory** combinado com **Closure**: a fábrica cria funções sob demanda, cada uma pré-configurada com o contrato da skill.

```python
def construir_ferramentas_dos_contratos(contratos):
    habilidades = contratos.get("habilidades", {}).get("habilidades", [])
    ferramentas = {}
    for habilidade in habilidades:
        nome = habilidade.get("nome")
        if nome:
            ferramentas[nome] = construir_ferramenta(habilidade)
    return ferramentas
```

Resultado: um dicionário `{"consultar_metricas": <função>, "buscar_logs": <função>, ...}`. O ciclo chama `ferramentas[nome_ferramenta](argumentos)` sem saber nada sobre o domínio.

### Propagação de evidências pelo histórico

```python
def extrair_evidencias_do_historico(historico):
    evidencias = {}
    for registro in historico:
        plano = registro.get("plano", {})
        resultado = registro.get("resultado_acao")
        nome_ferramenta = plano.get("nome_ferramenta")
        if resultado and resultado.get("sucesso") and nome_ferramenta:
            evidencias[nome_ferramenta] = resultado.get("dados", {})
    return evidencias

def montar_argumentos_mock(habilidade, historico):
    argumentos = {}
    evidencias = extrair_evidencias_do_historico(historico)
    for nome_campo, tipo_campo in habilidade.get("entrada", {}).items():
        if tipo_campo.lower() == "object" and evidencias:
            argumentos[nome_campo] = evidencias  # usa tudo que foi coletado
        else:
            argumentos[nome_campo] = _gerar_valor_fallback(tipo_campo, nome_campo)
    return argumentos
```

Quando `relatorio_incidente` precisa de `evidencia: object`, o `montar_argumentos_mock` passa todas as evidências coletadas pelas ferramentas anteriores. A cadeia de coleta de dados se torna automática — o histórico funciona como acumulador de contexto.

---

## executor.py — quatro responsabilidades

### 1. Ganchos (Observer pattern)

```python
def executar_gancho(nome, contrato_ganchos, **kwargs):
    ganchos = contrato_ganchos.get("ganchos", {})
    acao = ganchos.get(nome)
    if not acao:
        return

    carimbo_tempo = datetime.now().strftime("%H:%M:%S")
    detalhe = " ".join(f"{k}={v}" for k, v in kwargs.items())

    if acao == "log":
        print(f"  [{carimbo_tempo}] gancho:{nome} {detalhe}")
    elif acao == "alerta":
        print(f"  [{carimbo_tempo}] [ALERTA] gancho:{nome} {detalhe}")
```

O padrão **Observer** clássico: eventos do ciclo notificam observadores (ganchos). A diferença é que os observadores são declarados em `hooks.md`, não registrados no código. O runtime lê o contrato e sabe quem chamar e quando. Cinco pontos de observação, dois tipos de ação (`log` e `alerta`), zero código de domínio.

### 2. Validação de payload de entrada

```python
_MAPA_TIPOS = {
    "string": str,
    "int": (int,),
    "float": (int, float),  # int é aceito como float
    "bool": (bool,),
    "list": (list,),
    "object": (dict,),
}

def validar_payload(nome_ferramenta, argumentos, contratos):
    habilidade = next(
        (h for h in habilidades if h.get("nome") == nome_ferramenta), None
    )
    schema_entrada = habilidade.get("entrada", {})
    argumentos = argumentos or {}

    erros = []
    for campo, tipo_esperado in schema_entrada.items():
        if campo not in argumentos:
            erros.append(f"campo obrigatorio '{campo}' ausente")
            continue
        valor = argumentos[campo]
        tipos_python = _MAPA_TIPOS.get(tipo_esperado.lower())
        if tipos_python and valor is not None:
            if not isinstance(valor, tipos_python):
                erros.append(f"campo '{campo}': esperado {tipo_esperado}, recebido {type(valor).__name__}")

    return erros
```

O schema de tipos vem de `skills.md`. O mapa `_MAPA_TIPOS` traduz tipos YAML para tipos Python. Note que `float` aceita `int` (`(int, float)`) — uma decisão pragmática: `janela_tempo_minutos: 30` é um int mas faz sentido passar para um campo `float`. Os erros de payload são **registrados mas não bloqueiam** — graceful degradation.

### 3. Execução com retry

```python
def executar(nome_ferramenta, argumentos, ferramentas, contratos):
    if nome_ferramenta not in ferramentas:
        return {"sucesso": False, "erro": f"Ferramenta '{nome_ferramenta}' nao encontrada"}

    try:
        resultado = ferramentas[nome_ferramenta](argumentos or {})
    except Exception as erro:
        config_executor = contratos.get("executor", {}).get("execucao", {})
        if config_executor.get("tentar_novamente_em_falha"):
            try:
                resultado = ferramentas[nome_ferramenta](argumentos or {})
            except Exception as erro_nova_tentativa:
                return {"sucesso": False, "erro": str(erro_nova_tentativa)}
        else:
            return {"sucesso": False, "erro": str(erro)}

    return resultado
```

`tentar_novamente_em_falha: true` no `executor.md` ativa o retry. Uma segunda tentativa com os mesmos argumentos. Simples, mas cobre o caso mais comum: falha transitória de rede ou timeout da API. Em produção você implementaria backoff exponencial aqui — o contrato não mudaria.

### 4. Avaliação semântica

```python
def avaliar(plano, resultado_acao, contratos=None):
    if plano.get("proxima_acao") == "FINALIZAR":
        return {"objetivo_alcancado": True, "motivo": plano.get("criterio_sucesso", "")}

    if not resultado_acao or not resultado_acao.get("sucesso"):
        return {"objetivo_alcancado": False, "motivo": "etapa falhou", "qualidade": "falha"}

    # valida a saída contra o schema
    problemas_saida = validar_saida(nome_ferramenta, resultado_acao, contratos)

    if problemas_saida:
        return {
            "objetivo_alcancado": False,
            "motivo": f"etapa ok com ressalvas - {'; '.join(problemas_saida)}",
            "qualidade": "parcial",
            "problemas_saida": problemas_saida,
        }
    else:
        return {
            "objetivo_alcancado": False,
            "motivo": f"etapa ok - criterio: {criterio}",
            "qualidade": "completa",
        }
```

Três níveis de qualidade: `completa` (tudo OK), `parcial` (sucesso mas saída com problemas), `falha` (execução falhou). Essa classificação alimenta o histórico e o painel de KPIs.

```python
def validar_saida(nome_ferramenta, resultado, contratos):
    dados = resultado.get("dados", {})
    schema_saida = habilidade.get("saida", {})
    problemas = []
    for campo, tipo_esperado in schema_saida.items():
        if campo not in dados:
            problemas.append(f"campo de saida '{campo}' ausente")
        elif dados[campo] is None:
            problemas.append(f"campo '{campo}' retornou None")
        elif isinstance(dados[campo], str) and not dados[campo].strip():
            problemas.append(f"campo '{campo}' retornou string vazia")
        elif isinstance(dados[campo], list) and len(dados[campo]) == 0:
            problemas.append(f"campo '{campo}' retornou lista vazia")
    return problemas
```

Validação de saída não é sobre tipos — é sobre **dados úteis**. Um campo `eventos: list` que retorna `[]` é tecnicamente correto mas semanticamente inútil. O agente precisa saber disso para tomar uma decisão melhor na próxima iteração.

---

## telemetria.py — observabilidade estruturada

```python
class Telemetria:
    def __init__(self, agente, tipo_agente):
        self.trace_id = uuid.uuid4().hex[:12]
        self.inicio = time.time()
        self.eventos = []
        self.fases = []
        self.tokens = {"prompt": 0, "completion": 0, "total": 0}
        self.chamadas_llm = 0
        self.ferramentas_sucesso = 0
        self.ferramentas_falha = 0
        self.circuit_breaker_ativacoes = 0
        self.validacao_payload_falhas = 0
```

O `trace_id` é gerado com `uuid4().hex[:12]` — 12 caracteres hex, únicos por execução. Todos os eventos carregam esse ID, permitindo correlacionar logs de múltiplas execuções.

### Medição de fases

```python
def iniciar_fase(self, nome_fase, etapa):
    return {
        "fase": nome_fase,
        "etapa": etapa,
        "inicio": time.time(),
        "fim": None,
        "duracao_ms": None,
    }

def finalizar_fase(self, marcador):
    marcador["fim"] = time.time()
    marcador["duracao_ms"] = round((marcador["fim"] - marcador["inicio"]) * 1000, 1)
    self.fases.append(marcador)
    self.registrar("fase_concluida", {
        "fase": marcador["fase"],
        "etapa": marcador["etapa"],
        "duracao_ms": marcador["duracao_ms"],
    })
```

O padrão `marcador = iniciar_fase(...) → ... → finalizar_fase(marcador)` é simples e funciona porque Python passa dicionários por referência. Qualquer parte do código pode medir qualquer intervalo sem acoplamento.

### Os 4 streams de saída

<figure style="margin: 2rem 0; text-align: center;">
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 180" style="max-width:760px;width:100%;">
    <rect width="760" height="180" rx="10" fill="#0d1b2a"/>
    <text x="380" y="28" text-anchor="middle" font-family="monospace" font-size="13" fill="#aaa" font-weight="bold">trace.json — 4 streams de observabilidade</text>
    <!-- telemetry_stream -->
    <rect x="20" y="45" width="165" height="115" rx="8" fill="#0a1520" stroke="#00ff88" stroke-width="1.5"/>
    <text x="103" y="68" text-anchor="middle" font-family="monospace" font-size="10" fill="#00ff88" font-weight="bold">telemetry_stream</text>
    <text x="103" y="86" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">todos os eventos</text>
    <text x="103" y="101" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">timestamp + elapsed</text>
    <text x="103" y="116" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">trace_id em cada</text>
    <text x="103" y="148" text-anchor="middle" font-family="monospace" font-size="9" fill="#336644">→ debugging</text>
    <!-- audit_logs -->
    <rect x="200" y="45" width="165" height="115" rx="8" fill="#0a1520" stroke="#ffaa00" stroke-width="1.5"/>
    <text x="283" y="68" text-anchor="middle" font-family="monospace" font-size="10" fill="#ffaa00" font-weight="bold">audit_logs</text>
    <text x="283" y="86" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">plano_gerado</text>
    <text x="283" y="101" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">ferramenta_executada</text>
    <text x="283" y="116" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">circuit_breaker</text>
    <text x="283" y="131" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">confirmacao_humana</text>
    <text x="283" y="148" text-anchor="middle" font-family="monospace" font-size="9" fill="#886600">→ compliance</text>
    <!-- health_metrics -->
    <rect x="380" y="45" width="165" height="115" rx="8" fill="#0a1520" stroke="#ff4466" stroke-width="1.5"/>
    <text x="463" y="68" text-anchor="middle" font-family="monospace" font-size="10" fill="#ff4466" font-weight="bold">health_metrics</text>
    <text x="463" y="86" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">taxa_sucesso %</text>
    <text x="463" y="101" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">circuit_breaker count</text>
    <text x="463" y="116" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">payload_falhas count</text>
    <text x="463" y="131" text-anchor="middle" font-family="monospace" font-size="9" fill="#cc3355">chamadas_llm count</text>
    <text x="463" y="148" text-anchor="middle" font-family="monospace" font-size="9" fill="#882233">→ SRE / alertas</text>
    <!-- performance_data -->
    <rect x="560" y="45" width="180" height="115" rx="8" fill="#0a1520" stroke="#00ccff" stroke-width="1.5"/>
    <text x="650" y="68" text-anchor="middle" font-family="monospace" font-size="10" fill="#00ccff" font-weight="bold">performance_data</text>
    <text x="650" y="86" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">fases: media/max/total</text>
    <text x="650" y="101" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">tokens: prompt/completion</text>
    <text x="650" y="116" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">tempo_total_ms</text>
    <text x="650" y="131" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">chamadas_llm</text>
    <text x="650" y="148" text-anchor="middle" font-family="monospace" font-size="9" fill="#2a5a7a">→ perf tuning</text>
  </svg>
  <figcaption style="font-size:0.85rem;color:#888;margin-top:0.5rem;">Os 4 streams gravados no trace.json — cada um atende uma audiência diferente.</figcaption>
</figure>

```python
def telemetry_stream(self):
    """Todos os eventos em ordem cronológica."""
    return self.eventos

def audit_logs(self):
    """Apenas eventos auditáveis: decisões e ações."""
    tipos_auditoria = {
        "plano_gerado", "ferramenta_executada", "circuit_breaker",
        "validacao_payload_falha", "confirmacao_humana", "finalizado",
    }
    return [e for e in self.eventos if e["tipo"] in tipos_auditoria]

def health_metrics(self):
    """Taxa de sucesso, circuit breaker, payload falhas."""
    total = self.ferramentas_sucesso + self.ferramentas_falha
    taxa = round(self.ferramentas_sucesso / total * 100, 1) if total > 0 else 0.0
    return {
        "taxa_sucesso_ferramentas": taxa,
        "circuit_breaker_ativacoes": self.circuit_breaker_ativacoes,
        "validacao_payload_falhas": self.validacao_payload_falhas,
        "chamadas_llm": self.chamadas_llm,
    }

def performance_data(self):
    """Tempos por fase: média, max, total, contagem."""
    tempos_por_fase = {}
    for fase in self.fases:
        nome = fase["fase"]
        if nome not in tempos_por_fase:
            tempos_por_fase[nome] = {"total_ms": 0, "contagem": 0, "max_ms": 0}
        tempos_por_fase[nome]["total_ms"] += fase["duracao_ms"]
        tempos_por_fase[nome]["contagem"] += 1
        if fase["duracao_ms"] > tempos_por_fase[nome]["max_ms"]:
            tempos_por_fase[nome]["max_ms"] = fase["duracao_ms"]
    for d in tempos_por_fase.values():
        d["media_ms"] = round(d["total_ms"] / d["contagem"], 1)
    return {"fases": tempos_por_fase, "tokens": self.tokens}
```

| Stream             | Audiência          | Conteúdo                            |
| ------------------ | ------------------ | ----------------------------------- |
| `telemetry_stream` | Debugging          | Todo evento com timestamp e elapsed |
| `audit_logs`       | Compliance         | Apenas decisões e ações relevantes  |
| `health_metrics`   | SRE / alertas      | Taxas, contagens, ativações         |
| `performance_data` | Performance tuning | Tempos por fase com min/max/média   |

Esses quatro streams são salvos no `trace.json`. Uma ferramenta de análise (o `analisar` command) pode rodar um segundo agente sobre esse arquivo e produzir um relatório Markdown com diagnóstico da execução.

---

## validador.py — consistência entre contratos

```python
def validar(caminho_agente):
    # 1. todos os 9 arquivos existem e têm YAML válido?
    for nome, caminho_arquivo in arquivos_obrigatorios.items():
        yaml_data = carregar_yaml_do_md(caminho_arquivo)
        if not yaml_data:
            erros.append(f"[ERRO] {nome} existe mas nao contem YAML valido")

    # 2. ferramentas do toolbox existem em skills?
    for nome in nomes_toolbox - nomes_habilidades:
        erros.append(f"[ERRO] '{nome}' esta no toolbox mas nao em skills")

    # 3. ferramentas obrigatórias (rules) existem em skills?
    for nome in regras.get("ferramentas_obrigatorias", []):
        if nome not in nomes_habilidades:
            erros.append(f"[ERRO] ferramenta obrigatoria '{nome}' nao existe em skills")

    # 4. tipo do agente é válido?
    tipo = agente.get("tipo", "")
    if tipo and tipo not in {"task_based", "interactive", "goal_oriented", "autonomous"}:
        erros.append(f"[ERRO] tipo '{tipo}' invalido")

    # 5. limites por ferramenta (rules) referenciam ferramentas existentes?
    chamadas = regras.get("limites", {}).get("chamadas_ferramenta", {})
    for nome in chamadas:
        if nome != "total" and nome not in nomes_habilidades:
            avisos.append(f"[AVISO] limite para '{nome}' que nao existe em skills")
```

Um agente pode ser sintaticamente correto (YAML válido em todos os arquivos) mas semanticamente inválido:

- `toolbox.md` menciona `rollback_deploy` mas `skills.md` não define essa ferramenta — o runtime chamaria `ferramentas["rollback_deploy"]` e receberia `KeyError`
- `rules.md` declara `relatorio_incidente` como obrigatória mas `skills.md` a renomeou para `criar_relatorio` — o agente nunca conseguiria finalizar
- `rules.md` define `limite.chamadas_ferramenta.consultar_metricas_v2: 3` mas a ferramenta se chama `consultar_metricas` — limite nunca seria aplicado

O validador detecta esses erros **antes da execução**, não durante. Isso é parte do toolkit de developer experience do framework.

---

## main.py — a CLI

```python
python main.py rodar      --agente ../monitor-agent --entrada "alerta de latencia"
python main.py rodar      --agente ../monitor-agent --entrada "alerta" --modo interactive
python main.py rodar      --agente ../monitor-agent --entrada "deploy falhou" \
                          --modo autonomous --evento deploy_falhou
python main.py validar    --agente ../monitor-agent
python main.py rastreamento
python main.py replay     --agente ../monitor-agent
python main.py analisar   --agente ../trace-analyzer
```

### O comando `analisar` — um agente analisando outro

O mais interessante dos cinco comandos:

```python
def main():
    if argumentos.comando == "analisar":
        dados_trace = json.loads(caminho_trace.read_text(encoding="utf-8"))
        entrada_trace = _resumir_trace(dados_trace)  # serializa o trace como texto

        # roda um segundo agente (trace-analyzer) com o trace como entrada
        rodar(
            caminho_agente=argumentos.agente,  # ../trace-analyzer
            texto_entrada=entrada_trace,
            saida=caminho_analise,
        )

        # gera relatório Markdown a partir do trace original + resultado da análise
        dados_analise = json.loads(caminho_analise_path.read_text(encoding="utf-8"))
        relatorio = _gerar_relatorio_md(dados_trace, dados_analise)
        caminho_md.write_text(relatorio, encoding="utf-8")
```

`_resumir_trace` serializa o `trace.json` como texto estruturado, tipo:

```
TRACE_ID: a4f2c1e9b3d0
AGENTE: monitor-agent
TIPO: task_based
ETAPA 1: acao=CHAMAR_FERRAMENTA ferramenta=consultar_metricas sucesso=True qualidade=completa
ETAPA 2: acao=CHAMAR_FERRAMENTA ferramenta=buscar_logs sucesso=True qualidade=completa
...
HEALTH: taxa_sucesso=100% circuit_breaker=0 payload_falhas=0 chamadas_llm=5
```

Esse texto vira a entrada de um segundo agente (`trace-analyzer`) que tem suas próprias skills: `analisar_saude`, `analisar_performance`, `analisar_conformidade`, `detectar_anomalias`, `gerar_veredito`. O resultado é um relatório Markdown com diagnóstico completo da execução.

**Um agente analisando outro. O mesmo runtime. O mesmo ciclo.**

---

## Os 4 modos de operação

| Modo            | Comportamento                                                                              | Quando usar                    |
| --------------- | ------------------------------------------------------------------------------------------ | ------------------------------ |
| `task_based`    | Recebe tarefa, executa, entrega artefato. Sem perguntas.                                   | Incidentes com contexto claro  |
| `interactive`   | Antes de agir, usa `PERGUNTAR_USUARIO` para remover ambiguidades                           | Quando a entrada pode ser vaga |
| `goal_oriented` | Recebe objetivo amplo, decompõe em sub-objetivos, reavalia após cada etapa                 | Análises exploratórias         |
| `autonomous`    | Responde a eventos/triggers com limites rígidos. Nunca executa destrutivas sem confirmação | Automação em produção          |

A diferença está principalmente no `planejador.py`:

- Em `interactive`, o mock retorna `PERGUNTAR_USUARIO` na primeira etapa se não há histórico
- Em `autonomous`, o system prompt inclui instruções específicas sobre o evento trigger e segurança
- Em `goal_oriented`, o system prompt orienta a decomposição em sub-objetivos

O **ciclo é o mesmo** para todos os modos. Só o comportamento da LLM e do planejador muda.

---

## O fluxo completo de uma execução

Ao rodar `python runtime/main.py rodar --agente ../monitor-agent --entrada "alerta de latencia"`, acontece exatamente isso:

```
main.py
  └─ rodar(caminho_agente, texto_entrada)
       │
       ├─ contratos.py: carregar_contratos()
       │    └─ 9x carregar_yaml_do_md() → contratos dict
       │
       ├─ contratos.py: criar_estado()
       │    └─ lê limites de rules.md → estado inicial zerado
       │
       ├─ ferramentas.py: construir_ferramentas_dos_contratos()
       │    └─ 4x construir_ferramenta() → {"consultar_metricas": fn, ...}
       │
       ├─ telemetria.py: Telemetria(agente, tipo)
       │    └─ trace_id gerado
       │
       └─ LOOP (ciclo.py: rodar)
            │
            ├─ verificar limites: tempo / tokens / estagnação
            │
            ├─ [gancho: antes_da_etapa]
            │
            ├─ PERCEBER: planejador.py: perceber(estado)
            │    └─ monta string com alerta + histórico + ferramentas usadas
            │
            ├─ PLANEJAR: planejador.py: chamar_llm(percepcao, contratos)
            │    └─ system prompt gerado → gpt-4o-mini → JSON
            │    └─ acumula tokens no estado
            │
            ├─ CIRCUIT BREAKER: validar_resposta_llm(plano)
            │    └─ valida proxima_acao + ferramenta + argumentos
            │    └─ tenta auto-correção antes de encerrar
            │
            ├─ [verificar ferramentas obrigatórias se FINALIZAR]
            │
            ├─ AGIR:
            │    ├─ verificar limites por ferramenta
            │    ├─ verificar estagnação
            │    ├─ [verificar human-in-the-loop se ação sensível]
            │    ├─ executor.py: validar_payload()
            │    ├─ [gancho: antes_da_acao]
            │    ├─ executor.py: executar() [com retry se configurado]
            │    ├─ [gancho: apos_acao / em_erro]
            │    └─ acumula tokens da ferramenta
            │
            ├─ AVALIAR: executor.py: avaliar(plano, resultado)
            │    └─ validar_saida() → qualidade: completa|parcial|falha
            │
            ├─ atualiza histórico, estado["concluido"]
            ├─ [gancho: apos_etapa]
            └─ exibir_kpis() → painel no terminal
       │
       ├─ gerar_resumo_final()
       ├─ telemetria.resumo_completo()
       └─ salva trace.json
```

---

## Padrões de engenharia de software no runtime

Este framework é um catálogo vivo de padrões clássicos aplicados a agentes. Vale identificar cada um:

<figure style="margin: 2rem 0; text-align: center;">
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 300" style="max-width:760px;width:100%;">
    <rect width="760" height="300" rx="10" fill="#0d1b2a"/>
    <text x="380" y="28" text-anchor="middle" font-family="monospace" font-size="13" fill="#aaa" font-weight="bold">Padrões de Engenharia de Software no Runtime</text>
    <!-- Linha 1 -->
    <rect x="20" y="45" width="160" height="70" rx="8" fill="#0a1520" stroke="#ff4466" stroke-width="1.5"/>
    <text x="100" y="68" text-anchor="middle" font-family="monospace" font-size="10" fill="#ff4466" font-weight="bold">Circuit Breaker</text>
    <text x="100" y="84" text-anchor="middle" font-family="monospace" font-size="9" fill="#882233">validar_resposta_llm()</text>
    <text x="100" y="99" text-anchor="middle" font-family="monospace" font-size="9" fill="#882233">+ auto-correção</text>
    <rect x="200" y="45" width="160" height="70" rx="8" fill="#0a1520" stroke="#00ff88" stroke-width="1.5"/>
    <text x="280" y="68" text-anchor="middle" font-family="monospace" font-size="10" fill="#00ff88" font-weight="bold">State Machine</text>
    <text x="280" y="84" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">estado dict</text>
    <text x="280" y="99" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">etapa → concluido</text>
    <rect x="380" y="45" width="160" height="70" rx="8" fill="#0a1520" stroke="#ffaa00" stroke-width="1.5"/>
    <text x="460" y="68" text-anchor="middle" font-family="monospace" font-size="10" fill="#ffaa00" font-weight="bold">Observer (Hooks)</text>
    <text x="460" y="84" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">executar_gancho()</text>
    <text x="460" y="99" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">declarado em hooks.md</text>
    <rect x="560" y="45" width="180" height="70" rx="8" fill="#0a1520" stroke="#9b4fff" stroke-width="1.5"/>
    <text x="650" y="68" text-anchor="middle" font-family="monospace" font-size="10" fill="#9b4fff" font-weight="bold">Factory + Closure</text>
    <text x="650" y="84" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">construir_ferramenta()</text>
    <text x="650" y="99" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">fn por habilidade</text>
    <!-- Linha 2 -->
    <rect x="20" y="135" width="160" height="70" rx="8" fill="#0a1520" stroke="#00ccff" stroke-width="1.5"/>
    <text x="100" y="158" text-anchor="middle" font-family="monospace" font-size="10" fill="#00ccff" font-weight="bold">Strategy</text>
    <text x="100" y="174" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">4 modos de agente</text>
    <text x="100" y="189" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">planejador_mock()</text>
    <rect x="200" y="135" width="160" height="70" rx="8" fill="#0a1520" stroke="#00ff88" stroke-width="1.5"/>
    <text x="280" y="158" text-anchor="middle" font-family="monospace" font-size="10" fill="#00ff88" font-weight="bold">Template Method</text>
    <text x="280" y="174" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">ciclo.rodar()</text>
    <text x="280" y="189" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">esqueleto fixo</text>
    <rect x="380" y="135" width="160" height="70" rx="8" fill="#0a1520" stroke="#9b4fff" stroke-width="1.5"/>
    <text x="460" y="158" text-anchor="middle" font-family="monospace" font-size="10" fill="#9b4fff" font-weight="bold">Command</text>
    <text x="460" y="174" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">plano: proxima_acao</text>
    <text x="460" y="189" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">+ nome + argumentos</text>
    <rect x="560" y="135" width="180" height="70" rx="8" fill="#0a1520" stroke="#ff4466" stroke-width="1.5"/>
    <text x="650" y="158" text-anchor="middle" font-family="monospace" font-size="10" fill="#ff4466" font-weight="bold">Chain of Responsibility</text>
    <text x="650" y="174" text-anchor="middle" font-family="monospace" font-size="9" fill="#882233">CB → obrig. → limite</text>
    <text x="650" y="189" text-anchor="middle" font-family="monospace" font-size="9" fill="#882233">→ estag. → sensível</text>
    <!-- Linha 3 -->
    <rect x="20" y="225" width="160" height="60" rx="8" fill="#0a1520" stroke="#00ccff" stroke-width="1.5"/>
    <text x="100" y="250" text-anchor="middle" font-family="monospace" font-size="10" fill="#00ccff" font-weight="bold">Repository</text>
    <text x="100" y="266" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">contratos.py abstrai</text>
    <text x="100" y="281" text-anchor="middle" font-family="monospace" font-size="9" fill="#3a7a9a">acesso a .md + YAML</text>
    <rect x="200" y="225" width="160" height="60" rx="8" fill="#0a1520" stroke="#ffaa00" stroke-width="1.5"/>
    <text x="280" y="250" text-anchor="middle" font-family="monospace" font-size="10" fill="#ffaa00" font-weight="bold">Human-in-the-Loop</text>
    <text x="280" y="266" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">acoes_sensiveis</text>
    <text x="280" y="281" text-anchor="middle" font-family="monospace" font-size="9" fill="#aa7700">confirmação antes</text>
    <rect x="380" y="225" width="160" height="60" rx="8" fill="#0a1520" stroke="#00ff88" stroke-width="1.5"/>
    <text x="460" y="250" text-anchor="middle" font-family="monospace" font-size="10" fill="#00ff88" font-weight="bold">Graceful Degradation</text>
    <text x="460" y="266" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">sem API → mock</text>
    <text x="460" y="281" text-anchor="middle" font-family="monospace" font-size="9" fill="#4a9a6a">payload erro → registra</text>
    <rect x="560" y="225" width="180" height="60" rx="8" fill="#0a1520" stroke="#9b4fff" stroke-width="1.5"/>
    <text x="650" y="250" text-anchor="middle" font-family="monospace" font-size="10" fill="#9b4fff" font-weight="bold">Spec-Driven Dev</text>
    <text x="650" y="266" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">contrato muda →</text>
    <text x="650" y="281" text-anchor="middle" font-family="monospace" font-size="9" fill="#6a4a9a">comportamento muda</text>
  </svg>
  <figcaption style="font-size:0.85rem;color:#888;margin-top:0.5rem;">11 padrões de engenharia de software identificados no runtime.</figcaption>
</figure>

### Circuit Breaker

`validar_resposta_llm` + lógica de auto-correção em `ciclo.py`. Protege o executor de receber planos malformados da LLM. Tenta se recuperar antes de abrir o circuito definitivamente.

### State Machine

O `estado` dict é a máquina de estados do agente. Transições: `etapa = 0` → incrementa por iteração → `concluido = True`. As transições são controladas pelos guardrails (`max_etapas`, `sem_progresso`, etc.) e pela avaliação semântica (`objetivo_alcancado`).

### Observer (Hooks)

`executar_gancho` implementa o padrão Observer onde os observadores são declarados em `hooks.md`. O ciclo emite eventos (`antes_da_acao`, `em_erro`, etc.) e o executor notifica os handlers configurados.

### Factory com Closure

`construir_ferramenta` é uma factory que cria funções. O closure captura o prompt de sistema no momento da construção, criando uma função independente por ferramenta.

### Strategy

Os modos de agente (`task_based`, `interactive`, `goal_oriented`, `autonomous`) são estratégias diferentes de planejamento. O `planejador_mock` seleciona a estratégia baseado no tipo.

### Template Method

`ciclo.rodar` define o esqueleto do algoritmo (perceber → planejar → circuit breaker → agir → avaliar) e delega os detalhes para `planejador.py`, `ferramentas.py`, `executor.py`.

### Repository

`contratos.py` é um Repository que abstrai o acesso aos dados de configuração. O ciclo pede `contratos.get("regras", {})` sem saber que esses dados vêm de arquivos Markdown com YAML embarcado.

### Command

Cada plano retornado pela LLM (`{"proxima_acao": "CHAMAR_FERRAMENTA", "nome_ferramenta": "...", "argumentos_ferramenta": {...}}`) é um objeto Command que encapsula a intenção sem saber como ela será executada.

### Chain of Responsibility

A sequência de validações antes da execução: circuit breaker → verifica ferramentas obrigatórias → verifica limite total → verifica limite por ferramenta → verifica estagnação → verifica ação sensível → valida payload. Cada verificação pode interceptar e encerrar a cadeia.

### Spec-Driven Development

A abordagem macro: contratos definem comportamento, código interpreta contratos. Mudança de comportamento = mudança de contrato, não de código.

### Human-in-the-Loop

`pedir_confirmacao_humana` para ações sensíveis. Pausa a execução automatizada e espera input humano antes de continuar. Com `EOFError`, nega por default (fail-safe).

### Graceful Degradation

- Sem API key: `planejador_mock` entra, sem crash
- Payload inválido: registra, não bloqueia
- Ferramenta inexistente na LLM: fallback para a próxima disponível

---

## Da linha de código ao produto

A diferença entre um experimento de LLM e software de produção é exatamente o que este runtime implementa: limites, observabilidade, auditoria, contratos explícitos e tolerância a falhas.

Um agente sem `max_etapas` pode rodar eternamente. Sem `circuit breaker`, a LLM pode enviar lixo para o executor. Sem telemetria, você não sabe o que aconteceu quando algo falha. Sem validação cruzada de contratos, o `relatorio_incidente` obrigatório pode nunca ser chamado por um erro de nome.

Cada guardrail deste runtime existe porque alguém, em algum sistema de produção, sentiu a ausência dele.

O que o runtime entrega é simples de enunciar mas difícil de implementar:

> **Cada linha de YAML que você escreveu tem uma linha de Python que a lê.**

Você declara `tentar_novamente_em_falha: true` e o `executor.py` faz retry. Você declara `em_erro: alerta` e o `executor.py` imprime com destaque. Você declara `relatorio_incidente` como obrigatória e o ciclo nunca deixa o agente finalizar sem ela. Você declara `rollback_deploy` como ação sensível e o loop para e pede sua confirmação.

Isso é o que separa automação de responsabilidade: **comportamento declarado, não implícito**.

---

_Post escrito com base no código do módulo 4 do curso de IA Aplicada. O runtime e os contratos estão disponíveis para estudo — cada arquivo é um ponto de entrada para entender como um agente autônomo realmente funciona._
