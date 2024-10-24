# Projeto: Configuração e Documentação

## Pré-requisitos

Antes de iniciar o projeto, certifique-se de ter os seguintes softwares instalados em sua máquina:

- **Docker**: [Instruções de instalação](https://docs.docker.com/engine/install/)
- **Docker Compose**: [Instruções de instalação](https://docs.docker.com/compose/install/)
- **Git**: [Guia de instalação](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

---

## Clonando o Repositório

Se o repositório ainda não foi clonado no seu ambiente local, utilize o comando abaixo:

```bash
git clone https://github.com/josef-axer/docker_guess_game.git
```
Navegue até a pasta do projeto clonado:
```bash
cd projeto-docker
```
```bash
docker-compose up
```
A aplicação estará acessível via ```http://localhost``` na porta 80.

## Nota importante:
A porta 80 deve estar disponível em seu host. Se já houver outro serviço rodando nessa porta (por exemplo, Apache ou Nginx), será necessário modificar o arquivo docker-compose.yml para utilizar outra porta, alterando o mapeamento de portas.

# Estrutura do Projeto
* **Arquivo Docker Compose:** Localizado na raiz do projeto (docker-compose.yml).
* **Dockerfile do Backend:** Localizado na raiz do projeto (Dockerfile).
* **Dockerfile do Frontend:** Localizado na pasta frontend/ (frontend/Dockerfile).
* **Configuração do NGINX:** Localizado na raiz do projeto (nginx.conf).

# Configurações Detalhadas
1. **Arquivo** ```docker-compose.yml```
O arquivo ```docker-compose.yml``` define os serviços necessários para o funcionamento da aplicação.
1.1. Frontend
O **Dockerfile** do frontend está localizado na pasta frontend. O build é realizado a partir desse diretório:
```yaml
frontend:
  build:
    context: ./frontend
    dockerfile: Dockerfile
  restart: always
  environment:
    - REACT_APP_API_URL=http://localhost
  depends_on:
    - backend
```
1.2. Backend
O backend utiliza um Dockerfile na raiz do projeto. As variáveis de ambiente foram movidas para o arquivo docker-compose.yml, incluindo um healthcheck para garantir que o serviço esteja pronto antes de iniciar o balanceador de carga:
```yaml
backend:
  build: .
  restart: always
  environment:
    - APP_ENV=production
    - DB_HOST=postgres
    - DB_USER=user
    - DB_PASSWORD=password
    - DB_NAME=appdb
  depends_on:
    - postgres
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
    interval: 15s
    timeout: 5s
    retries: 3
```
O ```Dockerfile``` do backend inclui a instalação do ```curl``` para suportar o healthcheck:
```yaml
FROM python:3.9
WORKDIR /app
RUN apt-get update && apt-get install -y curl
COPY . .
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python", "app.py"]
```
1.3. Banco de dados (PostgreSQL)
O banco de dados PostgreSQL é configurado com as credenciais necessárias e tem seus dados persistidos em um volume:
```yaml
postgres:
  image: postgres:latest
  restart: always
  environment:
    POSTGRES_USER=user
    POSTGRES_PASSWORD=password
    POSTGRES_DB=appdb
  volumes:
    - db-data:/var/lib/postgresql/data
```
1.4. NGINX como Load Balancer
O NGINX foi configurado para fazer o balanceamento de carga entre o frontend e o backend:
```
nginx:
  image: nginx:latest
  restart: always
  ports:
    - "80:80"
  volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf
  depends_on:
    - frontend
    - backend
```
1.5. Volume para Persistência do Banco de Dados
Um volume nomeado ```db-data``` foi criado para armazenar os dados do PostgresSQL:
```yaml
volumes:
  db-data:
```

2. ### Configuração do NGINX
O arquivo ```nginx.conf``` define as regras para roteamento, com o frontend sendo servido na rota principal ```(/)```, e o backend em rotas específicas:
```yaml
events { }

http {
    server {
        listen 80;

        location / {
            proxy_pass http://frontend:3000;
            proxy_set_header Host $host;
        }

        location ~ ^/(api|user|data) {
            proxy_pass http://backend:5000;
            proxy_set_header Host $host;
        }
    }
}
```
3. ### Intruções de Atualização
3.1.  Atualizando o NGINX:
Se precisar modificar as rotas ou a configuração do balanceador de carga, edite o arquivo ```nginx.conf```. Para aplicar as alterações, derrube e reinicie os contêineres:
```bash
docker-compose down
docker-compose up
```
3.2. Atualizando o Frontend:
Qualquer modificação no frontend deve ser refletida no Dockerfile ou no arquivo ```docker-compose.yml``` localizado na pasta frontend. Após as alterações:
```bash
docker-compose down
docker-compose up
```
