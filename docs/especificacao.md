# Especificação do projeto RAG Neo4j Students

## 1. Objetivo deste documento

Este documento descreve a finalidade, a arquitetura, o modelo de dados, o fluxo
de execução, os contratos de integração e o roteiro evolutivo do projeto
`rag-neo4j-students`.

Ele foi escrito para servir como material de estudo e como referência para
futuras implementações. Por isso, diferencia:

- **Estado atual**: comportamento e estrutura encontrados no código.
- **Arquitetura-alvo**: comportamento que o projeto pretende oferecer quando os
  nós ainda incompletos forem implementados.

Essa distinção é necessária porque o `README.md` ainda documenta um gerador de
artigos, enquanto a implementação atual está migrando para um assistente de
analytics de vendas.

## 2. Visão geral

O projeto é uma API de perguntas e respostas sobre dados educacionais e
comerciais. Uma pessoa pode perguntar, por exemplo:

- Quais cursos estão disponíveis?
- Quem comprou determinado curso?
- Qual foi a receita de pagamentos com cartão?
- Quais alunos chegaram a 100% de progresso?
- Quais cursos são comprados em conjunto?

O sistema deve transformar a pergunta em uma consulta **Cypher**, executá-la no
Neo4j e converter os resultados em uma resposta analítica compreensível.

O fluxo conceitual é:

```text
Pergunta em linguagem natural
        |
        v
Extração da pergunta
        |
        v
Análise de complexidade
        |
        v
Geração de Cypher pelo LLM
        |
        v
Validação e execução no Neo4j
        |
        +--> Correção da consulta, se necessário
        |
        v
Resposta analítica em linguagem natural
```

Embora o nome contenha “RAG”, o estado atual não possui busca vetorial nem
embeddings. O mecanismo implementado é principalmente **Graph QA**:
recuperação de dados estruturados no Neo4j orientada por uma consulta gerada
por um modelo de linguagem.

## 3. Objetivos de aprendizagem

O projeto pode ser utilizado para estudar:

1. Construção de APIs com Fastify e TypeScript.
2. Modelagem de dados relacionais em um banco orientado a grafos.
3. Criação de consultas Cypher.
4. Conversão de linguagem natural em consultas de banco.
5. Orquestração de agentes com LangGraph.
6. Uso de saídas estruturadas com Zod.
7. Validação e autocorreção de consultas geradas por LLM.
8. Decomposição de perguntas complexas em múltiplas etapas.
9. Testes end-to-end envolvendo LLM, Neo4j e HTTP.
10. Observabilidade, segurança e controle de custos em aplicações de IA.

## 4. Escopo funcional

### 4.1 Funcionalidades previstas

- Receber uma pergunta via HTTP.
- Identificar a pergunta mais recente enviada pelo usuário.
- Classificar a pergunta como simples ou complexa.
- Dividir perguntas complexas em até três subperguntas.
- Gerar Cypher compatível com o esquema real do Neo4j.
- Validar a consulta com `EXPLAIN`.
- Executar a consulta com parâmetros quando aplicável.
- Tentar corrigir uma consulta inválida uma vez.
- Produzir uma resposta com dados, cálculos e comparações.
- Sugerir perguntas de acompanhamento.
- Informar erros de forma compreensível.

### 4.2 Fora do escopo atual

- Alteração de dados por perguntas do usuário.
- Autenticação e autorização de usuários.
- Dashboard visual.
- Treinamento ou fine-tuning de modelos.
- Indexação vetorial de documentos.
- Persistência de histórico de conversas.
- Garantia de que dados aleatórios representem dados reais de negócio.

## 5. Estrutura do projeto

```text
.
├── data/
│   ├── courses.json          Catálogo inicial de cursos
│   ├── seed.ts               Entrada do processo de carga
│   └── seedHelpers.ts        Geração e persistência dos dados
├── docs/
│   └── especificacao.md      Este documento
├── src/
│   ├── config.ts             Configuração da aplicação
│   ├── index.ts              Inicialização do servidor
│   ├── server.ts             API HTTP
│   ├── graph/
│   │   ├── factory.ts        Criação do grafo e dos serviços
│   │   ├── graph.ts          Estado e topologia do LangGraph
│   │   └── nodes/            Etapas do fluxo
│   ├── prompts/v1/           Prompts e schemas Zod
│   └── services/
│       ├── neo4jService.ts   Acesso ao banco
│       └── openrouterService.ts
├── storage/neo4j/            Dados persistidos pelo Docker
├── tests/                    Testes end-to-end
├── docker-compose.yaml       Infraestrutura local
├── langgraph.json            Configuração do LangGraph CLI
├── package.json              Scripts e dependências
└── tsconfig.json             Configuração TypeScript
```

