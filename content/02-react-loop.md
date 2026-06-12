---
tags: [fundamentos, react, loop, estado, módulo]
aliases: [Loop ReAct, loop de agente, loop explícito]
módulo: "02"
anterior: "[[01-function-calling]]"
próximo: "[[03-langchain-patterns]]"
---

# 02 — O loop ReAct na prática
**Fundamentos · v1.0 · junho 2026**

---

## Conceito central

Você já sabe o que é o loop. Agora vai ver ele em código real.

Este doc constrói o **Benchmark Agent do FounderLens do zero** — o agente que executa
o ingrediente 10 do Discover (pesquisar concorrentes, extrair features/preço/gap).

Ao final você terá visto:
- O loop completo com estado acumulado
- Como o agente decide quando parar
- Como tratar erros e timeouts
- Como o mesmo padrão fica em Python vs Swift
- Como o resultado é salvo no `project_state.json`

---

## O loop em pseudocódigo anotado

```python
def run_agent(objetivo, ferramentas, max_iteracoes=10):
    
    # ESTADO DE CURTO PRAZO
    # Tudo que o LLM precisa ver para raciocinar
    historico = [
        {"role": "user", "content": objetivo}
    ]
    
    iteracao = 0
    
    while iteracao < max_iteracoes:
        iteracao += 1
        
        # LLM raciocina com base em tudo que aconteceu até agora
        resposta = llm.call(
            messages=historico,
            tools=ferramentas
        )
        
        # CONDIÇÃO DE PARADA 1: LLM decidiu que terminou
        if resposta.stop_reason == "end_turn":
            return {"sucesso": True, "resultado": resposta.text}
        
        # CONDIÇÃO DE PARADA 2: LLM quer usar ferramenta
        if resposta.stop_reason == "tool_use":
            
            # Adiciona a decisão do LLM ao histórico
            historico.append({"role": "assistant", "content": resposta.content})
            
            resultados_ferramentas = []
            
            for chamada in resposta.tool_calls:
                try:
                    # SEU CÓDIGO executa a ação real
                    resultado = executar_ferramenta(chamada.nome, chamada.argumentos)
                    
                    resultados_ferramentas.append({
                        "tool_use_id": chamada.id,   # ← ID crítico para o LLM rastrear
                        "content": resultado,
                        "status": "sucesso"
                    })
                    
                except Exception as e:
                    # TRATAMENTO DE ERRO: devolve o erro pro LLM, não quebra o loop
                    resultados_ferramentas.append({
                        "tool_use_id": chamada.id,
                        "content": f"Erro: {str(e)}. Tente uma abordagem diferente.",
                        "status": "erro"
                    })
            
            # Adiciona os resultados ao histórico para o LLM ver na próxima rodada
            historico.append({"role": "user", "content": resultados_ferramentas})
    
    # CONDIÇÃO DE PARADA 3: atingiu limite de iterações
    return {"sucesso": False, "erro": "Limite de iterações atingido", "historico": historico}
```

**Três condições de parada:**
1. LLM retorna resposta final (`end_turn`) → sucesso
2. LLM continua pedindo ferramentas mas você limita as iterações → falha controlada
3. Erro não recuperável → você decide se retry ou abort

---

## O Benchmark Agent do FounderLens — completo

### Sistema e objetivo

```python
SYSTEM_PROMPT = """
Você é um analista de mercado especializado em produtos digitais.

Sua tarefa: dado o problema e contexto de um founder, identifique os top 3 concorrentes
diretos, extraia para cada um: (1) principais features, (2) modelo de preço,
(3) por que ganharam no mercado. Depois sintetize: qual gap nenhum deles cobre?

Regras:
- Use busca_web para cada concorrente. Nunca invente dados.
- Se não encontrar um fato, diga "não encontrado" — nunca assuma.
- Priorize concorrentes ativos no mercado-alvo do founder.
- O gap deve ser específico, não genérico ("melhor UX" não é gap).
- Responda em JSON estruturado conforme o schema fornecido.
"""

SCHEMA_RESPOSTA = """
{
  "concorrentes": [
    {
      "nome": "string",
      "features_principais": ["string"],
      "modelo_preco": "string",
      "por_que_ganharam": "string"
    }
  ],
  "gap_identificado": "string",
  "sinal_regulatorio": "string ou null"
}
"""

def criar_objetivo_benchmark(contexto_founder):
    return f"""
Contexto do founder:
{contexto_founder}

Pesquise os top 3 concorrentes e retorne no schema:
{SCHEMA_RESPOSTA}
"""
```

### A ferramenta de busca

