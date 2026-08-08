# Local PDF RAG Assistant

Aplicação local de RAG (*Retrieval-Augmented Generation*) para conversar com
documentos PDF. A interface é construída com Streamlit, os embeddings são
gerados com MiniLM e as respostas são produzidas pelo Qwen 2.5 3B por meio do
Transformers.

Depois que os modelos são baixados, a extração do PDF, a busca vetorial e a
geração das respostas acontecem localmente.

## Funcionalidades

- Upload de documentos PDF pela interface web.
- Extração e normalização do texto com `pypdf`.
- Embeddings locais com `sentence-transformers/all-MiniLM-L6-v2`.
- Índice vetorial em memória com distância cosseno ou euclidiana.
- Recuperação dos trechos mais relevantes para cada pergunta.
- Geração local com `Qwen/Qwen2.5-3B-Instruct`.
- Respostas exibidas em streaming.
- Cache do modelo e do índice durante a sessão do Streamlit.

## Como funciona

```text
PDF
 └─> extração do texto
      └─> separação em parágrafos
           └─> embeddings MiniLM
                └─> índice vetorial em memória
                     └─> recuperação dos trechos relevantes
                          └─> prompt com contexto
                               └─> Qwen 2.5 3B
                                    └─> resposta em streaming
```

O contexto enviado ao modelo contém somente os trechos recuperados do
documento. O prompt instrui o modelo a informar quando a resposta não estiver
presente no PDF.

## Modelos utilizados

| Finalidade | Modelo |
| --- | --- |
| Geração de texto | `Qwen/Qwen2.5-3B-Instruct` |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` |

O MiniLM gera vetores normalizados de 384 dimensões. Os modelos são baixados do
Hugging Face Hub na primeira execução e armazenados no cache local.

## Requisitos

- Python 3.13 recomendado.
- Espaço em disco para os modelos e o cache do Hugging Face.
- Memória suficiente para executar um modelo de 3 bilhões de parâmetros.
- Internet na primeira execução para autenticação e download dos modelos.

Uma GPU compatível melhora o desempenho, mas não é obrigatória. Em macOS com
Apple Silicon, a instalação não utiliza CUDA e o componente de embeddings da
implementação atual executa em CPU. Em máquinas NVIDIA, o PyTorch e os drivers
precisam ser compatíveis com a versão CUDA instalada.

## Instalação

Clone o repositório e entre no diretório:

```bash
git clone <URL_DO_REPOSITORIO>
cd rag-document
```

Crie e ative um ambiente virtual:

```bash
python3.13 -m venv .venv
source .venv/bin/activate
```

No Windows PowerShell, use:

```powershell
py -3.13 -m venv .venv
.venv\Scripts\Activate.ps1
```

Instale as dependências:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

## Autenticação no Hugging Face

A implementação lê o token da variável `HF_TOKEN`. Crie um arquivo `.env` na
raiz do projeto:

```dotenv
HF_TOKEN=hf_seu_token_aqui
```

Crie ou gerencie tokens nas configurações da sua conta do Hugging Face. Não
adicione o arquivo `.env` ao Git; ele já está listado no `.gitignore`.

Opcionalmente, para verificar a autenticação pelo CLI atual do Hugging Face:

```bash
hf auth login
hf auth whoami
```

O comando atual é `hf`; `huggingface-cli` está obsoleto.

## Execução

Com o ambiente virtual ativo, execute:

```bash
streamlit run main.py
```

Depois:

1. Abra o endereço informado pelo Streamlit.
2. Aguarde o carregamento do Qwen. A primeira execução pode demorar por causa
   do download.
3. Envie um PDF pela barra lateral.
4. Aguarde a extração do texto e a construção do índice.
5. Faça perguntas sobre o documento.

## Estrutura do projeto

```text
.
├── main.py               # interface e estado da aplicação Streamlit
├── local_llm.py          # carregamento do Qwen e geração em streaming
├── local_embedding.py    # embeddings MiniLM e recuperação de contexto
├── pdf_reader.py         # extração e normalização do texto dos PDFs
├── vector_index.py       # índice vetorial em memória
├── requirements.txt      # dependências Python
└── pdfs/                 # PDF de exemplo
```

## Componentes principais

### `PdfReader`

Extrai o texto das páginas, normaliza espaços e quebras de linha e retorna uma
lista de parágrafos.

### `LocalEmbedding`

Tokeniza os parágrafos, calcula os embeddings com *mean pooling*, normaliza os
vetores e alimenta o índice local.

### `VectorIndex`

Armazena vetores e documentos somente em memória. A busca suporta distância
cosseno e euclidiana, sem depender de um banco vetorial externo.

### `AiModel`

Carrega o Qwen, constrói o prompt de RAG e executa a geração em uma thread para
que o Streamlit possa consumir os tokens progressivamente.

## Limitações atuais

- O índice vetorial não é persistido entre execuções.
- Um novo PDF precisa ser processado novamente quando o servidor reinicia.
- A separação do documento é baseada em parágrafos, sem estratégia avançada de
  *chunking* ou sobreposição.
- O histórico é exibido na interface, mas perguntas anteriores não são enviadas
  ao modelo como memória da conversa.
- A resposta não apresenta citações ou números de página.
- PDFs compostos apenas por imagens precisam de OCR, que não está implementado.
- O desempenho em CPU pode ser lento, principalmente durante a geração.

## Solução de problemas

### `No matching distribution found for torch...+cu...`

Builds com o sufixo `+cu...` são específicos para CUDA/NVIDIA. Em macOS, use a
versão padrão declarada no `requirements.txt`:

```text
torch==2.11.0
```

### `Attempting Hugging Face login` falha

Confirme se `.env` existe na raiz e contém um token válido:

```dotenv
HF_TOKEN=hf_seu_token_aqui
```

### `WARNING: No GPU detected`

O teste atual verifica CUDA. Esse aviso é esperado em macOS e não impede a
execução em CPU.

### A primeira execução está lenta

Na primeira execução, o Transformers baixa e armazena os modelos. As próximas
execuções reutilizam o cache local do Hugging Face.

## Qualidade de código

O projeto pode ser verificado e formatado com Ruff:

```bash
ruff check .
ruff check --fix .
ruff format .
```

