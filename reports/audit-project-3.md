# 📑 Relatório de Auditoria Arquitetural — `task-manager-api`

**Data da Auditoria:** `2026-10-30`  
**Stack Detectada:** `Python 3` / `Flask 3.0.0 + Flask-SQLAlchemy 3.1.1`  
**Domínio da Aplicação:** `Task Manager (tarefas, usuários, categorias, relatórios de produtividade)`  

---

## 📊 1. Resumo Executivo por Severidade

| Severidade | Quantidade Identificada |
|---|---|
| 🚨 **CRITICAL** | `3` |
| ⚠️ **HIGH** | `3` |
| ⚡ **MEDIUM** | `6` |
| 🔵 **LOW** | `4` |
| **TOTAL** | **`16`** |

---

## 🔍 2. Detalhamento dos Achados

### Achado #1: Hardcoded Secrets
- **Severidade:** `CRITICAL`
- **Arquivo / Localização:** `app.py (Linhas 11 e 13)`, `services/notification_service.py (Linhas 7 a 10)`, `seed.py (Linhas 19, 26, 33)`
- **Descrição do Problema:** `SECRET_KEY = 'super-secret-key-123'` e credenciais SMTP (`taskmanager@gmail.com` / `senha123`) fixas no código. `python-dotenv` está nas dependências, mas não é utilizado. O seed cria um admin com senha `1234`.
- **Recomendação de Refatoração:** `config/settings.py` carregando `.env` com `load_dotenv()` e lendo `SECRET_KEY`, `DATABASE_URL`, `SMTP_*` de `os.environ`; `NotificationService` recebe as credenciais por injeção/configuração.