```python
import httpx  # ou qualquer lib HTTP

async def buscar_web(query: str) -> str:
    """
    Na prática: Brave Search API, Serper, ou similar.
    O founder usa a própria chave (BYOK) ou passa pelo servidor do FounderLens.
    """
    response = await httpx.get(
        "https://api.search.brave.com/res/v1/web/search",
        headers={"X-Subscription-Token": BRAVE_API_KEY},
        params={"q": query, "count": 5}
    )
    resultados = response.json()["web"]["results"]
    
    # Formata para o LLM — texto limpo, não JSON bruto
    return "\n\n".join([
        f"**{r['title']}**\n{r['description']}\nURL: {r['url']}"
        for r in resultados[:5]
    ])

FERRAMENTAS = [
    {
        "name": "busca_web",
        "description": """Busca informações atuais na web.

Use quando precisar de:
- Features e detalhes de concorrentes específicos
- Preços e modelos de monetização
- Tamanho de mercado e dados de adoção
- Notícias recentes sobre o setor

NÃO use para: informações que já estão no contexto do projeto.""",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Termos de busca específicos. Inclua o país/mercado quando relevante."
                }
            },
            "required": ["query"]
        }
    }
]
```

### O agente completo

```python
import anthropic
import json

client = anthropic.Anthropic(api_key=ANTHROPIC_KEY)

async def executar_benchmark_agent(contexto_founder: str) -> dict:
    
    objetivo = criar_objetivo_benchmark(contexto_founder)
    
    historico = [{"role": "user", "content": objetivo}]
    
    MAX_ITERACOES = 8  # 3 concorrentes × ~2 buscas cada + margem
    
    for i in range(MAX_ITERACOES):
        
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=SYSTEM_PROMPT,
            tools=FERRAMENTAS,
            messages=historico
        )
        
        # ── Agente terminou ──────────────────────────────────────────────
        if response.stop_reason == "end_turn":
            texto = next(b.text for b in response.content if hasattr(b, 'text'))
            
            try:
                resultado = json.loads(texto)
                return {"sucesso": True, "benchmark": resultado, "iteracoes": i + 1}
            except json.JSONDecodeError:
                # LLM não retornou JSON válido — pede de novo
                historico.append({"role": "assistant", "content": response.content})
                historico.append({
                    "role": "user",
                    "content": "Retorne APENAS o JSON, sem texto adicional."
                })
                continue
        
        # ── Agente quer usar ferramenta ──────────────────────────────────
        if response.stop_reason == "tool_use":
            historico.append({"role": "assistant", "content": response.content})
            
            tool_results = []
            
            for block in response.content:
                if block.type != "tool_use":
                    continue
                    
                if block.name == "busca_web":
                    try:
                        resultado = await buscar_web(block.input["query"])
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": resultado
                        })
                    except Exception as e:
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": f"Busca falhou: {e}. Tente uma query diferente."
                        })
            
            historico.append({"role": "user", "content": tool_results})
    
    return {"sucesso": False, "erro": "Benchmark não concluído no limite de iterações"}
```

### Salvando no project_state.json

```python
async def executar_e_salvar_benchmark(contexto_founder: str, project_state_path: str):
    
    resultado = await executar_benchmark_agent(contexto_founder)
    
    if not resultado["sucesso"]:
        raise Exception(f"Benchmark falhou: {resultado['erro']}")
    
    # Lê o state atual
    with open(project_state_path, "r") as f:
        state = json.load(f)
    
    # Adiciona o benchmark ao estado do projeto
    state["discover"]["benchmark"] = resultado["benchmark"]
    state["discover"]["benchmark_iteracoes"] = resultado["iteracoes"]
    state["discover"]["benchmark_status"] = "concluido"
    
    # Salva de volta
    with open(project_state_path, "w") as f:
        json.dump(state, f, indent=2, ensure_ascii=False)
    
    return resultado["benchmark"]
```

---

## O mesmo padrão em Swift (para Apple Foundation Models)

```swift
import FoundationModels

// A ferramenta
struct BuscaWebTool: Tool {
    static let name = "busca_web"
    static let description = """
        Busca informações atuais na web sobre concorrentes, preços e mercado.
        Use para cada concorrente identificado. Nunca invente dados.
        """
    
    struct Parameters: Codable {
        let query: String
    }
    
    func call(parameters: Parameters) async throws -> ToolOutput {
        let resultados = try await BraveSearchAPI.search(query: parameters.query)
        return ToolOutput(string: resultados.formatado)
    }
}

// O agente
class BenchmarkAgent {
    
    private let session: LanguageModelSession
    
    init() {
        self.session = LanguageModelSession(
            tools: [BuscaWebTool()],
            instructions: """
                Você é um analista de mercado. Dado o contexto do founder,
                identifique top 3 concorrentes com features, preço e gap.
                Retorne JSON estruturado. Nunca invente dados.
                """
        )
    }
    
    func executar(contextoFounder: String) async throws -> BenchmarkResult {
        
        // Foundation Models gerencia o loop internamente
        let resposta = try await session.respond(
            to: "Pesquise: \(contextoFounder)\n\nRetorne o schema JSON."
        )
        
        // Decodifica o resultado
        let json = try JSONDecoder().decode(BenchmarkResult.self, from: resposta.content.data(using: .utf8)!)
        return json
    }
}
```

