# Advanced RAG

## 1. O que é
É o Naive RAG com **camadas de otimização** adicionadas antes, durante e depois do retrieval. Não muda a ideia central (embed → retrieve → generate), mas adiciona etapas que corrigem os pontos fracos do Naive: retrieval impreciso, contexto irrelevante, queries mal formuladas.

Divide-se em três frentes: **pre-retrieval**, **retrieval** e **post-retrieval**.

## 2. Fluxo completo

```
Query → [Pre-retrieval] → Embed → Retrieve (hybrid) → [Post-retrieval: rerank + filter] → Prompt → LLM → Resposta
```

### 2.1 Pre-retrieval (otimizar a query antes de buscar)

**Query Rewriting / Query Expansion**
A query original do usuário é reescrita ou expandida pelo próprio LLM antes de virar embedding. Objetivo: melhorar a chance de match semântico.
- Exemplo: "reembolso 2024?" → reescrita para "Qual é a política de reembolso da empresa para o ano de 2024?"

**Query Decomposition**
Perguntas complexas (multi-hop) são quebradas em sub-perguntas menores, cada uma buscada separadamente.
- Exemplo: "Quem fundou a empresa que comprou a startup X?" vira duas buscas: (1) quem comprou a startup X, (2) quem fundou essa empresa.

**HyDE (Hypothetical Document Embeddings)**
Em vez de usar o embedding da query direto, o LLM gera uma **resposta hipotética** pra pergunta, e essa resposta é que vira o vetor de busca. Funciona porque respostas hipotéticas costumam ser semanticamente mais próximas dos documentos reais do que a pergunta crua.

### 2.2 Retrieval (buscar melhor, não só buscar)

**Hybrid Search**
Combina busca vetorial (**dense retrieval**, semântica) com busca por palavra-chave (**sparse retrieval**, ex: BM25). Um captura significado, o outro captura termos exatos (siglas, nomes próprios, códigos) que embeddings às vezes perdem.

**Metadata Filtering**
Filtra o retrieval por campos estruturados antes ou durante a busca (data, categoria, autor, tipo de documento), reduzindo o espaço de busca e evitando chunks fora de contexto.

### 2.3 Post-retrieval (limpar o que foi buscado, antes de mandar pro LLM)

**Reranking**
Um segundo modelo, geralmente um **cross-encoder** (mais lento e mais preciso que o embedding usado no retrieval), reavalia os chunks recuperados e reordena por relevância real à query. O retrieval inicial traz um conjunto maior (ex: top-50), o reranker reduz pra um conjunto final menor (ex: top-5).

**Compression / Filtering**
Remove partes irrelevantes de dentro dos chunks recuperados (em vez de descartar o chunk inteiro), reduzindo ruído e economizando tokens do contexto.

**Contextual reordering**
Reordena os chunks finais dentro do prompt — alguns modelos dão mais peso ao que está no início/fim do contexto (**lost in the middle**), então chunks mais relevantes são posicionados nessas posições.

## 3. Comparação direta com Naive RAG

| Etapa | Naive RAG | Advanced RAG |
|---|---|---|
| Query | usada crua | reescrita/expandida/decomposta |
| Busca | só vetorial | híbrida (vetorial + keyword) |
| Resultado do retrieval | usado direto | reranqueado e filtrado |
| Quantidade de chunks | top-k fixo, direto pro prompt | top-N amplo → reduzido a top-k relevante |

## 4. Limitações
- Mais etapas = mais latência e mais custo (cada etapa extra é uma chamada de modelo ou processamento adicional).
- Ainda é um pipeline **sequencial fixo**: não decide dinamicamente se deve buscar de novo ou mudar de estratégia — só executa as mesmas etapas de otimização toda vez.
- Reranking com cross-encoder é caro computacionalmente se aplicado a muitos candidatos.

## 5. Quando usar
Bases de conhecimento grandes ou heterogêneas, queries variadas (algumas simples, outras complexas), quando precisão importa mais que latência mínima, e quando Naive RAG já mostrou problemas de recall ou contexto irrelevante.