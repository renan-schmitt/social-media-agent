# Social Media Agent

Publica automaticamente o conteúdo dos apps no Facebook e no Instagram, a partir de uma
biblioteca de posts no Firebase. A visão geral e as decisões estão em [`docs/ideas.md`](docs/ideas.md).

## Requisitos

- [uv](https://docs.astral.sh/uv/) (instala o Python 3.12 sozinho, se precisar)

## Instalação

```bash
uv sync
```

## Testes e lint

```bash
uv run pytest
uv run ruff check
uv run ruff format --check
```
