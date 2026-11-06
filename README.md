# Laravel Reverb Server

A standalone **Laravel Reverb WebSocket server** built with **Laravel
13** and designed to run independently from the main Laravel backend or
frontend application.

This repository contains only the infrastructure required to run Laravel
Reverb as a dedicated real-time broadcasting service. It is
intentionally separated from the main application so the WebSocket
server can be deployed, maintained, restarted, and scaled independently.

The repository includes a production-ready Docker setup and is designed
to be deployed easily with **Coolify** or any other Docker-compatible
platform.

------------------------------------------------------------------------

## Purpose

In a typical Laravel application, Laravel Reverb can run inside the same
project as the main backend. However, for production environments it can
be useful to isolate the long-running WebSocket process from the main
application.

This repository provides that separation.

The architecture can be used with:

-   A Laravel backend with its own frontend
-   Laravel + Inertia
-   Laravel + React
-   Laravel + Vue
-   Laravel API + separate React / Next.js / Vue frontend
-   Multiple frontend applications connecting to the same Reverb service

The main application remains responsible for business logic,
authentication, broadcast events, channel authorization, queues, APIs,
and persistent data.

This repository is responsible only for the **Reverb WebSocket
service**.

------------------------------------------------------------------------

## Architecture

``` text
┌──────────────────────────────┐
│        Main Backend          │
│                              │
│ Laravel Application          │
│ Broadcast Events             │
│ Queue Workers                │
│ Channel Authorization        │
│ Business Logic               │
└──────────────┬───────────────┘
               │
               │ Publish broadcast events
               ▼
┌──────────────────────────────┐
│     Laravel Reverb Server    │
│      This Repository         │
│                              │
│ WebSocket Service            │
│ ws / wss connections         │
│ Port 8080 internally         │
└──────────────┬───────────────┘
               │
               │ Real-time events
               ▼
┌──────────────────────────────┐
│          Frontend            │
│                              │
│ Laravel Echo                 │
│ React / Vue / Inertia        │
│ Next.js / Separate SPA       │
└──────────────────────────────┘
```

The backend publishes events to Reverb, while frontend clients maintain
WebSocket connections with the Reverb server and receive those events in
real time.

For private and presence channels, authentication should normally remain
in the main backend application.

------------------------------------------------------------------------

## Technology Stack

-   Laravel 13
-   Laravel Reverb
-   PHP 8.4 Docker runtime
-   Laravel Echo compatible
-   Pusher protocol compatible
-   Docker
-   Coolify-ready deployment

------------------------------------------------------------------------

## Repository Responsibilities

### This repository handles

-   Running the Laravel Reverb WebSocket server
-   Accepting WebSocket connections
-   Receiving broadcast messages from the main backend
-   Broadcasting real-time events to connected clients
-   Managing Reverb application credentials
-   WebSocket origin restrictions
-   Optional Reverb horizontal scaling

### This repository does not handle

-   Application business logic
-   User authentication
-   Main application APIs
-   Broadcast event definitions
-   Private channel authorization logic
-   Application queue workers
-   Application database records
-   Frontend UI
-   Persistent chat/message storage

Those responsibilities should remain in the main backend or frontend
repositories.

------------------------------------------------------------------------

## Requirements

For local development:

-   PHP 8.3+
-   Composer

For container deployment:

-   Docker

For production:

-   Docker-compatible hosting such as Coolify
-   A domain or subdomain for the WebSocket endpoint
-   SSL/TLS termination for secure `wss://` connections

------------------------------------------------------------------------

## Installation

Clone the repository:

``` bash
git clone https://github.com/mahfuzahmedzisan/reverb_server.git
cd reverb_server
```

Install PHP dependencies:

``` bash
composer install
```

Create the environment file:

``` bash
cp .env.example .env
```

Generate the Laravel application key:

``` bash
php artisan key:generate
```

Configure the Reverb credentials in `.env`.

------------------------------------------------------------------------

## Environment Configuration

Example:

``` env
APP_NAME="Reverb Server"
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=https://ws.example.com

DB_CONNECTION=sqlite

SESSION_DRIVER=array
CACHE_STORE=array
QUEUE_CONNECTION=sync

REVERB_APP_ID=your-app-id
REVERB_APP_KEY=your-app-key
REVERB_APP_SECRET=your-app-secret

REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080

REVERB_HOST=ws.example.com
REVERB_PORT=443
REVERB_SCHEME=https

REVERB_ALLOWED_ORIGINS=example.com,www.example.com,localhost,127.0.0.1
```

### Important

The following credentials must match between the Reverb server and the
Laravel backend that publishes events:

``` env
REVERB_APP_ID
REVERB_APP_KEY
REVERB_APP_SECRET
```

The frontend needs the public Reverb connection information, including
the application key and public WebSocket hostname.

Never expose `REVERB_APP_SECRET` to the frontend.

------------------------------------------------------------------------

## Reverb Server Host vs Public Host

Laravel Reverb uses two different sets of host/port settings.

### Server binding

``` env
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080
```

These values define where the Reverb process listens inside the server
or Docker container.

### Public connection endpoint

``` env
REVERB_HOST=ws.example.com
REVERB_PORT=443
REVERB_SCHEME=https
```

These values represent the public endpoint used by the backend and
clients.

For a Docker/Coolify production deployment, the common setup is:

``` text
Public:
wss://ws.example.com:443

        ↓ reverse proxy

Container:
0.0.0.0:8080
```

------------------------------------------------------------------------

## Running Locally

Start the Reverb server:

``` bash
php artisan reverb:start
```

For debugging:

``` bash
php artisan reverb:start --debug
```

By default, the server is configured to listen on port `8080`.

Example:

``` text
ws://localhost:8080
```

------------------------------------------------------------------------

## Docker

This repository includes a Dockerfile specifically configured for
running the standalone Reverb service.

Build the image:

``` bash
docker build -t laravel-reverb-server .
```

Run the container:

``` bash
docker run \
    --env-file .env \
    -p 8080:8080 \
    laravel-reverb-server
```

The Reverb process listens internally on:

``` text
0.0.0.0:8080
```

The Docker image also includes a health check for the Reverb service.

Environment secrets should be injected at runtime instead of being
stored inside the Docker image.

------------------------------------------------------------------------

## Coolify Deployment

This repository is designed to be deployable directly from GitHub using
Coolify.

### Basic deployment flow

1.  Create a new resource/application in Coolify.
2.  Select the Git repository.
3.  Use the repository `Dockerfile` as the build method.
4.  Configure the required environment variables.
5.  Expose container port `8080`.
6.  Assign a WebSocket domain, for example:

``` text
ws.example.com
```

7.  Enable HTTPS for the domain.
8.  Deploy the application.

Coolify should proxy the public HTTPS/WSS connection to the Reverb
container running on port `8080`.

Typical production configuration:

``` env
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080

REVERB_HOST=ws.example.com
REVERB_PORT=443
REVERB_SCHEME=https
```

The public WebSocket URL will then be:

``` text
wss://ws.example.com
```

------------------------------------------------------------------------

## Connecting a Laravel Backend

The main Laravel application should use Reverb as its broadcast
connection.

Example backend environment:

``` env
BROADCAST_CONNECTION=reverb

REVERB_APP_ID=your-app-id
REVERB_APP_KEY=your-app-key
REVERB_APP_SECRET=your-app-secret

REVERB_HOST=ws.example.com
REVERB_PORT=443
REVERB_SCHEME=https
```

The credentials must match the standalone Reverb server.

The main backend can then broadcast normal Laravel events:

``` php
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

class OrderUpdated implements ShouldBroadcast
{
    // ...
}
```

The event is produced by the main application and delivered through the
standalone Reverb server.

If the event implements `ShouldBroadcast`, make sure the main
application's queue worker is running when using an asynchronous queue
connection.

------------------------------------------------------------------------

## Connecting a Frontend

Laravel Echo can connect directly to the standalone Reverb server.

Install the client dependencies:

