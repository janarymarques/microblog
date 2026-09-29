# Microblog

Aplicação de exemplo do [Flask Mega-Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world), de Miguel Grinberg. É um microblog no estilo Twitter: usuários se cadastram, publicam posts curtos, seguem uns aos outros e interagem por mensagens privadas.

O objetivo do projeto é **didático**: mostrar como montar uma aplicação Flask completa (autenticação, banco, e-mail, i18n, busca, filas, API REST), não ser um produto pronto para produção.

Versão deste recorte: **0.23**.

---

## Índice

- [Funcionalidades](#funcionalidades)
- [Stack](#stack)
- [Estrutura do projeto](#estrutura-do-projeto)
- [Requisitos](#requisitos)
- [Como executar localmente](#como-executar-localmente)
- [Primeiro uso](#primeiro-uso)
- [Variáveis de ambiente](#variáveis-de-ambiente)
- [Serviços opcionais](#serviços-opcionais)
- [Rotas da interface](#rotas-da-interface)
- [API REST](#api-rest)
- [Modelo de dados](#modelo-de-dados)
- [CLI](#cli)
- [Docker](#docker)
- [Referência](#referência)

---

## Funcionalidades

| Área | O que a aplicação faz |
| --- | --- |
| Autenticação | Cadastro, login, logout, “lembrar-me”, reset de senha por e-mail (JWT) |
| Posts | Publicar textos de até 140 caracteres; detecção de idioma (`langdetect`) |
| Social | Seguir / deixar de seguir; feed da home (próprios posts + de quem você segue); Explore (todos os posts) |
| Perfil | Editar username e “sobre mim”; avatar via Gravatar; último acesso |
| Mensagens | Mensagens privadas; contador de não lidas |
| Notificações | Polling em `/notifications` (mensagens e progresso de tarefas) |
| Busca | Full-text nos posts via Elasticsearch (se configurado) |
| Tradução | Traduzir posts com Microsoft Translator (se configurado) |
| Internacionalização | Flask-Babel; idiomas configurados: inglês e espanhol |
| Exportação | Job em background (Redis/RQ) que envia os posts em JSON por e-mail |
| API | REST para usuários, seguidores e tokens |
| Operação | Paginação, páginas 404/500, log em arquivo ou stdout, alerta de erro por SMTP |

---

## Stack

- **Python 3** + **Flask 3**
- **SQLAlchemy 2** + **Flask-Migrate** (Alembic)
- **SQLite** por padrão; Postgres via `DATABASE_URL` (`psycopg2`)
- **Flask-Login**, **Flask-WTF**, **Flask-Mail**, **Flask-Moment**, **Flask-Babel**
- **Elasticsearch** (busca, opcional)
- **Redis** + **RQ** (filas, opcional para o núcleo social)
- **Gunicorn** (produção / Docker)

---

## Estrutura do projeto

```
microblog-0.23/
├── app/
│   ├── __init__.py          # factory create_app(), extensões, blueprints
│   ├── models.py            # User, Post, Message, Notification, Task
│   ├── email.py             # envio de e-mail (síncrono ou via RQ)
│   ├── search.py            # indexação e consulta no Elasticsearch
│   ├── translate.py         # Microsoft Translator
│   ├── tasks.py             # worker: export_posts
│   ├── cli.py               # flask translate {init,update,compile}
│   ├── auth/                # login, registro, reset de senha
│   ├── main/                # feed, perfil, follow, busca, mensagens
│   ├── api/                 # REST /api
│   ├── errors/              # 404 e 500
│   ├── templates/           # Jinja2
│   ├── static/              # CSS
│   └── translations/        # catálogos Babel
├── migrations/              # histórico do banco
├── config.py                # configuração (lê .env)
├── microblog.py             # ponto de entrada Flask
├── .flaskenv                # FLASK_APP e FLASK_DEBUG
├── requirements.txt
├── Dockerfile / boot.sh
└── Procfile                 # web + worker (Heroku-style)
```

A app usa o padrão **application factory** (`create_app()` em `app/__init__.py`) e **blueprints**:

| Blueprint | Prefixo | Papel |
| --- | --- | --- |
| `main` | `/` | Interface autenticada |
| `auth` | `/auth` | Autenticação |
| `api` | `/api` | JSON REST |
| `errors` | — | Handlers HTTP |
| `cli` | — | Comandos Flask |

---

## Requisitos

- Python 3.10+ recomendado (o tutorial original usa 3.x)
- `pip` e um ambiente virtual

Para o **mínimo** (cadastrar, postar, seguir): só Python. Banco SQLite é criado automaticamente após `flask db upgrade`.

Opcional:

- Redis (exportação de posts e progresso de tarefas)
- Elasticsearch (busca)
- Servidor SMTP (reset de senha, exportação, alerta de erro)
- Chave Microsoft Translator (traduzir posts)

---

## Como executar localmente

No PowerShell, na raiz do projeto:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt
flask db upgrade
flask run
```

No Linux/macOS:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
flask db upgrade
flask run
```

Abra [http://127.0.0.1:5000](http://127.0.0.1:5000).

O arquivo `.flaskenv` já define:

```
FLASK_APP=microblog.py
FLASK_DEBUG=1
```

`config.py` carrega um `.env` na raiz, se existir. Sem `.env`, o banco padrão é `app.db` (SQLite) e a `SECRET_KEY` usa o fallback `you-will-never-guess`.

### Compilar traduções (opcional)

```powershell
flask translate compile
```

Sem isso, strings em espanhol podem aparecer em inglês (msgid).

### Usuário pelo shell

```powershell
flask shell
```

```python
from app import db
from app.models import User
u = User(username='alice', email='alice@example.com')
u.set_password('secret')
db.session.add(u)
db.session.commit()
```

---

## Primeiro uso

1. Acesse `/auth/register` e crie uma conta (o e-mail pode ser fictício se SMTP não estiver configurado).
2. Publique um post na home.
3. Faça logout, registre um segundo usuário e publique outro post.
4. Em **Explore**, veja todos os posts.
5. Abra o perfil do outro usuário e **Follow**.
6. Na home, o feed mostra posts seus e de quem você segue.
7. Teste **Edit Profile**, **Send private message** e **Messages**.

Recursos que **não funcionam** sem configuração extra: busca, tradução de posts, reset de senha por e-mail e exportação de posts.

---

## Variáveis de ambiente

Definidas em `config.py` (via `.env` ou o ambiente do SO):

| Variável | Padrão | Uso |
| --- | --- | --- |
| `SECRET_KEY` | `you-will-never-guess` | Sessões, CSRF, JWT de reset |
| `SERVER_NAME` | — | Hostname da app (útil atrás de proxy) |
| `DATABASE_URL` | `sqlite:///.../app.db` | URL SQLAlchemy. Prefixo `postgres://` é convertido para `postgresql://` |
| `LOG_TO_STDOUT` | — | Se definido, log vai para stdout em vez de `logs/microblog.log` |
| `MAIL_SERVER` | — | SMTP |
| `MAIL_PORT` | `25` | Porta SMTP |
| `MAIL_USE_TLS` | desligado | TLS se a variável existir |
| `MAIL_USERNAME` / `MAIL_PASSWORD` | — | Credenciais SMTP |
| `MS_TRANSLATOR_KEY` | — | Azure Translator |
| `ELASTICSEARCH_URL` | — | Se vazio, busca fica desligada |
| `REDIS_URL` | `redis://` | Fila RQ (`microblog-tasks`) |

Constantes no código:

- `ADMINS`: destinatários de e-mail de falha (`your-email@example.com`)
- `LANGUAGES`: `['en', 'es']`
- `POSTS_PER_PAGE`: `25`

---

## Serviços opcionais

### Redis + RQ (exportação)

1. Suba o Redis.
2. Defina `REDIS_URL` se não for `redis://localhost:6379`.
3. Em outro terminal, com o venv ativo:

```powershell
rq worker microblog-tasks
```

4. No perfil, use a exportação de posts. O worker monta um JSON e envia por e-mail (SMTP precisa estar ok).

### Elasticsearch (busca)

Defina `ELASTICSEARCH_URL` (ex.: `http://localhost:9200`). Posts passam a ser indexados no commit. A busca na navbar só retorna resultados com o serviço no ar.

### E-mail (reset e exportação)

Exemplo mínimo com um SMTP local ou de desenvolvimento:

```
MAIL_SERVER=localhost
MAIL_PORT=8025
```

O pacote `aiosmtpd` está no `requirements.txt` e pode ser usado para um servidor de debug.

### Tradução de posts

Defina `MS_TRANSLATOR_KEY`. A região no cliente está fixa em `westus` (`app/translate.py`).

---

## Rotas da interface

Prefixo de autenticação: `/auth`.

| Método | Rota | Descrição |
| --- | --- | --- |
| GET/POST | `/auth/login` | Login |
| GET | `/auth/logout` | Logout |
| GET/POST | `/auth/register` | Cadastro |
| GET/POST | `/auth/reset_password_request` | Pedir reset |
| GET/POST | `/auth/reset_password/<token>` | Definir nova senha |
| GET/POST | `/`, `/index` | Home + novo post |
| GET | `/explore` | Todos os posts |
| GET | `/user/<username>` | Perfil |
| GET | `/user/<username>/popup` | Popup do usuário |
| GET/POST | `/edit_profile` | Editar perfil |
| POST | `/follow/<username>` | Seguir |
| POST | `/unfollow/<username>` | Deixar de seguir |
| GET | `/search` | Busca (`?q=`) |
| POST | `/translate` | JSON: traduzir texto |
| GET/POST | `/send_message/<recipient>` | Enviar mensagem |
| GET | `/messages` | Caixa de entrada |
| GET | `/export_posts` | Disparar exportação |
| GET | `/notifications` | JSON de notificações (`?since=`) |

A home, o Explore e a maior parte das páginas exigem login (`@login_required`).

---

## API REST

Base: `/api`. Respostas em JSON.

**Autenticação**

1. Obter token (HTTP Basic: username + senha):

```http
POST /api/tokens
```

Resposta: `{ "token": "..." }`.

2. Nas demais rotas autenticadas, envie `Authorization: Bearer <token>`.
3. Revogar:

```http
DELETE /api/tokens
```

| Método | Rota | Auth | Descrição |
| --- | --- | --- | --- |
| POST | `/api/users` | não | Criar usuário (`username`, `email`, `password`) |
| GET | `/api/users` | token | Listar (paginado: `page`, `per_page` ≤ 100) |
| GET | `/api/users/<id>` | token | Detalhe |
| PUT | `/api/users/<id>` | token (só o próprio) | Atualizar |
| GET | `/api/users/<id>/followers` | token | Seguidores |
| GET | `/api/users/<id>/following` | token | Seguindo |
| POST | `/api/tokens` | Basic | Emitir token (~1 h) |
| DELETE | `/api/tokens` | token | Invalidar token |

Coleções seguem o formato `to_collection_dict`: `items`, `_meta` (página, totais) e `_links` (`self`, `next`, `prev`). O recurso de usuário inclui `_links` para `self`, `followers`, `following` e `avatar`.

---

## Modelo de dados

| Entidade | Função |
| --- | --- |
| `User` | Conta, senha hash, perfil, token de API, relações sociais |
| `Post` | Corpo, timestamp, idioma, autor; indexável (`SearchableMixin`) |
| `followers` | Tabela de associação many-to-many (seguir) |
| `Message` | Mensagem privada (remetente, destinatário, corpo) |
| `Notification` | Nome + payload JSON por usuário |
| `Task` | Job RQ (`id` do job, nome, descrição, `complete`) |

Migrations em `migrations/versions/` cobrem usuários, posts, follow, campos de perfil, idioma do post, mensagens, notificações, tasks e tokens.

---

## CLI

```powershell
flask translate init <lang>    # novo idioma
flask translate update         # extrair e atualizar .po
flask translate compile        # gerar .mo
flask db upgrade               # aplicar migrations
flask shell                    # shell com db, User, Post, Message, Notification, Task
```

---

## Docker

O `Dockerfile` instala dependências, copia a app e usa `boot.sh`:

1. `flask db upgrade` (com retry)
2. `gunicorn` na porta **5000**

Build e execução típicos:

```bash
docker build -t microblog .
docker run -p 5000:5000 -e SECRET_KEY=troque -e DATABASE_URL=... microblog
```

O `Procfile` descreve dois processos estilo Heroku:

- `web`: migrate + compile de traduções + Gunicorn
- `worker`: `rq worker microblog-tasks`

---

## Referência

- Tutorial: [The Flask Mega-Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world)
- Licença: ver `LICENSE`
- Código original: Miguel Grinberg

Este repositório não inclui suíte de testes automatizados; a verificação principal é usar a interface (e a API, se quiser) após `flask run`.
