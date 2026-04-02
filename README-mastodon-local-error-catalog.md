# Mastodon Local Installation -- Error Catalog

This document lists **common errors encountered during experimental
local installation of Mastodon using Docker without federation**, along
with their **causes and solutions**.

The goal is to help developers quickly diagnose problems when running
Mastodon locally.

------------------------------------------------------------------------

## Table of Contents

1.  [Yarn Version Mismatch (Corepack Required)](#1-yarn-version-mismatch)
2.  [Husky Pre-Commit Hook Failure](#2-husky-pre-commit-hook-failure)
3. [HTTP Request Sent to HTTPS Server](#3-http-request-sent-to-https-server)
4.  [Caddy Certificate Path Error](#4-caddy-certificate-path-error)
5.  [Missing Local TLS Certificates](#5-missing-local-tls-certificates)
6.  [Puma SSL Parsing Error](#6-puma-ssl-parsing-error)
7.  [Missing Rails Secret Key](#7-missing-rails-secret-key)
8.  [Missing ActiveRecord Encryption Keys](#8-missing-activerecord-encryption-keys)
9.  [Missing Database Tables](#9-missing-database-tables)
10. [Cannot Access Local Web Server](#10-cannot-access-local-web-server)
11. [Caddy Warning About Unnecessary Headers](#11-caddy-warning-about-unnecessary-headers)
12. [GitHub Actions CI Failures](#12-github-actions-ci-failures)

------------------------------------------------------------------------

# 1. Yarn Version Mismatch 
### Error
(Corepack Required)



    error This project's package.json defines "packageManager": "yarn@4.12.0".
    However the current global version of Yarn is 1.22.22.

    Presence of the "packageManager" field indicates that the project is meant
    to be used with Corepack.

    Corepack must currently be enabled by running corepack enable

### Cause

Mastodon uses **Yarn 4**, but the system had **Yarn 1.x installed
globally**.

Recent Node versions rely on **Corepack** to manage package manager
versions.

### Solution
 Enable Corepack: 

 ~~~Bash 
corepack enable
 ~~~


------------------------------------------------------------------------

# 2. Husky Pre Commit Hook Failure

### Error

    husky - pre-commit script failed (code 1)

Often followed by:

    oxfmt --no-error-on-unmatched-pattern [SIGKILL]

and

    'bin' is not recognized as an internal or external command
    operable program or batch file.

### Cause

The repository includes **pre-commit hooks** that run formatting and
linting tools. Some tools require a **Linux environment** and may fail
when running from Windows.

### Solution

For development forks or documentation repositories, these hooks can be bypassed:
 ~~~Bash 
git commit --no-verify
 ~~~

Or disable the hooks completely:

 ~~~Bash 
rm -rf .husky
 ~~~


    

------------------------------------------------------------------------

# 3. HTTP Request Sent to HTTPS Server

### Error

    Client sent an HTTP request to an HTTPS server.

### Cause

Caddy was configured for **HTTPS**, but the browser accessed the server
using **HTTP**.

### Solution

Use the correct protocol:

    https://localhost

instead of:

    http://localhost

------------------------------------------------------------------------

# 4. Caddy Certificate Path Error

### Error
Caddy container logs:

    Error: loading initial config: loading new config:
    loading tls app module: provision tls: loading certificates:
    read /cert.pem: is a directory

### Cause

Docker mounted a **directory instead of a certificate file**.  
This happened because the certificate files had been deleted but the mount path still existed.    
Example incorrect mount:


~~~YAML 
- ./localhost+1.pem:/cert.pem
 ~~~


  If the file does not exist, Docker creates a directory instead.

### Solution

Recreate the certificates and mount them correctly.


------------------------------------------------------------------------

# 5. Missing Local TLS Certificates

After removing the certificate files, **Caddy cannot start**.

### Solution
Generate local trusted certificates using mkcert.  
#### Install mkcert:

    choco install mkcert

Or download:

https://github.com/FiloSottile/mkcert

#### Install the local certificate authority:

    mkcert -install

#### Generate certificates:

    mkcert localhost 127.0.0.1

This generates:

    localhost+1.pem
    localhost+1-key.pem


These files are then mounted in Docker:

~~~YAML 
    volumes:
      - ./certs/localhost+1.pem:/cert.pem
      - ./certs/localhost+1-key.pem:/key.pem
 ~~~


------------------------------------------------------------------------

# 6. Puma SSL Parsing Error

### Error

    Puma::HttpParserError:
    Invalid HTTP format, parsing fails.
    Are you trying to open an SSL connection to a non-SSL Puma?

### Cause

HTTPS traffic was sent directly to **Puma**, which only expects
**HTTP**.  
Puma runs inside the web container and should only receive internal HTTP traffic.

### Solution
Ensure **Caddy terminates HTTPS** and forwards HTTP internally.

Correct architecture:

    Browser
       ↓ HTTPS
    Caddy
       ↓ HTTP
    Puma (web container)

------------------------------------------------------------------------

# 7. Missing Rails Secret Key
### Error

Rails containers failed to start due to missing secret configuration.

### Cause

Mastodon running in **production mode** requires:

    SECRET_KEY_BASE

Without it, Rails refuses to boot.

### Solution

Generate:

~~~Bash 
docker compose run --rm web bin/rails secret
 ~~~
    

Add to `.env.production`:

    SECRET_KEY_BASE=generated_secret_key_here

Restart the containers:

    docker compose up -d

------------------------------------------------------------------------

# 8. Missing ActiveRecord Encryption Keys

### Error

Rails could not initialize encryption.

### Cause

Mastodon requires database encryption keys in production.

### Solution

Generate them:

~~~Bash 
docker compose run --rm web bin/rails db:encryption:init
 ~~~

    

Add the output to `.env.production`:

    ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY=...
    ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT=...
    ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY=...

------------------------------------------------------------------------

# 9. Missing Database Tables

### Error

PostgreSQL logs:

    ERROR: relation "users" does not exist
    ERROR: relation "settings" does not exist

    
Sidekiq logs:  

~~~Bash 
PG::UndefinedTable: ERROR: relation "users" does not exist
 ~~~


### Cause

Database created but **migrations not executed**.

### Solution

Run database setup:

    docker compose run --rm web rails db:setup

or run migrations:

    docker compose run --rm web rails db:migrate

------------------------------------------------------------------------

# 10. Cannot Access Local Web Server

### Error

    Firefox can’t establish a connection to the server at 127.0.0.1:3000

### Cause

The Mastodon web server is not intended to be accessed directly.

It must be accessed through the reverse proxy (Caddy).

### Solution

Use:

    https://localhost

instead of

    http://127.0.0.1:3000

------------------------------------------------------------------------

# 11. Caddy Warning About Unnecessary Headers

### Warning

    Unnecessary header_up X-Forwarded-For
    Unnecessary header_up X-Forwarded-Proto

### Cause

Caddy automatically forwards these headers by default.

### Solution

Remove redundant header configuration.

------------------------------------------------------------------------

# 12. GitHub Actions CI Failures

### Error

    Ruby Testing: Some jobs were not successful
    End to End testing failed

### Cause

The upstream Mastodon repository contains extensive CI pipelines, including browser tests and multiple Ruby environments.

### Solution

Disable the workflow or run it manually.

------------------------------------------------------------------------

## Resulting Working Architecture

Running containers:

    mastodon-web-1
    mastodon-sidekiq-1
    mastodon-streaming-1
    mastodon-db-1
    mastodon-redis-1
    mastodon-caddy-1

Access Mastodon via:

    https://localhost

Caddy handles HTTPS and forwards traffic internally.
