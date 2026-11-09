# My Flask App

A flasky app.

## Docker Quickstart

This app can be run completely using `Docker` and `docker compose`. **Using Docker is recommended, as it guarantees the application is run using compatible versions of Python and Node**.

There are three services: `flask-dev` (development), `flask-prod` (production mode), and `manage` (Flask CLI commands).

### First installation (Windows / PowerShell)

Run all commands from the project root, with Docker Desktop running. Continue to the next command only if the previous command succeeds.

Before starting any container, prepare the local configuration and SQLite file:

```powershell
if (-not (Test-Path -LiteralPath .env)) { Copy-Item .env.example .env }
if (-not (Test-Path -LiteralPath dev.db)) { New-Item -ItemType File -Path .\dev.db }
```

Keep an existing `.env`: it may contain your own configuration. The default `DATABASE_URL=sqlite:////tmp/dev.db` refers to the path inside Docker.

If `dev.db` is already a **directory**, follow the troubleshooting section below before continuing.

Build the management image:

```powershell
docker compose build manage
```

If the project has no `migrations` directory yet, create the migration repository and the initial migration once:

```powershell
docker compose run --rm manage db init
docker compose run --rm manage db migrate -m "Initial database"
```

If `migrations` is already included in your checkout, use the existing migrations instead. Apply migrations to create or update the database tables:

```powershell
docker compose run --rm manage db upgrade
```

Start the development application:

```powershell
docker compose up --build -d flask-dev
```

Open `http://localhost:8080`.

### Subsequent starts

```powershell
docker compose up -d flask-dev
```

Use `--build` after changing the Dockerfile or dependencies. After pulling new migrations, run `docker compose run --rm manage db upgrade`.

To inspect logs or stop the project:

```powershell
docker compose logs --tail 100 flask-dev
docker compose down
```

### Troubleshooting

**`dev.db` is a directory:** Docker creates a directory when a bind-mount source is missing. SQLite needs a file. Stop and remove the project containers, remove the empty directory, and create the file:

```powershell
docker compose down
Remove-Item -LiteralPath .\dev.db
New-Item -ItemType File -Path .\dev.db
```

These repair commands apply only when `dev.db` is an empty directory. Do not run them against an existing database file. If the directory contains files, inspect and preserve them before removing it. Then continue with the database initialization steps above, using existing migrations if present.

**`Could not locate a Flask application`:** check that `.env` exists at the project root and contains `FLASK_APP=autoapp.py`. The local directory mounted at `/app` hides the `.env` created inside the image during its build. After correcting `.env`, run `docker compose restart flask-dev`.

**`pull access denied for my_flask_app-development`:** this image is built locally by this project. Run `docker compose up --build -d flask-dev` and check the subsequent build output if it fails.

The `environment:` variables in `docker-compose.yml` take precedence over variables loaded from `.env` by the application.

To run Flask CLI commands, append the command after `manage`, for example:

```powershell
docker compose run --rm manage routes
```

A docker volume `node-modules` is created to store NPM packages and is reused across the dev and prod versions of the application. For the purposes of DB testing with `sqlite`, the file `dev.db` is mounted to all containers. This volume mount should be removed from `docker compose.yml` if a production DB server is used.

### Running locally

Run the following commands to bootstrap your environment if you are unable to run the application using Docker

Run these commands from the project root with Python 3.13 and Node 22 installed. Prepare `.env` as described above, then change its `DATABASE_URL` to `sqlite:///dev.db` for local execution (the default `/tmp/dev.db` path is intended for Docker).

```powershell
pip install -r requirements/dev.txt
npm install
npm run-script build
```

Initialize the database as described below before running `npm start`, which starts Webpack and Flask together.

Go to `http://localhost:5000`. You will see a pretty welcome screen.

#### Database Initialization (locally)

SQLite requires no separate database server. If there is no `migrations` directory, run `flask db init` and `flask db migrate` once. Then apply existing migrations:

```bash
flask db upgrade
npm start
```

## Deployment

When using Docker, reasonable production defaults are set in `docker compose.yml`

```text
FLASK_ENV=production
FLASK_DEBUG=0
```

Complete the configuration and database setup above first. To run the production service locally, stop the development service to free port 8080:

```bash
docker compose stop flask-dev
docker compose up --build -d flask-prod
```

Before deploying publicly, set a strong `SECRET_KEY` and configure the production database. The supplied `.env.example` contains development values.

If running without Docker (Bash syntax)

```bash
export FLASK_ENV=production
export FLASK_DEBUG=0
export DATABASE_URL="<YOUR DATABASE URL>"
npm run build   # build assets with webpack
flask run       # start the flask server
```

## Shell

To open the interactive shell, run

```bash
docker compose run --rm manage shell
flask shell # If running locally without Docker
```

By default, you will have access to the flask `app`.

## Running Tests/Linter

To run all tests, run

```bash
docker compose run --rm manage test
flask test # If running locally without Docker
```

To run the linter, run

```bash
docker compose run --rm manage lint
flask lint # If running locally without Docker
```

The `lint` command will attempt to fix any linting/style errors in the code. If you only want to know if the code will pass CI and do not wish for the linter to make changes, add the `--check` argument.

## Migrations

Whenever a database migration needs to be made. Run the following commands

```bash
docker compose run --rm manage db migrate
flask db migrate # If running locally without Docker
```

This will generate a new migration script. Then run

```bash
docker compose run --rm manage db upgrade
flask db upgrade # If running locally without Docker
```

To apply the migration.

For a full migration command reference, run `docker compose run --rm manage db --help`.

If you will deploy your application remotely (e.g on Heroku) you should add the `migrations` folder to version control.
You can do this after `flask db migrate` by running the following commands

```bash
git add migrations/*
git commit -m "Add migrations"
```

Make sure folder `migrations/versions` is not empty.

## Asset Management

Files placed inside the `assets` directory and its subdirectories
(excluding `js` and `css`) will be copied by webpack's
`file-loader` into the `static/build` directory. In production, the plugin
`Flask-Static-Digest` zips the webpack content and tags them with a MD5 hash.
As a result, you must use the `static_url_for` function when including static content,
as it resolves the correct file name, including the MD5 hash.
For example

```html
<link rel="shortcut icon" href="{{static_url_for('static', filename='build/favicon.ico') }}">
```

If all of your static files are managed this way, then their filenames will change whenever their
contents do, and you can ask Flask to tell web browsers that they
should cache all your assets forever by including the following line
in ``.env``:

```text
SEND_FILE_MAX_AGE_DEFAULT=31556926  # one year
```
