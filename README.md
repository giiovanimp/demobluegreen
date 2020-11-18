# Blue/Green Deployment

Este tutorial tem como objetivo a demonstração de uma atualização de aplicações utilizando o método Bluegreen em um Cluster Kuebernetes. Ao final será capaz de:
1. Gerar um build Docker de uma aplicação.
2. Executar um container Docker localmente.
3. Adicionar um container Docker ao Docker Hub.
4. Criar deployments Kubernetes de suas aplicações Docker.
5. Entender o funcionamento de uma atualização de aplicação através do Serviço Kubernetes utilizando Blue/Green.

Para o início do tutorial é necessário que os seguintes pré-requisitos:
- [Instalar Docker](https://docs.docker.com/docker-for-windows/install/)
- Ter acesso a um cluster Kubernetes.
- [Instalar kubetcl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) para ter acesso aos comandos Kubernetes no Cluster.

## Criar e publicar um container Docker
### Criando a Dockerfile

Na pasta raiz da aplicação a ser contenerizadas, adicione um novo arquivo chamado [Dockerfile](https://github.com/giiovanimp/demobluegreen/blob/main/DemoBlueGreen/Dockerfile). Este arquivo será responsável pela criação e configuração da imagem docker que será gerada. Dentro dela são definidos os comandos que serão executados pelo docker. Cada Dockerfile pode variar dependendo do tamanho, linguagem e recursos que a aplicação possui.

```bash
# Adiciona o SDK do .Net Core para posterior publicação e acessa o diretório contendo a aplicação.
FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build-env
WORKDIR /src

# Copia a solução e os projetos para que possa executar um restore de suas dependências.
COPY *.sln ./
COPY DemoBlueGreen/DemoBlueGreen.csproj DemoBlueGreen/
RUN dotnet restore

# Copia todos os arquivos para a pasta que será feita a montagem e publicação da aplicação.
COPY . .
WORKDIR "DemoBlueGreen"
#RUN mv appsettings.Development.json appsettings.json
RUN dotnet publish "DemoBlueGreen.csproj" -c Release -o /app

# Copia a runtime do .Net para a pasta que a dll da aplicação foi gerada.
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim
WORKDIR /app
COPY --from=build-env /app .
ENTRYPOINT ["dotnet", "DemoBlueGreen.dll"]

# Define a porta de acesso ao container.
EXPOSE 80
```

### Executando o container.

Execute o Docker Desktop e aguarde até o ícone indicar que o Docker está rodando.

![Docker Running](https://github.com/giiovanimp/demobluegreen/blob/main/Images/dockerrunning.png?raw=true)

Abra um Prompt de comando na pasta do projeto para que possam ser executados os comandos Docker.

Primeiramente vamos executar **"docker build"** indicando que criaremos uma tag **"-t"**, isto é, uma versão da aplicação, escreva o nome da imagem que será gerada acompanhada do nome da tag, nesse caso v1, **"giiovanimp/demobluegreen:v1"**, ao final indique a localização da Dockerfile, por estar no mesmo diretório coloque um **"."**.
 
O comando completo ficará assim:
```
docker build -t giiovanimp/demobluegreen:v1 .
```
Com isso o Docker irá executar todos os comandos listados na Dockerfile e exibirá na tela. Caso tudo esteja certo indicará que gerou a build e a tag com sucesso.

![Successfully Built](https://github.com/giiovanimp/demobluegreen/blob/main/Images/successfullybuilt.png?raw=true)

Com a imagem gerada execute o container no docker localmente utilizando o comando **"docker run"**, primeiramente indique o direcionamento de portas para acessar o container, nesse exemplo utilize a porta 8080 para referenciar a porta 80 do container, resultando em **"-p 8080:80"**. Logo após indique o container que será executado juntamente com sua tag **"giiovanimp/demobluegreen:v1"**. O Comando completo ficará assim:
```
docker run -p 8080:80 giiovanimp/demobluegreen:v1
```

![Container Running](https://github.com/giiovanimp/demobluegreen/blob/main/Images/containerrunning.png?raw=true)

Após a confirmação de execução, acesse localmente a aplicação indicando a porta configurada. Em um navegador acesse: http://localhost:8080/weatherforecast

No caso dessa aplicação, ela retornará a previsão do tempo dos próximos 5 dias.

![Localhost v1](https://github.com/giiovanimp/demobluegreen/blob/main/Images/localhostv1.png?raw=true)

Para parar a execução do container, na linha de comando, liste os containers em execução para ter acesso ao id do container.
```
docker container ls
```
Agora com o id execute o comando substituindo o id do container:
```
docker kill containerId
```
Em caso de sucesso o Docker retornará o ID do container deletado.

### Publicando a imagem no DockerHub

Para isso execute o comando **"docker login"**, caso não tenha logado ainda o Docker irá pedir as credenciais da conta DockerHub.
Após logado faça um **"push"** com a imagem gerada utilizando o comando:
```
docker push giiovanimp/demobluegreen:v1
```

## Publicando e acessando aplicação no Kubernetes
### Criando Namespace e Contexto

Para esta etapa, certifique que o **kubectl** está devidamente configurado e apontando para o Cluster. Para isso no Prompt de Comando execute:
```
kubectl cluster-info
```
Ele exibirá as URLs configuradas.

Assim como Dockerfile, o Kubernetes também precisa de alguns arquivos de configuração para serem executados. Inicialmente crie um namespace e um contexto para facilitar a separação da aplicação dos outros elementos padrões no Kubernetes. Para criar um Namespace, crie um arquivo [namespace.yaml](https://github.com/giiovanimp/demobluegreen/blob/main/Kubernetes/namespace.yaml). Nele é definido qual o tipo de elemento que está sendo criado e também seu respectivo nome.

```
apiVersion: v1
kind: Namespace
metadata:
  name: giiovanimp
```
Lembrando que para um arquivo YAML a indentação é primordial para a leitura correta do documento. Sendo definido 2 espaços para cada elemento filho.

Com o documento criado utilize o comando **"kubectl apply"** para aplicar as configurações descritas no arquivo. Para isso esteja com o Prompt de Comando na pasta em que os arquivos foram criados para que possa referenciar corretamente. Este comando será usado durante todas modificações e aplicações de YAML no Kubernetes.

O comando ficará assim:
```
kubectl apply -f namespace.yaml
```

Agora que já possui um Namespaces, configure o Contexto para esse Namespace. Para isso execute o comando abaixo colocando o nome do contexto desejado e apontando para o Namespace criado préviamente.
```
kubectl config set-context giiovanimp-demo --namespace=giiovanimp
```
Agora acesse o contexto.
```
kubectl config use-context giiovanimp-demo
```

### Gerando deploy da aplicação e do serviço

Com o contexto configurado pode-se executar alguns comandos básicos, como listar Pods ou Services. Para isso utilize o verbo **"get"** no comando **"kubectl"**. Exemplo:
```
kubectl get pods
```
ou
```
kubectl get services
```
Como ainda não foi publicado nenhum deployment, é normal não exibir nenhum Pod. Igualmente os Services onde tem-se apenas o serviço interno do Kubernetes.

![Get Pods e Services](https://github.com/giiovanimp/demobluegreen/blob/main/Images/getpodsservices.png?raw=true)

Para publicar a primeira versão da aplicação crie um arquivo chamado [deploymentv1.yaml](https://github.com/giiovanimp/demobluegreen/blob/main/Kubernetes/deploymentv1.yaml), nele serão feitas todas configurações necessárias para o Deployment.

Inicialmente defina qual tipo de elemento será aplicar, nesse caso, um Deployment, que será publicado no **"namespace: giiovanimp"** com o nome **"demo-blue-green-dep"**. Aqui também é definida a label, classificação e versão da app, sendo **"demo-bg"**, **"app"** na **"v1"**. Resultando em:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: giiovanimp
  name: demo-blue-green-dep
  labels:
    app: demo-bg
    tier: app
    version: v1
```
Essas informações serão utilizadas nas próximas linhas também, para identificar o deploy a ser feito.

```bash
spec: 
  selector: #Define o seletor da label criada anteriormente.
    matchLabels:
      app: demo-bg
      tier: app
      version: v1
  replicas: 1 #Define a quantidade de réplicas (pods) que serão criadas
  template: #Detalha o template usado para o deploy
    metadata:
      labels: #Sendo a mesma label criada acima
        app: demo-bg
        tier: app
        version: v1
    spec:
      containers: #Especificação do container que esse Pod possuirá
        - name: demobg
          image: giiovanimp/demobluegreen:v1 #Busca no DockerHub a imagem gerada anteriormente
          ports: #Expondo a porta que será usada.
            - name: http 
              containerPort: 80
              protocol: TCP
          resources: #Define os recursos que cada Pod irá exigir inicialmente. Podendo possuir limites também.
            requests:
              cpu: 10m #10 milicores equivalem a 10% de um Core, que possui 1000 milicores.
              memory: 100Mi #100 MebiBytes, equivalente a 104,858 Megabytes
          imagePullPolicy: IfNotPresent #A política de busca da imagem. Neste caso, caso não já não exista.
      restartPolicy: Always #Indica que o Pod reiniciará sempre que algo der errado.
  strategy: #Estratégia de deploy.
    type: RollingUpdate #Neste caso Rolling Update, onde atualiza réplica a replica.
```

Com o arquivo configurado aplique ele para o Kubernetes utilizando:
```
kubectl apply -f deploymentv1.yaml
```
Ao executar o comando para recuperar os pods, pode-se visualizar a aplicação na lista.

![Apply Deployment v1](https://github.com/giiovanimp/demobluegreen/blob/main/Images/applydeploymentv1.png?raw=true)

Agora com o Pod já publicado no cluster é preciso criar um serviço que o acesse, para isso crie um novo arquivo [service.yaml](https://github.com/giiovanimp/demobluegreen/blob/main/Kubernetes/service.yaml). Nele é definido um serviço externo que guiará para o Pod publicado.

```bash
apiVersion: v1
kind: Service #Utilizado o tipo Service
metadata:
  namespace: giiovanimp #Também estará no mesmo namespace
  name: demo-blue-green-svc
spec:
  type: LoadBalancer #Sendo do tipo balanceador de carga
  ports: #Direcionará para a porta 80 do Pod.
    - port: 80
  selector: #Define qual aplicação irá referenciar e sua versão
    app: demo-bg
    tier: app
    version: v1
```

Aplique o arquivo como feito nos anteriores.
```
kubectl apply -f service.yaml
```
Ao recuperar os serviços pode-se observar o que acabou de ser criado. Ele possui uma propriedade **"EXTERNAL-IP"**, esse IP que é utilizado para acessar a aplicação externamente.

![Apply Service](https://github.com/giiovanimp/demobluegreen/blob/main/Images/applyservice.png?raw=true)

Em um navegador acesse http://20.62.216.3/weatherforecast, substituindo o IP gerado no serviço.

![Pod v1](https://github.com/giiovanimp/demobluegreen/blob/main/Images/podv1.png?raw=true)

É válido lembrar que todas essas configurações podem ser feitas em um único arquivo YAML, separando as definições com ***"---"***, assim como exemplificado no arquivo [k8sdeploy.yaml](https://github.com/giiovanimp/demobluegreen/blob/main/Kubernetes/k8sdeploy.yaml).

## Aplicando atualização Blue/Green
### Gerando ambiente Green

O Ambiente Green é a atualização da aplicação que atualmente está publicada (Blue). É necessário gerar uma nova imagem Docker, uma nova versão e publica-la no Docker Hub. Portanto repita os passos do primeiro tópico mudando a versão da tag gerada.

![Docker images](https://github.com/giiovanimp/demobluegreen/blob/main/Images/dockerimagesv2.png?raw=true)

Crie um novo arquivo [deploymentv2.yaml](https://github.com/giiovanimp/demobluegreen/blob/main/Kubernetes/dockerimagesv2.yaml) que será praticamente idêntico ao v1.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: giiovanimp
  name: demo-blue-green-v2 #Altere o nome do deployment para que seja criado um deployment da nova versão
  labels:
    app: demo-bg
    tier: app
    version: v2 #Altere a versão da app e todos lugares que a referenciam.
...
        spec:
          containers:
            - name: demobg
              image: giiovanimp/demobluegreen:v2 #Altere a tag buscada no DockerHub
```

Aplique esse novo deployment como feito na primeira versão e observe que agora a lista de Pods contém os dois deployments. Sendo o primeiro o Blue e o segundo o Green.

![Apply Deployment v2](https://github.com/giiovanimp/demobluegreen/blob/main/Images/applydeploymentv2.png?raw=true)

Para a atualização Blue/Green altere o serviço que foi aplicado anteriormente, porém apontando para a nova versão no seletor.

```bash
...
  selector:
    app: demo-bg
    tier: app
    version: v2
```

Com isso o Kubernetes irá atualizar o serviço e passará a direcionar as novas requisições para o novo ambiente. Note que isso poderá levar alguns minutos e requisições que começaram no ambiente antigo ficarão por lá até serem finalizadas, portanto é normal por alguns momentos obter respostas diferentes em diferentes dispositivos.

Com a atualização a aplicação retorna somente um dia.

![Pod v2](https://github.com/giiovanimp/demobluegreen/blob/main/Images/podv2.png?raw=true)

Caso encontre algum problema na nova versão o serviço poderá ser alterado novamente retornando as chamadas para a primeira versão até que as correções necessárias sejam feitas. Caso tudo ocorra como planejado apague o deployment da primeira versão para que não fique consumindo recursos no Cluster sem necessidade. Para isso execute:

```
kubectl delete deployment demo-blue-green-dep
```

# Troubleshooting
### Contexto não tem acesso ao Cluster

Caso ao alterar para o novo contexto o kubectl exiba uma mensagem de "connection refused", verifique as configurações do arquivo **{user}/.kube/config** e certifique que o contexto criado está com o mesmo usuário e cluster configurados que os demais contextos.
