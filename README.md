# Stremio-Server-Raspberry-PI
Here is a guide on setting up a streaming server for Stremio which is accessible on the local network and the PI itself. This works should work for pretty much any device with web browser.

## 1. Install Docker
You will need docker for this as it allows us to host `stremio-server`
https://docs.docker.com/engine/install/debian/#install-using-the-repository

```bash
## Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

## Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Now install the required `docker-engine` packages:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Now test the installation with hello world example:

```
sudo docker run hello-world
```

If the output shows "Hello World" then you're good to continue.

## 2. Install Docker Compose
Upon installing docker engine and testing it works with the "Hello World" project you can now install `docker-compose`
```bash
sudo apt-get install docker-compose
```

## 3. Creating `docker-compose.yaml`
You may want to modify the below definition to not use reverse proxy. 
```yaml
version: '3.8'
services:
  stremio-server:
    image: stremio/server:latest
    ports:
      - "11470:11470" # http
      # - "12470:12470" # https UNCOMMENT IF NOT USING NGINX
    environment:
      - NO_CORS=1
      - APP_PATH=/app/config
    volumes:
      - ./stremio:/app/config
    restart: unless-stopped

  ## You can ignore this if you don't need clients with HTTPS support unlike apple devices
  nginx:
    image: nginx:latest
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro # Map your SSL certificate directory
    ports:
      - "443:443"
    depends_on:
      - stremio-server
    restart: unless-stopped
```

## 4. Setting up `stremio-server`
You will want to create a new folder relative to your `docker-compose.yaml` file called `stremio`. In this folder we will create a new file called `server-settings.json`. You can modify this configuration as required.

```json
{
    "serverVersion": "4.20.5",
    "appPath": "/app/config",
    "cacheRoot": "/app/config",
    "cacheSize": 2147483648,
    "btMaxConnections": 400,
    "btHandshakeTimeout": 25000,
    "btRequestTimeout": 6000,
    "btDownloadSpeedSoftLimit": 8388608,
    "btDownloadSpeedHardLimit": 78643200,
    "btMinPeersForStable": 10,
    "remoteHttps": "{IF NO NGINX HOSTNAME OR IP OTHERWISE LEAVE IT BLANK}", 
    "localAddonEnabled": false,
    "transcodeHorsepower": 0.75,
    "transcodeMaxBitRate": 0,
    "transcodeConcurrency": 1,
    "transcodeTrackConcurrency": 1,
    "transcodeHardwareAccel": true,
    "transcodeProfile": true,
    "transcodeMaxWidth": 1920,
    "btProfile": "ultra_fast"
}
```

## 5. Setting up `nginx`
This part is optional and is only required if you are going to use `stremio-server` on devices that require a HTTPS connection for example Apple devices. You will need to update the `sever_name` field.

```R
events {}

http {
    server {
        listen 443 ssl;
        server_name {HOSTNAME OR IP};

        ssl_certificate /etc/nginx/certs/server.crt;
        ssl_certificate_key /etc/nginx/certs/server.key;

        location / {
            proxy_pass http://stremio-server:11470;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}

```

The next part of this step is to generate the certificates in the `nginx/certs` folder relative to your `docker-compose.yaml` file. You can generate a self-signed certificate using the openssl command:

```bash
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout server.key -out server.crt
```

**IMPORANT:** It's important that you set the Common Name (FQDN) to `*` otherwise you may run into issues.
## 6. Pre-flight Checks
Before we go ahead and run our compose file, validate your directory looks like this:

```bash
stremio-server
	docker-compose.yaml
	stremio/
		server-settings.json
	nginx/
		nginx.conf
		certs/
			server.crt
			server.key			
```

## 7. Up the docker compose

Now we're ready to bring the compose stack up:

```bash
sudo docker-compose up -d
```

**IMPORTANT:** If you're having any issues remove the `-d` of the command to get the output from both `stremio-server` and `nginx`.

## 8. Validate it's working
You can now navigate to your `https://{HOSTNAME OR IP}:12470/` and you will be redirected to the stremio app. 

**IMPORTANT:** When the redirect happens it will set the streaming server to `http` you will need to edit the URL and change it to `https://{HOSTNAME OR IP}:12470/`. Don't forget to re-enter the port. You can then bookmark this link for ease of use later.

Everything should now work, apart from casting.

## Further improvements:
1. Implement Let's Encrypt so the self-signed certificate errors are not annoying.
2. Support for hardware encoding
3. Host `stremio-web` inside of the stack

