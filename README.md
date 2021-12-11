# synapse-element-chat-docker

Start your own self-hosted, privacy-respectful, end-to-end encrypted chat and call application.

- Web app (included)
- Mobile app ([Google Play](https://play.google.com/store/apps/details?id=im.vector.app) or [build from source](https://github.com/vector-im/element-android))
- Desktop app ([download](https://element.io/get-started) or [build from source](https://github.com/vector-im/element-desktop))

## Install

### Initialization

Generating fresh config file and certificates (edit `SYNAPSE_SERVER_NAME` according to your configuration below) :

```bash
docker-compose run --rm -e SYNAPSE_SERVER_NAME=localhost -e SYNAPSE_REPORT_STATS=no synapse generate

# Temporary changing permissions for editing conf files
sudo chown -R $USER ./synapse_data
sudo chmod -R 755 ./synapse_data
```

3 files are generated under `synapse_data/` :

- `homeserver.yaml` : synapse global config (including SSL)
- `localhost.log.config` : logging strategy
- `localhost.signing.key` : signing key

**Edit** the following line as-is in `localhost.log.config` :

- `filename: /data/homeserver.log`

**Uncomment and edit** the following line in `homeserver.yaml` :

- `enable_registration: true` (you may want to disable it later)

**Edit** the `config.localhost.json` filename if needed by your hostname in `docker-compose.yml` :

```yml
  volumes:
    - ./element-config.json:/app/config.localhost.json:ro
```

### SSL certificates

We're going to enable SSL certificates

```bash
openssl genrsa -out ./synapse_data/localhost.tls.key 2048
openssl req -new -x509 -sha256 -days 1095 -subj "/C=FR/ST=IDF/L=PARIS/O=EXAMPLE/CN=Synapse" -key ./synapse_data/localhost.tls.key -out ./synapse_data/localhost.tls.crt
```

In `./synapse_data/homeserver.yaml`, uncomment the following lines :

- `tls_certificate_path: "/data/localhost.tls.crt"`
- `tls_private_key_path: "/data/localhost.tls.key"`

**Remove** these lines :

```yml
trusted_key_servers:
  - server_name: "matrix.org"

- port: 8008
  tls: false
  type: http
  x_forwarded: true
  
  resources:
    - names: [client, federation]
      compress: false
```

**Add** these lines and edit accordingly with `./synapse_data/localhost.signing.key` content :

  ```yml
  trusted_key_servers:
    - server_name: "localhost"
      verify_keys:
        "ed25519:xxxxxxx": xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  ```

  ```yml
  listeners:
    - port: 8448
      type: http
      tls: true
      resources:
        - names: [client, federation]
  ```

  ```yml
  # If using multiple Synapse instances
  trusted_key_servers:
    - server_name: "localhost"
    - server_name: "mydomain2"
    - server_name: "mydomain3"
  ```

**Edit** the SSL certificates lines accordingly to your host in `./nginx/nginx.conf` :

```conf
ssl_certificate     /synapse_data/localhost.tls.crt;
ssl_certificate_key /synapse_data/localhost.tls.key;
```

### Postgres configuration

**Remove** these lines :

```yml
database:
  name: sqlite3
  args:
    database: /data/homeserver.db
```

**Add** these lines :

```yml
database:
  name: psycopg2
  args:
    user: synapse_user
    password: synapse_password
    database: synapse
    host: database
    cp_min: 5
    cp_max: 10
```

### Start

```bash
# Final conf file permissions
sudo chmod -R 711 ./synapse_data
sudo chown -R 991 ./synapse_data

docker-compose up -d
```

Add a user as server admin :

```bash
docker-compose exec synapse register_new_matrix_user -u <USERNAME> -p <PASSWORD> -a https://localhost:8448 -c /data/homeserver.yaml
```

Access the UI at `localhost:8448`
