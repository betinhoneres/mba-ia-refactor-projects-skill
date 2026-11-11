# 📑 Relatório de Auditoria Arquitetural — `code-smells-project`

**Data da Auditoria:** `2026-10-30`  
**Stack Detectada:** `Python 3` / `Flask 3.1.1 (+ flask-cors 5.0.1)`  
**Domínio da Aplicação:** `E-commerce (produtos, usuários, pedidos, relatórios de vendas)`  

---

## 📊 1. Resumo Executivo por Severidade

| Severidade | Quantidade Identificada |
|---|---|
| 🚨 **CRITICAL** | `5` |
| ⚠️ **HIGH** | `2` |
| ⚡ **MEDIUM** | `4` |
| 🔵 **LOW** | `4` |
| **TOTAL** | **`15`** |

> **APIs Deprecated:** nenhuma das APIs listadas no catálogo (`@app.before_first_request`, `json.loads(request.data)`) foi encontrada. O projeto usa `request.get_json()` corretamente.

---

## 🔍 2. Detalhamento dos Achados

### Achado #1: Hardcoded Secrets
- **Severidade:** `CRITICAL`
- **Arquivo / Localização:** `app.py (Linha 7)`, `controllers.py (Linhas 284 a 289)`, `database.py (Linhas 75 a 79)`
- **Descrição do Problema:** O `SECRET_KEY` do Flask está fixo no código e, pior, é **devolvido em texto plano** pelo endpoint público `GET /health` (junto com `db_path` e `debug`). O seed cria usuários com senhas fixas (`admin123`, `123456`, `senha123`).
- **Recomendação de Refatoração:** Mover segredos para `config/settings.py` lendo variáveis de ambiente (`os.environ` / `.env`). Remover qualquer dado de configuração da resposta do health check. Seed só para ambiente de desenvolvimento, com senhas vindas de env.

### Achado #2: SQL Injection Vulnerability
- **Severidade:** `CRITICAL`
- **Arquivo / Localização:** `models.py (Linhas 28, 47 a 50, 57 a 61, 68, 92, 109 a 111, 126 a 129, 140, 148 a 166, 174, 188, 192, 220, 224, 279 a 281, 289 a 297)`
- **Descrição do Problema:** Praticamente todas as queries são montadas por concatenação de strings com dados vindos do request. Em `login_usuario` (L109-111) um payload como `email = "' OR 1=1 --"` autentica sem senha; em `buscar_produtos` (L291-293) o parâmetro `q`/`categoria` da query string é injetado diretamente.
- **Recomendação de Refatoração:** Reescrever a camada Model com queries parametrizadas (`cursor.execute("... WHERE id = ?", (id,))`), centralizadas em repositórios (`models/produto_model.py`, `models/usuario_model.py`, `models/pedido_model.py`).

### Achado #3: Endpoints administrativos sem autenticação (execução de SQL arbitrário)
- **Severidade:** `CRITICAL`
- **Arquivo / Localização:** `app.py (Linhas 47 a 78)`
- **Descrição do Problema:** `POST /admin/query` executa **qualquer SQL** enviado no corpo da requisição e `POST /admin/reset-db` apaga todas as tabelas — ambos sem autenticação nem autorização. Equivale a acesso total ao banco para qualquer cliente HTTP.
- **Recomendação de Refatoração:** Remover `/admin/query` por completo. Se o reset for necessário, mantê-lo apenas como comando CLI de desenvolvimento (`flask reset-db`) ou protegido por middleware de autenticação/role `admin`.

### Achado #4: Senhas armazenadas e expostas em texto plano
- **Severidade:** `CRITICAL`
- **Arquivo / Localização:** `models.py (Linhas 83, 99, 109 a 111, 126 a 129)`, `database.py (Linha 31)`
- **Descrição do Problema:** Senhas são gravadas sem hash e comparadas em texto plano no login. `GET /usuarios` e `GET /usuarios/<id>` retornam o campo `senha` de todos os usuários.
- **Recomendação de Refatoração:** Usar `werkzeug.security.generate_password_hash` / `check_password_hash` no Controller de usuários; o Model nunca deve serializar o campo de senha (criar `to_dict()` / serializer sem o campo).

