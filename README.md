# move-tech-cloud-application

API de micro e-commerce com pedidos e itens, construída em Python com FastAPI —
versionada, conteinerizada e publicada na **Magalu Cloud**.

> Parte do curso **Move Tech** — Magalu × Prósper Digital Skills  
> Formação em Cloud Computing para iniciantes

---

## O que tem aqui

Uma API de pedidos e itens que persiste os dados em banco relacional via SQLAlchemy,
expõe métricas no formato Prometheus e roda em um cluster Kubernetes na Magalu Cloud,
com deploy automatizado pelo GitHub Actions.

| Camada | Tecnologia |
|--------|-----------|
| API | FastAPI + Pydantic |
| Persistência | SQLAlchemy 2.0 (PostgreSQL em produção, SQLite local) |
| Container | Docker (`python:3.11-slim`) |
| Orquestração | Kubernetes (`k8s/app.yaml`) |
| Observabilidade | Prometheus (ServiceMonitor) + logs em JSON |
| CI/CD | GitHub Actions (`.github/workflows/deploy.yml`) |

### Endpoints disponíveis

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET` | `/health` | Verifica a API e a conexão com o banco |
| `GET` | `/stats` | Contadores de pedidos e itens |
| `GET` | `/metrics` | Métricas no formato Prometheus |
| `GET` | `/docs` | Documentação interativa (Scalar) |
| `POST` | `/orders` | Cria um novo pedido |
| `GET` | `/orders` | Lista todos os pedidos |
| `GET` | `/orders/{id}` | Retorna um pedido com seus itens |
| `DELETE` | `/orders/{id}` | Cancela um pedido |
| `POST` | `/orders/{id}/items` | Adiciona um item ao pedido |
| `GET` | `/orders/{id}/items` | Lista os itens de um pedido |

A estrutura das tabelas está documentada em [`docs/data-model.md`](docs/data-model.md).

---

## Entregas da Competência 3

- [x] Publicar a imagem no Container Registry da Magalu Cloud
- [x] Criar o manifest Kubernetes (`k8s/app.yaml`)
- [x] Fazer o deploy no cluster Kubernetes da Magalu Cloud
- [x] Configurar o pipeline de CI/CD no GitHub Actions

---

## O Dockerfile

O `Dockerfile` define como a aplicação é empacotada em uma imagem Docker:

```dockerfile
FROM python:3.11-slim          # Imagem base com Python 3.11

WORKDIR /app                   # Diretório de trabalho dentro do container

RUN pip install poetry==1.8.3  # Instala o gerenciador de dependências

COPY pyproject.toml poetry.lock* ./
RUN poetry config virtualenvs.create false && \
    poetry install --without dev --no-root  # Instala apenas as dependências de produção

COPY app/ ./app/               # Copia o código da aplicação

EXPOSE 8000                    # Porta que a aplicação vai escutar

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

O `docker-compose.yml` usa esse Dockerfile para construir e rodar a aplicação localmente.
Na nuvem, o pipeline faz o mesmo — constrói a imagem e publica no registry.

> **Referência:** [Dockerfile — Documentação oficial Docker](https://docs.docker.com/reference/dockerfile/)

---

## Como rodar localmente

**Pré-requisito:** [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado
(Mac e Windows) ou [Docker Engine](https://docs.docker.com/engine/install/) (Linux).

```bash
docker compose up --build
```

Acesse a documentação interativa em: http://localhost:8000/docs

Sem a variável `DATABASE_URL` definida, a aplicação usa SQLite em arquivo (`orders.db`).
Para apontar para um PostgreSQL:

```bash
DATABASE_URL=postgresql+psycopg2://usuario:senha@host:5432/pedidos docker compose up --build
```

### Testes

```bash
poetry install
poetry run pytest
```

---

## Deploy

O workflow `Deploy` (acionado manualmente em **Actions → Deploy → Run workflow**):

1. roda os testes com pytest;
2. constrói a imagem e publica no Container Registry da Magalu Cloud;
3. cria o Secret `db-secret` com a URL do banco;
4. aplica os manifests em `k8s/` e acompanha o rollout.

Secrets necessários no repositório:

| Secret | Para que serve |
|--------|----------------|
| `MGC_REGISTRY_USER` / `MGC_REGISTRY_PASSWORD` | Login no Container Registry |
| `MGC_REGISTRY_NAME` | Nome do registry (compõe o caminho da imagem) |
| `MGC_KUBECONFIG` | Acesso ao cluster Kubernetes |
| `DATABASE_URL` | URL de conexão com o PostgreSQL |

---

## Observabilidade

- **Métricas:** o `prometheus-fastapi-instrumentator` expõe `/metrics`, e o
  `k8s/servicemonitor.yaml` registra o serviço no kube-prometheus-stack.
- **Logs:** saída em JSON (`JsonFormatter` em `app/main.py`), pronta para coleta.
- **Health checks:** `/health` valida a conexão com o banco e alimenta as probes
  de liveness e readiness do Deployment.

---

> Inspirado no tutorial [Construindo APIs robustas utilizando Python](https://github.com/luizalabs/tutorial-python-brasil) do LuizaLabs.
