# Graph RAG

## 1. O que é
É a variante de RAG onde a base de conhecimento não é só um conjunto de vetores soltos, mas um **grafo de conhecimento (Knowledge Graph)**: nós representam **entidades** (pessoas, empresas, conceitos) e arestas representam **relações** entre elas (ex: "Empresa X" —adquiriu→ "Empresa Y").

O objetivo é resolver o principal ponto fraco do RAG vetorial puro: perguntas que dependem de **relações estruturais entre entidades**, não só de similaridade semântica de texto.

## 2. Fluxo completo

```
Documentos → Extração de entidades/relações → Construção do grafo
                                                        ↓
Query → Identificar entidades na query → Traversal no grafo → Recuperar subgrafo relevante → Prompt → LLM → Resposta
```

### 2.1 Construção do grafo (offline, acontece na indexação)

**Extração de entidades (Named Entity Recognition)**
O texto dos documentos é processado (geralmente por um LLM) para identificar entidades: pessoas, organizações, locais, conceitos, datas.

**Extração de relações**
O mesmo processo identifica como essas entidades se relacionam entre si, gerando triplas do tipo `(entidade_A, relação, entidade_B)`.
- Exemplo: `(Empresa X, adquiriu, Startup Y)`, `(Startup Y, fundada_por, João)`.

**Construção do grafo**
Essas triplas viram nós (entidades) e arestas (relações) num banco de grafo (ex: Neo4j). Cada nó pode carregar também um embedding, permitindo busca híbrida (vetorial + estrutural).

**Comunidades / clusters (opcional, técnica usada no Microsoft GraphRAG)**
O grafo é dividido em subgrupos de entidades fortemente conectadas (**community detection**), e um resumo (**summary**) é gerado pra cada comunidade — permitindo responder perguntas gerais sobre o dataset inteiro, não só sobre entidades específicas.

### 2.2 Query time

**Passo 1 — Entity linking**
A query é analisada e as entidades mencionadas nela são identificadas e ligadas aos nós correspondentes no grafo.

**Passo 2 — Graph traversal**
A partir dos nós identificados, o sistema navega pelas arestas do grafo (1 ou mais "saltos" — **multi-hop traversal**) pra encontrar entidades e relações conectadas relevantes à pergunta.
- Exemplo: pergunta "Quem fundou a empresa que a Empresa X comprou?" → parte do nó "Empresa X" → segue aresta "adquiriu" → chega em "Startup Y" → segue aresta "fundada_por" → chega em "João".

**Passo 3 — Recuperação do subgrafo**
O sistema extrai o subgrafo relevante (nós + relações + textos de origem associados) em vez de chunks de texto soltos.

**Passo 4 — Augmentation + Generation**
O subgrafo é serializado em texto (ex: lista de triplas ou descrição narrativa) e inserido no prompt junto com a query. O LLM gera a resposta usando essa estrutura relacional como contexto.

## 3. Diferença central vs RAG vetorial (Naive/Advanced)

| Aspecto | RAG vetorial | Graph RAG |
|---|---|---|
| Unidade de busca | chunk de texto | entidade + relação |
| Tipo de busca | similaridade semântica | traversal estrutural (+ opcionalmente semântica) |
| Bom para | perguntas factuais diretas | perguntas multi-hop, relacionais, "conecte A com B" |
| Visão do dataset | fragmentada (chunks isolados) | conectada (relações explícitas entre fatos) |

## 4. Limitações
- **Custo de construção alto**: extrair entidades/relações de todo o corpus geralmente exige múltiplas chamadas de LLM — caro e lento pra atualizar.
- **Qualidade depende da extração**: se a extração de entidades/relações for ruim ou incompleta, o grafo fica com buracos e a busca falha.
- **Manutenção complexa**: adicionar novos documentos exige atualizar o grafo (não é só inserir um vetor novo).
- **Overkill para perguntas simples**: se a pergunta não depende de relações entre entidades, o custo extra não compensa.

## 5. Quando usar
Domínios com forte estrutura relacional (jurídico, biomedicina, compliance, redes societárias, investigação), perguntas multi-hop ("quem está conectado com quem"), e casos onde entender **como** os fatos se relacionam importa tanto quanto os fatos em si.