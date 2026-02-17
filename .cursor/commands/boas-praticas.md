#  Python Best Practices Guide
##  1. Arquitetura e Modelagem (Pydantic V2)
Sempre valide entradas e saídas. Não confie em dicionários (`dict`) puros para dados estruturados.

- **Type Safety:** Use `BaseModel` para definir contratos de dados claros.
- **Settings Management:** Utilize `pydantic-settings` para gerenciar variáveis de ambiente.
- **Strict Mode:** Sempre que possível, utilize `model_validate(data, strict=True)`.

---

##  2. Tooling & DX (Developer Experience)
O ecossistema mudou. Para máxima performance no desenvolvimento:

* **Gerenciador de Pacotes:** Use `uv` ou `Poetry` em vez de `pip` direto.
* **Linter & Formatter:** Utilize **Ruff**. Ele substitui Flake8, Isort e Black, sendo 10-100x mais rápido.
* **CLI:** Para comandos de terminal, o **Typer** é o padrão ouro por ser baseado em Type Hints.

---

## 📝 3. Estilo de Código & Tipagem
O Python moderno é estritamente tipado na prática (Static Typing).

* **Annotated Types:** Use `Annotated` para adicionar metadados a argumentos (ex: validações do Typer).
* **F-Strings:** Prefira sempre f-strings: `f"Olá {nome}"`.
* **Pathlib:** Use `pathlib.Path` em vez de `os.path`.
* **Type Hints:** Funções devem ter assinaturas completas:
    ```python
    def process_data(items: list[str]) -> bool:
    ```
#  Python Best Practices Guide
##  1. Arquitetura e Modelagem (Pydantic V2)
Sempre valide entradas e saídas. Não confie em dicionários (`dict`) puros para dados estruturados.

- **Type Safety:** Use `BaseModel` para definir contratos de dados claros.
- **Settings Management:** Utilize `pydantic-settings` para gerenciar variáveis de ambiente.
- **Strict Mode:** Sempre que possível, utilize `model_validate(data, strict=True)`.

---

##  2. Tooling & DX (Developer Experience)
O ecossistema mudou. Para máxima performance no desenvolvimento:

* **Gerenciador de Pacotes:** Use `uv` ou `Poetry` em vez de `pip` direto.
* **Linter & Formatter:** Utilize **Ruff**. Ele substitui Flake8, Isort e Black, sendo 10-100x mais rápido.
* **CLI:** Para comandos de terminal, o **Typer** é o padrão ouro por ser baseado em Type Hints.

---

## 📝 3. Estilo de Código & Tipagem
O Python moderno é estritamente tipado na prática (Static Typing).

* **Annotated Types:** Use `Annotated` para adicionar metadados a argumentos (ex: validações do Typer).
* **F-Strings:** Prefira sempre f-strings: `f"Olá {nome}"`.
* **Pathlib:** Use `pathlib.Path` em vez de `os.path`.
* **Type Hints:** Funções devem ter assinaturas completas:
    ```python
    def process_data(items: list[str]) -> bool:
    ```

---

## 🧪 4. Testes e Qualidade
Código sem teste é código legado no momento em que é escrito.

* **Framework:** Use **Pytest**.
* **Mocking:** Utilize a fixture `mocker` do `pytest-mock`.
* **Coverage:** Mantenha uma cobertura mínima de 80%, focando nos caminhos críticos (Happy Path e Edge Cases).

---

## 🚀 5. Performance e Concorrência
* **AsyncIO:** Use `async/await` para tarefas de I/O Bound (Chamadas de API, DB).
* **Multiprocessing:** Use para tarefas de CPU Bound (Cálculos pesados, Processamento de Imagem).
* **Generators:** Use `yield` para processar grandes volumes de dados sem estourar a memória RAM.

---

## 🔒 6. Segurança
* **Secrets:** Nunca coloque chaves de API no código. Use arquivos `.env`.
* **Dependências:** Execute `pip-audit` ou ferramentas similares regularmente.
* **Sanitização:** O Pydantic já ajuda, mas sempre valide tipos de arquivos e tamanhos de upload.

---

## 📂 7. Estrutura de Projeto Sugerida
```text
projeto/
├── src/
│   ├── models/      # Modelos Pydantic
│   ├── core/        # Lógica de negócio pura
│   ├── api/         # Endpoints ou CLI Commands
│   └── main.py      # Entry point
├── tests/           # Suíte de testes
├── .env             # Variáveis sensíveis (ignored)
├── pyproject.toml   # Configurações de ferramentas (Ruff, Pytest, UV)
└── README.md
---

## 🧪 4. Testes e Qualidade
Código sem teste é código legado no momento em que é escrito.

* **Framework:** Use **Pytest**.
* **Mocking:** Utilize a fixture `mocker` do `pytest-mock`.
* **Coverage:** Mantenha uma cobertura mínima de 80%, focando nos caminhos críticos (Happy Path e Edge Cases).

---

## 🚀 5. Performance e Concorrência
* **AsyncIO:** Use `async/await` para tarefas de I/O Bound (Chamadas de API, DB).
* **Multiprocessing:** Use para tarefas de CPU Bound (Cálculos pesados, Processamento de Imagem).
* **Generators:** Use `yield` para processar grandes volumes de dados sem estourar a memória RAM.

---

## 🔒 6. Segurança
* **Secrets:** Nunca coloque chaves de API no código. Use arquivos `.env`.
* **Dependências:** Execute `pip-audit` ou ferramentas similares regularmente.
* **Sanitização:** O Pydantic já ajuda, mas sempre valide tipos de arquivos e tamanhos de upload.

---

## 📂 7. Estrutura de Projeto Sugerida
```text
projeto/
├── src/
│   ├── models/      # Modelos Pydantic
│   ├── core/        # Lógica de negócio pura
│   ├── api/         # Endpoints ou CLI Commands
│   └── main.py      # Entry point
├── tests/           # Suíte de testes
├── .env             # Variáveis sensíveis (ignored)
├── pyproject.toml   # Configurações de ferramentas (Ruff, Pytest, UV)
└── README.md