## 6. Arquitetura técnica atual

### 6.1 API

A aplicação usa Fastify. O servidor é criado por `createServer()` em
`src/server.ts` e escuta a porta `4000` quando iniciado por `src/index.ts`.

O endpoint atual é:

```http
POST /sales
Content-Type: application/json
```

Corpo:

```json
{
  "question": "List all available courses"
}
```

Resposta-alvo:

```json
{
  "answer": "There are 10 available courses...",
  "followUpQuestions": [
    "Which courses generate the most revenue?",
    "Which students completed their courses?"
  ],
  "query": "MATCH (c:Course) RETURN c.name AS courseName",
  "error": null
}
```

O campo `query` é útil para estudo e depuração, mas deve ser ocultado em
ambientes públicos caso revele detalhes internos do banco.

### 6.2 LangGraph

O estado principal é definido em `src/graph/graph.ts` e inclui:

| Campo | Tipo | Finalidade |
|---|---|---|
| `messages` | `BaseMessage[]` | Mensagens de entrada e saída |
| `question` | `string` opcional | Pergunta extraída |
| `query` | `string` opcional | Cypher atual |
| `originalQuery` | `string` opcional | Consulta antes de correções |
| `dbResults` | `any[]` opcional | Resultados do Neo4j |
| `correctionAttempts` | `number` opcional | Quantidade de tentativas |
| `validationError` | `string` opcional | Erro de validação |
| `needsCorrection` | `boolean` opcional | Indica falha corrigível |
| `isMultiStep` | `boolean` opcional | Indica decomposição |
| `subQuestions` | `string[]` opcional | Subperguntas |
| `currentStep` | `number` opcional | Etapa atual |
| `subQueries` | `string[]` opcional | Consultas das subetapas |
| `subResults` | `any[][]` opcional | Resultados das subetapas |
| `answer` | `string` opcional | Resposta final |
| `followUpQuestions` | `string[]` opcional | Sugestões de continuação |
| `error` | `string` opcional | Erro funcional |

Topologia atual:

```text
START
  -> extractQuestion
  -> queryPlanner
  -> cypherGenerator
  -> cypherExecutor
  -> [cypherCorrection] (condicional)
  -> analyticalResponse
  -> END
```

O `queryPlanner` usa uma saída estruturada definida em
`src/prompts/v1/queryAnalyzer.ts`. A classificação prevê no máximo três
subperguntas, mas o suporte completo ao processamento multi-etapas ainda não
está concluído.

### 6.3 Serviços

`Neo4jService` encapsula:

- Inicialização lazy do `Neo4jGraph`.
- Recuperação do esquema.
- Validação com `EXPLAIN`.
- Execução de consultas.
- Limpeza do banco.
- Encerramento da conexão.

`OpenRouterService` encapsula um `ChatOpenAI` apontando para a API compatível
com OpenAI do OpenRouter. Ele fornece geração estruturada usando agente e
`toolStrategy`.

## 7. Modelo de dados do Neo4j

### 7.1 Nós

#### `Course`

| Propriedade | Tipo | Descrição |
|---|---|---|
| `name` | string | Nome único usado pelo seed |
| `url` | string | URL pública do curso |

#### `Student`

| Propriedade | Tipo | Descrição |
|---|---|---|
| `id` | string | UUID do aluno |
| `name` | string | Nome completo |
| `email` | string | E-mail |
| `phone` | string | Telefone |

### 7.2 Relacionamento `PURCHASED`

```text
(Student)-[PURCHASED]->(Course)
```

Propriedades:

| Propriedade | Tipo | Descrição |
|---|---|---|
| `status` | `paid` ou `refunded` | Estado da compra |
| `paymentMethod` | `pix` ou `credit_card` | Forma de pagamento |
| `paymentDate` | ISO string | Data da transação |
| `amount` | número | Valor da compra |

### 7.3 Relacionamento `PROGRESS`

```text
(Student)-[PROGRESS]->(Course)
```

Propriedade:

| Propriedade | Tipo | Descrição |
|---|---|---|
| `progress` | inteiro de 0 a 100 | Percentual concluído |

O seed cria progresso somente para compras com status `paid`. Portanto, a
pergunta “alunos com progresso sem compra” normalmente deve retornar zero com
os dados gerados pelo seed; ela é útil como caso de borda, mas não representa
necessariamente uma inconsistência possível nesse gerador.

## 8. Processo de carga de dados

O comando `npm run seed`:

