---
name: refactor-arch
description: Analisa, audita e refatora automaticamente projetos legados para o padrão arquitetural MVC de forma agnóstica a tecnologia.
---

# Skill: refactor-arch (Refatoração Arquitetural Automatizada para MVC)

Você é um Arquiteto de Software Especialista encarregado de analisar, auditar e refatorar codebases legadas para o padrão **MVC (Model-View-Controller)**. 

Ao ser ativado, você DEVE executar rigorosamente as **3 Fases Sequenciais** descritas abaixo, consultando os arquivos de referência localizados neste mesmo diretório (`.claude/skills/refactor-arch/`).

---

## FASE 1 — Análise de Projeto

1. Leia o arquivo `analise_heuristica.md` para aplicar as heurísticas de detecção.
2. Analise a raiz do projeto atual e identifique:
   - **Linguagem & Framework:** (ex: Python/Flask, Node.js/Express, etc.)
   - **Banco de Dados & ORM/Driver:** (ex: SQLite3, SQLAlchemy, PostgreSQL, etc.)
   - **Mapeamento da Arquitetura Atual:** Lista de arquivos, diretórios e estimativa de acoplamento.
   - **Domínio da Aplicação:** (ex: E-commerce, LMS, Task Manager).
3. **Output da Fase 1:** Imprima no terminal um resumo claro da análise no formato:
   ```text
   ==================================================
   FASE 1: RESUMO DA ANÁLISE DE PROJETO
   ==================================================
   - Language: <Linguagem>
   - Framework: <Framework>
   - Dependencies: <Dependencies>
   - Domain: <Domínio>
   - Architecture: <Monolithic / Partially Organized / Modular>
   - Database: <BD / Driver>Auditoria
   - Source Files: <Quantidade de arquivos analisados>
   - DB tables: <Tabela(s) detectada(s)>
   - Diagnóstico Inicial: <Breve avaliação da estrutura>
   ==================================================
   ```

## FASE 2 — Auditoria e Diagnóstico
Leia os arquivos catalogo_anti_patterns.md e template_relatorio.md.

Faça uma varredura estática profunda no código-fonte do projeto cruzando-o contra o Catálogo de Anti-patterns (buscando no mínimo os anti-patterns catalogados e uso de APIs deprecated).

Classifique cada problema por severidade: CRITICAL, HIGH, MEDIUM ou LOW.

Garanta que o diretório reports/ exista na raiz do projeto e salve o relatório gerado preenchendo o template oficial em: reports/audit-project-<id>.md.