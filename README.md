## Startup services

#### docker-compose file
```yaml
# docker-compose.yml
version: '3.5'
services:
  envconsul:
    # for test
    image: hashicorp/envconsul:0.12.1
    entrypoint: sh
    tty: true
    stdin_open: true
  vault:
    image: vault:1.10.0
    ports:
      - '8200:8200'
  consul:
    image: consul:1.8
    ports:
      - '8500:8500'
```

```bash
docker-compose up -d

# check vault token from log, same like:
# ...
# vault_1            | Unseal Key: DrkBU8OcjbHuLZXY/gYmtALN9wK+LbTOhIbllyhyfKs=
# vault_1            | Root Token: $VAULT_TOKEN

export VAULT_TOKEN=`docker-compose logs vault | grep "Token:" | awk '{print $5}'`

```

### Test for consul

#### create base config

```bash
curl -X 'PUT' -H 'X-Consul-Token: '  'http://localhost:8500/v1/kv/dev-zhipeng/apps/base/app?dc=dc1' --data 'base'
curl -X 'PUT' -H 'X-Consul-Token: '  'http://localhost:8500/v1/kv/dev-zhipeng/apps/base/stage?dc=dc1' --data 'dev-zhipeng'
```

#### create app1 config

```bash
curl -X 'PUT' -H 'X-Consul-Token: '  'http://localhost:8500/v1/kv/dev-zhipeng/apps/app1/app?dc=dc1' --data 'app1'
curl -X 'PUT' -H 'X-Consul-Token: '  'http://localhost:8500/v1/kv/dev-zhipeng/apps/app1/name?dc=dc1' --data 'app1'
```

#### show base envs

```bash
docker-compose exec envconsul envconsul  -consul-addr=consul:8500 -wait=2s:5s -log-level=debug -pristine -sanitize -upcase -no-prefix=true  -prefix=dev-zhipeng/apps/base -exec="env"
# APP=base
# STAGE=dev-zhipeng
```

#### run with app1 envs(without base)

```bash
docker-compose exec envconsul envconsul  -consul-addr=consul:8500 -wait=2s:5s -log-level=debug -pristine -sanitize -upcase -no-prefix=true  -prefix=dev-zhipeng/apps/app1 -exec="env"
# NAME=app1
# APP=app1
```

#### run with app1 envs(inherited from base)

```bash
docker-compose exec envconsul envconsul  -consul-addr=consul:8500 -wait=2s:5s -log-level=debug -pristine -sanitize -upcase -no-prefix=true  -prefix=dev-zhipeng/apps/base -prefix=dev-zhipeng/apps/app1 -exec="env"
# APP=app1
# STAGE=dev-zhipeng
# NAME=app1
```


### Test for vault
#### create base config

```bash
curl  -X 'PUT'  -H "X-Vault-Token: $VAULT_TOKEN" 'http://localhost:8200/v1/secret/data/dev-zhipeng/apps/base' --data-raw '{"data":{"app":"baseapp","name":"basename","stage":"dev-zhipeng"}}'
```

```bash
curl  -X 'PUT'  -H "X-Vault-Token: $VAULT_TOKEN" 'http://localhost:8200/v1/secret/data/dev-zhipeng/apps/app1' --data-raw '{"data":{"app":"app1","name":"app1-name"}}'
```

#### run with base envs

```bash
docker-compose exec envconsul envconsul  -vault-addr=http://vault:8200 -vault-token=$VAULT_TOKEN -wait=2s:5s  -log-level=debug -pristine -sanitize -upcase -no-prefix=true  -secret=secret/dev-zhipeng/apps/base -exec="env"
# STAGE=dev-zhipeng
# APP=baseapp
# NAME=basename
```

#### run with app1 envs(without base)

```bash
docker-compose exec envconsul envconsul  -vault-addr=http://vault:8200 -vault-token=$VAULT_TOKEN -wait=2s:5s  -log-level=debug -pristine -sanitize -upcase -no-prefix=true  -secret=secret/dev-zhipeng/apps/app1 -exec="env"
# APP=app1
# NAME=app1-name
```

#### run with app1 envs(inherited from base)

```bash
docker-compose exec envconsul envconsul  -vault-addr=http://vault:8200 -vault-token=$VAULT_TOKEN -wait=2s:5s  -log-level=debug -pristine -sanitize -upcase -no-prefix=true  -secret=secret/dev-zhipeng/apps/base -secret=secret/dev-zhipeng/apps/app1 -exec="env"
# APP=app1
# NAME=app1-name
# STAGE=dev-zhipeng
```

### Manual Install envconsul
```bash
# Dockerfile
FROM python:alpine
ENV NAME=envconsul
ENV VERSION=0.12.1
ENV HASHICORP_RELEASES=https://releases.hashicorp.com
ENV ARCH=amd64

RUN wget -O ${NAME}-${VERSION}.zip ${HASHICORP_RELEASES}/${NAME}/${VERSION}/${NAME}_${VERSION}_linux_${ARCH}.zip && \
    unzip ${NAME}-${VERSION}.zip && \
    mv ${NAME} /usr/local/bin/${NAME} && \
    rm ${NAME}-${VERSION}.zip
RUN ${NAME} -v
RUN echo '#!/bin/sh' > /entrypoint.sh; \
    echo 'echo $@; ${NAME} $@'  > /entrypoint.sh; \
    chmod +x /entrypoint.sh

ENTRYPOINT /entrypoint.sh $@
```

```bash
docker-compose.dev.yml 
version: '3.5'
services:
  app1:
    build: .
    command: "- -vault-addr=http://vault:8200 -vault-token=$VAULT_TOKEN -pristine -sanitize -upcase -no-prefix=true  -secret=secret/dev-zhipeng/apps/base -secret=secret/dev-zhipeng/apps/app1 -exec=\"env\" "
    #command: ["", "-v"]
  envconsul:
    image: hashicorp/envconsul:0.12.1
    entrypoint: sh
    tty: true
    stdin_open: true
  vault:
    image: vault:1.10.0
    ports:
      - '8200:8200'
  consul:
    image: consul:1.8
    ports:
      - '8500:8500'
      - '8501:8501'
      - '8600:8600/udp'
```

```bash
docker-compose -f docker-compose.dev.yml up -d --build app1
docker-compose -f docker-compose.dev.yml logs app1

#app1_1             | NAME=app1-name
#app1_1             | STAGE=dev-zhipeng
#app1_1             | APP=app1
```
