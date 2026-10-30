---

#### 6. `playbook_refatoracao.md` (Playbook de Transformação)

```markdown
# 📖 Playbook de Refatoração (Exemplos Antes vs. Depois)

Este documento contém os padrões concretos de código para corrigir cada um dos anti-patterns catalogados durante a Fase 3.

---

### 1. Hardcoded Secrets → Environment Variables

**ANTES (Inseguro):**
```python
# app.py
app.config["SECRET_KEY"] = "super-secret-key-123"
```

**DEPOIS (Refatorado):**

```python
# src/config/config.py
import os

class Config:
    SECRET_KEY = os.getenv("SECRET_KEY", "default-dev-key")
```