
# ELT Pipeline - Marca de Carros - Vendas e análises

- Este projeto mostra uma pipeline ELT (Extract - Load - Transform) de um banco de dados PostgreSQL (on premises) referente à vendas de uma marca de carros, este banco de dados "é vivo" e recebe valores transacionais assim que uma venda é realizada, o que pode ser à qualquer momento. 
- O objetivo deste projeto foi integrar ferramentas dos três estágios da ELT, e então responder as questões de negócio abaixo:

## Questões de negócio (geração de valor):

### 1. Realizar a análise de vendas por concessionária;
### 2. Quais foram as vendas por modelo de veículo;
### 3. Vendas por vendedor;
### 4. Vendasss em análise temporal (por mês ou por ano por exemplo);.


## Pré requisitos:
- Docker (caso queira ter um database baseado em postgres personalizado);
- PgAdmin;
- Conta AWS (cuidado: pode ocasionar gastos dependendo da EC2 escolhida);
- Conta DBT;
- Conta no Snowflake (possui $400 doláres de free tier);
- Conta na google (para utilizar o Looker como ferramenta de BI).

## Banco de dados Postgres exemplo para conexão (mudar de acordo com o que você tiver):

host: 159.223.187.110
dbname: novadrive
user: etlreadonly
password: novadrive376A@

Com essas informações você já consegue conectar ao Postgres de exemplo com a ferramenta PgAdmin4 e realizar querys para consultar as tabelas de origem.

# Orquestrador: Apache Airflow

Existem muitas formas de utilizar o apache airflow na nuvem para ingestão ou migração de dados de um ambiente local (on premises) para a nuvem, das quais podemos citar as mais comuns:
- Airflow gerenciado pela nuvem (AWS - MWAA / GCP Composer): esta é a forma que vai te trazer menos dor de cabeça, é escalável e gerenciada pelas cloud providers, a desvantagem é o custo que chega a ser bem alto;
- Airflow com K8s: No kubenetes tem-se o melhor do open source e gerenciamento sob seu comando, entretanto é necessário uma curva de aprendizado bem elevada para setar esse ambiente, sem contar a manutenção;
- Utilizar o Astro CLI: esta talvez seja a forma mais simples de estudar airflow, por ser local, e bem abstraída. Mas como foi dito é algo local e não na nuvem. Entretanto, você pode realizar todo o desenvolvimento local em seguida iniciar uma Virtual machine na cloud de sua preferência (EC2 por exemplo) e realizar o pull da imagem docker local para o repositório de imagens docker e dentro da sua VM realizar um Push dessa imagem.
- Entretanto a forma que utilizaremos aqui, apesar de possuir comandos shell um tanto complexos, foi a que uniu o melhor dos dois mundos (custo e escalabilidade) para este estudo: instanciar uma EC2 e dentro dessa máquina virtual instalar o docker, docker compose e persistir o volume o passo a passo será explicado a seguir.

## provisionar o airflow em uma AWS EC2.
- Assumindo que você tenha uma conta na AWS, no painel inicial procure por EC2, escolha pela opção "Ubuntu", para o estudo de caso utilizei a "t2.large" e os gastos estimados ficaram em 27 cents de dolar.
- Improtante, é necessário criar uma "key pair" no momento de criação da instância pois essa chave vai ser a nossa conexão via SSH. Marque a opção "Allow SSH traffic from" aí fica a seu critério escolher somente o seu IP ou "anywhere", por segurança acosnelho escolher apenas o seu IP para conexão, mas o anywhere "0.0.0.0/0" também funcionará. Por fim, clique em instanciar e aguarde até que o status esteja "running".

- Uma vez que a instancia estiver disponível você consegue clicar nela, na aba "connect" clicar em "SSH" e copiar o comando que vai ser no formato a seguir:

`ssh -i "airflow.pem" ubuntu@ec2-34-201-61-202.compute-1.amazonaws.com` 
Cole esse comando em um terminal, por exemplo git-bash. Entretanto, você tem que ter a chave "key-pair" no mesmo diretório para que a conexão funcione corretamente.
- Nota: Ao pausar a isntancia e liga-la novamente o comando de coneção vai mudar, pois o nome `@ec2-34-201-61-202` geralmente muda também.

- Uma vez conectado, digite os comandos abaixo seguindo a ordem. Os comandos basicamente realizam a instalação do docker, persistem volume, e fazem o pull da imagem docker-compose oficial do airflow para a sua instalação dentro da EC2.

### 1. Atualizar a lista de pacotes do APT:
`sudo apt-get update`;
### 2. Instalar pacores necessários para adicionar um novo repositório via HTTPS:
`sudo apt-get install ca-certificates curl gnupg lsb-release`
### 3. Criar diretório para armazenar as chaves de repositórios: 
`sudo mkdir -m 0755 -p /etc/apt/keyrings`
### 4. Adicionar a chave GPG do repositório do Docker:
`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg`
### 5. dicionar o repositório do Docker às fontes do APT:
`echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null `
### 6. Atualiza a lista de pacotes após adicionar o novo repositório do Docker:
`sudo apt-get update`
### 7. Instalar o Docker e componentes:
`sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
### 8. Baixar o arquivo docker-compose.yaml do Airflow:
`curl -LfO 'https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml'`
### 9. Criar diretórios para DAGs, logs e plugins:
`mkdir -p ./dags ./logs ./plugins`
### 10.  Criar um arquivo .env com o UID do usuário, usado pelo docker para permissões:
`echo -e "AIRFLOW_UID=$(id -u)" > .env`
### 11. inicia o airflow
`sudo  docker compose up airflow-init`
### 12 -subir o Airflow em modo desacoplado:
`sudo docker compose up -d`

- Aguardar os containers entrarem no status "Healthy" você pode realizar essa chegagem via terminal com o comando:
`sudo docker ps`

- Assim que todos os containers estiverem com o status Healthy, você pode acessar o airflow copiando o nome da sua EC2 e adicionando `:8080`em um navegador de sua preferência:
`http://ec2-35-175-126-189.compute-1.amazonaws.com:8080/`

- O proximo passo não é mandatório, mas deixa o seu ambiente no airflow mais "clean", no caso vamos marcar a opção de FALSE para que ele não suba os exemplos nativos, editandop o docker-compose.yaml. Digite o comando abaixo, procure por "AIRFLOW__CORE__LOAD_EXAMPLES='true'" e troque por false, utilizando o editor nano:
`nano /home/ubuntu/docker-compose.yaml`

- reinicie o seu airflow:
```
#reiniciar
sudo docker compose stop
sudo docker compose up -d
sudo docker ps

```





