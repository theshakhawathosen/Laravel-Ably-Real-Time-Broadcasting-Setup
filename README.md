<div align="center">

<img src="./logo.png" alt="Ably Logo" width="180"/>

# Laravel + Ably — Real-Time Broadcasting Setup

**A complete, copy-paste-ready guide to integrate Ably WebSocket broadcasting into a fresh Laravel project.**

![Laravel](https://img.shields.io/badge/Laravel-12.x-FF2D20?style=flat-square&logo=laravel&logoColor=white)
![Ably](https://img.shields.io/badge/Ably-WebSocket-orange?style=flat-square)
![PHP](https://img.shields.io/badge/PHP-8.x-777BB4?style=flat-square&logo=php&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)

</div>

---

## 📋 Table of Contents

- [Prerequisites](#-prerequisites)
- [Step 1 — Create Laravel Project](#step-1--create-laravel-project)
- [Step 2 — Install Packages](#step-2--install-packages)
- [Step 3 — Ably Dashboard Setup](#step-3--ably-dashboard-setup)
- [Step 4 — Configure .env](#step-4--configure-env)
- [Step 5 — Check broadcasting.php](#step-5--check-broadcastingphp)
- [Step 6 — Create BroadcastServiceProvider](#step-6--create-broadcastserviceprovider)
- [Step 7 — Register in bootstrap/app.php](#step-7--register-in-bootstrapappphp)
- [Step 8 — Create routes/channels.php](#step-8--create-routeschannelsphp)
- [Step 9 — Configure app.js](#step-9--configure-appjs)
- [Step 10 — Create an Event](#step-10--create-an-event)
- [Step 11 — Fire Event from Route](#step-11--fire-event-from-route)
- [Step 12 — Listen in Blade](#step-12--listen-in-blade)
- [Step 13 — Clear Cache & Run](#step-13--clear-cache--run)
- [Troubleshooting](#-troubleshooting)
- [Architecture Flow](#-architecture-flow)

---

## ✅ Prerequisites

| Requirement | Version |
|-------------|---------|
| PHP | >= 8.0 |
| Composer | Latest |
| Node.js & NPM | Latest LTS |
| Ably Account | [Free tier available](https://ably.com) |

---

## Step 1 — Create Laravel Project

```bash
composer create-project laravel/laravel my-project
cd my-project
```

---

## Step 2 — Install Packages

### Server-side (PHP)
```bash
composer require ably/ably-php ably/laravel-broadcaster
```

> `ably/ably-php` — core PHP SDK that sends messages to Ably server.  
> `ably/laravel-broadcaster` — bridge that connects Laravel's `broadcast()` system to Ably.

### Client-side (JS) — Choose ONE approach:

#### Option A: NPM (recommended for Vite-based projects)
```bash
npm install ably
```

#### Option B: CDN (no Node.js / npm required)
```html
<script src="https://cdn.ably.com/lib/ably.min-2.js"></script>
```

> ⚠️ **Do NOT install `laravel-echo`** — it has known compatibility issues with Ably. Use the Ably JS client directly instead.

---

## Step 3 — Ably Dashboard Setup

1. Log in at [ably.com](https://ably.com)
2. Click **Create App**
3. Go to your App → **API Keys**

### ⚠️ Important: Use the Root API Key

You will see a **Root API Key** at the top of the API Keys list. This is the key you need.

> The Root API Key looks like: `appId.keyId:secretValue`  
> Example: `NHRNYA.8Nz07w:wuleIitSy...`

### Enable All Capabilities

Click on your Root API Key → under **Capabilities**, make sure all are checked:

| Capability | Required |
|------------|----------|
| ✅ Publish | **Yes** |
| ✅ Subscribe | **Yes** |
| ✅ Presence | Yes |
| ✅ History | Yes |

> ❌ If **Publish** is not enabled, you will get `Unauthorized to publish to channel` error (Error Code: 40160).

Click **Save** after making changes.

---

## Step 4 — Configure .env

```env
BROADCAST_CONNECTION=ably

# Full Root API Key (used by Laravel server-side)
ABLY_KEY=appId.keyId:yourSecretHere

# Same key exposed to Vite (used by JS client)
VITE_ABLY_KEY=appId.keyId:yourSecretHere
```

> Both `ABLY_KEY` and `VITE_ABLY_KEY` should hold the **same full Root API Key**.

---

## Step 5 — Check broadcasting.php

Open `config/broadcasting.php` and make sure the `ably` connection exists inside `connections`:

```php
'ably' => [
    'driver' => 'ably',
    'key'    => env('ABLY_KEY'),
],
```

If it's missing, add it manually.

---

## Step 6 — Create BroadcastServiceProvider

Laravel 11+ does not include this file by default. Create it manually:

**`app/Providers/BroadcastServiceProvider.php`**

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Broadcast;
use Illuminate\Support\ServiceProvider;

class BroadcastServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Broadcast::routes();
        require base_path('routes/channels.php');
    }
}
```

---

## Step 7 — Register in bootstrap/app.php

Add `->withProviders([...])` after `->withRouting(...)`:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withProviders([
        App\Providers\BroadcastServiceProvider::class, // 👈 Add this
    ])
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

---

## Step 8 — Create routes/channels.php

If this file doesn't exist, create it:

**`routes/channels.php`**

```php
<?php

use Illuminate\Support\Facades\Broadcast;

Broadcast::channel('App.Models.User.{id}', function ($user, $id) {
    return (int) $user->id === (int) $id;
});
```

---

## Step 9 — Configure Client-side

### Option A: NPM (app.js via Vite)

Replace the contents of `resources/js/app.js`:

```js
import * as Ably from 'ably';

const realtime = new Ably.Realtime({ key: import.meta.env.VITE_ABLY_KEY });

window.AblyClient = realtime;
```

Make sure `VITE_ABLY_KEY` is set in your `.env`.

### Option B: CDN (no app.js needed)

Skip `app.js` entirely. Just add the CDN script directly in your Blade file (see Step 12 — Option B).

---

## Step 10 — Create an Event

```bash
php artisan make:event MessageSent
```

**`app/Events/MessageSent.php`**

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;
use Illuminate\Queue\SerializesModels;

class MessageSent implements ShouldBroadcastNow
{
    use SerializesModels;

    public string $message;

    public function __construct(string $message)
    {
        $this->message = $message;
    }

    public function broadcastOn(): Channel
    {
        return new Channel('chat');
    }

    public function broadcastAs(): string
    {
        return 'message.sent';
    }
}
```

> Using `ShouldBroadcastNow` instead of `ShouldBroadcast` — broadcasts **immediately** without requiring a queue worker.

---

## Step 11 — Fire Event from Route

**`routes/web.php`**

```php
use App\Events\MessageSent;

Route::get('/send', function () {
    broadcast(new MessageSent('Hello from Laravel + Ably!'));
    return 'Event fired!';
});
```

---

## Step 12 — Listen in Blade

### Option A: NPM (using app.js via Vite)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Ably Test</title>
    @vite(['resources/js/app.js'])
</head>
<body>

    <h1>Listening for messages...</h1>

    <script type="module">
        const channel = window.AblyClient.channels.get('public:chat');

        channel.subscribe('message.sent', (msg) => {
            console.log('Received:', msg.data);
        });
    </script>

</body>
</html>
```

> ⚠️ Always use `<script type="module">` — without it, `window.AblyClient` will be `undefined` because `app.js` loads asynchronously as an ES module.

---

### Option B: CDN (no npm / Vite needed)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Ably Test</title>
</head>
<body>

    <h1>Listening for messages...</h1>

    <script src="https://cdn.ably.com/lib/ably.min-2.js"></script>

    <script>
        const realtime = new Ably.Realtime({
            key: '{{ config("broadcasting.connections.ably.key") }}'
        });

        const channel = realtime.channels.get('public:chat');

        channel.subscribe('message.sent', (msg) => {
            console.log('Received:', msg.data);
        });
    </script>

</body>
</html>
```

> ✅ No `type="module"` needed — CDN script loads synchronously so `Ably` is available immediately.  
> ⚠️ **Channel name must be `public:chat`** — Laravel's Ably broadcaster automatically adds the `public:` prefix when broadcasting to a public channel. Your frontend must subscribe to `public:chat`, not `chat`.

---

## Step 13 — Clear Cache & Run

```bash
php artisan config:clear
php artisan cache:clear
```

```bash
# Terminal 1 — Vite dev server
npm run dev

# Terminal 2 — Laravel server
php artisan serve
```

### Test it

1. Open `http://127.0.0.1:8000` in your browser
2. Open DevTools → Console (F12)
3. In a new tab, visit `http://127.0.0.1:8000/send`
4. Check the first tab's console — you should see the message ✅

---

## 🐛 Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Unauthorized to publish to channel` (Code 40160) | API Key missing Publish capability | Ably Dashboard → API Keys → Enable all Capabilities |
| `invalid key parameter` | Incomplete key in `.env` | Use full key: `appId.keyId:secret` |
| `Cannot read properties of undefined (reading 'channel')` | Missing `type="module"` on script tag | Add `type="module"` to your inline `<script>` |
| `Broadcaster string ably is not supported` | Wrong version of laravel-echo | Remove laravel-echo; use Ably JS client directly |
| `Ably error: Unauthorized` on server | `ABLY_KEY` missing or wrong in `.env` | Check `.env` and run `php artisan config:clear` |
| Message not received even though event fires | Wrong channel name on frontend | Use `public:chat` not `chat` — Laravel adds `public:` prefix automatically |

---

## 🏗 Architecture Flow

```
┌─────────────────────────────────────────────────────────┐
│                        BROWSER                          │
│                                                         │
│   Page loads → Ably JS client connects to Ably server   │
│             → Subscribes to 'chat' channel              │
└────────────────────────┬────────────────────────────────┘
                         │  WebSocket (persistent)
                         ▼
┌─────────────────────────────────────────────────────────┐
│                     ABLY SERVER                         │
│                  (Cloud infrastructure)                 │
└──────────┬──────────────────────────────────────────────┘
           │  Publish message
           │
┌──────────┴──────────────────────────────────────────────┐
│                   LARAVEL SERVER                        │
│                                                         │
│   GET /send                                             │
│     → broadcast(new MessageSent(...))                   │
│       → Ably PHP client publishes to 'chat' channel     │
└─────────────────────────────────────────────────────────┘
```

---

## 📦 Quick Reference

```
Server package      →  ably/ably-php + ably/laravel-broadcaster
Client (NPM)        →  ably  (npm install ably)
Client (CDN)        →  https://cdn.ably.com/lib/ably.min-2.js
Laravel Echo        →  ❌ Not used (compatibility issues)
Broadcast interface →  ShouldBroadcastNow  (no queue needed)
Script type (NPM)   →  type="module"  (required)
Script type (CDN)   →  regular <script>  (no module needed)
Channel prefix      →  public:chat  (Laravel adds public: prefix automatically)
API Key type        →  Root API Key with all Capabilities enabled
```

---

<div align="center">

Made with ❤️ for the Laravel community  
[Ably Docs](https://ably.com/docs) · [Laravel Broadcasting Docs](https://laravel.com/docs/broadcasting) · [ably/laravel-broadcaster](https://github.com/ably/laravel-broadcaster)

</div>