**Diferença crítica em Swift**: o `LanguageModelSession` roda o loop ReAct internamente.
Você não vê as iterações. Pro FounderLens isso pode ser aceitável para agentes simples,
mas para o Benchmark Agent você vai querer **visibilidade** — saber quantas buscas foram
feitas, o que encontrou, onde falhou. Isso favorece escrever o loop explicitamente
(mais próximo do exemplo Python) mesmo em Swift, usando a API da Anthropic diretamente.

---

## Padrões importantes que este agente demonstra

### 1. O sistema prompt define o contrato de saída

O agente retorna JSON estruturado porque o system prompt exige. Isso é essencial
para o FounderLens: o resultado do Benchmark Agent vai alimentar o Validation Auditor —
os dois precisam falar a mesma língua. Schema explícito no prompt = output previsível.

### 2. Erros devolvidos pro LLM, não quebram o loop

Quando uma busca falha, o agente devolve o erro como `tool_result`. O LLM vê o erro,
adapta a query, tenta de novo. Um agente resiliente não para no primeiro erro —
ele informa o LLM e deixa ele decidir o próximo passo.

### 3. `max_iteracoes` é segurança, não limite de qualidade

Você não quer que um agente rode indefinidamente (custo, tempo). Mas o limite deve ser
generoso o suficiente para o caso de uso. Para o Benchmark Agent: 3 concorrentes ×
~2 buscas = 6 iterações mínimas. Limite de 8-10 dá margem sem ser perigoso.

### 4. O contexto acumulado é o "raciocínio" do agente

O LLM na iteração 4 vê tudo: o objetivo original, os resultados das 3 buscas anteriores,
e os tool_use_ids das chamadas. Sem esse histórico, cada iteração seria cega.
Por isso o histórico cresce a cada rodada — e por isso agentes longos ficam caros.

### 5. Agentes fazem um trabalho → passam para o próximo

O Benchmark Agent não faz o scoring. Ele produz o benchmark e salva.
O Validation Auditor lê o benchmark salvo e faz o scoring.
Cada agente tem responsabilidade única. Isso é o padrão que você vai ver no CrewAI
(doc 04) formalizado como Tasks.

---

## Trace de execução real (para o app de cancelamento de voo)

```
[iter 1] Thought: vou buscar o mercado geral primeiro
         Action: busca_web("apps direitos passageiros cancelamento voo Brasil 2026")
         Obs: AirHelp, Resolvvi, LiberFly aparecem nos resultados

[iter 2] Thought: AirHelp é o maior. Vou detalhar
         Action: busca_web("AirHelp features pricing review 2026")
         Obs: 35% success fee, 3M+ passengers, 12 anos, app de tracking

[iter 3] Thought: preciso da Resolvvi — foco em Brasil
         Action: busca_web("Resolvvi app passageiros Brasil preço features")
         Obs: 35% success fee, análise grátis, R$130M recuperados, foco digital

[iter 4] Thought: preciso da LiberFly — modelo diferente (express)
         Action: busca_web("LiberFly express modelo preço R$1000 48h")
         Obs: express paga R$1000 em 48h, tradicional 30% success fee

[iter 5] Thought: tenho os 3. Vou sintetizar o gap.
         stop_reason: end_turn
         
Output: {
  "concorrentes": [...],
  "gap_identificado": "Todos atuam APÓS o fato — nenhum ajuda no momento do cancelamento",
  "sinal_regulatorio": "OAB notificou Resolvvi e LiberFly em 2020 por publicidade irregular"
}
```

5 iterações. Resultado fundamentado. Zero invenção.

---

## Ponto de checagem

1. Por que o histórico de mensagens precisa incluir tanto o `tool_use` do LLM quanto o
   `tool_result` do seu código?
2. O que acontece se o LLM retornar JSON inválido na resposta final? Como o agente
   acima trata isso?
3. Por que o `max_iteracoes` do Benchmark Agent é 8 e não 3?
4. Qual a diferença de visibilidade entre usar `LanguageModelSession` da Apple e
   escrever o loop explicitamente? Por que isso importa para o FounderLens?

---

## Próximo

→ **[03 — LangChain patterns](./03-langchain-patterns.md)**
O vocabulário que todo artigo, tutorial e Stack Overflow de agentes usa. Chains, Tools,
Memory, Callbacks. Você vai reconhecer tudo que aprendeu aqui, com nomes formais.

## Relacionados

- [[react-pattern]] — o padrão que este doc implementa
- [[tool-use-id]] — usado em cada iteração do loop
- [[max-iterations]] — o `MAX_ITERACOES = 8` explicado
- [[state-memoria]] — o `historico` (curto prazo) + `project_state.json` (longo prazo)
- [[callbacks]] — o próximo passo: adicionar observabilidade a este loop
- [[loop-explicito-vs-abstraido]] — por que este estilo é preferido no FounderLens
- [[benchmark-agent]] — o agente completo construído neste doc
- [[MOC-agentes]] — mapa completo do grafo


---

*Última atualização: junho 2026*
