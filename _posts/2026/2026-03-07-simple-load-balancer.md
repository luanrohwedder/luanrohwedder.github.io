---
title: Aprendendo Load Balancer
date: 2026-07-03
categories: [System Design, BackEnd, Web]
tags: [system design, docker, golang, nginx]
---

Hoje em dia, é muito comum serviços terem dificuldade de entregar um serviço eficiente, sem travar a conexão, cair, etc. Existem diversas estratégias para isso que são estudadas em System Design, cada uma sendo melhor para um caso, sempre com um _trade off_, mas hoje quero falar especificamente do **Load Balancer**.

## O problema

Antes de explicar sobre o Load Balancer, vamos entender primeiro o problema que ele tenta resolver. Vamos supor que existe apenas um único servidor para um certo SaaS. No começo, esse único servidor é suficiente, visto que o nosso SaaS possui poucos clientes. Mas do dia pra noite, o acesso foi de 100 clientes para 100 mil clientes, e o resultado foi que o servidor não suportou toda essa carga.

Além desse problema, com apenas um servidor nós vamos ter um Single Point of Failure, onde, caso o servidor fique fora de serviço por outro motivo, o sistema inteiro fica fora do ar. Podemos citar também a dificuldade de escalar um único servidor, que será limitado ao escalonamento vertical.

![Modelo de um servidor sobrecarregado](https://pub-b617dac63aca4aac960f4e53b352ffe6.r2.dev/Untitled-2026-07-03-1555.png)
_Modelo de um servidor sobrecarregado_

Então para resumir, com apenas um servidor, sem balancear a carga de clientes acessando o nosso serviço, teremos os seguintes problemas:

- Single Point of Failure
- Sobrecarga do servidor
- Escalabilidade limitada 

## Load Balancer

Para resolver esse problema, uma das técnicas utilizadas é o **Load Balancer**. Ele distribui as solicitações do cliente entre vários servidores, evitando a sobrecarga deles, e essa distribuição é feita utilizando diferentes algoritmos, que avaliam como o servidor está no momento e transferem a solicitação para o servidor certo.

Além de resolver a sobrecarga do servidor, resolve também o Single Point of Failure, visto que se um servidor falhar, basta o Load Balancer distribuir entre os servidores restantes.

![Modelo usando um Load Balancer](https://pub-b617dac63aca4aac960f4e53b352ffe6.r2.dev/load_balancer.png)

### Tipos de Load Balancer

Existem basicamente três tipos de Load Balancer:

- **Hardware:** São equipamentos que fazem esse balanceamento de cargas, usados muito em data centers.
- **Software:** São softwares que rodam em um servidor e distribuem as requisições para os outros servidores. Ex: Nginx, HAProxy.
- **Cloud:** Serviços cloud que oferecem um Load Balancer, como os serviços da AWS, Microsoft e Google.

### Load Balancer no Modelo OSI

Existem dois Load Balancers mais comuns para os layers do modelo OSI, o layer 4 e a layer 7.

- **Layer 4:** Atua diretamente na camada de transporte, dessa forma, ele sabe apenas de onde vem a requisição (origem) e para onde deve ir a requisição (destino). Isso o torna mais eficiente e rápido para transferir grande volume de dados.
- **Layer 7:** Atua na camada de aplicação, então o Load Balancer analisa o conteúdo das requisições, como headers HTTP, URL, etc. Isso torna o roteamento mais inteligente, conseguindo tomar decisões baseadas no conteúdo.

## Prática

Para não ficar apenas na teoria, decidi testar de forma prática esse conceito do Load Balancer, obviamente em uma escala minúscula, apenas para simular como isso pode funcionar. Para isso, primeiro criei um servidor simples em **Go** que ficou da seguinte forma:

```
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
)

func main() {
	port, server := os.Getenv("PORT"), os.Getenv("SERVER")
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "<h1>Servidor ", server, "</h1>")
	})

	log.Fatal(http.ListenAndServe(fmt.Sprintf(":%s", port), nil))
}
```

Ele vai escutar na porta que eu vou definir ainda no docker-compose, e vai escrever na página falando em qual servidor ele está, bem simples. Depois, criei um **Dockerfile** que vai compilar esse código em binário e copiar o binário para um container **scratch** (container vazio, apenas para executar o binário).

```Dockerfile
FROM golang:1.26.4 AS builder

WORKDIR /app

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -o servidor .

FROM scratch

COPY --from=builder /app/servidor /servidor

CMD ["/servidor"]
```

Como pode ver, ele pega uma imagem com um compilador golang, copia tudo para o diretório /app e então executa `CGO_ENABLED=0 GOOS=linux go build -o servidor .` para compilar o arquivo. Logo em seguida, cria um container scratch, copia do container builder o binário e executa ele, que vai ficar escutando na porta definida no docker-compose, criando dessa forma um container leve para o servidor.

```
x-servidor: &servidor
  image: servidor-go
  build: .

services:
  servidor1:
    <<: *servidor
    environment:
      PORT: 8081
      SERVER: 1

  servidor2:
    <<: *servidor
    environment:
      PORT: 8082
      SERVER: 2

  servidor3:
    <<: *servidor
    environment:
      PORT: 8083
      SERVER: 3

  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - servidor1
      - servidor2
      - servidor3
```

Aqui eu pedi ajuda para o ChatGPT em como fazer com que eu usasse somente uma vez a imagem que eu iria criar do **Dockerfile**, e usasse ela para criar três serviços, o servidor em Go. Veja que cada serviço usa a imagem criada no começo do compose, e define as variáveis de ambiente que serão usadas no servidor, como a porta em que ele irá ficar escutando e o número do servidor, para verificar se todos os servidores estão no ar. Por fim, temos o serviço do *nginx*, o load balancer, que vai ficar escutando no localhost e posteriormente vai redirecionar para os outros três servidores.

Para o **nginx** funcionar, é necessário um arquivo de configuração bem simples (para esse caso de estudo), que vai mapear os servidores que estão no container para a porta certa, fazendo o balanceamento de carga.

```
events {}

http {
    upstream backend {
        server servidor1:8081;
        server servidor2:8082;
        server servidor3:8083;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

Assim, basta rodar `docker-compose up -d` e acessar o localhost, que o nginx vai fazer o seu trabalho. Nesse caso, como não foi configurado o tipo de algoritmo usado, por padrão ele irá usar o Round Robin, em que as requisições são passadas de forma sequencial para cada servidor. E o resultado final fica assim, onde para cada F5 vai mudando o server utilizado.

![Resultado final](https://pub-b617dac63aca4aac960f4e53b352ffe6.r2.dev/load_balancer_funcionando.gif)


## Conclusão

A motivação para esse post foi um vídeo do [Renato Augusto](https://www.youtube.com/watch?v=hhy6EDDjy-o) sobre o tema Load Balancer, tanto que o exemplo prático simples é praticamente igual ao do vídeo dele. A diferença é que eu subi os servidores em um docker para também me familiarizar com os conceitos do Docker que raramente uso.

Outra motivação foi em relação às IAs, visto que fazer código hoje em dia com IAs é barato. Eu acho interessante aprender fundamentos, engenharia de software e conceitos que precisam de tomada de decisão, onde talvez as IAs ainda não sejam tão boas.
