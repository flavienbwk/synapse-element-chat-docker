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

In `localhost.log.config` edit the following line as-is :

- `filename: /data/homeserver.log`

In `homeserver.yaml`, uncomment and edit the following line :

- `enable_registration: true` (you may want to disable it later)

### SSL certificates

We're going to enable SSL certificates

In `homeserver.yaml`, uncomment the following lines :

- `tls_certificate_path: "/data/localhost.tls.crt"`
- `tls_private_key_path: "/data/localhost.tls.key"`

Then let's generate SSL certificates :

```bash
openssl genrsa -out ./synapse_data/localhost.tls.key 2048
openssl req -new -x509 -sha256 -days 1095 -subj "/C=FR/ST=IDF/L=PARIS/O=EXAMPLE/CN=Synapse" -key ./synapse_data/localhost.tls.key -out ./synapse_data/localhost.tls.crt
```

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

**Add** these lines and edit accordingly with `localhost.signing.key` content :

```yml
trusted_key_servers:
  - server_name: "localhost"
    verify_keys:
      "ed25519 xxxxxxx xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
- port: 8448
  type: http
  tls: true
  resources:
    - names: [client, federation]
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

Access the UI at `localhost:8448`
