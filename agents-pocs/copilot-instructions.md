# Copilot Instructions

Este arquivo orienta o GitHub Copilot (e agentes similares) ao gerar código neste repositório. Mantenha-o curto, direto e atualizado.

## Visão geral do projeto

Projeto de **engenharia de dados em Python**. Descreva em 2-3 linhas o que o projeto faz, quais fontes de dados ele consome e qual é o destino final (data warehouse, data lake, API, dashboard, etc.).

> Exemplo: "Pipeline que ingere dados de APIs públicas e arquivos CSV, transforma com pandas/PySpark e carrega em um data warehouse PostgreSQL. Orquestração via Airflow."

## Stack e ferramentas principais

- **Linguagem:** Python 3.11+
- **Gerenciamento de dependências:** `uv` / `poetry` / `pip-tools` (escolha uma)
- **Manipulação de dados:** `pandas`, `polars`, `pyarrow`
- **Banco/armazenamento:** `sqlalchemy`, `psycopg2`, `boto3` (S3)
- **Orquestração:** `Airflow` / `Prefect` / `Dagster` (se aplicável)
- **Testes:** `pytest`, `pytest-cov`
- **Lint/format:** `ruff`, `black`, `mypy`

## Convenções de código

- Siga **PEP 8** e use `ruff` + `black` antes de commitar.
- **Type hints são obrigatórios em TODAS as funções, métodos e assinaturas de módulos** — inclusive funções internas/privadas. Não gere código sem anotações de tipo.
- Use tipos modernos: `list[str]` em vez de `List[str]`, `dict[str, int]` em vez de `Dict[str, int]`, `X | None` em vez de `Optional[X]` (Python 3.10+).
- Use `snake_case` para variáveis/funções e `PascalCase` para classes.
- **Docstrings são obrigatórias no estilo NumPy**, sempre em português. Toda função, classe ou módulo público deve ter docstring.
- Prefira **funções puras** e pequenas; evite efeitos colaterais escondidos.

### Exemplo de função bem documentada (NumPy style + type hints)

```python
import pandas as pd


def normalizar_colunas(df: pd.DataFrame, colunas: list[str]) -> pd.DataFrame:
    """
    Normaliza colunas numéricas para o intervalo [0, 1].

    Parameters
    ----------
    df : pandas.DataFrame
        DataFrame de entrada contendo as colunas a serem normalizadas.
    colunas : list of str
        Nomes das colunas numéricas que serão normalizadas.

    Returns
    -------
    pandas.DataFrame
        Uma cópia do DataFrame com as colunas indicadas normalizadas.

    Raises
    ------
    KeyError
        Se alguma coluna em `colunas` não existir em `df`.
    ValueError
        Se alguma coluna indicada não for numérica.

    Examples
    --------
    >>> df = pd.DataFrame({"a": [1, 2, 3], "b": [10, 20, 30]})
    >>> normalizar_colunas(df, ["a", "b"])
       a    b
    0  0.0  0.0
    1  0.5  0.5
    2  1.0  1.0
    """
    ...
```

## Estrutura de pastas sugerida

```
.
├── src/
│   ├── ingestion/      # extração das fontes
│   ├── transform/      # limpeza e modelagem
│   ├── load/           # carga no destino
│   └── utils/          # helpers compartilhados
├── tests/
├── notebooks/          # exploração (não versionar outputs)
├── dags/               # se usar Airflow
└── pyproject.toml
```

## Padrões para dados

- **Nunca** commite dados reais no repositório. Use `.gitignore` para `data/`, `*.csv`, `*.parquet`.
- Sempre validar schemas na entrada (ex.: `pydantic`, `pandera`).
- Logs estruturados com `logging` (nível `INFO` por padrão, `DEBUG` opcional via env var).
- Credenciais apenas via variáveis de ambiente ou secret manager — **nunca** hardcoded.
- Datas em **UTC** internamente; converter só na camada de apresentação.

## O que o Copilot deve evitar

- Não sugerir `pandas` quando o dataset for claramente grande (>1M linhas) — prefira `polars` ou `PySpark`.
- Não usar `print()` para logs — use `logging`.
- Não usar `os.system` ou `subprocess.shell=True` sem necessidade.
- Não gerar queries SQL concatenando strings — use parâmetros ou SQLAlchemy Core.
- Não criar funções gigantes; quebrar em unidades testáveis.

## Logging

Logs devem ajudar a entender o que aconteceu em produção sem poluir a saída. Regra geral: **se você precisaria dessa linha às 3h da manhã para debugar, logue. Caso contrário, não.**

### Diretrizes

- **Use sempre a função utilitária de logging do projeto** (responsável por inicializar e configurar o `logging`). Nunca chame `logging.basicConfig` nem crie `StreamHandler` manualmente no meio do código.
- O logger deve ser obtido dessa função utilitária e atribuído a:
  - `self.logger` quando estiver dentro de uma classe de ETL.
  - Uma variável de módulo/função chamada `logger` em funções avulsas.
