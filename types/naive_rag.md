# Naive RAG

## 1. O que é
É a implementação mais simples e direta de RAG. Segue um pipeline linear, sequencial, sem etapas de correção, validação ou decisão intermediária. Cada query passa exatamente pelo mesmo caminho, não importa o quão simples ou complexa ela seja.

É o "RAG didático" — o ponto de partida antes de qualquer otimização.

## 2. Fluxo completo

```
Query → Embed + Pool → Retrieve (top-k) → Concatenar contexto → Prompt → LLM → Resposta
```

### 2.1 Indexação (offline, acontece antes de qualquer query)
Essa etapa roda uma vez (ou periodicamente), não a cada pergunta:

1. **Ingestão**: documentos brutos (PDF, HTML, txt, etc.) são coletados.
2. **Chunking**: os documentos são divididos em pedaços menores (**chunks**), geralmente por tamanho fixo (ex: 512 tokens) com sobreposição (**overlap**) entre chunks vizinhos, pra não cortar frases/ideias no meio.
3. **Embedding**: cada chunk passa por um **embedding model** e vira um vetor numérico.
4. **Armazenamento**: os vetores (+ texto original + metadata) são salvos num **vector database**.

### 2.2 Query time (acontece a cada pergunta do usuário)

**Passo 1 — Embed da query**
A pergunta do usuário é transformada em vetor, usando o **mesmo embedding model** usado na indexação (crítico: se usar modelos diferentes, os vetores não são comparáveis).

**Passo 2 — Retrieval (busca)**
O vetor da query é comparado com todos os vetores do banco via uma métrica de similaridade (geralmente **cosine similarity** ou **dot product**). Retorna os **top-k** chunks mais similares (ex: top-5).

**Passo 3 — Augmentation (montagem do prompt)**
Os chunks recuperados são concatenados em texto puro e inseridos no prompt, geralmente com uma instrução tipo:
```
Contexto:
{chunk_1}
{chunk_2}
...

Pergunta: {query}
Responda usando apenas o contexto acima.
```

**Passo 4 — Generation**
O LLM recebe o prompt completo (contexto + pergunta) e gera a resposta em linguagem natural, condicionado ao contexto fornecido.

## 3. Características
- **Sem julgamento**: não avalia se os chunks recuperados são realmente relevantes.
- **Sem correção**: se o retrieval falhar (trouxer chunks irrelevantes), o LLM tenta responder mesmo assim — pode alucinar ou responder errado.
- **Sem iteração**: busca uma vez só, não busca de novo mesmo que o contexto seja insuficiente.
- **k fixo**: o número de chunks recuperados costuma ser definido manualmente e não muda por query.

## 4. Limitações principais
- **Recall baixo em queries complexas**: perguntas que precisam de múltiplas fontes de informação (multi-hop) frequentemente falham, porque o retrieval é feito de uma vez só, baseado apenas na similaridade semântica direta da query original.
- **Sensível a chunking ruim**: se o chunk cortar uma informação no meio, a resposta fica incompleta mesmo que o documento certo tenha sido encontrado.
- **Sem reranking**: a ordem de similaridade bruta pode não refletir relevância real pra responder a pergunta.
- **Contexto ruim = resposta ruim**: não existe camada de proteção contra chunks irrelevantes indo parar no prompt.

## 5. Quando usar
Bom para: bases de conhecimento pequenas/médias, perguntas diretas (fato único, não multi-hop), protótipos e MVPs, ou quando latência/custo são mais prioritários que precisão máxima.