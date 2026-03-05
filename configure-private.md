To start mastodon from a Docker container, first download Docker if you are in Linux or instal Docker Desktop in Windows. 

Docker Desktop: 

Key Details and Requirements:

    What it is: A comprehensive tool allowing developers to run Linux or Windows containers on Windows, featuring built-in Kubernetes and easy integration with development tools.
    System Requirements:
        OS: Windows 10/11 64-bit, Home/Pro/Enterprise/Education (22H2 or higher).
        Hardware: 64-bit processor with Second Level Address Translation (SLAT).
        RAM: 4GB minimum, though more is recommended.
        Virtualization: Enabled in BIOS/UEFI.
    Backend Technology: WSL 2 (Windows Subsystem for Linux 2) is highly recommended for better performance and speed compared to Hyper-V.
    Installation: Requires Administrator privileges for installation, but can be used by standard users afterward
https://docs.docker.com/desktop/setup/install/windows-install/

The you will have the Docker Engine plus the GUI. 

Clone this repository and run in the directory:

docker compose -f docker-compose-local.yml up

That will download the necesary Images and start the containers. It will take several minutes the first time depending on your machine. 

### Mastodon issues

Mastodon many components are very strict about certificate keys, and that made it more secure.
However that is an issue if we want just explore the code and see how it works without spend on a Domain or a certificate. Then I found this to be a solution to the problem.


Best Solution for Local HTTPS

🥇 Option 1 (Recommended): Use mkcert

This is the cleanest solution.

# What mkcert does:

- Generates a local Certificate Authority (CA)

- Installs it in your system trust store

- Generates trusted local SSL certs

- Your browser will NOT show warnings

# Step 1 — Install mkcert

On Windows (PowerShell):

choco install mkcert

Or download from:
https://github.com/FiloSottile/mkcert

# 2 — Install local CA
localhost+1-key.pem

This generates:

localhost+1.pem
localhost+1-key.pem

------------

🔥 Even Better (Ultra Clean)

Use Caddy instead of Nginx.

Caddy auto-handles HTTPS beautifully.

Example:

caddy:
  image: caddy:2
  ports:
    - "443:443"
  volumes:
    - ./Caddyfile:/etc/caddy/Caddyfile
    - ./localhost+1.pem:/cert.pem
    - ./localhost+1-key.pem:/key.pem

Caddyfile:

https://localhost {
  tls /cert.pem /key.pem
  reverse_proxy web:3000
}

Much simpler than Nginx.


Detailed steps: 

 mkcert -install
 mkcert localhost 127.0.0.1
Created a new local CA 💥
The local CA is now installed in the system trust store! ⚡️
Note: Firefox support is not available on your platform. ℹ️


Created a new certificate valid for the following names 📜
 - "localhost"
 - "127.0.0.1"

The certificate is at "./localhost+1.pem" and the key at "./localhost+1-key.pem" ✅

It will expire on 4 June 2028 🗓





## The Keys problem

Mastodon is running in production mode inside Docker.

Rails expects ENV["SECRET_KEY_BASE"] (or via credentials).

Neither .env.production nor credentials contain a secret_key_base.

Web (puma) and Sidekiq crash immediately because Rails refuses to boot.

Without this, any attempt to generate encryption keys with db:encryption:init will fail, because Rails can’t even load the application to run the task.

How to fix


Option 1 — Generate a secret key manually

Generate a random key:

docker compose run --rm web bin/rails secret

It will output a long string, e.g.:

a7f6c4b8e2f… (96 chars)

Add it to your .env.production:

SECRET_KEY_BASE=a7f6c4b8e2f...

Rebuild/restart Docker Compose:

docker compose -f docker-compose-local.yml up -d

Now Rails can start.










Before run the docker-compose-local, or if they are running  run this:

docker compose -f docker-compose-local.yml down


Then: 

docker compose run --rm web bin/rails db:encryption:init
It will show the next text that has to be copied to .env.production
Off course the key will vary.

