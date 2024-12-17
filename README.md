# Projeto: Sistema de Balanceamento de Carga com Backend, Frontend e NGINX

## Estrutura do Repositório

Este projeto foi desenvolvido para orquestrar um sistema de **backend** em Flask, um **frontend** em React e um **NGINX** configurado como proxy reverso e balanceador de carga. Todos os serviços são orquestrados utilizando **Docker Compose**.

### Estrutura de Arquivos:
- **Dockerfile (Backend)**: Configuração do container Python/Flask.
- **Dockerfile (Frontend)**: Configuração do container React com build via NGINX.
- **docker-compose.yml**: Orquestra todos os serviços (backend, frontend, banco de dados, e NGINX).
- **nginx.conf**: Configuração do NGINX para proxy reverso e balanceamento de carga.

---

## 1. Pré-requisitos
Certifique-se de ter os seguintes requisitos instalados:

- **Docker**: [Instalação do Docker](https://docs.docker.com/get-docker/)
- **Docker Compose**: [Instalação do Docker Compose](https://docs.docker.com/compose/install/)
- Git: [Instalação do Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

---

## 2. Clonando o Projeto
Clone o repositório em sua máquina:
```bash
git clone <URL_DO_REPOSITORIO>
cd <NOME_DO_REPOSITORIO>
```

---

## 3. Configuração do Backend (Flask)
O **backend** utiliza Python 3.9 e Flask. O arquivo `Dockerfile` foi configurado para copiar e instalar as dependências.

### Dockerfile - Backend:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

# Copia e instala as dependências
COPY requirements.txt .
RUN pip install --no-cache-dir --trusted-host pypi.org --trusted-host pypi.python.org --trusted-host files.pythonhosted.org -r requirements.txt

# Copia o restante do código
COPY . .

# Variáveis de ambiente
ENV FLASK_APP=/app/run.py \
    FLASK_RUN_HOST=0.0.0.0 \
    FLASK_RUN_PORT=5000

# Healthcheck
HEALTHCHECK CMD curl --fail http://localhost:5000/health || exit 1

# Expõe a porta do Flask
EXPOSE 5000

CMD ["flask", "run"]
```

---

## 4. Configuração do Frontend (React)
O **frontend** foi configurado para ser compilado e servido pelo NGINX.

### Dockerfile - Frontend:
```dockerfile
FROM node:18.17.0 AS build

ARG REACT_APP_BACKEND_URL
ENV REACT_APP_BACKEND_URL=${REACT_APP_BACKEND_URL:-http://localhost/api}

WORKDIR /app

COPY package.json package-lock.json ./
RUN yarn config set strict-ssl false && CYPRESS_INSTALL_BINARY=0 yarn install

COPY . .
RUN yarn build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 5. Configuração do NGINX
O NGINX foi configurado como **proxy reverso** e **balanceador de carga** entre múltiplas réplicas do backend.

### nginx.conf:
```nginx
events {}

http {
    include /etc/nginx/mime.types;
    sendfile on;

    upstream backend {
        least_conn;
        server backend:5000;
        server backend:5000;
        server backend:5000;
    }

    server {
        listen 80;

        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri /index.html;
        }

        location /api/ {
            rewrite ^/api/(.*)$ /$1 break;
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        error_page 502 503 504 = /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
}
```

---

## 6. Orquestração com Docker Compose
O `docker-compose.yml` organiza os serviços e define réplicas para o backend e frontend.

### docker-compose.yml:
```yaml
version: '3.8'

services:
  db1:
    image: postgres:14.6
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secretpass
      POSTGRES_DB: guess_game
    volumes:
      - db_data1:/var/lib/postgresql/data
    networks:
      - app-network
    restart: always

  backend:
    build:
      context: .
      dockerfile: Dockerfile
    deploy:
      replicas: 3
    environment:
      FLASK_APP: run.py
      FLASK_DB_USER: postgres
      FLASK_DB_PASSWORD: secretpass
      FLASK_DB_NAME: guess_game
      FLASK_DB_HOST: db1
    networks:
      - app-network
    depends_on:
      - db1
    restart: always

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    deploy:
      replicas: 3
    networks:
      - app-network
    restart: always

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    networks:
      - app-network
    depends_on:
      - backend
      - frontend
    restart: always

networks:
  app-network:

volumes:
  db_data1:
    name: persistent_postgres_data1
```

---

## 7. Rodando o Projeto
### Construir e iniciar os serviços:
Execute o seguinte comando na raiz do projeto:
```bash
docker-compose up --build
```

### Acesso à aplicação:
- Frontend: `http://<IP_DO_SERVIDOR>`
- Backend: `http://<IP_DO_SERVIDOR>/api`

---

## 8. Decisões de Design
1. **NGINX** como proxy reverso e balanceador de carga.
2. **Docker Compose** para orquestração e escalabilidade de serviços.
3. **Banco de dados PostgreSQL** configurado com persistência de dados via volumes.
4. **Deploy de múltiplas réplicas** (3 réplicas para backend e frontend) para garantir resiliência e balanceamento.

---

## 9. Atualização dos Componentes
Para atualizar qualquer componente (backend, frontend, ou banco de dados):
1. Altere a versão da imagem no `Dockerfile` ou `docker-compose.yml`.
2. Execute novamente:
   ```bash
   docker-compose up --build
   ```

---

## 10. Observações
- Todos os containers se comunicam corretamente utilizando a **rede app-network**.
- Em caso de falha de uma réplica, as outras continuam funcionando devido ao balanceamento de carga no NGINX.

---

**FIM**