1. Limpa o banco com `MATCH (n) DETACH DELETE n`.
2. Gera 20 alunos.
3. Gera entre 1 e 10 compras por aluno.
4. Escolhe cursos aleatoriamente do `courses.json`.
5. Define status, forma de pagamento, data e valor.
6. Cria progresso para as compras pagas.
7. Insere os dados usando `UNWIND` e parâmetros.

Como os dados são aleatórios, testes que dependem de nomes, valores ou
quantidades exatas podem ser instáveis. Testes de integração devem validar
invariantes, como formato da resposta, existência de campos e consistência
entre resultados, ou utilizar um dataset fixo dedicado a testes.

## 9. Prompts e contratos de LLM

### 9.1 Análise de complexidade

`QueryAnalysisSchema` exige:

```json
{
  "complexity": "simple",
  "requiresDecomposition": false,
  "subQuestions": [],
  "reasoning": "Direct retrieval"
}
```

### 9.2 Geração de Cypher

O prompt de `cypherGenerator.ts` orienta o modelo a:

- Usar aliases explícitos.
- Retornar campos planos.
- Preferir `elementId()` a `id()`.
- Usar `CASE` para contagens condicionais.
- Usar `EXISTS { ... }` para existência.
- Filtrar cedo.
- Limitar a profundidade dos relacionamentos.
- Retornar somente Cypher, sem bloco Markdown.

### 9.3 Correção

`CypherCorrectionSchema` exige:

```json
{
  "correctedQuery": "MATCH ...",
  "explanation": "A variável foi propagada por WITH."
}
```

A correção deve preservar a intenção original e corrigir somente problemas de
sintaxe, escopo, agregação, propriedades ou valores nulos.

### 9.4 Resposta analítica

`AnalyticalResponseSchema` exige:

```json
{
  "answer": "Texto baseado nos resultados reais.",
  "followUpQuestions": [
    "Pergunta relacionada 1",
    "Pergunta relacionada 2"
  ]
}
```

O modelo deve responder no idioma da pergunta, não no idioma dos dados.
Também deve evitar inventar valores, consultas ou conclusões que não estejam
nos resultados recebidos.

## 10. Requisitos não funcionais

### Segurança

- Nunca permitir que o LLM execute comandos de escrita.
- Rejeitar consultas que contenham `CREATE`, `MERGE`, `DELETE`, `SET`, `DROP`,
  `REMOVE` ou `CALL` não autorizado.
- Preferir parâmetros a interpolação de texto.
- Não expor credenciais, prompts internos ou stack traces ao cliente.
- Validar tamanho e conteúdo da pergunta.
- Limitar tempo, número de tentativas e quantidade de resultados.

### Confiabilidade

- Falhas do OpenRouter devem produzir erro explícito.
- Falhas do Neo4j não devem ser convertidas em resposta de sucesso.
- A autocorreção deve ter limite fixo, atualmente uma tentativa planejada.
- O estado deve preservar a pergunta original e a consulta que falhou.

### Observabilidade

Registrar, sem dados sensíveis:

- Identificador da requisição.
- Tempo total.
- Tempo do LLM.
- Tempo do Neo4j.
- Modelo utilizado.
- Quantidade de resultados.
- Número de correções.
- Tipo de erro.

Não registrar a chave da API, e-mails completos ou conteúdo desnecessário de
mensagens pessoais.

## 11. Execução local

Pré-requisitos:

- Node.js compatível com `package.json`.
- Docker e Docker Compose.
- Chave do OpenRouter.

Exemplo de `.env`:

```env
OPENROUTER_API_KEY=sk-or-v1-...
OPENROUTER_MODELS=google/gemma-4-31b-it:free
```

Comandos:

```bash
npm install
npm run docker:infra:up
npm run seed
npm start
```

Requisição:

```bash
curl -X POST http://localhost:4000/sales \
  -H 'Content-Type: application/json' \
  -d '{"question":"List all available courses"}'
```

Para encerrar:

```bash
npm run docker:infra:down
```

O script `docker:infra:cleanup` remove os volumes e o diretório local de
armazenamento; deve ser usado somente quando for desejado perder os dados.

## 12. Estratégia de testes

### Testes unitários recomendados

- Extração de pergunta de uma mensagem.
- Pergunta vazia ou sem mensagens.
- Validação de consultas somente leitura.
- Formatação de respostas estruturadas.
- Tratamento de resultados vazios.
- Limite de tentativas de correção.
- Classificação simples versus complexa com respostas fixas do LLM.

### Testes de integração

- Conexão com Neo4j.
- Recuperação do esquema.
- `EXPLAIN` de consultas válidas.
- Execução de consultas parametrizadas.
- Carga completa do seed.

