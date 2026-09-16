# RAG

## 1. O que é
**RAG (Retrieval-Augmented Generation)**: técnica que combina um sistema de **busca/recuperação de informação (retrieval)** com um **LLM (Large Language Model)**, para gerar respostas com base em documentos externos em vez de depender só do conhecimento memorizado nos pesos do modelo (**parametric knowledge**).

**Fluxo básico:**
1. Usuário faz uma pergunta (**query**)
2. Sistema busca os trechos mais relevantes numa base de dados (**retrieval**)
3. Esses trechos viram **contexto**, inseridos no prompt (**augmentation**)
4. O LLM gera a resposta usando esse contexto (**generation**)

**Exemplo prático:**
Pergunta: *"Qual a política de reembolso da empresa X em 2024?"*
- Sem RAG: o modelo responde do que aprendeu no treino (pode estar desatualizado ou alucinar).
- Com RAG: o sistema busca o PDF/manual da empresa, recupera o trecho sobre reembolso de 2024, injeta esse trecho no prompt, e o modelo responde baseado nele — reduzindo **alucinação** e permitindo conhecimento atualizado/específico sem re-treinar o modelo.

Analogia: é como um aluno fazendo prova de consulta — em vez de responder só de memória, ele procura no material a informação exata antes de escrever a resposta.

## 2. Etapas
Todo RAG tem essas etapas; os "tipos" de RAG mudam apenas o *meio técnico* de executar cada uma.

### 2.1 ETAPA 1 — QUANDO BUSCAR (Retrieval Trigger)
Decide **se** é necessário buscar contexto externo antes de responder.
- Naive RAG: sempre busca, para toda query.
- Agentic RAG: o modelo (**agente**) decide dinamicamente se busca ou responde direto, baseado na necessidade.

### 2.2 ETAPA 2 — O QUE BUSCAR (Retrieval)
1. A query é transformada em um **embedding** (vetor numérico).
2. Esse vetor é comparado com os embeddings da base (**similarity search**, geralmente por **cosine similarity**).
3. Retorna os **chunks** (pedaços de texto) mais similares semanticamente.

### 2.3 ETAPA 3 — QUANDO PARAR (Stopping Criterion)
Não é uma regra universal, depende da implementação. Métodos mais comuns:
- **Top-k**: pega os k chunks com maior score de similaridade.
- **Threshold**: corta por um score mínimo de similaridade.
- **Reranking + cutoff**: reordena os resultados com um modelo secundário (**reranker**) e corta os piores.
- **Budget de contexto**: limita pela janela de tokens disponível do modelo.

Não existe "parar quando a resposta estiver certa" — é sempre um critério de quantidade/score definido antes, não uma checagem de qualidade da resposta.

## 3. Vector Databases
Cada chunk de texto é convertido num **embedding**: vetor denso que representa seu significado semântico, gerado por um **embedding model** (ex: BERT, OpenAI embeddings).

**Pooling**: como os vetores de tokens individuais (ex: cada palavra) são condensados em **um único vetor** por chunk:
- **Mean pooling**: média de todos os vetores de token.
- **CLS pooling**: usa o vetor de um token especial que resume a sequência.

O **vector database** (ex: Pinecone, Weaviate, FAISS) armazena esses vetores como pares chave-valor, similar a um dicionário: `{id: chunk} → {embedding, metadata, texto original}`. A busca compara o vetor da query com todos os vetores armazenados e retorna os mais próximos no espaço vetorial.

## 4. Tipos

- **Naive RAG**: pipeline direto — embed → retrieve (top-k) → generate. Sem correção ou julgamento intermediário.
- **Advanced RAG**: adiciona etapas de otimização — **query rewriting**, **reranking**, filtros de **metadata**, **hybrid search** (combina busca vetorial + busca por keyword/BM25).
- **Graph RAG**: a base de conhecimento é um **grafo de conhecimento (knowledge graph)**, com entidades e relações explícitas, não só vetores soltos. A busca considera conexões estruturais, não só similaridade semântica.
- **Agentic RAG**: um **agente** (LLM com capacidade de decisão e uso de ferramentas) decide dinamicamente se busca, onde busca (múltiplas fontes/ferramentas), e pode iterar (buscar de novo) se o contexto recuperado for insuficiente.