### Achado #5: God File / God Class
- **Severidade:** `CRITICAL`
- **Arquivo / Localização:** `models.py (Linhas 1 a 314)`, `app.py (Linhas 1 a 88)`
- **Descrição do Problema:** `models.py` concentra acesso a dados de 4 entidades, serialização, regras de negócio (cálculo de desconto L256-262, validação/baixa de estoque L139-166) e relatórios. `app.py` mistura bootstrap, registro de rotas, rotas administrativas com SQL e configuração.
- **Recomendação de Refatoração:** Separar em `models/` (um arquivo por entidade, apenas persistência), `controllers/` (regras: pedido, estoque, desconto, relatório), `routes/` (Blueprints por recurso) e `config/` (settings + conexão).

### Achado #6: Bypass de Camadas / Regras em Rotas
- **Severidade:** `HIGH`
- **Arquivo / Localização:** `app.py (Linhas 47 a 78)`, `controllers.py (Linhas 264 a 274)`, `models.py (Linhas 139 a 146, 256 a 262)`, `controllers.py (Linhas 208 a 210, 247 a 250)`
- **Descrição do Problema:** Handlers HTTP executam SQL diretamente (`reset_database`, `executar_query`, `health_check`). Ao mesmo tempo, regras de negócio (desconto por faixa de faturamento, verificação de estoque) estão na camada de dados, e efeitos colaterais de notificação (e-mail/SMS/push simulados) estão no controller.
- **Recomendação de Refatoração:** Rotas apenas recebem request e delegam ao Controller; Controller aplica regras e chama Models; Models apenas persistem. Notificações extraídas para um serviço (`services/notificacao_service.py`).

### Achado #7: Modo DEBUG ativo e servidor exposto em 0.0.0.0
- **Severidade:** `HIGH`
- **Arquivo / Localização:** `app.py (Linhas 8 e 88)`
- **Descrição do Problema:** `DEBUG = True` com `host="0.0.0.0"` habilita o debugger interativo do Werkzeug na rede, o que permite execução remota de código em caso de exceção.
- **Recomendação de Refatoração:** Ler `DEBUG`, `HOST` e `PORT` de variáveis de ambiente em `config/settings.py`, com `DEBUG=False` por padrão.

### Achado #8: Queries N+1 no Banco
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `models.py (Linhas 171 a 201, 203 a 233, 139 a 166, 239 a 254)`
- **Descrição do Problema:** `get_todos_pedidos` e `get_pedidos_usuario` fazem 1 query por pedido + 1 query por item para buscar o nome do produto (1 + N + N·M queries). `criar_pedido` consulta cada produto duas vezes. `relatorio_vendas` faz 5 `COUNT`/`SUM` separados.
- **Recomendação de Refatoração:** Usar `JOIN` entre `pedidos`, `itens_pedido` e `produtos` em uma única query e agrupar em memória; no relatório, uma única query com `SUM(CASE WHEN status = ... )`.

### Achado #9: Tratamento Inseguro de Exceções
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `controllers.py (Linhas 10 a 12, 21 a 22, 60 a 62, 95 a 96, 108 a 109, 125 a 126, 133 a 134, 143 a 144, 164 a 165, 185 a 186, 218 a 220, 226 a 227, 234 a 235, 254 a 255, 261 a 262, 291 a 292)`, `app.py (Linhas 77 a 78)`
- **Descrição do Problema:** Todas as funções usam `except Exception as e: return jsonify({"erro": str(e)}), 500`, vazando mensagens internas do SQLite (incluindo trechos de SQL) para o cliente. Payload de erro não é padronizado.
- **Recomendação de Refatoração:** Criar `middlewares/error_handler.py` com `@app.errorhandler` global, exceções de domínio (`NotFoundError`, `ValidationError`) e resposta padrão `{"sucesso": false, "erro": "<mensagem segura>"}`, logando o detalhe internamente.

### Achado #10: Conexão global compartilhada e ausência de transações
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `database.py (Linhas 4 a 11)`, `models.py (Linhas 133 a 169)`
- **Descrição do Problema:** Uma única conexão SQLite global com `check_same_thread=False` é compartilhada entre todas as threads/requests. `criar_pedido` faz vários `INSERT`/`UPDATE` sem rollback em caso de falha e a verificação de estoque não é atômica (condição de corrida entre requests concorrentes).
- **Recomendação de Refatoração:** Conexão por request via `flask.g` + `teardown_appcontext` em `config/database.py`; criação de pedido dentro de transação (`with conn:`) com `UPDATE ... WHERE estoque >= ?`.

