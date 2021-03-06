# Contentorização Podman de um servidor PostgreSQL+PostGIS+pgRouting e com pgAdmin 4

Criação de containers Docker / Podman num sistema operativo Fedora.

## Imagem PostgreSQL+Postgis

Para criar imagem PostgreSQL + PostGIS (12.3 + 3.0),  construímos primeiro este ficheiro chamado *Dockerfile*:

```dockerfile
FROM postgres:12.3

LABEL maintainer="<localuser>, after PostGIS Project dockerfile"

ENV POSTGIS_MAJOR 3
ENV POSTGIS_VERSION 3.0.0+dfsg-2~exp1.pgdg100+1

RUN apt-get update \
      && apt-cache showpkg postgresql-$PG_MAJOR-postgis-$POSTGIS_MAJOR \
      && apt-get install -y --no-install-recommends \
        postgresql-$PG_MAJOR-postgis-$POSTGIS_MAJOR=$POSTGIS_VERSION \
        postgresql-$PG_MAJOR-postgis-$POSTGIS_MAJOR-scripts=$POSTGIS_VERSION \
        postgresql-$PG_MAJOR-ogr-fdw postgresql-$PG_MAJOR-cron \
        postgresql-plpython3-$PG_MAJOR postgresql-$PG_MAJOR-pgrouting \
      && rm -rf /var/lib/apt/lists/*
```

As instruções foram parcialmente copiadas da imagem Docker **postgis/postgis**. Foi forçada a versão 12.3 do PostgreSQL e foram removidos os comandos de inicialização da base de dados *template_postgis*, desnecessária.

Para fazer o **build** da imagem, atribuindo-lhe duas tags, *<localuser>/postgis:12-3.0* e *<localuser>/postgis:latest*:

```
sudo podman build -t <localuser>/postgis:12-3.0 -t <localuser>/postgis:latest -f ./Dockerfile
```

ATENÇÃO: Ao contrário do que acontece com Docker, com o **podman** (Fedora/Red Hat/CentOS 7 com Extras), as imagens que irão aceder a recursos de acesso privilegiado tem de ser manipuladas pelo user **root**.


## Mapeamento diretoria *dados*

Convém separar por versão, para ser possível ter containers a correr diferentes versões, se necessário, nomeadamente em momentos de upgrading de versão (Postgresql ou PostGIS).

Neste caso temos, para a versão PG 12.3 e PostGIS 3 o caminho:

    <dir.local>/postgresqlData/12-3

mapeado para ```/var/lib/postgresql/data``` no comando "run" do container como:

    -v <dir.local>/postgresqlData/12-3:/var/lib/postgresql/data

Irá ser criada automaticamente a diretoria **pgdata** em:

    <dir.local>/postgresqlData/12-3/pgdata

## Iniciar

```
sudo podman run \
    --name pgcont12-3 \
    --rm \
    -p 5432:5432 \
    -e "POSTGRES_PASSWORD=******" \
    -e "PGDATA=/var/lib/postgresql/data/pgdata" \
    -v <dir.local>/postgresqlData/12-3:/var/lib/postgresql/data \
    --detach <localuser>/postgis
```  

Onde se refere **podman** poderia estar **docker**. 

O *container* desta forma iniciado vai disponibilizar o *listener* do PostgreSQL na porta TCP 5432 do sistema hospedeiro (*host*), como esperado. 
Para lançar múltiplos *containers* destes, teríamos de alterar a linha do script anterior ...

```
    -p 5432:5432 \
```  

para


```
    -p 5433:5432 \
```   

para que um novo *container* oferecesse o seu serviço na porta 5433.
Teriamos também de alterar as linhas 

```
    --name pgcont12-3 \
 ...
    -v <dir.local>/postgresqlData/12-3:/var/lib/postgresql/data \ 
```

... para dar um nome diferente e uma localização diferente para os dados no dísco do *host*.


## PgAdmin contentorizado

A melhor solução para manter pgAdmin é também através da contentorização.

Comecei por baixar a imagem da minha preferência:

    docker pull dpage/pgadmin4

Criei depois o contentor respetivo com:
```
sudo podman run --name pgadmin4 \
	--rm \
	-p 5050:80 \
	-v <dir.local>/ContainerStorage/dpage_pgadmin4/lib/pgadmin:/var/lib/pgadmin \
	-e "PGADMIN_DEFAULT_EMAIL=xyztabc@gmail.com" \
	-e "PGADMIN_DEFAULT_PASSWORD=********" \
	--detach dpage/pgadmin4
```

O **pgAdmin** ficará assim disponível em ```http://localhost:5050```.

## Tudo em execução

O comando ```sudo podman ps``` retorna:

| CONTAINER ID | IMAGE                             | COMMAND     | CREATED        | STATUS               | PORTS                  | NAMES      |
| ------------ | --------------------------------- | ----------- | -------------- | -------------------- | ---------------------- | ---------- |
| d9e305482ec6 | localhost/&lt;localuser&gt;/postgis:12-3.0 | postgres    | 51 minutes ago | Up 51 minutes ago    | 0.0.0.0:5432->5432/tcp | pgcont12-3 |
| d26721a68705 | docker.io/dpage/pgadmin4:latest |  | 5 hours ago | Up 5 hours ago | 0.0.0.0:5050->80/tcp | pgadmin4 |


## Criar tablespaces

Usar **pgAdmin** para criar users e tablespaces.

No **pgAdmin**, teremos de indicar, como caminho dentro da imagem:

    /var/lib/postgresql/data/tablespaces/<tablespace>

Caminho físico correspondente será:

    <dir.local>/postgresqlData/12-3/tablespaces/<tablespace>

No Fedora, estas diretorias devem ter group:owner igual a:

    systemd-timesync:root
