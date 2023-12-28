# Docker Guide - Getting Started

Guia de Estudo elaborado a partir da documentação oficial do [Docker Docs](https://docs.docker.com/get-started/).

Esse guia de estudos aborda os seguintes tópicos:
- [Visão Geral do Docker](#visão-geral-do-docker)
- [Aplicação em Container](#aplicação-em-container)
- [Atualizando e Compartilhamento](#atualização-e-compartilhamento)
- [Volumes](#volumes)
- [Networks e Multi-Containers](#networks-e-multi-containers)
- [Docker Compose](#docker-compose)

## Visão Geral do Docker

Docker é uma plataforma que possibilita o desenvolvimento e execução de aplicações em um ambiente isolado e gerenciado. Esse ambiente isolado é chamado de Container e entrega uma infraestrutura especialmente gerenciada e configurada para a execução de determinado serviço.

O Docker pode ser utilizado nos fluxos de Integração e Entrega Contínuas (CI/CD), entregando aos desenvolvedores um ambiente de desenvolvimento consistente e padronizado, um ambiente de testes automatizados e um ambiente de produção confiável, para o qual a imagem da aplicação é publicada após as validações.

Exemplo:

```
$ docker run -it ubuntu /bin/bash
```

O comando acima cria e executa interativamente `-it` um container com a imagem do ubuntu, executando o comando `/bin/bash`. A imagem do ubuntu é baixada do Docker Hub se já não estiver presente no host.

### Objetos Docker

Ao utilizar o Docker, diversos objetos são criados e gerenciados, alguns deles são: Imagens, Containers, Volumes e Networks.

#### Imagem

Uma imagem é um template read-only com instruções para a criação de um container. Uma imagem possui todas as dependências, configurações, variáveis de ambiente e outras informações necessárias para a criação de um container que roda um serviço específico.

#### Container

Um container é uma instância executável de uma imagem. É possível criar, iniciar, parar, remover containers de acordo com a necessidade. Um container é necessariamente criado a partir de uma imagem. O container é um processo isolado rodando no host. Esse isolamento do processo é alcançável através da utilização de `namespaces`, `chroots` e `cgroups`.

`namespaces` criam uma camada de abstração que envolvem os recursos globais do sistema, fazendo com que o processo pareça rodar em uma instância isolada desses recursos. `chroots` atribui um diretório root a um determinado processo, de maneira que este não consiga acessar outros locais do sistema; o root de um container é o próprio sistema de arquivos da imagem. `cgroups` são utilizados para atribuir um processo a determinado grupo com limitações específicas aos recursos do sistema.

Para entender melhor o funcionamento do Container por de baixo dos panos, assista [esse vídeo](https://www.youtube.com/watch?v=8fi7uSYlOdc). Volumes e Networks serão abordados com mais detalhes posteriormente nesse guia.

### Arquitetura

Docker funciona com uma arquitetura Cliente-Servidor, em que o cliente (Docker API e CLI) enviam requisições HTTP, através de UNIX sockets ou network interfaces, ao Docker Daemon (processo intermitente rodando em plano de fundo).

![Arquitetura Docker](/images/docker-architecture.webp?raw=true)

O Docker Daemon escuta por requisições do Client e gerencia os Objetos Docker, como Imagens, Containers, Volumes e Networks. Além disso, o Docker Daemon também pode acessar Docker Registries, onde são armazenadas as Imagens. O Docker Hub é um registry público, onde o Docker busca as imagens por padrão.

## Aplicação em Container

Esse guia apresenta uma aplicação de exemplo implementada em Node.js. A aplicação é um Todo List. No root do repositório há um arquivo sem extensão chamado Dockerfile, que contém as instruções para o build da imagem da aplicação:

```
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
EXPOSE 3000
```

A aplicação tem como imagem base o `node:18-alpine`, baixado do Docker Hub. Além disso, instruções de comandos a serem executados e portas são especificadas. Para construir uma imagem dessa aplicação, e em seguida rodar um container com a imagem, basta executar os seguintes comandos:

```
$ docker build -t getting-started .
$ docker run -dp 127.0.0.1:3000:3000 getting-started
```
O primeiro comando faz o build da imagem da aplicação, já o segundo cria e roda um container, a partir da imagem, realizando um mapeamento da porta 3000 do container para a porta 3000 do host. Liste os containers em execução com o comando `docker ps`, ou todos com `docker ps -a`.

## Atualização e Compartilhamento

Após a implementação de uma funcionalidade na aplicação, é possível construir a nova imagem com o mesmo comando de build utilizado na seção anterior. Para parar um container em execução e removê-lo, utilize os seguintes comandos:

```
$ docker stop <container-id>
$ docker rm <container-id>
```

> Alternativamente, é possível utilizar a flag `-f` com o segundo comando para remover um container forçadamente, sem pará-lo.

Para compartilhar uma imagem, crie uma conta no [Docker Hub](https://hub.docker.com/). Em seguida, crie um repositório, faça login com `docker login -u <user-name>`, e envie a imagem para o Docker Hub:

```
$ docker push <user-name>/getting-started
```

> Certifique-se de que o nome da imagem apresenta o seguinte formato: `<user-name>/<image-name>`

Pronto, agora a imagem da sua aplicação está disponível publicamente no Docker Hub e pode ser baixada com o comando `docker pull <user-name>/getting-started`.

## Volumes

Dados criados ou alterados durante a execução de um container não são persistidos uma vez que o container é removido e outro é inciado, mesmo que utilizem a mesma imagem. Isso ocorre pois o sistema de arquivos de um container é isolado de todo o resto do sistema.

Para que dados sejam mantidos através da remoção e criação de containers, utilizam-se volumes. Volumes são caminhos que podem ser mapeados para a máquina do host e acessados pelo container. Existem dois tipos de volumes, `volume mounts` e `bind mounts`.

Para criar um volume e rodar um container utilizando ele, utilize os comandos:

```
$ docker volume create todo-db
$ docker run -dp 127.0.0.1:3000:3000 \
    --mount type=volume,src=todo-db,target=/etc/todos \
    getting-started
```

A flag `--mount` é utilizada para montar o volume previamente criado no container. `type=volume` especifica o tipo como volume mounts. `src=todo-db` indica o volume a ser utilizado. `target=/etc/todos` atribui o caminho do sistema de arquivos do container a ser mapeado para o volume, que é criado em um caminho definido pelo Docker na máquina do host.

Bind Mounts é um outro tipo de volume, utilizado para compartilhar um diretório do sistema de arquivos do host para o container. Qualquer modificação realizada no diretório do host é refletida no container, e vice-versa. Esse tipo de mount pode ser utilizado para ouvir por mudanças no código e reinicializar a aplicação. No exemplo a ferramenta utilizada é o `nodemon`.

```
$ docker run -dp 127.0.0.1:3000:3000 \
    -w /app --mount type=bind,src="$(pwd)",target=/app \
    node:18-alpine \
    sh -c "yarn install && yarn run dev"
```

O comando acima roda o container da aplicação especificando o tipo de mount como `bind`. `-w /app` define o diretório de trabalho do container. `src="$(pwd)"` monta o diretório da aplicação do host para o `target=/app` do container.

Para visualizar os logs de reinicialização do servidor utilize o comando `docker logs -f <container-id>`.

## Networks e Multi-Containers

Na maioria das vezes, as aplicações que desenvolvemos não podem ser contidas dentro de um único projeto ou container. Diversos serviços integram a aplicação, como o Frontend, o Backend, um Banco de Dados etc. A abordagem mais apropriada nesses casos seria utilizar diferentes containers para diferentes serviços.

Ao utilizar esse tipo de abordagem, diversos benefícios são alcançados, como: facilidade do controle de versões através das mudanças; escalabilidade eficiente e facilitada; diminuição da complexidade, uma vez que um container inicia apenas um processo, e a execução de múltiplos processos exigiria um gerenciador.

Porém, para que esses múltiplos processos possam se comunicar entre si, por exemplo o Frontend enviar uma requisição HTTP para o Backend e este consultar o Banco de Dados, é necessário que todos estejam conectados na mesma rede, permitindo a comunicação, uma vez que são isolados de outros processos. Para isso são utilizadas as Docker Networks.

![Application Network](/images/multi-container.webp?raw=true)

No nosso exemplo, criaremos uma network em que o container da aplicação e um container do BD MySQL estejam ambos conectados.

```
$ docker network create todo-app
$ docker run -d \
    --network todo-app --network-alias mysql \
    -v todo-mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=todos \
    mysql:8.0
```

O primeiro comando acima cria uma network chamada `todo-app`. O segundo comando inicializa um container com a imagem do MySQL, conectando ele à rede previamente criada. A tag `--network-alias mysql` cria um alias para o endereço que será atribuído a esse container na rede. Nesse caso, `mysql` será o hostname desse serviço MySQL na network, que resolverá o endereço de IP.

Para confirmar que o container do banco de dados está rodando, execute `docker exec -it <mysql-container-id> mysql -u root -p`. Talvez seja necessário [especificar explicitamente a porta e protocolo de conexão](https://stackoverflow.com/questions/23234379/installing-mysql-in-docker-fails-with-error-message-cant-connect-to-local-mysq).

Agora, basta criar um container da aplicação, conectando à mesma rede e definindo variáveis de ambiente para a conexão com o container do MySQL.

```
docker run -dp 127.0.0.1:3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:18-alpine \
  sh -c "yarn install && yarn run dev"
```

Note que a variável de ambiente `MYSQL_HOST` recebe o valor `mysql`, que é o alias/ hostname que resolve o endereço de IP do container rodando o serviço de banco de dados.

## Docker Compose

Todo esse trabalho de gerenciamento e configuração dos containers realizado até agora através da CLI, pode ser rapidamente efetuado através de um único arquivo que especifica os serviços que serão rodados, bem como a imagem do serviço, comandos, mapeamento de portas, volumes, networks, diretório de trabalho, variáveis de ambiente entre outras configurações.

Nesse repositório você encontrará um arquivo chamado `compose.yaml`, com o seguinte conteúdo:

```
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 127.0.0.1:3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

Esse arquivo define dois serviços, que gerarão dois containers, o `app` e o `mysql`, ambos gerados com as imagens `node:18-alpine` e `mysql:8.0`, respectivamente. Mapeamento de porta, diretório de trabalho, variáveis de ambiente e volumes também são configurados para os dois serviços.

Quando múltiplos containers são gerados a partir do Docker Compose, não é necessário especificar uma network, pois o Docker criará uma automaticamente para esses serviços, que podem ser acessados através de seus nomes.

Para iniciar a aplicação ou encerrá-la, utilize os seguintes comandos:

```
$ docker compose up
$ docker compose down
```

A flag `-d` pode ser passada ao primeiro comando para rodar tudo em background. Quando encerrada a aplicação, os volumes continuarão existindo, para removê-los, utilize a flag `--volumes`.