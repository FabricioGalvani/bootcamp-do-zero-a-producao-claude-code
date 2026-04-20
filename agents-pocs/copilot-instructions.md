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
