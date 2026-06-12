---
tags: [fundamentos, function-calling, tool-use, módulo]
aliases: [Function Calling, Tool Use, chamada de ferramenta]
módulo: "01"
anterior: "[[00-o-que-e-um-agente]]"
próximo: "[[02-react-loop]]"
---

# 01 — Function Calling
**Fundamentos · v1.0 · junho 2026**

---

## Conceito central

Function calling é **o mecanismo pelo qual um LLM usa ferramentas**.

A ilusão: parece que o LLM "busca na web" ou "lê um arquivo".
A realidade: o LLM produz JSON descrevendo o que quer fazer.
Seu código lê esse JSON e executa a ação real.

```
LLM → { "tool": "busca_web", "args": { "query": "AirHelp pricing" } }
                    ↓
              SEU CÓDIGO
                    ↓
            executa a busca
                    ↓
         devolve resultado pro LLM
```

O LLM **nunca** toca na internet, no sistema de arquivos, ou em nenhuma API.
Ele só produz e consome texto (incluindo JSON estruturado).
Você é o único executor. O LLM é o tomador de decisão.

---

## Como funciona: o protocolo

### 1. Você define as ferramentas disponíveis

Cada ferramenta tem: nome, descrição (para o LLM entender quando usar), e parâmetros.

```json
{
  "name": "busca_web",
  "description": "Busca informações atuais na web. Use quando precisar de dados
                  de concorrentes, preços de mercado, ou fatos que podem ter mudado.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "O que buscar"
      }
    },
    "required": ["query"]
  }
}
```

A **descrição** é crítica. O LLM lê isso para decidir quando e como usar a ferramenta.
Uma descrição vaga = ferramenta mal usada.

### 2. Você envia pro LLM com as ferramentas declaradas

```python
resposta = anthropic.messages.create(
    model="claude-opus-4",
    messages=[{"role": "user", "content": "Pesquise os top 3 concorrentes de..."}],
    tools=[busca_web_definition, ler_artefato_definition]  # ferramentas disponíveis
)
```

### 3. O LLM responde com uso de ferramenta (não texto)

```json
{
  "type": "tool_use",
  "name": "busca_web",
  "input": { "query": "AirHelp pricing model 2026" }
}
```

### 4. Seu código executa e devolve o resultado

```python
if resposta.stop_reason == "tool_use":
    tool_name = resposta.content[0].name      # "busca_web"
    tool_input = resposta.content[0].input    # { "query": "..." }
    
    resultado = executar_ferramenta(tool_name, tool_input)
    
    # devolve pro LLM continuar raciociando
    messages.append({"role": "assistant", "content": resposta.content})
    messages.append({
        "role": "user",
        "content": [{
            "type": "tool_result",
            "tool_use_id": resposta.content[0].id,
            "content": resultado
        }]
    })
```

### 5. O LLM continua (ou para)

Se o LLM tem o suficiente, responde com texto normal (`stop_reason == "end_turn"`).
Se precisa de mais dados, faz outra chamada de ferramenta. O loop continua.

---

## Anthropic API — function calling em detalhe

A Anthropic chama isso de **Tool Use**. Está disponível em todos os modelos Claude.

**Definindo uma ferramenta:**
```python
tools = [
    {
        "name": "busca_web",
        "description": "Busca informações atuais na web.",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Termos de busca"}
            },
            "required": ["query"]
        }
    }
]
```

**Enviando e tratando a resposta:**
```python
import anthropic

client = anthropic.Anthropic(api_key="sk-ant-...")

def run_with_tools(prompt, tools, executar_ferramenta):
    messages = [{"role": "user", "content": prompt}]
    
    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )
        
        # LLM terminou — retorna resposta final
        if response.stop_reason == "end_turn":
            return response.content[0].text
        
        # LLM quer usar uma ferramenta
        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})
            
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    resultado = executar_ferramenta(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(resultado)
                    })
            
            messages.append({"role": "user", "content": tool_results})
```

**Nota importante**: Claude pode chamar múltiplas ferramentas numa mesma rodada
(parallel tool use). Seu código precisa tratar isso.

---

## Apple Foundation Models — function calling em Swift

A Apple introduziu tool calling no framework `FoundationModels` com Apple Intelligence.
A API é diferente em estilo (Swift, declarativa) mas o mecanismo é o mesmo.

```swift
import FoundationModels

// 1. Defina a ferramenta
struct BuscaWebTool: Tool {
    static let name = "busca_web"
    static let description = "Busca informações atuais na web sobre concorrentes e mercado."
    
    struct Parameters: Codable {
        let query: String
    }
    
    func call(parameters: Parameters) async throws -> ToolOutput {
        // Aqui vai o código real de busca
        let resultados = try await buscarNaWeb(query: parameters.query)
        return ToolOutput(string: resultados)
    }
}

// 2. Configure a sessão com a ferramenta
let session = LanguageModelSession(tools: [BuscaWebTool()])

// 3. Execute — o framework gerencia o loop automaticamente
let resposta = try await session.respond(
    to: "Pesquise os top 3 concorrentes de apps de direitos de passageiros no Brasil"
)

// 4. O resultado já é a resposta final (o framework fez o loop internamente)
print(resposta.content)
```

