# rTorrent + ruTorrent Docker Setup with Caddy

This guide provides a comprehensive walkthrough for setting up **rTorrent + ruTorrent** using Docker, with **Caddy** as a reverse proxy. This setup includes a web UI, CLI access, Basic Authentication, and the ability to mount host folders for downloads, movies, and configuration/cache.

Once set up, you can access the ruTorrent web UI via:

-   **Local LAN:** `http://<HOST_IP>:8085`
-   **Domain:** `https://<YOUR_DOMAIN>` (optional, if you configure HTTPS)

---

## Getting Started

This setup is designed to be straightforward. Here's a high-level overview of the steps:

1.  **Prerequisites:** Ensure you have Docker and Docker Compose installed on your system.
2.  **Folder Structure:** Create the necessary folder structure for your setup.
3.  **Environment Variables:** Create a `.env` file and configure the necessary environment variables, such as your username and password for Basic Authentication.
4.  **Docker Compose:** Configure your `docker-compose.yml` file to define the rTorrent and Caddy services.
5.  **Caddyfile:** Configure your `Caddyfile` to set up the reverse proxy and Basic Authentication.
6.  **Start the Stack:** Run `docker compose up -d` to start the services.
7.  **Access ruTorrent:** Access the ruTorrent web UI in your browser.

---

## Folder Structure

Create the following folder structure on your host machine:

```text
~/docker/rutorrent/
├─ docker-compose.yml
├─ Caddyfile
├─ .env
├─ config/           # rTorrent config (from host, optional)
├─ cache/            # rTorrent cache (from host, optional)
├─ downloads/        # Mounted downloads folder
├─ Movies/           # Mounted movies folder
├─ caddy_data/       # Caddy persistent data
├─ caddy_config/     # Caddy config
└─ watch/            # Folder for auto-loading torrents (optional)
```

---

## 1. Environment Variables

Create a `.env` file in the `~/docker/rutorrent/` directory. This file will store your sensitive information, such as your username and password for Basic Authentication.

```
BASIC_USER=<YOUR_USERNAME>
BASIC_HASH=<YOUR_BCRYPT_HASH>
```

**Important:**

-   Replace `<YOUR_USERNAME>` with your desired username.
-   Replace `<YOUR_BCRYPT_HASH>` with a B-crypt hash of your password.
-   To generate a B-crypt hash, you can use the following command:

    ```
    docker compose run --rm caddy caddy hash-password --plaintext '<YOUR_PASSWORD>'
    ```

-   When you generate the hash, you will get a string that looks like this: `$2a$14$jW/YJRXe.ANGPxXwsqzWv.7eIz7o08n.rciV/mRIZktL3mwpivHOK`.
-   You need to **double all the `$` characters** in the hash before adding it to your `.env` file. For example, the hash above would become:

    ```
    $$2a$$14$$jW/YJRXe.ANGPxXwsqzWv.7eIz7o08n.rciV/mRIZktL3mwpivHOK
    ```

---

## 2. Docker Compose

Create a `docker-compose.yml` file with the following content:

```yaml
services:
  rutorrent:
    image: lscr.io/linuxserver/rutorrent:latest
    container_name: rutorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Kathmandu
      - UMASK=002
    volumes:
      - ./downloads:/downloads
      - ./Movies:/Movies
      - ./config:/config
      - ./cache:/cache
    ports:
      - "8085:80"
    restart: unless-stopped

  caddy:
    image: caddy:latest
    container_name: caddy-rutorrent
    depends_on:
      - rutorrent
    ports:
      - "8085:8085"
      # - "443:443" # Uncomment if you want to use HTTPS
    environment:
      - RUTORRENT_DOMAIN=<YOUR_DOMAIN> # Replace with your domain
      - BASIC_USER=${BASIC_USER}
      - BASIC_HASH=${BASIC_HASH}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy_data:/data
      - ./caddy_config:/config
    restart: unless-stopped
```

**Notes:**

-   Make sure to replace `<YOUR_DOMAIN>` with your actual domain name if you want to use HTTPS.
-   The `volumes` section maps the folders you created on your host machine to the corresponding folders in the Docker containers.
-   The `ports` section maps the ports on your host machine to the ports in the Docker containers.

---

## 3. Caddyfile

Create a `Caddyfile` with the following content:

### For Local LAN Access (HTTP)

```
:8085 {
    encode zstd gzip
    basicauth {
        {$BASIC_USER} {$BASIC_HASH}
    }
    reverse_proxy rutorrent:80
}
```

### For Domain Access (HTTPS)

```
<YOUR_DOMAIN> {
    encode zstd gzip
    basicauth {
        {$BASIC_USER} {$BASIC_HASH}
    }
    reverse_proxy rutorrent:80
}
```

**Notes:**

-   If you are using a domain, make sure to replace `<YOUR_DOMAIN>` with your actual domain name.
-   The `basicauth` directive enables Basic Authentication, which will prompt you for a username and password when you access the ruTorrent web UI.
-   The `reverse_proxy` directive forwards all requests to the rTorrent container.

---

## 4. Starting the Stack

To start the rTorrent and Caddy services, run the following command:

```
docker compose up -d
```

This will start the containers in detached mode, which means they will run in the background.

To check the logs for the containers, you can use the following commands:

```
docker logs rutorrent
docker logs caddy-rutorrent
```

---

## 5. Accessing ruTorrent

Once the containers are running, you can access the ruTorrent web UI in your browser:

-   **Local LAN:** `http://<HOST_IP>:8085`
-   **Domain:** `https://<YOUR_DOMAIN>`

You will be prompted for the username and password that you configured in your `.env` file.

---

## 6. rTorrent CLI Access

You can access the rTorrent command-line interface (CLI) by running the following command:

```
docker exec -it rutorrent bash
rtorrent -n -o import=/config/rtorrent/rtorrent.rc
```

**Warning:** Only run one instance of rTorrent at a time, as multiple instances can cause issues with your session files.

---

## 7. Troubleshooting

-   **Permissions Issues:** If you encounter any permissions issues, make sure that the folders you created on your host machine are writable by the user with UID `1000` and GID `1000`.
-   **Incorrect Password:** If you are unable to log in, double-check that you have correctly configured your username and password in the `.env` file.
-   **HTTPS Issues:** If you are having trouble with HTTPS, make sure that port `443` is not being used by another application on your host machine.

---

## 8. Stopping and Restarting

To stop the containers, run the following command:

```
docker compose down
```

To restart the containers, run the following command:

```
docker compose up -d
```


To add new user
add this to caddy file
:8085 {
    encode zstd gzip
    basicauth {
        romin {$BASIC_HASH}
        alice $$2a$$14$$<Alice_Hash_Here>
    }
    reverse_proxy rutorrent:80
}

Generate a bcrypt hash for the new user

docker compose run --rm caddy caddy hash-password --plaintext 'alicepassword'

docker compose restart caddy



