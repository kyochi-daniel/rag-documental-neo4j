# RAG documental com Neo4j

Protótipo de consulta a documentos PDF com Retrieval-Augmented Generation (RAG). O projeto combina embeddings executados localmente, busca vetorial no Neo4j e geração de respostas por um modelo acessado pelo OpenRouter.

Projeto acadêmico realizado durante minha pós-graduação em Engenharia de Software com IA Aplicada, a partir do material didático da Unipds.

O exemplo consulta o arquivo `tensores.pdf` para responder perguntas sobre TensorFlow.js e machine learning. A execução ocorre pelo terminal, com perguntas definidas no código e respostas salvas em Markdown.

## Fluxo de processamento

1. Carrega o PDF com `PDFLoader` e extrai seu conteúdo textual.
2. Divide o conteúdo em trechos de 1.000 caracteres, com sobreposição de 200 caracteres.
3. Gera embeddings com `HuggingFaceTransformersEmbeddings` e armazena os trechos no Neo4j.
4. Recupera até três trechos por pergunta e mantém aqueles com score superior a `0.5`.
5. Monta o prompt com a pergunta, os trechos recuperados e as instruções educacionais.
6. Consulta o modelo pelo OpenRouter e salva as respostas em `respostas/`.

O Neo4j é usado como banco vetorial. Este exemplo não implementa recuperação por relações de um grafo de conhecimento.

## Tecnologias

| Tecnologia | Uso neste projeto |
| --- | --- |
| Node.js e TypeScript | Execução do pipeline e gravação das respostas |
| LangChain | Carregamento do PDF, divisão em trechos, busca vetorial e composição da sequência RAG |
| Hugging Face Transformers | Execução local do modelo de embeddings selecionado |
| Neo4j Community | Armazenamento e recuperação dos vetores |
| OpenRouter | Acesso ao modelo de geração por API compatível com OpenAI |
| Docker Compose | Ambiente local do Neo4j |

## Pré-requisitos

- Node.js `22.13.1`, versão indicada no `package.json`, e npm.
- Docker e Docker Compose para iniciar o banco incluído no repositório.
- Conexão à internet para instalar dependências, baixar o modelo de embeddings e acessar o OpenRouter.
- Conta e chave de API do OpenRouter, com acesso ao modelo escolhido.
- Recursos locais suficientes para executar o modelo de embeddings selecionado.

Use um **banco dedicado e descartável para este exemplo**. A inicialização executa `clearAll()`, que apaga todos os nós com o rótulo `Chunk` e suas relações antes de repopular a base. A chamada atual não usa `await`, portanto a exclusão pode concorrer com a inserção dos novos documentos.

O Compose mantém dados em `./neo4j/data`, exige a senha definida em `.env` e restringe as portas publicadas ao endereço local `127.0.0.1`.

## Configuração

Na raiz do projeto, copie o arquivo de exemplo e preencha as variáveis abaixo:

```bash
cp .env.example .env
```

Defina uma senha para o banco, sua chave do OpenRouter e o identificador do modelo de geração. Não publique `.env`; ele está incluído no `.gitignore`.

| Variável | Configuração |
| --- | --- |
| `NEO4J_URI` | Endereço Bolt do banco; para o Compose local, `bolt://localhost:7687` |
| `NEO4J_USER` | Usuário definido no `NEO4J_AUTH` do Compose |
| `NEO4J_PASSWORD` | Senha correspondente ao usuário do banco |
| `EMBEDDING_MODEL` | Identificador de um modelo de embeddings compatível com o carregador Hugging Face Transformers; o arquivo de exemplo sugere `Xenova/all-MiniLM-L6-v2` |
| `NLP_MODEL` | Identificador do modelo de geração disponível para sua conta no OpenRouter |
| `OPENROUTER_API_KEY` | Sua chave de API do OpenRouter |
| `OPENROUTER_SITE_URL` | URL de identificação enviada no cabeçalho `HTTP-Referer` |
| `OPENROUTER_SITE_NAME` | Nome de identificação enviado no cabeçalho `X-Title` |

As variáveis são lidas em `src/config.ts`. O código não valida sua presença antes de iniciar; confira os valores antes da execução.

## Execução

Execute os comandos a partir da raiz deste projeto:

```bash
npm ci
docker compose up -d --wait
npm start
```