### Achado #11: Duplicação de código
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `models.py (Linhas 12 a 21, 31 a 40, 304 a 313 / 79 a 86, 95 a 102 / 171 a 201 vs 203 a 233)`, `controllers.py (Linhas 28 a 50 vs 72 a 90)`
- **Descrição do Problema:** Mapeamento row→dict de produto e usuário repetido 3 e 2 vezes; `get_pedidos_usuario` e `get_todos_pedidos` são praticamente idênticas; validações de criação e atualização de produto duplicadas (e divergentes — o update não valida categoria nem tamanho do nome).
- **Recomendação de Refatoração:** Funções `_row_to_produto()` / `_row_to_usuario()` no Model; uma única função de listagem de pedidos com filtro opcional; validador único `validar_produto(dados)` no Controller.

### Achado #12: Logs e Print Statements Dispersos
- **Severidade:** `LOW`
- **Arquivo / Localização:** `controllers.py (Linhas 8, 11, 57, 61, 106, 161, 179, 182, 208 a 210, 219, 248, 250)`, `app.py (Linhas 56, 83 a 86)`
- **Descrição do Problema:** Rastreamento feito com `print()`, incluindo e-mails de usuários em logs de login.
- **Recomendação de Refatoração:** Usar o módulo `logging` (`app.logger`) com níveis e sem dados pessoais.

### Achado #13: Magic Numbers soltos pelo código
- **Severidade:** `LOW`
- **Arquivo / Localização:** `models.py (Linhas 257 a 262)`, `controllers.py (Linhas 47 a 52, 242)`, `app.py (Linha 88)`
- **Descrição do Problema:** Faixas de desconto (`10000`, `5000`, `1000`, `0.1`, `0.05`, `0.02`), limites de nome (`2`, `200`), listas de categorias/status e porta `5000` embutidos no código.
- **Recomendação de Refatoração:** Constantes nomeadas (`FAIXAS_DESCONTO`, `CATEGORIAS_VALIDAS`, `STATUS_PEDIDO`) em `config/constants.py` ou no Controller correspondente.

### Achado #14: Nomenclatura ruim de variáveis
- **Severidade:** `LOW`
- **Arquivo / Localização:** `controllers.py (Linhas 14, 56, 64, 98, 136, 160)`, `models.py (Linhas 24, 187 a 193, 219 a 225)`
- **Descrição do Problema:** Parâmetro `id` sobrescreve o builtin do Python; cursores `cursor2`, `cursor3`; variável `prod`.
- **Recomendação de Refatoração:** Renomear para `produto_id`, `usuario_id`, `itens_cursor`, `produto` etc.

### Achado #15: Imports não utilizados e validação de tipos frágil
- **Severidade:** `LOW`
- **Arquivo / Localização:** `models.py (Linha 2)`, `database.py (Linha 2)`, `controllers.py (Linhas 43 a 46, 118 a 121, 170)`
- **Descrição do Problema:** `import sqlite3` em `models.py` e `import os` em `database.py` não são usados. `preco < 0` com string no JSON gera `TypeError` → 500; `float(preco_min)` inválido → 500; `login` quebra com corpo vazio (`dados` é `None`).
- **Recomendação de Refatoração:** Remover imports mortos; validar tipos no Controller (ou com schema) retornando 400.

---

## 🏗️ 3. Plano de Reestruturação para MVC

1. **Camada Model:** `src/models/produto_model.py`, `usuario_model.py`, `pedido_model.py` — somente acesso a dados com queries parametrizadas, `JOIN` para pedidos+itens+produtos, serialização sem campo de senha.
2. **Camada Controller:** `src/controllers/produto_controller.py`, `usuario_controller.py` (hash de senha, login), `pedido_controller.py` (validação de itens, estoque atômico em transação, mudança de status), `relatorio_controller.py` (regra de desconto com constantes nomeadas).
3. **Camada Routes / Views:** `src/routes/` com Blueprints `produtos`, `usuarios`, `pedidos`, `relatorios`, `health` — handlers finos que só leem o request e devolvem JSON padronizado. Remover `/admin/query`; `/admin/reset-db` vira comando CLI de desenvolvimento.
4. **Configurações:** `src/config/settings.py` (lendo `SECRET_KEY`, `DEBUG`, `DB_PATH`, `PORT` do ambiente / `.env`), `src/config/database.py` (conexão por request via `flask.g`, criação de schema e seed separados), `src/middlewares/error_handler.py` (handler global de erros + `logging`).