# Do NOT change these variables once they are set
ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY=xGC3cy0IyAyCoCS4d3YArUZ5bsxkZwN9
ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT=h8nNktLoEr9tP48b406vAzcEJvRhER60
ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY=R9zeCLo7odZsos6LO7VU7xuoforckZd8


Error database miss tables: 


db-1         | 2026-03-04 21:05:39.467 UTC [136] ERROR:  relation "settings" does not exist at character 523
db-1         | 2026-03-04 21:05:39.467 UTC [136] STATEMENT:  SELECT a.attname, format_type(a.atttypid, a.atttypmod),
db-1         |         pg_get_expr(d.adbin, d.adrelid), a.attnotnull, a.atttypid, a.atttypmod,                                                                                                      
db-1         |         c.collname, col_description(a.attrelid, a.attnum) AS comment,
db-1         |         attidentity AS identity,                                                                                                                                                     
db-1         |         attgenerated as attgenerated
db-1         |    FROM pg_attribute a                                                                                                                                                               
db-1         |    LEFT JOIN pg_attrdef d ON a.attrelid = d.adrelid AND a.attnum = d.adnum
db-1         |    LEFT JOIN pg_type t ON a.atttypid = t.oid                                                                                                                                         
db-1         |    LEFT JOIN pg_collation c ON a.attcollation = c.oid AND a.attcollation <> t.typcollation                                                                                           
db-1         |   WHERE a.attrelid = '"settings"'::regclass
db-1         |     AND a.attnum > 0 AND NOT a.attisdropped                                                                                                                                          
db-1         |   ORDER BY a.attnum
db-1         |                                                                                                                                                                                      
db-1         | 2026-03-04 21:05:40.411 UTC [136] ERROR:  relation "users" does not exist at character 523
sidekiq-1    | PG::UndefinedTable: ERROR:  relation "users" does not exist
sidekiq-1    | LINE 10:  WHERE a.attrelid = '"users"'::regclass

## 💡 Caused by missing tables. Running rails db:setup (or rails db:migrate) inside your web container will fix it. After that, restart the web and Sidekiq containers.

Solution: 


Make sure your database container is running

Check that your db service is up and accessible:

docker-compose ps

You should see db running. If it’s not, start it:

docker-compose up -d db
2️⃣ Create the database

If this is the first time you’re running Mastodon, you need to create the database:

docker-compose run --rm web rails db:setup

This will:

- Create the database

- Load the schema

- Run initial seeds

Tip: If you already have a database but the tables are missing, you might need just:

docker-compose run --rm web rails db:migrate

or:

docker-compose run --rm web rails db:setup

[+]  2/2t 2/22
 ✔ Container mastodon-redis-1 Created                                                                                                                                                          0.2s 
 ✔ Container mastodon-db-1    Running                                                                                                                                                          0.0s 
Container mastodon-web-run-f524dd4e7df7 Creating 
Container mastodon-web-run-f524dd4e7df7 Created 
Database 'postgres' already exists

docker-compose run --rm web rails db:migrate

[+]  2/2t 2/2
 ✔ Container mastodon-redis-1 Running                                                                                                                                                          0.0s 
 ✔ Container mastodon-db-1    Running                                                                                                                                                          0.0s 
Container mastodon-web-run-bc80d8dc3b52 Creating 
Container mastodon-web-run-bc80d8dc3b52 Created 
PS C:\Users\carlo\OneDrive\Desktop\workSearch\LEARNING\svalbard-salg\mastodon> 





CREATE ACCOUNT:

docker compose -f docker-compose-local.yml exec web bin/tootctl accounts create carlo --email carlo@localhost --confirmed

resolve a generated password that has to be copied.

New password: cda02eea6de5...............

Now make that user admin:

docker compose -f docker-compose-local.yml exec web bin/tootctl accounts modify carlo --role Owner


# Role options:

- User

- Moderator

- Admin

- Owner ← full control (recommended)

Now log in at:

https://localhost