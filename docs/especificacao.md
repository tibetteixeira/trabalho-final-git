# Especificação do projeto

## 1. Visão geral

Sistema mínimo de **vendas** de produtos de um catálogo fixo. É o menor escopo que pensamos capaz de gerar os 3 indicadores do dashboard (#6/#12), os 3 filtros combináveis (#4/#10) e a exportação CSV (#5/#11).

O mesmo CRUD é acessado por **Web** (`index.html`) e **CLI** (`cli.py`), cada um com seus próprios endpoints (`/api/web/*` e `/api/cli/*`) sobre a mesma API (`api.py`) e o mesmo banco (`db.py`).

## 2. Stack e decisões técnicas

| Item | Decisão | Motivo |
|---|---|---|
| Backend / API | Python 3.12 + Flask | Simples, cliente de testes embutido |
| Banco | PostgreSQL 16 + psycopg 3 | Serviço próprio no Compose ("Web, API e banco") |
| Frontend | HTML + CSS + JS puro em um único `index.html` | Exigência da avaliação |
| CLI | Python 3.12 em um único `cli.py`, só biblioteca padrão (`argparse`, `urllib`); serviço `cli` no Compose | Sem dependência nova; roda via `docker compose run` |
| Servidor Web | nginx servindo `index.html` e fazendo proxy de `/api/` | Mesma origem, sem CORS |
| Autenticação Web | Usuário inicial via env; senha com hash scrypt; token assinado (itsdangerous), 8 h | Sem senha em texto puro |
| Autenticação CLI | `POST /api/cli/token` com o mesmo usuário da Web; token assinado (itsdangerous, salt próprio), 8 h, enviado como Bearer e lido de `TOKEN_CLI` (issue #7) | Mesmo mecanismo da Web; o salt próprio separa os tokens dos dois clientes |
| Logs | JSON por linha no stdout, camadas `api` e `db`, `request_id` compartilhado | Seção 3.3 da avaliação |
| Testes | pytest + pytest-cov, 100% de linhas em `api.py`, `db.py` (e `cli.py` a partir da #7) | Seção 4 da avaliação |

## 3. Fluxo de trabalho

Regras completas em [`../CLAUDE.md`](../CLAUDE.md). Resumo:

- Uma **git worktree por issue** em `tmp/issue-N`, na branch `issue-N` (corretivas: `issue-N-fix-K`). `tmp/` é ignorado pelo `.gitignore`.
  - Ex.: `git worktree add tmp/issue-1 -b issue-1 upstream/main`
- **Somente as 12 issues existentes**; nenhuma issue nova. PRs corretivas usam `Refs #N`.
- PR para o `main` do repositório principal com o template `.github/PULL_REQUEST_TEMPLATE.md`, `Closes #N` e aprovação dos 2 colegas.
- Histórico **linear**: branches atualizadas com `rebase`, integração só por **Rebase and merge**.
- Branches **não são apagadas** após o merge.
- Tag SemVer + release manual após cada PR principal (`v0.N.0`, a 12ª é `v1.0.0`).

## 4. Modelo de dados

Decisão da equipe: **3 tabelas — `usuario`, `produto` e `venda`.**

- `usuario` guarda quem faz login.
- `produto` é um catálogo próprio, para evitar o mesmo produto com nomes diferentes (ex.: "Biscoito" e "Recheado") aparecendo separado no dashboard.
- `venda` é o recurso `records` da avaliação e referencia `produto`. Guarda o `valor_unitario` praticado; o `valor_total` é calculado pelo banco.
- Todas as tabelas são criadas na **#1** (critério de aceite: "Tabela de registros criada no banco com os campos do domínio escolhido pela equipe"). As rotas e telas que as usam vêm nas issues seguintes.
- Esquema e carga inicial ficam em `db.py`, dentro de `inicializar_banco` (sem arquivo `.sql`), de forma idempotente (`CREATE TABLE IF NOT EXISTS`, `ON CONFLICT`).

```
usuario            produto                venda
-------            -------                -----
id                 id  <──────────────┐   id
usuario            nome               └── produto_id
nome               categoria              quantidade
senha_hash         preco                  valor_unitario
criado_em                                 valor_total (gerado)
                                          data_venda
```

### 4.1. `usuario` (implementado na #1)

| Campo | Tipo | Observação |
|---|---|---|
| `id` | SERIAL PK | |
| `usuario` | TEXT UNIQUE NOT NULL | Login (`ADMIN_USUARIO`) |
| `nome` | TEXT NOT NULL | Nome exibido após o login (`ADMIN_NOME`) |
| `senha_hash` | TEXT NOT NULL | Hash scrypt; nunca a senha (`ADMIN_SENHA`) |
| `criado_em` | TIMESTAMPTZ | |

### 4.2. `produto` (implementado na #1)

Catálogo fixo de 12 produtos (3 por categoria), definido em `PRODUTOS_INICIAIS` no `db.py` e carregado na inicialização com `ON CONFLICT (nome) DO NOTHING` (sem CRUD de produtos, que nenhuma issue pede).

| Campo | Tipo | Regra |
|---|---|---|
| `id` | SERIAL PK | |
| `nome` | TEXT UNIQUE NOT NULL | |
| `categoria` | TEXT NOT NULL | Alimentos, Bebidas, Limpeza ou Higiene |
| `preco` | NUMERIC(10,2) NOT NULL | > 0; sugerido como `valor_unitario` no formulário de venda |

### 4.3. `venda` (= `records`, implementado na #1)

| Campo | Tipo | Regra | No formulário? |
|---|---|---|---|
| `id` | SERIAL PK | | não |
| `produto_id` | INTEGER NOT NULL FK → `produto.id` | produto existente (select) | sim (obrigatório) |
| `quantidade` | INTEGER NOT NULL | ≥ 1 (CHECK no banco) | sim (obrigatório) |
| `valor_unitario` | NUMERIC(10,2) NOT NULL | > 0 (CHECK no banco); o formulário sugere o `preco` do produto, editável (descontos) | sim (obrigatório) |
| `valor_total` | NUMERIC(12,2) | coluna gerada: `quantidade * valor_unitario` (`GENERATED ALWAYS ... STORED`) | não |
| `data_venda` | DATE NOT NULL | data válida; não pode ser futura (validação na API); o formulário sugere a data de hoje | sim (obrigatório) |

Todos os campos do formulário podem ser editados (#2/#8). O formulário tem **4 campos obrigatórios**, dentro da regra de 4–6. Como o valor fica gravado na venda, mudar o `preco` de um produto não altera vendas antigas. A equipe tem 3 integrantes, então não há `status` (#13–#16 não se aplicam).

**Divisão entre issues:**

| Issue | Parte do modelo |
|---|---|
| #1 | Tabelas `usuario`, `produto` (com carga inicial) e `venda`; login |
| #2 | `GET /api/web/produtos` para o select; formulário e rotas de criar/editar venda |
| #8 | `GET /api/cli/produtos` (rota própria da CLI) e comandos de cadastro/edição de venda |
| #3–#6, #9–#12 | Joins de `venda` com `produto` na listagem, busca, CSV e indicadores |

### 4.4. Filtros (#4 / #10) — proposta

1. `produto_id` (igual)
2. `categoria` do produto (igual, via join com `produto`)
3. Período de `data_venda` (`data_inicio` / `data_fim`)

### 4.5. Indicadores (#6 / #12) — proposta

1. **Faturamento por mês** — soma de `valor_total` agrupada por mês (gráfico de linha/barras)
2. **Faturamento por categoria** (barras)
3. **Top 5 produtos por quantidade vendida** (barras horizontais)
