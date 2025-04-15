+++
title = "Self-Hosting HedgeDoc"
date = "2025-04-15"
+++

## Introduction

→ <https://docs.goauthentik.io/integrations/services/hedgedoc/>

## Reverse Proxy: Caddy

<details>

<summary>caddy.yml</summary>

```yml
services:
  caddy:
    image: lucaslorentz/caddy-docker-proxy:2.9.2-alpine
    restart: unless-stopped
    ports:
      - 80:80
      - 443:443
    environment:
      - CADDY_INGRESS_NETWORKS=caddy-net
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./volumes/caddy:/data/caddy
    networks:
      - caddy-net
    healthcheck:
      test: ["CMD-SHELL", "curl --fail http://localhost:2019/metrics || exit 1"]
      start_period: 20s
      interval: 20s
      timeout: 5s
      retries: 3

networks:
  caddy-net:
    name: caddy-net
```

</details>

## SSO: Authentik

<details>

<summary>authentik.yml</summary>

```yml
services:
  postgresql:
    image: docker.io/library/postgres:16.8-alpine
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    networks:
      - authentik-net
    volumes:
      - ./volumes/authentik-postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${PG_PASS:?Variable is not set or empty}
      POSTGRES_USER: ${PG_USER:-authentik}
      POSTGRES_DB: ${PG_DB:-authentik}
  redis:
    image: docker.io/library/redis:7.4.2-alpine
    command: --save 60 1 --loglevel warning
    restart: unless-stopped
    networks:
      - authentik-net
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    volumes:
      - ./volumes/authentik-redis:/data
  server:
    image: ghcr.io/goauthentik/server:2025.2.4
    restart: unless-stopped
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS:?Variable is not set or empty}
      AUTHENTIK_LOG_LEVEL: warning
    volumes:
      - ./volumes/authentik-media:/media
      - ./volumes/authentik-custom-templates:/templates
    networks:
      - caddy-net
      - authentik-net
    labels:
      caddy: https://${AUTHENTIK_HOSTNAME:?Variable is not set or empty}
      caddy.reverse_proxy: "{{upstreams 9000}}"
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
  worker:
    image: ghcr.io/goauthentik/server:2025.2.4
    restart: unless-stopped
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS:?Variable is not set or empty}
      AUTHENTIK_LOG_LEVEL: warning
    # Removing `user: root` also prevents the worker from fixing the permissions
    # on the mounted folders, so when removing this make sure the folders have the correct UID/GID
    # (1000:1000 by default)
    user: root
    volumes:
      - ./volumes/authentik-media:/media
      - ./volumes/authentik-certs:/certs
      - ./volumes/authentik-custom-templates:/templates
    networks:
      - authentik-net
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy

networks:
  # TODO: Can this be removed?
  caddy-net:
    name: caddy-net
  authentik-net:
    name: authentik-net
```

</details>

## HedgeDoc

<details>

<summary>hedgedoc.yml</summary>

```yml
services:
  hedgedoc:
    # Make sure to use the latest release from https://hedgedoc.org/latest-release
    # TODO:
    image: quay.io/hedgedoc/hedgedoc:1.10.2
    environment:
      # TODO:
      CMD_DB_URL: postgres://hedgedoc:password@database:5432/hedgedoc
      CMD_DOMAIN: ${HEDGEDOC_HOSTNAME:?Variable is not set or empty}
      CMD_URL_ADDPORT: false
      CMD_PROTOCOL_USESSL: true
      CMD_ALLOW_ANONYMOUS: false
      CMD_EMAIL: false
      CMD_LOGLEVEL: warn
      CMD_OAUTH2_PROVIDERNAME: authentik
      # TODO: Use environment variables
      CMD_OAUTH2_CLIENT_ID: ''
      # TODO:
      CMD_OAUTH2_CLIENT_SECRET: ''
      CMD_OAUTH2_SCOPE: "openid email profile"
      CMD_OAUTH2_USER_PROFILE_URL: https://${AUTHENTIK_HOSTNAME:?Variable is not set or empty}/application/o/userinfo/
      CMD_OAUTH2_TOKEN_URL: https://${AUTHENTIK_HOSTNAME:?Variable is not set or empty}/application/o/token/
      CMD_OAUTH2_AUTHORIZATION_URL: https://${AUTHENTIK_HOSTNAME:?Variable is not set or empty}/application/o/authorize/
      CMD_OAUTH2_USER_PROFILE_USERNAME_ATTR: preferred_username
      CMD_OAUTH2_USER_PROFILE_DISPLAY_NAME_ATTR: name
      CMD_OAUTH2_USER_PROFILE_EMAIL_ATTR: email
    volumes:
      - ./volumes/hedgedoc-uploads:/hedgedoc/public/uploads
    restart: unless-stopped
    networks:
      - caddy-net
      - hedgedoc-net
    labels:
      caddy: https://${HEDGEDOC_HOSTNAME:?Variable is not set or empty}
      caddy.reverse_proxy: "{{upstreams 3000}}"
  database:
    image: postgres:13.4-alpine
    environment:
      # TODO: Use environment variables
      - POSTGRES_USER=hedgedoc
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=hedgedoc
    volumes:
      - ./volumes/hedgedoc-db:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - hedgedoc-net

networks:
  hedgedoc-net:
    name: hedgedoc-net
```

</details>
