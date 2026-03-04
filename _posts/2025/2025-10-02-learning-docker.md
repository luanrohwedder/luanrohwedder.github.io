---
title: Aprendendo Docker
date: 2025-10-02
categories: [Aprendendo]
tags: [aprendendo, docker, tecnologia]
image: /assets/img/docker-logo.png
---

Bom, como havia dito no post de [apresentação]({% post_url 2025-09-28-hello %}), eu ia falar sobre o docker. Eu estava postergando aprender o docker por alguns anos já, aproveitei a criação desse blog para tirar do meu **TODO LIST** de coisas que preciso aprender. Para deixar claro, eu já tive algumas experiências usando o docker no passado, mas nada muito profundo, apenas seguindo tutorial sem aprofundar muito sobre o assunto. Pois então, chegou a hora de aprender pelo menos o básico de ele funciona.

> **Disclaimer:** Meu primeiro contato com o docker, pode ter muita coisa errada aqui!! (e eu só queria usar prompt de aviso também)
{: .prompt-warning}

## O que é o Docker

O docker chegou para solucionar o famoso problema de *"Assim, na minha máquina ele funcionou legal"*, que inclusive é a minha motivação de aprender o docker. Pois bem, de que forma ele soluciona esse problema? Simples, criando um tipo ambiente que funciona igual para todos que forem utilizar. A forma que ele faz isso é utilizando imagens, que são uma espécie de modelo, geralmente um ambiente linux, que por sua vez é instanciado por um container, que cria um ambiente isolado do sistema operacional utilizando a imagem especificada.

Dessa forma, esse ambiente isolado criado pelo docker, sempre vai ser o mesmo para todos que forem utilizar, desde que se utilize das mesmas configuração. Uma introdução bem simples do que é o docker. Agora, eu quero explicar sobre algumas componentes dele, como a imagem e o container, além de falar um pouco também sobre o dockerfile e docker-compose.yml.

### Imagens

Como explicado anteriormente, a imagem é um tipo de modelo pronto para ser usado pelo container. Nela é contida todas as dependências, arquivos, bibliotecas, etc., que o container precisa para funcionar. Dessa forma, fica fácil compartilhar a mesma imagem para várias pessoas, que vai se utilizar do mesmo sistema, evitando o problema da máquina local. Alguns pontos legais de citar das imagens são:

- Somente para leitura (read-only) após sua criação
- Contém camadas (layers), para melhor armazenamento e melhorar o tempo de build (só vai buildar novamente o layer alterado)

Em todo docker nós vamos utilizar uma imagem base (acho que não tem como fazer um dockerfile sem imagem base, não sei ainda), que geralmente pegamos de um hub de imagem, como **dockerhub**, e a partir dessa imagem base, vamos montando nossa imagem com algumas adições de código, execuções, formando uma imagem pronta para ser utilizada no final. Cada adição que fazemos, conta como um layer. Essa parte de montagem da imagem, vou explicar melhor quando falar sobre o Dockerfile.

Em resumo, uma imagem é uma receita, um modelo pronto e configurado, de fácil compartilhamento, pronto para ser utilizado por um container.

### Container

O container é um ambiente isolado da máquina host, com todas as configurações necessárias, como códigos, bibliotecas, etc., para serem executadas dentro dele, de forma isolada. Isso é bom pois garante o mesmo ambiente para todos que forem utilizar daquele container. Assim, o docker é uma instância em execução da imagem que definimos. Esse é o conceito simples dos containers, ele pode escalar muito mais, como fazer uma conexão entre dois ou mais containers isolados, etc.

### Dockerfile

É aqui que é feito a especificação da imagem, onde escolhemos a imagem base, copiamos arquivos locais para a imagem, executa comandos, etc. Existem várias coisas que podem ser feito em um dockerfile. Aqui um exemplo de dockerfile que utilizei para aprender.

```Dockerfile
FROM python:3

WORKDIR /app

RUN apt-get update && apt-get install -y postgresql-client && rm -rf /var/lib/apt/lists/*

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

COPY init.sh /init.sh
RUN chmod +x /init.sh

CMD ["/app/init.sh"]
```

Vamos por partes:

- **FROM** - Aqui é definido a imagem base que o container vai rodar, no exemplo, uma imagem do python3.
- **WORKDIR:** - Diretório onde será trabalhado, funciona como um *cd* até esse diretório.
- **COPY:** - Copia arquivos locais até o WORKDIR
- **RUN:** - Executa comando durante o processo de build da imagem
- **CMD:** - Executa um comando após o processo de build da imagem

Então olhando para o dockerfile de exemplo, primeiro escolhemos a imagem base, que no caso é uma imagem do python3, após isso esoclhemos o diretório de trabalho, onde vai ser copiados os arquivos locais. Logo após, executamos um comando durante o processo de build, para atualizar o sistema e instalar o postgresql-client (que será utilizado depois no compose, eu acho). Em seguida, é copiado o requirements.txt e executado o mesmo para instalar as dependências necessárias do código python. Por fim, copiamos o restante dos arquivos locais no diretório raíz onde se encontra o dockerfile (. ., significa tudo), e executamos um script para inicializar o banco (não é imporante essa parte então vou pular).

Por enquanto é isso sobre o dockerfile, foi o que eu consegui aprender, coisa bem básicas, mas que já dá pra fazer bastante coisas.

### docker-compose.yml

O docker-compose, ou compose para ficar mais fácil, é o que vai orquestrar vários containers de um app só. Veja o seguinte arquivo compose como exemplo:

```
version: "3.9"

services:
  web:
    build: .
    container_name: flask_app
    ports:
      - "5000:5000"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=meubanco
      - POSTGRES_HOST=db
      - POSTGRES_PORT=5432
    depends_on:
      - db

  db:
    image: postgres:15
    container_name: postgres_db
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=meubanco
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  db_data:
```

Aqui definimos dois serviços, um web e outro um banco de dados (ainda não sei se esses nomes são padrões, acredito que sim). Dessa forma, irá ser criado dois containers, um exclusivamente para rodar a aplicação web e outro para rodar o banco de dados. Não vou entrar em detalhes dos parâmetros, porque acredito que dependendo da imagem eles mudam, então é mais coisa de ler a documentação mesmo. Uma coisa que vale mencionar, mas que eu não entendi por completo ainda, é que podemos criar volumes, que é uma forma de armazenamento persistente, visto que quando derrubamos um container, ele perde todos os seus dados. Acredito que isso seja interessante para self host, coisas que vou brincar futuramente.

## Conclusão

Finalmente aprendi um pouco sobre o docker, conseguindo até criar um para onde eu trabalho, e sim, eu trabalho em uma empresa pequena, que não utiliza docker, o que está tudo bem. Mas essa falta de ter um docker na empresa, me motivou a aprender, para tentar solucionar alguns problemas que estávamos enfrentando. Ainda não está 100%, mas estou pegando o jeito. Assim, tiro mais um item da minha lista, ~docker~.