**Diferença importante**: `LanguageModelSession` da Apple gerencia o loop ReAct
internamente. Você não precisa escrever o `while True`. Mas também tem menos controle
sobre o que acontece em cada iteração.

Com Claude/Anthropic API, você escreve o loop — mais trabalho, mais controle.
Com Foundation Models, o loop é abstraído — menos código, menos visibilidade.

---

## A descrição da ferramenta é um prompt

Esse é o ponto mais subestimado de function calling.

O LLM decide **quando** e **como** usar cada ferramenta baseado na descrição.
Uma descrição boa = ferramenta bem usada. Uma descrição ruim = chamadas erradas.

**Ruim:**
```json
{
  "name": "busca",
  "description": "Busca coisas"
}
```

**Bom:**
```json
{
  "name": "busca_web",
  "description": "Busca informações atuais na internet. Use SOMENTE quando precisar
                  de dados que podem ter mudado (preços, features de concorrentes,
                  estatísticas de mercado, notícias recentes). NÃO use para
                  informações que você já tem no contexto da conversa."
}
```

O "NÃO use para..." é tão importante quanto o "use quando". Define o escopo preciso.

---

## Mapeamento ao FounderLens

### Ferramentas que o FounderLens vai precisar

| Ferramenta | Usado por | Descrição |
|---|---|---|
| `busca_web(query)` | Benchmark Agent, Market Sizing Agent | Busca dados reais de concorrentes e mercado |
| `ler_contexto_projeto()` | Todos os auditores | Lê o estado atual do projeto (fases concluídas) |
| `ler_artefato(nome)` | Development Auditor | Lê uma spec específica (dados, backend, frontend) |
| `avaliar_rubrica(criterios, dados)` | Validation Auditor | Aplica scoring estruturado |

### O Benchmark Agent com function calling real

```
Ferramentas disponíveis: [busca_web]

Prompt do sistema:
"Você é um analista de mercado. Dado o problema do founder,
 identifique os top 3 concorrentes, extraia features + preço + por que ganharam,
 e sintetize o gap não coberto por nenhum deles.
 Use busca_web para cada concorrente. Nunca invente dados — se não encontrar, diga."

Turno 1 — usuário: "[contexto do founder: app de cancelamento de voo]"

Turno 2 — LLM chama: busca_web("apps direitos passageiros cancelamento voo Brasil 2026")
Turno 3 — resultado da busca devolvido
Turno 4 — LLM chama: busca_web("AirHelp features pricing model")
Turno 5 — resultado devolvido
Turno 6 — LLM chama: busca_web("Resolvvi Liberfly pricing comparison")
Turno 7 — resultado devolvido
Turno 8 — LLM retorna: síntese estruturada do benchmark
```

Isso é code que você vai escrever. Swift chamando Anthropic API (Opção B) ou
Foundation Models (Opção C). O mecanismo é idêntico; só a API muda.

---

## Erros comuns em function calling

**1. Não tratar parallel tool use**
Claude pode chamar múltiplas ferramentas de uma vez. Se seu código só trata
a primeira, perde chamadas.

**2. Não incluir o resultado da ferramenta no histórico**
O LLM precisa ver o resultado para continuar. Esquecer de adicionar ao `messages`
faz o loop travar ou alucinar.

**3. Ferramentas com nomes genéricos demais**
`search`, `get`, `fetch` — o LLM não sabe quando usar qual. Nomes descritivos
e descriptions precisas resolvem.

**4. Loop sem limite**
Um agente pode chamar ferramentas indefinidamente. Sempre defina um `max_iterations`
e trate o caso de timeout.

**5. Não validar o input da ferramenta**
O LLM pode passar argumentos inválidos. Valide sempre antes de executar.

---

## Ponto de checagem

1. Se o LLM "quer buscar na web", o que ele realmente produz?
2. O que acontece com esse output no seu código?
3. Por que a descrição de uma ferramenta é um prompt?
4. Qual a diferença entre Apple Foundation Models e Anthropic API em relação
   ao loop de function calling?

---

## Próximo

→ **[02 — O loop ReAct na prática](./02-react-loop.md)**
Como construir o loop completo em código, gerenciar estado entre iterações,
definir condições de parada, e tratar erros. O Benchmark Agent do FounderLens
implementado passo a passo.

## Relacionados

- [[tool-use-id]] — o campo que liga tool_use a tool_result
- [[tool-description-as-prompt]] — a descrição é um prompt para o LLM
- [[react-pattern]] — o ciclo onde function calling acontece
- [[loop-explicito-vs-abstraido]] — quem escreve o loop (você ou o framework)
- [[benchmark-agent]] — ferramenta `busca_web` em uso real
- [[MOC-agentes]] — mapa completo do grafo


---

*Última atualização: junho 2026*
