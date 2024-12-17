# **Documentação do Projeto**

## **Visão Geral**

Este projeto implementa uma aplicação escalável utilizando Docker,Docker Compose em ambiente Linux(Ubuntu Server)
Projetado para oferecer alta disponibilidade, facilidade de manutenção e balanceamento de carga entre serviços.

No decorrer das configuraçoes utilizamos alguns servicoes tais como:

- **PostgreSQL**
- **Backend**
- **Frontend**
- **NGINX**
- **Linux Server e Samba**

## **Decisões de Design**

### **1. Serviços Utilizados**

- **PostgreSQL**: Um dos mais utilizados banco relacionais gratuito suprindo o objetivo da entrega
- **Backend**: Utiliza o Flask para disponibilizar endpoints REST, que são consumidos pelo frontend.
- **Frontend**: Aplicação estática construída e servida via NGINX.
- **NGINX**: Configurado como proxy reverso e balanceador de carga, distribuindo o tráfego uniformemente entre os serviços backend e frontend.
- **Linux Samba Server**: Utilizado para compartilhar arquivos facilitando a manipulção em ambiente local Microosft.

### **2. Volumes**

- Cada instância do banco de dados utiliza um volume persistente para garantir que os dados não sejam perdidos após a remoção dos containers.
- Volumes criados:
  - `db_data1`
  - `db_data2`
  - `db_data3`

### **3. Redes**

- Todos os serviços são configurados na mesma rede Docker (`app-network`) para permitir a comunicação entre eles

### **4. Balanceamento de Carga**

- O **NGINX** utiliza a diretiva `least_conn` para distribuir o tráfego:
- **Backend**: Balanceia as requisições API entre 3 réplicas do serviço backend.
- **Frontend**: Balanceia as requisições de acesso ao frontend entre 3 réplicas.

Essa abordagem garante escalabilidade horizontal e melhor uso dos recursos disponíveis.

## **Estrutura do Projeto**

A estrutura do projeto segue uma organização simples e clara:

```plaintext
jogo4/
├── backend/
│   ├── Dockerfile          # Dockerfile para construção do backend
│   ├── run.py              # Código principal do backend (Flask)
│   └── requirements.txt    # Dependências do backend
├── frontend/
│   ├── Dockerfile          # Dockerfile para construção do frontend
│   └── dist/               # Arquivos estáticos do frontend
├── nginx/
│   └── nginx.conf          # Configuração do NGINX
├── samba/
│   └── smb.conf            # Configuração do Linux Samba Server
├── docker-compose.yml      # Arquivo principal do Docker Compose
└── README.md               # Documentação do projeto
```

## **Instruções de Instalação e Execução**

### **1. Pré-requisitos**

- **Docker**
- **Docker Compose**:
- **Servidor Linux com Samba**

### **2. Clonar o Repositório**

Clone o repositório para a máquina local e execurção

```bash
git clone https://gitlab.com/levysaraiva/game.git
cd jogo4
```

Verificação dos Conteiners

```bash
cd jogo4
docker ps -a
```

### **3. Configurar o Samba Server para editar os arquivos e configuração**

```bash
1. sudo apt update
2. sudo apt install samba
3. sudo nano /etc/samba/smb.conf
4. [home]
    path = /home
    valid users = devops
    read only = no
    browseable = yes
    writable = yes
    create mask = 0775
    directory mask = 0775
    force user = devops
5. sudo smbpasswd -a devops
6. sudo systemctl restart smbd
7. sudo chmod 755 /home
8. sudo chown -R :sambashare /home
9. sudo chmod -R 775 /home
10. sudo ufw allow samba
11. sudo systemctl restart smbd nmbd
```

### **4. Construir e Iniciar os Serviços**

Execute o seguinte comando para construir as imagens e iniciar os serviços em background:

```bash
docker-compose up --build -d
```

### **5. Verificar os Containers**

Confirme que os containers estão em execução:

```bash
docker-compose ps -a
```

### **6. Acessar os Serviços**

- **Frontend**: Acesse [http://localhost]
- **Backend API**: Acesse [http://localhost/api/]



Para atualizar a imagem do backend:

1. Modifique a versão da imagem no `docker-compose.yml`:
   ```yaml
   backend:
     image: meu-registro/backend:1.1.0
   ```

2. Execute o comando abaixo para atualizar apenas o serviço backend:
   ```bash
   docker-compose up -d backend
   ```

### **2. Atualizar o Frontend**

Para atualizar a imagem do frontend:

1. Modifique a versão da imagem no `docker-compose.yml`:
   ```yaml
   frontend:
     image: meu-registro/frontend:1.2.0
   ```

2. Reinicie o serviço frontend:
   ```bash
   docker-compose up -d frontend
   ```

### **3. Atualizar o Banco de Dados**

Caso precise atualizar a imagem do PostgreSQL:

1. Substitua a versão no `docker-compose.yml`:
   ```yaml
   db1:
     image: postgres:14.7
   ```

2. Reinicie o serviço:
   ```bash
   docker-compose up -d db1
   ```
## **Considerações Finais**

Este projeto foi estruturado para ser escalável, resiliente e fácil de atualizar. Com o uso de múltiplas réplicas e balanceamento de carga pelo NGINX, ele consegue lidar com um alto volume de tráfego, mantendo a aplicação estável.
