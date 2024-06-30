# Sobre a atividade 

Esta atividade é parte do meu roteiro pessoal na formação em Engenharia de Software. Trata-se de uma experiência simples
em orquestração de containers utilizando Kubernetes. Não possui caráter profissional e seu objetivo é unicamente o de ser
parte de um acervo pessoal de códigos e experiências. 

### Pré-requisitos

Utilizarei o Minikube como implementação do Kubernetes e o *kubectl* como ferramenta de linha de comando. Os links a 
seguir orientam sobre como realizar as instalações necessárias:

- [Python](https://www.python.org/)
- [PostgreSQL](https://www.postgresql.org/download/)
- [Docker](https://docs.docker.com/engine/install/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

### Aplicação

A aplicação será um sistema de registro de pessoas. Não atentarei para a realização de um *CRUD* completo, pois, o 
objetivo é trabalhar uma estrutura de microserviços. Vou somente realizar a exibição das pessoas salvas no 
banco de dados e o registro de novas pessoas.

Utilizarei o *Django framework* para [construir a aplicação](./readme/django.md) e o PostgreSQL como SGBD.  

### Imagem

- Criando arquivo *Dockerfile*:

```dockerfile
FROM python:3

WORKDIR /usr/src/app

COPY requirements.txt ./

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENTRYPOINT gunicorn --bind 0.0.0.0:8080 siteSetup.wsgi

EXPOSE 8080
```

- Criando arquivo *.dockerignore*:

```.dockerignore
venv
.vscode
.idea
.gitignore
db.sqlite
staticfiles
```

- Criando a imagem Docker da aplicação:
> ``docker build -t app_django:v1 .``

Para continuar é preciso ter um perfil no Docker Hub, pois, será necessário realizar a publicação da imagem em um 
repositório público para que o kubernetes possa utilizá-la.

- Realizando o login no Docker Hub:
> ``docker login``

- Criando uma imagem para o repositório:
> ``docker tag app_django:v1 usuario/app_django:v1``

- Empurrando a imagem para o Docker Hub:
> ``docker push usuario/app_django:v1``

### Kubernetes

- #### Preparando o ambiente do nosso Cluster:
> ``minikube delete``

- #### Iniciando o nosso Cluster:
> ``minikube start``

Trabalharei com dois serviços, um para o banco de dados e outro para a aplicação, começarei com a edição dos arquivos 
necessários para o banco de dados. Os comandos devem ser executados em um terminal aberto no diretório onde os arquivos
forem salvos.

### Serviço de banco de dados

- #### 1. Definindo o arquivo *postgres-configmap.yaml*:
ConfigMaps são objetos da API do Kubernetes que armazenam dados de 
configuração que utilizarei para armazenar detalhes da conexão com banco de dados.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-secret
  labels:
    app: postgres
data:
  POSTGRES_DB: "laboratorio"
  POSTGRES_USER: "estudante"
  POSTGRES_PASSWORD: "212223"
```
Executando o arquivo:
> ``kubectl apply -f postgres-configmap.yaml``

- #### 2. Definindo o arquivo *postgres-pv.yaml*:
Para persistência de dados, deve-se utilizar os recursos do Kubernetes Persistent Volume (PV) e Persistent Volume Claim (PVC).
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-volume
  labels:
    type: local
    app: postgres
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /data/postgresql
```

Executando o arquivo:
> ``kubectl apply -f postgres-pv.yaml``


- #### 3. Definindo o arquivo *postgres-pvc.yaml*
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-volume-claim
  labels:
    app: postgres
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

Executando o arquivo:
> ``kubectl apply -f postgres-pvc.yaml``


- #### 4. Definindo o arquivo *postgres-deployment.yaml*:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: 'postgres:14'
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
          envFrom:
            - configMapRef:
                name: postgres-secret
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: pgdata
      volumes:
        - name: pgdata
          persistentVolumeClaim:
            claimName: postgres-volume-claim
```

Executando o arquivo:
> ``kubectl apply -f postgres-deployment.yaml``

- #### 5. Definindo o arquivo *postgres-service.yaml*:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: ClusterIP
  ports:
    - port: 5432
  selector:
    app: postgres
```

Executando o arquivo:
> ``kubectl apply -f postgres-servide.yaml``

### Serviço da aplicação Django

