# On-Premise Deployment

An on-premise deployment of CoreUpdate is a self-administered instance that can be run behind a firewall.

## Accessing the CoreUpdate Container

After signing up you will have received a `.dockercfg` file containing your credentials to the `quay.io/coreos/coreupdate` repository.
Save this file to your CoreOS machine in `/home/core/.dockercfg` and `/root/.dockercfg`.
You should now be able to execute `docker pull quay.io/coreos/coreupdate` to download the container.

## Database Server

CoreUpdate requires an instance of a Postres database server.
You can use an existing instance if you have one, or use [the official Postgres docker image](https://registry.hub.docker.com/_/postgres/).

Postgres can be run on CoreOS with a systemd unit file similar to this one: 
```
[Service]
User=core
ExecStartPre=-/usr/bin/docker kill postgres
ExecStartPre=-/usr/bin/docker rm postgres
ExecStart=/usr/bin/docker run --rm --name postgres \
    -v /opt/coreupdate/postgres/data:/var/lib/postgresql/data \
    --net="host" \
    postgres:9.4
ExecStop=/usr/bin/docker kill postgres

[Install]
WantedBy=multi-user.target
```

It is recommended to mount a volume from your host machine for data storage.
The above example uses `/opt/coreupdate/postgres/data`.

Start the Postgres service by running:  
```bash
sudo cp postgres.service /etc/systemd/system
sudo systemctl start postgres.service
```

View the logs and verify it is running:  
```bash
sudo journalctl -u postgres.service -f
```

CoreUpdate needs a database to connect to, so you may need to initialize a new database on the Postgres server.
You can do this manually, or execute a similar command using another instance of the Postgres container:  

```bash
docker run --net="host" postgres:9.4 bash -c "\
  psql -h localhost -U postgres --command \"CREATE USER coreos WITH SUPERUSER 'coreos';\" && \
  psql -h localhost -U postgres --command \"CREATE DATABASE coreupdate OWNER coreos;\""
```

The username, password, and database name can be anything you choose as long as they match the `DB_URL` field in the config file.

## Web Service

Once your database server is configured and running properly you can configure the web service.

### Configuration File

All CoreUpdate configuration options can be stored in a `.yaml` file.
You will need to save this somewhere on your host machine such as `/etc/coreupdate/config.yaml`.

Below is a configuration file template. Customize the values as needed:  

```yaml
# Published base URL of the web service.
# Required if using DNS, Load Balancer, or http->https redirections.
BASE_URL: http://localhost:8000

# (required) Unique secret session string.
# You can generate a UUID from the command line using the `uuidgen` command
SESSION_SECRET: "a-long-unique-string"

# Set this to 'false' if using Google authentication.
DISABLE_AUTH: true

# Enables Google OAuth, otherwise set DISABLE_AUTH to 'true'
# Configure at https://console.developers.google.com
#GOOGLE_OAUTH_CLIENT_ID:
#GOOGLE_OAUTH_CLIENT_SECRET:
#GOOGLE_OAUTH_REDIRECT_URL:

# Address and port to listen on.
LISTEN_ADDRESS: ":8000"

# Postgres database settings.
# Format: postgres://username:password@host:port/database-name
DB_URL: "postgres://coreos:coreos@localhost:5432/coreupdate?sslmode=disable"
DBTIMEOUT: 0
DBMAXIDLE: 0
DBMAXACTIVE: 100

# (Optional) sets a path to enable CoreUpdate's static package serving feature.
# Comment out to disable.
#STATIC_PACKAGES_DIR: /packages

# (Optional) enables uploading of package payloads to the server.
#ENABLE_PACKAGE_UPLOADS: true

# (Optional) Enable if syncing with upstream CoreUpdate instances.
# Zero value is disabled.
# This should be disabled if you plan to synchronize packages manually.
UPSTREAM_SYNC_INTERVAL: 0

# (Optional) enables TLS
#TLS_CERT_FILE:
#TLS_KEY_FILE:
```

#### Package Payload Hosting

By default the CoreUpdate database only stores meta-data about application packages.
This enables you to host the package payloads using the file storage technology of your choice.

If you prefer you can store and serve package payloads from the same machine the CoreUpdate web service is running on.
To do so ensure the following settings exist in your configuration file:  

```bash
STATIC_PACKAGES_DIR: /packages
ENABLE-PACKAGE-UPLOADS: true
```

### Initializing the Application

The CoreUpdate web service can be run with a systemd unit file such as:  

```
[Service]
User=core
ExecStartPre=-/usr/bin/docker kill coreupdate-%i
ExecStartPre=-/usr/bin/docker rm coreupdate-%i
ExecStart=/usr/bin/docker run --net="host" --rm --name coreupdate-%i \
    # mount the location of the config file
    -v /etc/coreupdate:/etc/coreupdate \
    # (optional) mount the location of the package payload directory
    #-v /opt/packages:/packages \
    --net="host" \
    # container to run
    quay.io/coreos/coreupdate:latest \
    # binary inside the container to execute
    /opt/coreupdate/bin/coreupdate \
    # path to configuration file
    --yaml=/etc/coreupdate/config.yaml
ExecStop=/usr/bin/docker kill coreupdate-%i

[Install]
WantedBy=multi-user.target

[X-Fleet]
X-Conflicts=coreupdate@*
```

Start the service by running:  
```bash
sudo cp coreupdate@.service /etc/systemd/system
sudo systemctl start coreupdate@.service
```

View the logs and verify it is running:  
```bash
sudo journalctl -u coreupdate@.service -f
```

#### Create Admin Users

Now that the server is running the first user must be initialization.
Do this using the `updateservicectl` tool.

This will generate an `admin` user and an api `key`, make note of the key for subsequent use of `updateservicectl`.  
```bash
updateservicectl --server=http://localhost:8000 database init
```

Create the first control panel user:  
```bash
updateservicectl --server=http://localhost:8000 --user=admin --key=<previously-generated-key> admin-user create google.apps.email@example.com
```

#### Create the "CoreOS" Application

To sync the "CoreOS" application it must exist and have the same application id as the public CoreUpdate instance.
NOTE: the application id must match exactly what is listed here:  
```bash
updateservicectl --server=http://localhost:8000 --user=admin --key=<previously-generated-key> app create --label=CoreOS --app-id=e96281a6-d1af-4bde-9a0a-97b76e56dc57
```

You can now point your browser to `http://localhost:8000` to view the control panel.
