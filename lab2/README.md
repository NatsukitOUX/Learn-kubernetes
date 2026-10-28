# Lab 2 — Docker Build & Run

## What this lab was about

Build a custom Docker image from a `Dockerfile` and run a named container from it with port mapping so the app responds on `localhost:8081`. The application is a minimal PHP page served by Apache.

**Dockerfile:**
```dockerfile
FROM php:7.4-apache
WORKDIR /var/www/html
COPY . /var/www/html
EXPOSE 80
CMD ["apache2-foreground"]
```

**index.php:**
```php
<?php
echo "Hello, World!";
?>
```

---

## Errors & How I Fixed Them

### ❌ Error 1 — Running the image before building it

**Command run:**
```bash
sudo docker run lab2 .
```

**Error message:**
```
Error: initializing source docker://lab2:latest: reading manifest
latest in docker.io/library/lab2: requested access to the resource is denied
```

**Root cause:**  
Docker tried to pull `lab2` from Docker Hub (`docker.io/library/lab2`) because the image didn't exist locally yet — no image had been built. Additionally, the `.` at the end is wrong for `docker run`; it belongs only in `docker build`.

**✅ Fix — Build first, then run:**
```bash
# Step 1: build the image
docker build -f Dockerfile -t lab2 .

# Step 2: run the container
docker run -d --name lab2 -p 8081:80 lab2
```

---

### ❌ Error 2 — Port mapping was wrong / wrong image running

After stopping and removing a previous `lab2` container, curling `localhost:8081` returned:
```
curl: (7) Failed to connect to localhost port 8081: Could not connect to server
```

**Root cause:**  
The previous `lab2` container was actually running from the `nginx` image, and the port mapping was reversed: `0.0.0.0:46099->8081/tcp` meant host port 46099 mapped to container port 8081 — the opposite of what was intended.

**✅ Fix — Stop, remove, and re-run with the correct image and correct port order:**
```bash
docker stop lab2
docker rm lab2
docker run -d --name lab2 -p 8081:80 lab2
```

> Port mapping syntax: `-p HOST_PORT:CONTAINER_PORT`  
> Apache inside the container listens on **80**, so `8081:80` = host 8081 → container 80.

---

## What I Learned

| Concept | Notes |
|---|---|
| `docker build` vs `docker run` | `build` creates an image from a Dockerfile. `run` starts a container from an existing image. You must build first. |
| Port mapping syntax | `-p HOST:CONTAINER` — host port comes first. Getting it backwards means `curl` can't reach the service. |
| Local vs registry images | Docker tries to pull unknown image names from Docker Hub. Build locally with `-t name` to use a local image. |
| Named containers | `--name lab2` makes `stop`/`rm`/`inspect` much easier than using random names like `angry_antonelli`. |
| `-d` flag (detached) | Runs the container in the background so the terminal stays free. |
| Grade script checks | The lab verifies: image built ✓, container `lab2` running ✓, port 8081 listening ✓ |

---

## Commands Reference

| Command | What it does |
|---|---|
| `docker build -f Dockerfile -t lab2 .` | Build an image named `lab2` from the Dockerfile in the current directory |
| `docker run -d --name lab2 -p 8081:80 lab2` | Start a detached container, mapping host 8081 → container 80 |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (including stopped ones) |
| `docker stop lab2` | Gracefully stop the named container |
| `docker rm lab2` | Remove the stopped container |
| `curl localhost:8081` | Test that the app is reachable on the mapped host port |

---

## Result

```
[OK]   Image creation successful
[OK]   Container lab2 is running
[OK]   Port 8081 on localhost is listening

[CONGRATS] You have successfully completed this lab!
```
