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

## Primeiro passo: Criar e publicar um container Docker
### Criando a Dockerfile

Na pasta raiz da aplicação a ser contenerizadas, adicione um novo arquivo chamado [Dockerfile](https://github.com/giiovanimp/demobluegreen/blob/main/DemoBlueGreen/Dockerfile). Este arquivo será responsável pela criação e configuração da imagem docker que será gerada. Dentro dela são definidos os comandos que serão executados pelo docker. Cada Dockerfile pode variar dependendo do tamanho, linguagem e recursos que a aplicação possui.

Neste exemplo temos:

Primeiramente a definição da versão .Net que será utilizada para Build da aplicação.
```
FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build-env
```
Logo em seguida acessamos a pasta principal para poder iniciar a cópia dos arquivos para dentro da imagem.
```
WORKDIR /src
```
Neste passo é definido a solução e os projetos a serem copiados para a imagem. E a partir disso feito um restore para restaurar as dependências e ferramentas dos projetos.
```
COPY *.sln ./
COPY DemoBlueGreen/DemoBlueGreen.csproj DemoBlueGreen/
RUN dotnet restore
```
A partir disso, acessamos a pasta do projeto e é feito a cópia do arquivo de configuração, para posteriormente gerar um publicável da aplicação.
```
COPY . .
WORKDIR "DemoBlueGreen"
#RUN mv appsettings.Development.json appsettings.json
RUN dotnet publish "DemoBlueGreen.csproj" -c Release -o /app
```
Com o executável criado copiamos a runtime do .Net para uma pasta onde ele executará a aplicação.
```
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim
WORKDIR /app
COPY --from=build-env /app .
ENTRYPOINT ["dotnet", "DemoBlueGreen.dll"]
```
Agora com tudo pronto configuramos o container para expor sua porta 80 para receber as chamadas.
```
EXPOSE 80
```

### Executando o container.

Agora com o Dockerfile configurado, execute o Docker Desktop e aguarde até o ícone indicar que o Docker está rodando.

![alt text](https://github.com/giiovanimp/demobluegreen/blob/main/Images/dockerrunning.png?raw=true)

Após a execução, abra um Prompt de comando na pasta do projeto para que possam ser executados os comandos Docker.

Primeiramente vamos executar **docker build** indicando que criaremos uma tag, isto é, uma versão da aplicação **-t**, escrevemos o nome da imagem que será gerada acompanhada do nome da tag, nesse caso v1, **giiovanimp/demobluegreen:v1**, ao final indicamos a localização da Dockerfile, neste caso estamos no repositório então colocamos um **.** indicamos que está nesse diretório.
 
O comando completo ficará assim:
```
docker build -t giiovanimp/demobluegreen:v1 .
```
Com isso o Docker irá executar todos os comandos listados na Dockerfile e exibirá na tela, caso tudo esteja certo indicará que gerou a build e a tag com sucesso.

![alt text](https://github.com/giiovanimp/demobluegreen/blob/main/Images/successfullybuilt.png?raw=true)

Com a imagem gerada podemos executar o container no docker localmente utilizando o comando **docker run**, para esse comando primeiramente indicamos o direcionamento de portas para acessar o container, nesse exemplo vamos utilizar a porta 8080 para referenciar a porta 80 do container, ficando assim **-p 8080:80**. Logo após indicamos o container que será executado juntamente com sua tag **giiovanimp/demobluegreen:v1**. O Comando completo ficará assim:
```
docker run -p 8080:80 giiovanimp/demobluegreen:v1
```

![alt text](https://github.com/giiovanimp/demobluegreen/blob/main/Images/containerrunning.png?raw=true)

Após a confirmação de execução, podemos acessar localmente a aplicação indicando a porta configurada. Em um navegador acesse: http://localhost:8080/weatherforecast

No caso dessa aplicação, ela retornará a previsão do tempo dos próximos 5 dias.

![alt text](https://github.com/giiovanimp/demobluegreen/blob/main/Images/localhostv1.png?raw=true)

Para parar a execução do container, voltamos a linha de comando e listamos os containers em execução para ter acesso ao id do container.
```
docker container ls
```
Agora com o id em mãos execute o comando substituindo o id do container:
```
docker kill containerId
```
Em caso de sucesso o Docker irá retornar o ID do container deletado.

### Publicando a imagem no DockerHub

Agora que temos a imagem funcionando vamos enviar para o DockerHub. Para isso execute o comando **docker login** para caso não esteja, o docker entre em sua conta.
Após logado vamos fazer um **push** com a imagem gerada utilizando o comando:
```
docker push giiovanimp/demobluegreen:v1
```

Com a imagem publicada no DockerHub podemos passar para o Kubernetes.

## Segundo Passo: Publicando e acessando aplicação no Kubernetes
### Criando Namespace e Contexto

Para esse passo, certifique que o **kubectl** está devidamente configurado e apontando para o Cluster. Para isso no Prompt de Comando execute:
```
kubectl cluster-info
```
Ele exibirá as URLs configuradas.

Assim como Dockerfile, o Kubernetes também precisa de alguns arquivos de configuração para serem executados. Inicialmente criaremos um namespace e um contexto, para facilitar a separação da aplicação dos outros elementos padrões no Kubernetes.
Para criar um Namespace, crie um arquivo [namespace.yaml](https://github.com/giiovanimp/demobluegreen/blob/main/Kubernetes/namespace.yaml). Nele definimos qual o tipo de elemento que estamos criando no **kind** e damos o nome dele dentro de **metadata**.

```
apiVersion: v1
kind: Namespace
metadata:
  name: giiovanimp
```
Lembrando que para um arquivo YAML a indentação é primordial para a leitura correta do documento. Sendo definido 2 espaços para cada elemento filho.

Com o documento criado utilizaremos o comando **kubectl apply** para aplicar as configurações descritas no arquivo. Para isso esteja com o Prompt de Comando na pasta em que os arquivos foram criados para que possamos referenciar corretamente. Este comando será usado durante todas modificações e aplicações de YAML no Kubernetes.

O comando ficará assim:
```
kubectl apply -f namespace.yaml
```

Agora que já possuimos um Namespaces, vamos configurar um Contexto para esse Namespace. Para isso execute o comando abaixo colocando o nome do contexto desejado e apontando para o Namespace criado préviamente.
```
kubectl config set-context giiovanimp-demo --namespace=giiovanimp
```
Agora acesse o contexto.
```
kubectl config use-context giiovanimp-demo
```

### Gerando deploy da aplicação e do serviço

Com o contexto configurado podemos executar alguns comandos básicos, como listar Pods ou Services. Para isso utilizamos o verbo Get no comando **kubectl**. Exemplo:
```
kubectl get pods
kubectl get services
```
Como ainda não publicamos nenhum deployment, é normal não exibir nenhum Pod. Igualmente os services onde temos apenas o serviço interno do Kubernetes publicado.

![alt text](https://github.com/giiovanimp/demobluegreen/blob/main/Images/getpodsservices.png?raw=true)

Para publicar a primeira versão da aplicação crie um arquivo chamado [deploymentv1.yaml](https://github.com/giiovanimp/demobluegreen/blob/main/Kubernetes/deploymentv1.yaml), nele faremos todas configurações necessárias para o nosso Deployment.

Inicialmente definimos qual tipo de elemento vamos aplicar, nesse caso, um Deployment, que será publicado no **namespace: giiovanimp** com o nome **demo-blue-green-dep**. Aqui também é definida a label, classificação e versão da app, sendo **demo-bg**, **app"** na **v1**. Ficando assim:
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
Essas informações serão utilizadas nas próximas linhas também, para identificar o deploy a ser feito. Com isso começamos a especificar melhor nosso deploy.

```bash
spec: 
  selector: #Aqui é definido o seletor da label criada anteriormente.
    matchLabels:
      app: demo-bg
      tier: app
      version: v1
  replicas: 1 #Definindo também a quantidade de réplicas (pods) que serão criadas.
  template: #Agora definimos o template usado para o deploy
    metadata:
      labels: #possuindo a mesma label criada acima.
        app: demo-bg
        tier: app
        version: v1
    spec:
      containers: #Aqui é especificado o container que esse Pod possuirá
        - name: demobg
          image: giiovanimp/demobluegreen:v1 #buscando no dockerHub a imagem gerada no primeiro passo
          ports: #Expondo também as portas que serão usadas.
            - name: http 
              containerPort: 80
              protocol: TCP
          resources: #Aqui é definido os recursos que cada pode irá exigir inicialmente. Podendo possuir limites também.
            requests:
              cpu: 10m #10 milicores equivalem a 10% de um Core, que possui 1000 milicores.
              memory: 100Mi #100 MebiBytes, equivalente a 104,858 Megabytes
          imagePullPolicy: IfNotPresent #A política de busca da imagem. Neste caso, caso não já não exista.
      restartPolicy: Always #Indica que o Pod reiniciará sempre que algo der errado.
  strategy: #Estratégia de deploy.
    type: RollingUpdate #Neste caso Rolling Update, onde atualiza réplica a replica.
```

Como arquivo configurado podemos aplicar ele para o Kubernetes utilizando:
```
kubectl apply -f deploymentv1.yaml
```
Com o deployment criado, ao executar o comando para recuperar os pods, pode-se visualizar a aplicação na lista.

![alt text](https://github.com/giiovanimp/demobluegreen/blob/main/Images/applydeploymentv1.png?raw=true)

Agora com o Pod já publicado no cluster é preciso criar um serviço, para isso crie um novo arquivo [service.yaml](https://github.com/giiovanimp/demobluegreen/blob/main/Kubernetes/service.yaml). Nele iremos definir um serviço externo que guiará para o Pod publicado.

```bash
apiVersion: v1
kind: Service #Agora utilizamos o tipo Service
metadata:
  namespace: giiovanimp #Também estará no mesmo namespace
  name: demo-blue-green-svc
spec:
  type: LoadBalancer #Sendo do tipo balanceador de carga
  ports: #direcionará para a porta 80 do Pod.
    - port: 80
  selector: #Definimos qual aplicação ele irá referenciar e sua versão
    app: demo-bg
    tier: app
    version: v1
```

Aplique o arquivo como feito nos anteriores.
```
kubectl apply -f service.yaml
```
Ao recuperar os serviços pode-se observar o que acabamos de criar. Ele possui uma propriedade **EXTERNAL-IP**, esse IP que utilizaremos para acessar nossa aplicação.

![alt text](https://github.com/giiovanimp/demobluegreen/blob/main/Images/applyservice.png?raw=true)

Em um navegador acesse http://20.62.216.3/weatherforecast, substituindo o IP gerado no serviço.

![alt text](https://github.com/giiovanimp/demobluegreen/blob/main/Images/podv1.png?raw=true)

É válido lembrar que todas essas configurações podem ser feitas em um único arquivo YAML, separando as definições com ***---***, assim como exemplificado no arquivo [k8sdeploy.yaml](https://github.com/giiovanimp/demobluegreen/blob/main/Kubernetes/k8sdeploy.yaml)
