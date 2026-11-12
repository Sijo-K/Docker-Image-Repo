AK PRINTERS - RUN THE EXPORTED DOCKER IMAGES
============================================================

Image files:
  Desktop\AKPrintersDockerImages\akprinters-frontend-local.tar
  Desktop\AKPrintersDockerImages\akprinters-backend-v2.tar

These are application images. PostgreSQL is a separate container.
The image files do NOT include your existing products, database, or
uploaded product images. The first-time setup below creates a fresh
database. Keep the existing containers and volumes if you need their
current data. Never run "docker volume rm" or "docker compose down -v"
unless you intentionally want to permanently delete saved data.

FIRST-TIME SETUP (PowerShell)
-----------------------------

1. Load both application images:

docker load -i "$HOME\Desktop\AKPrintersDockerImages\akprinters-frontend-local.tar"
docker load -i "$HOME\Desktop\AKPrintersDockerImages\akprinters-backend-v2.tar"

2. Create a private Docker network and persistent data volumes:

docker network create akprinters-net
docker volume create akprinters-postgres
docker volume create akprinters-media

3. Start PostgreSQL:

docker run -d --name akprinters-db `
  --network akprinters-net --network-alias database `
  -e POSTGRES_DB=akprinters `
  -e POSTGRES_USER=akprinters `
  -e POSTGRES_PASSWORD=local-only-change-me `
  -v akprinters-postgres:/var/lib/postgresql/data `
  --restart unless-stopped `
  postgres:16-alpine

4. Wait for PostgreSQL to be ready:

$ready = $false
for ($i = 0; $i -lt 30; $i++) {
  docker exec akprinters-db pg_isready -U akprinters -d akprinters
  if ($LASTEXITCODE -eq 0) { $ready = $true; break }
  Start-Sleep -Seconds 2
}
if (-not $ready) { throw "PostgreSQL did not become ready. Check: docker logs akprinters-db" }

5. Start the Django backend:
   Replace the example secret with a private, long random value.

docker run -d --name akprinters-backend `
  --network akprinters-net `
  -p 127.0.0.1:8012:8000 `
  -e "DJANGO_SECRET_KEY=replace-this-with-a-long-random-secret" `
  -e DJANGO_DEBUG=true `
  -e "DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1" `
  -e DB_NAME=akprinters `
  -e DB_USER=akprinters `
  -e DB_PASSWORD=local-only-change-me `
  -e DB_HOST=database `
  -e DB_PORT=5432 `
  -v akprinters-media:/app/media `
  --restart unless-stopped `
  akprinters-backend:v2

The backend runs database migrations and collects static files when it
starts. Check its startup with:

docker logs -f akprinters-backend

Press Ctrl+C to stop following the logs; this does not stop the container.

6. Start the frontend:

docker run -d --name akprinters-frontend `
  --network akprinters-net `
  -p 8080:80 `
  -v akprinters-media:/app/media:ro `
  --restart unless-stopped `
  akprinters-frontend:local

7. Open the website:

  Storefront:    http://localhost:8080
  Staff panel:   http://localhost:8080/admin-panel
  Django admin:  http://localhost:8080/admin/
  Backend API:   http://127.0.0.1:8012

Create a staff login if the fresh database has no staff account:

docker exec -it akprinters-backend python manage.py createsuperuser

CHECK CONTAINERS
----------------

docker ps
docker logs --tail 100 akprinters-backend
docker logs --tail 100 akprinters-frontend

START THEM AGAIN LATER
----------------------

The database and uploaded media persist in the named volumes. After
restarting the computer or Docker Desktop, start the existing containers:

docker start akprinters-db
docker start akprinters-backend
docker start akprinters-frontend

If a container already exists, do not repeat its "docker run" command;
use "docker start <container-name>" instead.

IMPORTANT: EXISTING PRODUCT DATA
--------------------------------

Docker image archives contain application code, not the database or
uploaded files from a running container. The existing products are stored
in the existing backend container's database, and their photos are stored
in its media directory. The fresh PostgreSQL setup above will not contain
those existing products unless the database and media are separately
backed up and restored. Do not remove the old containers before backing
up data you want to keep.