### Testes end-to-end

Os cenários em `tests/sales.e2e.test.ts` devem verificar:

- Status HTTP.
- Existência de `answer`.
- Existência de `followUpQuestions`.
- Perguntas sobre cursos, compras, receita e progresso.
- Tratamento de perguntas fora do domínio.
- Falhas do LLM ou do Neo4j.

Idealmente, os testes devem usar um dataset determinístico. Testes contra um
LLM real devem ser separados dos testes rápidos e possuir limites de custo e
tempo.

## 13. Estado atual e lacunas conhecidas

Os seguintes componentes ainda precisam ser completados antes de considerar o
fluxo funcional:

1. Implementação da geração de Cypher.
2. Implementação da execução no nó executor.
3. Implementação do nó de correção.
4. Implementação da resposta analítica real.
5. Implementação completa de consultas multi-etapas.
6. Tratamento consistente de resultados vazios.
7. Validação de comandos somente leitura.
8. Atualização do README para refletir o projeto de sales analytics.
9. Alinhamento da versão de Node.js entre `langgraph.json` e `package.json`.
10. Remoção de chamadas de demonstração automáticas em `src/index.ts`.
11. Definição de um contrato de erro HTTP consistente.
12. Separação entre configuração de produção e credenciais locais.

Enquanto essas lacunas existirem, o projeto deve ser tratado como uma base
educacional em evolução, e não como uma API pronta para produção.

## 14. Roteiro recomendado de evolução

### Fase 1 — Tornar o fluxo mínimo funcional

- Implementar geração de Cypher estruturada.
- Implementar executor usando `validateQuery` e `query`.
- Implementar resposta analítica estruturada.
- Remover o retorno de demonstração `"hello world"`.
- Criar um teste para “listar cursos”.

### Fase 2 — Segurança e confiabilidade

- Criar validador de consultas somente leitura.
- Adicionar limites de tempo e resultados.
- Padronizar erros.
- Adicionar logs com correlação.
- Testar indisponibilidade do Neo4j e do OpenRouter.

### Fase 3 — Perguntas complexas

- Executar subperguntas em ordem.
- Armazenar cada consulta e resultado no estado.
- Sintetizar resultados com um prompt específico.
- Garantir que uma falha em uma etapa seja explicitada.

### Fase 4 — RAG e cache semântico

- Definir o que será indexado: perguntas, respostas, consultas ou documentos.
- Escolher modelo de embeddings.
- Criar índice vetorial no Neo4j.
- Definir limiar de similaridade.
- Invalidar cache quando o dado de origem mudar.
- Diferenciar claramente cache semântico de consulta ao grafo.

### Fase 5 — Produção

- Adicionar autenticação.
- Remover detalhes internos da resposta pública.
- Criar métricas e tracing.
- Fixar versões e validar compatibilidade de Node.js.
- Usar dados e segredos apropriados para cada ambiente.
- Executar testes determinísticos em CI.

## 15. Exemplos de consultas Cypher de referência

Listar cursos:

```cypher
MATCH (c:Course)
RETURN c.name AS courseName, c.url AS url
ORDER BY c.name
```

Alunos que compraram um curso:

```cypher
MATCH (s:Student)-[p:PURCHASED]->(c:Course)
WHERE c.name = $courseName AND p.status = 'paid'
RETURN s.name AS studentName, s.email AS email
ORDER BY s.name
```

Receita por curso:

```cypher
MATCH (s:Student)-[p:PURCHASED]->(c:Course)
WHERE p.status = 'paid'
RETURN c.name AS courseName, SUM(p.amount) AS revenue
ORDER BY revenue DESC
```

Alunos com progresso completo:

```cypher
MATCH (s:Student)-[p:PROGRESS]->(c:Course)
WHERE p.progress = 100
RETURN s.name AS studentName, c.name AS courseName
ORDER BY s.name
```

Esses exemplos são somente leitura e devem ser usados como base para testes e
para avaliar a qualidade das consultas produzidas pelo LLM.

## 16. Glossário

- **Cypher**: linguagem de consulta do Neo4j.
- **Graph QA**: perguntas respondidas consultando um grafo de conhecimento.
- **LLM**: modelo de linguagem usado para interpretar e gerar texto ou código.
- **LangGraph**: biblioteca para orquestrar fluxos de agentes com estado.
- **RAG**: geração aumentada por recuperação de informação.
- **Structured output**: resposta do modelo validada por um schema, neste caso
  usando Zod.
- **Seed**: processo que popula o banco com dados iniciais.
- **Autocorreção**: nova chamada ao LLM para corrigir uma consulta inválida.