### Achado #2: Exposição do hash de senha nas respostas da API
- **Severidade:** `CRITICAL`
- **Arquivo / Localização:** `models/user.py (Linha 21)`, `routes/user_routes.py (Linhas 33, 85, 129, 209)`
- **Descrição do Problema:** `User.to_dict()` inclui o campo `password`, e esse dicionário é retornado por `GET /users/<id>`, `POST /users`, `PUT /users/<id>` e `POST /login`. Como o hash é MD5 sem salt (Achado #5), ele é trivialmente quebrável.
- **Recomendação de Refatoração:** Remover `password` da serialização do Model (ou usar um schema `marshmallow` — já presente nas dependências — com `load_only=True` no campo).

### Achado #3: Autenticação falsa e escalonamento de privilégio
- **Severidade:** `CRITICAL`
- **Arquivo / Localização:** `routes/user_routes.py (Linhas 52, 71 a 72, 119 a 122, 210)`, todas as rotas em `routes/`
- **Descrição do Problema:** O login devolve `'fake-jwt-token-' + id` (previsível e nunca validado). Nenhuma rota exige autenticação, e qualquer cliente pode criar um usuário com `role: "admin"` ou promover-se via `PUT /users/<id>`.
- **Recomendação de Refatoração:** Emitir token assinado (ex.: `itsdangerous`/JWT com `SECRET_KEY` de ambiente), `middlewares/auth.py` com decorator `@login_required` / `@admin_required`; `role` só alterável por admin.

### Achado #4: Bypass de Camadas / Regras em Rotas
- **Severidade:** `HIGH`
- **Arquivo / Localização:** `routes/task_routes.py (Linhas 11 a 299)`, `routes/user_routes.py (Linhas 10 a 211)`, `routes/report_routes.py (Linhas 12 a 223)`
- **Descrição do Problema:** Embora existam pastas `models/`, `routes/` e `services/`, os handlers fazem queries ORM, validações, cálculo de atraso, estatísticas e exclusão em cascata manual (`user_routes.py` L140-142). `NotificationService` nunca é chamado. `report_routes.py` também abriga o CRUD de categorias, que não é relatório.
- **Recomendação de Refatoração:** Criar `controllers/task_controller.py`, `user_controller.py`, `report_controller.py`, `category_controller.py`; rotas ficam só com parsing de request e resposta; mover CRUD de categorias para `routes/category_routes.py`; cascata via `relationship(..., cascade='all, delete-orphan')`.

### Achado #5: Hashing de senhas obsoleto (MD5 sem salt)
- **Severidade:** `HIGH`
- **Arquivo / Localização:** `models/user.py (Linhas 27 a 32)`, `routes/user_routes.py (Linhas 64, 115)`
- **Descrição do Problema:** `hashlib.md5` sem salt é quebrável por tabelas rainbow em segundos; mínimo de senha de 4 caracteres agrava o problema.
- **Recomendação de Refatoração:** `werkzeug.security.generate_password_hash` / `check_password_hash` (scrypt/pbkdf2) e política mínima de 8 caracteres.

### Achado #6: Modo DEBUG ativo e servidor exposto em 0.0.0.0
- **Severidade:** `HIGH`
- **Arquivo / Localização:** `app.py (Linha 34)`
- **Descrição do Problema:** `app.run(debug=True, host='0.0.0.0')` expõe o debugger interativo do Werkzeug na rede (execução remota de código).
- **Recomendação de Refatoração:** `DEBUG`, `HOST`, `PORT` vindos do ambiente com `DEBUG=False` por padrão.

### Achado #7: Queries N+1 no Banco
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `routes/task_routes.py (Linhas 41 a 57)`, `routes/user_routes.py (Linha 22)`, `routes/report_routes.py (Linhas 15 a 28, 53 a 68, 159 a 164)`
- **Descrição do Problema:** `GET /tasks` faz 1 query de usuário + 1 de categoria **por tarefa**; `GET /users` dispara lazy-load de `u.tasks` por usuário; o resumo faz 12 `COUNT` separados e 1 query de tarefas por usuário; `GET /categories` faz 1 `COUNT` por categoria.
- **Recomendação de Refatoração:** `joinedload(Task.user)`, `joinedload(Task.category)`; agregações com `db.session.query(Task.status, func.count()).group_by(Task.status)`; contagens por usuário/categoria com `GROUP BY`.

### Achado #8: Falta de paginação em listagens
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `routes/task_routes.py (Linhas 14, 266, 281)`, `routes/user_routes.py (Linha 12)`, `routes/report_routes.py (Linhas 30, 53, 159)`
- **Descrição do Problema:** `Task.query.all()` / `User.query.all()` carregam a tabela inteira em memória e na resposta.
- **Recomendação de Refatoração:** `db.paginate(select(Task), page=..., per_page=...)` com limite máximo e metadados de paginação na resposta; filtros de atraso feitos no SQL (`Task.due_date < now`).

### Achado #9: Tratamento Inseguro de Exceções
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `routes/task_routes.py (Linhas 62, 137, 204, 236)`, `routes/user_routes.py (Linhas 130, 149)`, `routes/report_routes.py (Linhas 186, 207, 221)`, `utils/helpers.py (Linhas 46, 49, 88)`, `services/notification_service.py (Linhas 23 a 25)`
- **Descrição do Problema:** `except:` sem tipo captura até `KeyboardInterrupt`/`SystemExit` e engole a causa real sem log; `GET /tasks` transforma qualquer bug em "Erro interno" opaco.
- **Recomendação de Refatoração:** Capturar exceções específicas (`ValueError`, `SQLAlchemyError`), handler global em `middlewares/error_handler.py` com logging e payload padrão.

### Achado #10: APIs Deprecated / Obsoletas
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `Query.get()` → `routes/task_routes.py (Linhas 42, 51, 67, 117, 122, 158, 188, 195, 227)`, `routes/user_routes.py (Linhas 29, 94, 136, 155)`, `routes/report_routes.py (Linhas 105, 192, 213)`; `datetime.utcnow()` → `models/task.py (Linhas 15, 16, 52)`, `models/user.py (Linha 14)`, `models/category.py (Linha 11)`, `routes/task_routes.py (Linhas 31, 72, 215, 285)`, `routes/user_routes.py (Linha 172)`, `routes/report_routes.py (Linhas 35, 42, 45, 71, 133)`, `utils/helpers.py (Linha 38)`, `services/notification_service.py (Linha 35)`
- **Descrição do Problema:** `Model.query.get()` é API legada no SQLAlchemy 2.x (usado pelo Flask-SQLAlchemy 3.1) e emite `LegacyAPIWarning`. `datetime.utcnow()` está deprecated desde o Python 3.12.
- **Recomendação de Refatoração:** `db.session.get(Task, task_id)` / `db.get_or_404(Task, task_id)`; `datetime.now(timezone.utc)`.

### Achado #11: Duplicação de código
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `models/task.py (Linhas 50 a 60)` vs `routes/task_routes.py (Linhas 30 a 39, 71 a 80, 283 a 287)`, `routes/user_routes.py (Linhas 171 a 180)`, `routes/report_routes.py (Linhas 34 a 36, 132 a 135)`; `routes/task_routes.py (Linhas 17 a 28)` vs `models/task.py (Linhas 23 a 36)`; `routes/user_routes.py (Linhas 61, 106)` vs `utils/helpers.py (Linhas 19 a 23)`; `utils/helpers.py (Linhas 57 a 116)`
- **Descrição do Problema:** A regra de "tarefa atrasada" está reimplementada 7 vezes, apesar de existir `Task.is_overdue()`. A serialização da tarefa é refeita manualmente em vez de usar `to_dict()`. Regex de e-mail duplicada; `process_task_data()` e as constantes `VALID_STATUSES`, `MAX_TITLE_LENGTH` etc. existem em `helpers.py` mas nunca são usadas, enquanto as rotas repetem as mesmas validações em `create_task`/`update_task`.
- **Recomendação de Refatoração:** Uma única fonte de verdade: `Task.is_overdue()` + `to_dict()` incluindo `overdue`; validações de tarefa/usuário centralizadas no Controller (ou schemas marshmallow) usando as constantes.

### Achado #12: Validação de entrada frágil (erros 500 em vez de 400)
- **Severidade:** `MEDIUM`
- **Arquivo / Localização:** `routes/task_routes.py (Linhas 96, 113, 167, 182, 261, 264)`, `routes/report_routes.py (Linhas 196 a 202)`, `routes/user_routes.py (Linha 125)`
- **Descrição do Problema:** `priority` como string gera `TypeError` na comparação; `int(priority)` na busca levanta `ValueError` não tratado; `update_category` quebra com corpo vazio (`data` é `None`); `active` aceita qualquer tipo; cor de categoria não é validada (há `is_valid_color` não utilizado).
- **Recomendação de Refatoração:** Schemas de entrada (marshmallow) no Controller, retornando 400 com mensagens claras.

### Achado #13: Logs e Print Statements Dispersos
- **Severidade:** `LOW`
- **Arquivo / Localização:** `routes/task_routes.py (Linhas 149, 153, 219, 234)`, `routes/user_routes.py (Linhas 83, 89, 147)`, `services/notification_service.py (Linhas 21, 24)`, `utils/helpers.py (Linhas 39 a 41)`
- **Descrição do Problema:** Rastreamento via `print()` sem níveis nem destino configurável.
- **Recomendação de Refatoração:** `logging` padrão / `app.logger`.

### Achado #14: Imports e dependências não utilizados
- **Severidade:** `LOW`
- **Arquivo / Localização:** `app.py (Linha 7: os, sys, json)`, `routes/task_routes.py (Linha 7: json, os, sys, time)`, `routes/user_routes.py (Linha 6: hashlib, json)`, `routes/report_routes.py (Linhas 7 a 8: format_date, json)`, `models/task.py (Linha 3)`, `utils/helpers.py (Linhas 3 a 7)`, `requirements.txt (marshmallow, requests, python-dotenv)`
- **Descrição do Problema:** Imports mortos e dependências declaradas que não são usadas poluem o código e a superfície de ataque.
- **Recomendação de Refatoração:** Remover imports mortos; usar de fato `marshmallow` e `python-dotenv` na refatoração ou removê-los, e remover `requests`.

### Achado #15: Magic Numbers soltos pelo código
- **Severidade:** `LOW`
- **Arquivo / Localização:** `routes/task_routes.py (Linhas 96, 99, 104, 113, 167, 169, 182)`, `routes/user_routes.py (Linhas 64, 115)`, `routes/report_routes.py (Linhas 24 a 28, 45, 129, 180)`
- **Descrição do Problema:** Limites `3`, `200`, `1..5`, `4`, janela de `7` dias, `priority <= 2` como "alta prioridade" e cor `'#000000'` espalhados, embora as constantes já existam em `utils/helpers.py`.
- **Recomendação de Refatoração:** Consolidar em `config/constants.py` e referenciar nos Controllers.

### Achado #16: Nomenclatura ruim de variáveis e código verboso
- **Severidade:** `LOW`
- **Arquivo / Localização:** `routes/report_routes.py (Linhas 24 a 28: p1..p5)`, `routes/task_routes.py (Linhas 16, 51: t, cat)`, `routes/user_routes.py (Linha 14: u)`, `models/category.py (Linha 14: d)`, `models/task.py (Linhas 38 a 60)`, `models/user.py (Linhas 34 a 38)`
- **Descrição do Problema:** Nomes de uma letra/abreviados; métodos `validate_status`/`validate_priority` de instância que não usam `self`; `if cond: return True else: return False` em vez de `return cond`.
- **Recomendação de Refatoração:** Nomes descritivos (`task`, `user`, `priority_counts`), validações como funções/constantes de domínio e expressões booleanas diretas.

---

## 🏗️ 3. Plano de Reestruturação para MVC

1. **Camada Model:** `src/models/task.py`, `user.py`, `category.py` — apenas entidades SQLAlchemy, relacionamentos com `cascade='all, delete-orphan'`, `to_dict()` sem senha e com `overdue` derivado de `is_overdue()`, timestamps com `datetime.now(timezone.utc)`.
2. **Camada Controller:** `src/controllers/task_controller.py` (CRUD, busca, estatísticas com `GROUP BY`, paginação, `joinedload`), `user_controller.py` (hash com werkzeug, login com token assinado, regra de roles), `category_controller.py`, `report_controller.py` (resumo e relatório por usuário com agregações SQL); validações via schemas marshmallow e constantes centralizadas; `services/notification_service.py` com credenciais injetadas.
3. **Camada Routes / Views:** `src/routes/task_routes.py`, `user_routes.py`, `category_routes.py`, `report_routes.py`, `auth_routes.py` — Blueprints finos que leem request, chamam o Controller e devolvem JSON; rotas protegidas pelos decorators de `middlewares/auth.py`.
4. **Configurações:** `src/config/settings.py` (`load_dotenv()`, `SECRET_KEY`, `DATABASE_URL`, `SMTP_*`, `DEBUG`), `src/config/database.py` (instância `db`), `src/middlewares/error_handler.py` (handlers globais + `logging`), `src/app.py` como application factory (`create_app()`); `seed.py` lendo senhas do ambiente.
