# Agentic RAG

## 1. O que é
É a variante de RAG onde o **LLM atua como agente**: em vez de seguir um pipeline fixo (embed → retrieve → generate), ele **decide dinamicamente** cada etapa — se precisa buscar, onde buscar, quantas vezes buscar, e quando parar. O modelo tem acesso a **ferramentas (tools)** e raciocina sobre qual usar e quando.

É o único tipo de RAG com **loop de decisão e correção**, em vez de execução linear.

## 2. Fluxo completo

```
Query → Agente avalia → [decide: responder direto | usar tool] 
                                    ↓
                          Tool call (retrieval, web search, SQL, API, etc.)
                                    ↓
                          Agente avalia resultado → [suficiente? → responde | insuficiente? → nova tool call]
                                    ↓
                                Resposta final
```

Esse ciclo (avaliar → agir → observar → reavaliar) é geralmente implementado com o padrão **ReAct (Reasoning + Acting)**.

### 2.1 Etapa de decisão (routing)
Antes de qualquer busca, o agente analisa a query e decide:
- Responde direto (não precisa de contexto externo)?
- Precisa buscar? Se sim, em qual fonte?

Isso é feito com um **LLM como router**, que classifica a intenção da pergunta e escolhe entre múltiplas ferramentas disponíveis.

### 2.2 Ferramentas disponíveis (tools)
Diferente dos outros tipos de RAG (que têm uma única fonte: o vector DB), o Agentic RAG pode ter várias fontes registradas como ferramentas:
- **Vector search** (retrieval clássico)
- **Web search** (informação em tempo real)
- **SQL query** (dados estruturados em banco relacional)
- **API calls** (sistemas externos, ex: CRM, calendário)
- **Graph traversal** (se combinado com Graph RAG)

Cada ferramenta tem uma descrição que o agente usa pra decidir se ela é relevante pra query atual.

### 2.3 Execução da tool call
O agente formata os parâmetros necessários (ex: a query de busca, filtros) e invoca a ferramenta escolhida. O resultado retorna como **observação (observation)**.

### 2.4 Avaliação do resultado (self-reflection / grading)
Aqui está a diferença central do Agentic RAG: o agente **julga** se o resultado recuperado é suficiente pra responder a pergunta.
- Se **sim**: segue pra geração da resposta final.
- Se **não**: pode reformular a query e buscar de novo, trocar de ferramenta, ou decompor a pergunta em sub-perguntas e repetir o ciclo pra cada uma.

Esse é o mecanismo que falta no Naive e no Advanced RAG — ambos buscam uma vez e seguem em frente, mesmo se o resultado for ruim.

### 2.5 Geração final
Depois de reunir contexto suficiente (de uma ou várias iterações/fontes), o agente monta o prompt final e gera a resposta.

## 3. Arquiteturas comuns

**Single-agent**
Um agente só, com várias ferramentas disponíveis, decidindo tudo sozinho em loop.

**Multi-agent**
Vários agentes especializados (ex: um agente de retrieval vetorial, um agente de SQL, um agente de web search), coordenados por um **agente orquestrador** que decide qual sub-agente acionar e agrega os resultados.

## 4. Comparação com os outros tipos

| Aspecto | Naive/Advanced RAG | Graph RAG | Agentic RAG |
|---|---|---|---|
| Decisão de buscar | sempre busca | sempre busca | decide se busca |
| Número de fontes | uma (vector DB) | uma (grafo) | múltiplas (tools) |
| Iteração | nenhuma | nenhuma | busca de novo se necessário |
| Avaliação do resultado | não existe | não existe | agente julga suficiência |
| Fluxo | linear | linear | loop (ReAct) |

## 5. Limitações
- **Latência alta**: múltiplas iterações e chamadas de LLM (uma pra decidir, uma pra avaliar, uma pra responder) tornam o tempo de resposta bem maior.
- **Custo alto**: cada iteração do loop é uma chamada de modelo adicional.
- **Comportamento menos previsível**: como o agente decide dinamicamente o caminho, é mais difícil garantir consistência entre execuções da mesma pergunta.
- **Risco de loop longo/ineficiente**: sem um limite de iterações bem definido, o agente pode ficar buscando repetidamente sem convergir.

## 6. Quando usar
Perguntas que exigem múltiplas fontes de dados heterogêneas, tarefas complexas de pesquisa (multi-step reasoning), sistemas que precisam de informação em tempo real combinada com base interna, e quando a qualidade da resposta importa mais que velocidade/custo.