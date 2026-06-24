# Jogo de Adivinhação com Docker Compose

Este repositório empacota o projeto demo [guess_game](https://github.com/fams/guess_game) em uma arquitetura Docker Compose. O código da aplicação foi preservado: esta entrega adiciona somente a infraestrutura de containers necessária para executar o backend Flask, o frontend React, o proxy NGINX e o banco PostgreSQL.

## Arquitetura

O sistema é composto pelos seguintes serviços:

| Serviço | Tecnologia | Responsabilidade |
| --- | --- | --- |
| `frontend` | NGINX + React | Compila e serve os arquivos estáticos React. Também é o único serviço exposto ao host e atua como proxy reverso para a API. |
| `backend-1`, `backend-2`, `backend-3` | Python 3.12 + Flask + Gunicorn | Executam o backend do jogo em três instâncias. |
| `postgres` | PostgreSQL 16 | Armazena os jogos e tentativas de forma persistente. |

O NGINX recebe as requisições do navegador na porta `8080`. Arquivos do React são entregues diretamente pelo NGINX; requisições iniciadas em `/api/` são distribuídas, em *round-robin*, entre as três instâncias do backend.

```text
Navegador
    |
    | http://localhost:8080
    v
NGINX / frontend
    |-- serve React
    `-- /api/* --> backend-1 ─┐
                   backend-2 ─┼--> PostgreSQL (volume postgres_data)
                   backend-3 ─┘
```

Todos os serviços pertencem à rede interna `guess-net`. Apenas o NGINX publica uma porta para a máquina host; o banco e os backends não ficam expostos diretamente.

## Pré-requisitos

- Docker Engine com Docker Compose v2 (comando `docker compose`)
- Porta `8080` livre na máquina host

Não é necessário instalar Python, Node.js, PostgreSQL ou NGINX localmente. As versões usadas pelos containers são Python 3.12, Node 18, NGINX 1.27 e PostgreSQL 16.

## Como executar

1. Clone o repositório e acesse a pasta do projeto.

   ```bash
   git clone https://github.com/BryanWille/puc_guess_game.git
   cd puc_guess_game
   ```

2. Construa as imagens locais e inicie os serviços.

   ```bash
   docker compose up --build -d
   ```

3. Confira se todos os containers estão em execução.

   ```bash
   docker compose ps
   ```

4. Abra o jogo em **http://localhost:8080**.

Para acompanhar a inicialização ou diagnosticar problemas, use:

```bash
docker compose logs -f
```

Para encerrar os containers sem apagar os jogos salvos:

```bash
docker compose down
```

> Não use `docker compose down -v` se quiser manter os dados do jogo. A opção `-v` remove também o volume persistente do PostgreSQL.

## Persistência de dados

O serviço `postgres` monta o volume nomeado `postgres_data` em `/var/lib/postgresql/data`. Assim, os dados permanecem disponíveis mesmo após parar, remover ou recriar os containers.

Para listar o volume:

```bash
docker volume ls
```

Para apagar intencionalmente todos os dados do banco:

```bash
docker compose down -v
```

## Balanceamento e resiliência

A configuração do NGINX está em [`frontend/default.conf`](frontend/default.conf). O bloco `upstream backend_pool` contém `backend-1`, `backend-2` e `backend-3`; o NGINX distribui as chamadas da API entre elas.

O bloco `location /api/` repassa, por exemplo, `/api/create` para `/create` em uma das instâncias Flask. Ele também usa `proxy_next_upstream` para tentar outra instância em caso de erro ou indisponibilidade temporária.

Todos os serviços usam a política `restart: always` no `docker-compose.yml`. Portanto, se um processo em um container falhar, o Docker tentará reiniciá-lo automaticamente.

## Atualização dos componentes

As versões estão declaradas de forma explícita para tornar a manutenção previsível:

| Componente | Onde alterar | Versão atual |
| --- | --- | --- |
| Backend | `Dockerfile.backend` | `python:3.12-slim` |
| Frontend (build) | `frontend/Dockerfile` | `node:18-alpine` |
| Frontend/proxy | `frontend/Dockerfile` | `nginx:1.27-alpine` |
| Banco de dados | `docker-compose.yml` | `postgres:16-alpine` |

Para atualizar uma imagem base, substitua apenas a sua tag no arquivo correspondente e recrie os serviços:

```bash
docker compose up --build -d
```

Exemplo para atualizar o PostgreSQL: altere `postgres:16-alpine` para a versão desejada em `docker-compose.yml` e execute o comando anterior. Antes de atualizar versões maiores do PostgreSQL, consulte as instruções oficiais de migração do banco; volumes de dados não devem ser reutilizados entre versões incompatíveis sem migração.

As imagens locais da aplicação também possuem tags no Compose: `guess-game-backend:1.0.0` e `guess-game-frontend:1.0.0`. Ao preparar uma nova versão da aplicação, atualize a tag correspondente e execute `docker compose up --build -d`, mantendo o processo de atualização isolado por componente.

## Arquivos da entrega

- `docker-compose.yml`: define os serviços, a rede, o volume persistente, a porta pública e as políticas de reinício.
- `Dockerfile.backend`: cria a imagem Python/Flask executada com Gunicorn.
- `frontend/Dockerfile`: compila o React com Node 18 e gera a imagem NGINX que serve o frontend.
- `frontend/default.conf`: configura o NGINX como servidor de arquivos estáticos, proxy reverso e balanceador de carga.

## Observação de segurança

As credenciais do PostgreSQL presentes no Compose são apropriadas somente para demonstração acadêmica e desenvolvimento local. Em um ambiente real, elas devem ser fornecidas por variáveis de ambiente ou um mecanismo de gerenciamento de segredos.