O `npm start` carrega `.env` e executa `src/index.ts` diretamente com o suporte de remoção de tipos do Node.js. Não há etapa de build configurada.

O Neo4j Browser fica disponível em `http://localhost:7474`. O terminal exibe o andamento da ingestão, os scores de recuperação e as respostas. As respostas geradas são gravadas em arquivos com o padrão `respostas/resposta-<índice>-<timestamp>.md`; essa pasta é ignorada pelo Git.

Para encerrar o banco:

```bash
docker compose down
```

Também existem os scripts `npm run infra:up` e `npm run infra:down`, que usam o comando `docker-compose` com hífen. Eles exigem que esse comando esteja disponível. O segundo inclui `--volumes`; como os dados do Neo4j usam diretórios locais montados, eles permanecem nesses diretórios.

## Perguntas e documentos

Edite o array `questions` em [`src/index.ts`](src/index.ts) para definir as perguntas da execução. O exemplo inclui perguntas sobre conversão de objetos JavaScript em tensores, normalização, redes neurais e treinamento.

Em [`src/config.ts`](src/config.ts), ajuste:

- `pdf.path` para selecionar outro PDF com conteúdo textual extraível.
- `textSplitter` para alterar tamanho e sobreposição dos trechos.
- `similarity.topK` para alterar a quantidade de candidatos recuperados.
- `output` para alterar a pasta e o prefixo dos arquivos de resposta.

As instruções de resposta ficam em [`prompts/answerPrompt.json`](prompts/answerPrompt.json) e o template em [`prompts/template.txt`](prompts/template.txt). Ao trocar o assunto do documento, ajuste também essas instruções, que atualmente tratam de TensorFlow.js e machine learning.

Cada nova execução repete a ingestão do documento e a tentativa de exclusão dos chunks existentes.

## Custos e dados enviados

Os embeddings são calculados localmente. A primeira utilização do modelo pode exigir download de seus arquivos.

A geração utiliza o OpenRouter e pode consumir créditos conforme o modelo selecionado e os tokens processados. O código configura até duas novas tentativas em caso de falha, mas não implementa controle de gastos nem relatório de consumo. Consulte as condições do modelo em sua conta antes de executar; este repositório não pressupõe disponibilidade gratuita.

As perguntas e os trechos recuperados do PDF são enviados ao OpenRouter e ao provedor do modelo escolhido. Use documentos adequados a esse envio.

## Limitações conhecidas

- A exclusão dos chunks não é aguardada antes da inserção. Há risco de perda de trechos e de resultados inconsistentes durante a ingestão.
- Quando a busca retorna candidatos, mas todos são descartados pelo filtro de score, o pipeline ainda chama o modelo com contexto vazio. A interrupção só ocorre quando a busca retorna zero candidatos.
- O processamento preserva apenas `source` nos metadados e descarta a informação de página. As respostas não incluem citações verificáveis por página.
- O limiar de score é fixo e não foi acompanhado, neste repositório, de uma avaliação de qualidade da recuperação.
- As instruções pedem respostas baseadas apenas no contexto, mas isso não garante fidelidade factual nem impede alucinações.
- Não há suíte de testes ou benchmark de qualidade configurado. Este README não apresenta métricas de precisão, latência ou custo.
- A aplicação é um exemplo por terminal, com um PDF e perguntas definidas no código. Não oferece interface de chat nem API HTTP.

Essas limitações descrevem o código preservado do exemplo acadêmico; não representam correções já implementadas.

## Origem, créditos e licença

Este projeto foi organizado para apresentação como exercício acadêmico a partir do exemplo `exemplo-13-embeddings-neo4j-rag` do [material didático da Unipds](https://github.com/unipds-engenharia-de-ia-aplicada/engenharia-de-software-com-ia-aplicada).

O material original e os prompts mantêm seus créditos, incluindo a autoria de Erick Wendel. A lógica do pipeline foi preservada. A organização desta cópia inclui este README, um exemplo de variáveis de ambiente, ajustes de configuração local e metadados, e a exclusão de respostas geradas anteriormente. Não é apresentada como criação original do pipeline.

A licença original está preservada em [`LICENSE.md`](LICENSE.md) e identifica o material como **CC BY-NC-ND 4.0**. Consulte esse arquivo para as condições de atribuição, uso não comercial e distribuição.
