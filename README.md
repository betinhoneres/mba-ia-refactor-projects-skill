# 🚀 Refactor-Arch: Skill para Refatoração Arquitetural Automatizada

Bem-vindo ao **refactor-arch**, uma *skill* customizada para o Claude Code CLI desenvolvida para analisar, auditar e refatorar autonomamente projetos legados (Python/Flask, Node.js/Express) em direção a uma arquitetura limpa e desacoplada no padrão **MVC (Model-View-Controller)**.

## 📂 Estrutura do Repositório

O ambiente contém a raiz do repositório com três subprojetos legados, além dos arquivos de configuração da *skill*:

```
mba-ia-refactor-projects-skill/
├── .claude/
│   └── skills/
│       └── refactor-arch/
│           ├── SKILL.md                     # Orquestração principal da skill (Fases 1 e 2 configuradas)
│           ├── analise_heuristica.md        # Heurísticas de detecção e mapeamento inicial
│           ├── catalogo_anti_patterns.md    # Catálogo de débitos técnicos e anti-patterns
│           ├── guideline_arquitetura_mvc.md # Blueprint da arquitetura MVC alvo
│           └── template_relatorio.md        # Template oficial para relatórios de auditoria
│           └── playbook_refatoracao.md      # Playbook de Refatoração (Exemplos Antes vs. Depois)
├── code-smells-project/                # Python / Flask 3.1.1 (SQL nativo / Alto acoplamento)
├── ecommerce-api-legacy/               # Node.js / Express 4.18 (God Class / BD em memória)
└── task-manager-api/                   # Python / Flask 3.0 / SQLAlchemy (Parcialmente organizado)
```

## 🛠️ Ambiente e Pré-requisitos

* **Node.js:** `v20.17.0` (Versão LTS recomendada)
* **Gerenciador de Pacotes:** `npm` (v10.8.2+)
* **Ferramenta CLI:** `@anthropic-ai/claude-code` (v2.1.314)
* **Autenticação:** Conta no Anthropic Console via faturamento de uso de API (`claude login`)

## ⚙️ Como Instalar e Usar a Skill

### 1. Instalar o Claude Code CLI Globalmente

Caso ainda não esteja instalado no seu ambiente:

```bash
npm install -g @anthropic-ai/claude-code
```

### 2. Autenticar

Execute o fluxo de login para vincular sua conta do Anthropic Console (faturamento por API):

```bash
claude login
```

### 3. Executar a Skill no Seu Projeto

Abra o terminal na pasta raiz do repositório ou navegue até o diretório de um subprojeto específico e acione a skill:

```bash
claude "/refactor-arch"
```

## 📋 Fases de Execução

A skill `refactor-arch` opera de forma sequencial através de três fases principais:

* **Fase 1 — Análise de Projeto:** Mapeia a estrutura, detectando a linguagem, framework, driver de banco de dados, modelo arquitetural e métricas iniciais do código.
* **Fase 2 — Auditoria e Diagnóstico:** Varredura estática profunda utilizando o catálogo de débitos técnicos (`catalogo_anti_patterns.md`), classificando vulnerabilidades (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`) e gerando um relatório estruturado em `reports/audit-project-<id>.md`, finalizando com uma pausa de segurança obrigatória.
* **Fase 3 — Refatoração Automatizada:** Executa a separação modular seguindo as diretrizes do padrão MVC após a confirmação do usuário *(aguardando gatilho manual)*.