- Nunca use `print` para logar.
- **Um log por evento significativo**, não um log por linha de código.
- **Níveis corretos:**
  - `DEBUG`: detalhes internos úteis só durante investigação (valores intermediários, queries montadas).
  - `INFO`: marcos do pipeline (início/fim de etapa, quantidade de registros processados, arquivo gerado).
  - `WARNING`: algo inesperado mas recuperável (registro descartado por schema inválido, retry feito).
  - `ERROR`: falha que impede a operação atual (não conseguiu carregar tabela, API retornou 500).
  - `CRITICAL`: pipeline inteiro inviabilizado (banco fora do ar, credencial ausente).
- **Inclua contexto útil** na mensagem: nome da etapa, quantidade de linhas, identificador do lote, tempo decorrido. Sem contexto, o log é inútil.
- **Nunca logue dados sensíveis**: senhas, tokens, PII, conteúdo bruto de registros. Log apenas IDs ou contagens.
- Prefira **mensagens parametrizadas** (`logger.info("Processados %d registros em %.2fs", n, elapsed)`) em vez de f-strings — evita custo de formatação quando o nível está desligado.
- Em loops grandes, **não logue a cada iteração**. Logue no início, no fim e em batches (ex.: a cada 10.000 registros).
- **PySpark: nunca chame `df.count()` (ou qualquer ação que force execução, como `.collect()`, `.show()`) dentro de logs em nível `INFO`** — isso dispara um job inteiro só para gerar a mensagem e degrada o pipeline. Contagens de DataFrames Spark só são aceitáveis em `DEBUG`, e ainda assim com parcimônia. Para marcos de pipeline, prefira logar o **nome da etapa, parâmetros e tempo decorrido**, não o volume.

### Exemplo — função avulsa

```python
import time

from meu_projeto.logging_utils import get_logger  # função utilitária do projeto

logger = get_logger(__name__)


def carregar_lote(lote_id: str, registros: list[dict]) -> int:
    """Carrega um lote de registros no destino e retorna a contagem inserida."""
    inicio = time.perf_counter()
    logger.info("Iniciando carga do lote %s (%d registros)", lote_id, len(registros))

    try:
        inseridos = _inserir(registros)
    except Exception:
        logger.exception("Falha ao carregar lote %s", lote_id)  # já inclui traceback
        raise

    elapsed = time.perf_counter() - inicio
    logger.info("Lote %s carregado em %.2fs", lote_id, elapsed)
    return inseridos
```

### Exemplo — classe de ETL (PySpark)

```python
import time

from pyspark.sql import DataFrame

from meu_projeto.logging_utils import get_logger


class MeuETL:
    def __init__(self, origem: str, destino: str) -> None:
        self.origem = origem
        self.destino = destino
        self.logger = get_logger(self.__class__.__name__)

    def transformar(self, df: DataFrame) -> DataFrame:
        self.logger.info("Iniciando transformação (origem=%s)", self.origem)
        inicio = time.perf_counter()

        df_out = df.filter("ativo = true").dropDuplicates(["id"])

        # df_out.count() aqui seria uma ação cara — evite em INFO.
        self.logger.debug("Schema pós-transformação: %s", df_out.schema.simpleString())

        self.logger.info("Transformação concluída em %.2fs", time.perf_counter() - inicio)
        return df_out
```

### O que evitar

- `logger.info("entrou na função X")` / `logger.info("saiu da função X")` — ruído puro.
- Logar o mesmo evento em vários níveis da call stack.
- Usar `logger.error()` + `raise` no mesmo lugar com a exceção crua — use `logger.exception()` que já anexa o traceback.
- Mensagens vagas como `"erro ao processar"` sem identificador, contagem ou causa.
- Chamar `df.count()`, `df.collect()` ou `df.show()` dentro de mensagens de log em `INFO`/`WARNING` em pipelines PySpark — força execução e degrada o desempenho do ETL.
- Reconfigurar o `logging` dentro de módulos (ex.: `logging.basicConfig(...)`) — a configuração é responsabilidade exclusiva da função utilitária do projeto.

## Testes

- Todo novo módulo em `src/` deve ter teste correspondente em `tests/`.
- Use `pytest` e `pytest.fixture` para dados de exemplo.
- Meta de cobertura mínima: **80%**.

## Commits

Use [Conventional Commits](https://www.conventionalcommits.org/):

- `feat: adiciona ingestão da API X`
- `fix: corrige parsing de datas no CSV Y`
- `refactor: extrai utilitário de normalização`
- `test: cobre cenário de schema inválido`
- `docs: atualiza README da camada de transformação`