``` bash
npm install laravel-echo pusher-js
```

Example Vite environment:

``` env
VITE_REVERB_APP_KEY=your-app-key
VITE_REVERB_HOST=ws.example.com
VITE_REVERB_PORT=443
VITE_REVERB_SCHEME=https
```

Example Echo configuration:

``` javascript
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT ?? 80,
    wssPort: import.meta.env.VITE_REVERB_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

The frontend may live inside the same Laravel repository as the backend
or in a completely separate repository.

------------------------------------------------------------------------

## Private and Presence Channels

The Reverb server transports WebSocket messages, but application-level
authorization should remain in the main Laravel backend.

For example:

``` text
Frontend
   │
   ├── WebSocket connection ──────────────> Reverb Server
   │
   └── /broadcasting/auth request ────────> Main Laravel Backend
```

The main backend should define its channel authorization rules in:

``` text
routes/channels.php
```

This keeps authentication and authorization close to the application's
users, permissions, and business rules.

------------------------------------------------------------------------

## Allowed Origins

Production deployments should restrict which frontend hosts may connect
to Reverb.

Example:

``` env
REVERB_ALLOWED_ORIGINS=example.com,app.example.com,www.example.com
```

For development, localhost may also be allowed:

``` env
REVERB_ALLOWED_ORIGINS=example.com,localhost,127.0.0.1
```

Avoid using unrestricted origins in production unless there is a
deliberate reason to do so.

------------------------------------------------------------------------

## Restarting Reverb

Reverb is a long-running process.

After configuration or application changes, restart the process:

``` bash
php artisan reverb:restart
```

When deployed through Docker/Coolify, redeploying or restarting the
container will also restart the Reverb process.

------------------------------------------------------------------------

## Scaling

A single Reverb instance is sufficient for many applications.

If the application later requires multiple Reverb instances, Reverb
supports horizontal scaling through Redis.

Example:

``` env
REVERB_SCALING_ENABLED=true

REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
```

All Reverb instances must use the same shared Redis server when
horizontal scaling is enabled.

The instances can then be placed behind a load balancer.

------------------------------------------------------------------------

## Example Deployment Architecture

``` text
                        ┌─────────────────────┐
                        │      Frontend       │
                        │ React / Vue / Next  │
                        │ Inertia / Laravel   │
                        └──────────┬──────────┘
                                   │
                       wss://ws.example.com
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │ Coolify Proxy / TLS │
                        └──────────┬──────────┘
                                   │
                                   │ :8080
                                   ▼
                        ┌─────────────────────┐
                        │ Standalone Reverb   │
                        │      Server         │
                        └──────────▲──────────┘
                                   │
                                   │ Broadcast API
                                   │
                        ┌──────────┴──────────┐
                        │   Laravel Backend   │
                        │ Events / Queues /   │
                        │ Channel Auth / API  │
                        └─────────────────────┘
```

------------------------------------------------------------------------

## Why Keep Reverb Separate?

Running Reverb as an independent service provides several advantages:

-   Independent deployment from the main application
-   Independent restart lifecycle
-   Cleaner separation of responsibilities
-   WebSocket failures do not require restarting the main application
    container
-   Easier infrastructure management
-   Easier horizontal scaling later
-   The same Reverb service pattern can be reused with different
    Laravel/frontend architectures
-   Main backend containers can remain focused on HTTP/API and queue
    workloads

This repository therefore acts as a dedicated **real-time transport
layer**, while the main Laravel application remains the source of
business logic and persistent application data.

------------------------------------------------------------------------

## Security Notes

-   Never commit production `.env` files.
-   Never expose `REVERB_APP_SECRET` to browser clients.
-   Use `wss://` in production.
-   Restrict allowed origins.
-   Keep application authorization in the main backend.
-   Store production credentials in Coolify environment variables or
    another secure secret store.
-   Use different credentials between development, staging, and
    production environments.

------------------------------------------------------------------------

## License

This project is based on Laravel and Laravel Reverb.

Laravel is open-sourced software licensed under the MIT license.
