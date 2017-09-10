---
layout: post
title:  "Multi Stages builds no docker"
date:   2017-09-03 22:48:45
description: Multi Stages builds é uma forma de construir imagens utilizando outras para realizar a compilação dos arquivos necessários para sua app, visando ter uma imagem pequena e somente com os binários da app.
categories:
- docker
- nodejs
---


Opa galera, tudo certo? 

Hoje vamos falar de um assunto legal sobre **docker**, para quem acompanha as atualizações do docker, não é um assunto tão novo, mas é um assunto interessante e que nem todo mundo conhece. Sabemos que a [Docker Inc][docker] é uma máquina de fazer código e temos novas versões literalmente toda semana. Com todas essas atualizações, eles lançaram na versão: [17.05][1705] o conceito chamado de **Multi Stages Builds** em uma tradução literal fica parecido com: **construção em múltiplos estágios**. 

## Mas o que é uma construção em múltiplos estágios? 

Fazendo uma anologia com algo que nós devs estamos acostumado, seria algo parecido com o **pipeline** que temos em nossas ferramentas de **CI** como: [Jenkins][jenkins], [Travis][travis] etc. Conseguimos definir vários passos antes de ter o nosso binário pronto para execução. 

Estamos falando de docker, então nosso binário será uma [imagem][imagem].

Imagem ilustrando como funciona o multi stage: 
![](/assets/img{{page.id}}/multi-stage-build.png)


Para demonstrar a facilidade e os benefícios que o **multi stage build** oferece, vou mostrar um cenário que passei alguns dias atrás.


Eu estava precisando gerar alguns dashboards com os indicadores que era retornado de uma API Rest, como minhas habilidades com front não são as melhores do mundo, fui dar uma pesquisada no github, encontrei um cara chamado: [freeboard][freboard]. O freeboard é um framework para criação de dashboards onde é possível plugar vários datasources.

## Freeboard sem Multi Stage

{% gist 1cdd70e2c19cdb833c1cd51352b1a905 %}

Olhando o **Dockerfile** parece tudo ok. Mas olhando carinhosamente temos alguns problemas:

* Instalamos o git, node e npm
* Utilizamos somente uma vez os recursos instalados

Todas essas ferramentas são somente para compilação do projeto, não precisamos delas em produção, precisamos apenas dos arquivos estáticos que foram gerados. Porém mesmo não precisando mais dessas ferramentas, elas fazem parte da nossa imagem e com isso ela fica grande.

![](/assets/img{{page.id}}/imagem-sem-multi-stage.png)

## Freeboard com Multi Stage

{% gist 2a47fb74ecf5cbd62f8579ccedc52b04 %}

Explicando as mudanças no dockerfile:

Agora estamos falando para o docker que iremos utilizar a imagem `node:alpine`, utilizamos a instrução `as` para dar um nome ao container que será criado utilizando essa imagem, no nosso caso demos o nome `temp`. 

Logo abaixo instalamos o **git**, não precisamos mais instalar o **node** e **npm** pois já estamos usando uma imagem com eles instalados.

Em seguida falamos para o docker executar os passos de compilação do projeto

```docker
WORKDIR /app
RUN git clone https://github.com/Freeboard/freeboard.git
RUN cd /app/freeboard && npm install -g
```
Agora sabemos que existe um container chamado de `temp` e que dentro desse container estão todos os arquivos estáticos que precisamos para nossa **app** funcionar em produção. Então o que precisamos é de um `webserver` para disponibilizar esses arquivos estáticos, nesse caso vamos utilizar o **nginx**

```docker
FROM nginx:1.12-alpine

COPY --from=temp /app/freeboard/css/ /usr/share/nginx/html/css/
COPY --from=temp /app/freeboard/js/ /usr/share/nginx/html/js/
COPY --from=temp /app/freeboard/img/ /usr/share/nginx/html/img/
COPY --from=temp /app/freeboard/plugins/ /usr/share/nginx/html/plugins/
COPY --from=temp /app/freeboard/index.html /usr/share/nginx/html/
```

Nessa parte do nosso `Dockerfile` estamos informando ao docker que queremos utilizar a imagem do **nginx**, na sequência estamos falando que ele deve copiar os arquivos do container que acabamos de criar, como podemos ter vários containers precisamos utilizar a instrução `--from` essa instrução recebe o **identificador** do container, nesse caso passamos o identifcador: `temp`. 

Caso não tivessemos utilizado o comando `as` para definir um nome ao container o docker iria atribuir automaticamente, por default o docker utiliza um contador que começa no `0`. Então nossa instrução do `COPY` iria ficar assim:

## Vantagens

Agora nossa imagem tem somente o que é necessário para app funcionar, com isso reduzimos seu tamanho, nesse caso a diferença de tamanho das imagens foi `60MB`:

![](/assets/img{{page.id}}/imagem-com-multi-stage.png)

1. Centraliza os passos para geração do binário
2. Redução no tamanho da imagem
3. Docker push/pull mais rápido já que o tamanho da imagem é menor
4. Ter uma imagem somente com o que **realmente** é necessário para execução


É isso galera, espero que tenham gostado da leitura :D

![](https://media.giphy.com/media/8g63zqQ5RPt60/giphy.gif)


[docker]: https://www.docker.com/
[imagem]:    https://hub.docker.com/u/alefh/
[jenkins]: https://jenkins.io/
[travis]: https://travis-ci.org/
[freeboard]: http://freeboard.io
[1705]: https://docs.docker.com/release-notes/docker-ce/#17061-ce-2017-08-15