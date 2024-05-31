# Laravel Reverb React Chat Application

Este projeto é um exemplo de aplicação de bate-papo em tempo real usando Laravel 11, Reverb, e React.js. A aplicação suporta autenticação de usuários, transmissão de mensagens em tempo real via WebSockets e interface de usuário com React.

## Pré-requisitos

Certifique-se de ter as seguintes ferramentas instaladas:

- **PHP**: versão 8.2 ou superior (`php -v` para verificar a versão)
- **Composer** (`composer` para verificar se está instalado)
- **Node.js**: versão 20 ou superior (`node -v` para verificar a versão)
- **MySQL**: versão 5.7 ou superior (`mysql --version` para verificar se está instalado)

## Etapas Gerais

1. Instalando o Laravel 11
2. Adicionando autenticação
3. Instalando e configurando o Reverb
4. Criando componentes React e escutando eventos no frontend

## Instalação do Laravel

Para começar, instale o Laravel 11 usando o Composer:

```sh
composer create-project laravel/laravel:^11.0 laravel-reverb-react-chat && cd laravel-reverb-react-chat/
```

Verifique o aplicativo executando o comando:

```sh
php artisan serve
```

## Criando o Modelo e Migração

Gere um modelo e uma migração para as mensagens:

```sh
php artisan make:model -m Message
```

Configure o modelo `Message` com o seguinte código:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Message extends Model
{
    use HasFactory;

    public $table = 'messages';
    protected $fillable = ['id', 'user_id', 'text'];

    public function user(): BelongsTo {
        return $this->belongsTo(User::class, 'user_id');
    }

    public function getTimeAttribute(): string {
        return date(
            "d M Y, H:i:s",
            strtotime($this->attributes['created_at'])
        );
    }
}
```

Configure a migração para a tabela `messages`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void {
        Schema::create('messages', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained();
            $table->text('text')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void {
        Schema::dropIfExists('messages');
    }
};
```

Configure o arquivo `.env` para o banco de dados:

```dotenv
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=database_name
DB_USERNAME=username
DB_PASSWORD=password
```

Otimize o cache e execute as migrações:

```sh
php artisan optimize
php artisan migrate:fresh
```

## Adicionando Autenticação

Instale o pacote Laravel UI:

```sh
composer require laravel/ui
```

Implemente os ativos relacionados ao React:

```sh
php artisan ui react --auth
```

Instale os pacotes NPM e compile os ativos de frontend:

```sh
npm install && npm run build
```

Inicie o servidor Laravel:

```sh
php artisan optimize && php artisan serve
```

## Configuração das Rotas

Adicione as rotas no arquivo `routes/web.php`:

```php
<?php

use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\HomeController;

Route::get('/', function () { return view('welcome'); });

Auth::routes();

Route::get('/home', [HomeController::class, 'index'])->name('home');
Route::get('/messages', [HomeController::class, 'messages'])->name('messages');
Route::post('/message', [HomeController::class, 'message'])->name('message');
```

## Configuração de Eventos e Jobs no Laravel

Crie um evento `GotMessage`:

```sh
php artisan make:event GotMessage
```

Implemente o evento `GotMessage`:

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class GotMessage implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(public array $message) {
        //
    }

    public function broadcastOn(): array {
        return [
            new PrivateChannel("channel_for_everyone"),
        ];
    }
}
```

Crie um job `SendMessage`:

```sh
php artisan make:job SendMessage
```

Implemente o job `SendMessage`:

```php
<?php

namespace App\Jobs;

use App\Events\GotMessage;
use App\Models\Message;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class SendMessage implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Message $message) {
        //
    }

    public function handle(): void {
        GotMessage::dispatch([
            'id' => $this->message->id,
            'user_id' => $this->message->user_id,
            'text' => $this->message->text,
            'time' => $this->message->time,
        ]);
    }
}
```

Implemente os métodos do controlador:

```php
<?php

namespace App\Http\Controllers;

use App\Jobs\SendMessage;
use App\Models\Message;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class HomeController extends Controller
{
    public function __construct() {
        $this->middleware('auth');
    }

    public function index() {
        $user = User::where('id', auth()->id())->select([
            'id', 'name', 'email',
        ])->first();

        return view('home', [
            'user' => $user,
        ]);
    }

    public function messages(): JsonResponse {
        $messages = Message::with('user')->get()->append('time');

        return response()->json($messages);
    }

    public function message(Request $request): JsonResponse {
        $message = Message::create([
            'user_id' => auth()->id(),
            'text' => $request->get('text'),
        ]);
        SendMessage::dispatch($message);

        return response()->json([
            'success' => true,
            'message' => "Message created and job dispatched.",
        ]);
    }
}
```

## Instalação do Laravel Reverb

Instale o Reverb:

```sh
php artisan install:broadcasting
```

Certifique-se de que as variáveis de ambiente específicas do Reverb foram adicionadas ao arquivo `.env`:

```dotenv
BROADCAST_CONNECTION=reverb

###

REVERB_APP_ID=795051
REVERB_APP_KEY=s3w3thzezulgp5g0e5bs
REVERB_APP_SECRET=gncsnk3rzpvczdakl6pz
REVERB_HOST="localhost"
REVERB_PORT=8080
REVERB_SCHEME=http

VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST="${REVERB_HOST}"
VITE_REVERB_PORT="${REVERB_PORT}"
VITE_REVERB_SCHEME="${REVERB_SCHEME}"
```

Adicione o canal ao arquivo `routes/channels.php`:

```php
<?php

use Illuminate\Support\Facades\Broadcast;

Broadcast::channel('channel_for_everyone', function ($user) {
    return true;
});
```

Otimize o cache:

```sh
php artisan optimize
```

## Personalização das Visualizações do Laravel

Configure a visualização `home.blade.php`:

```php
@extends('layouts.app')

@section('content')
    <div class="container">
        <div id="main" data-user="{{ json_encode($user) }}"></div>
    </div>
@endsection
```

## Configuração do Frontend com React

Crie o componente `Main.jsx`:

```jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import '../../css/app.css';
import ChatBox from "./ChatBox.jsx";

if (document.getElementById('main')) {
    const rootUrl = "http://127.0.0.1:8000";
    
    ReactDOM.createRoot(document.getElementById('main')).render(
        <React.StrictMode>
            <ChatBox rootUrl={rootUrl} />
        </React.StrictMode>
    );
}
```

Remova o arquivo `Example.jsx` e importe o `Main.jsx` no `app.js`:

```jsx
import './bootstrap';
import './components/Main.jsx';
```

Crie o componente `Message.jsx`:

```jsx
import React from "react";

const Message = ({ userId, message }) => {
    return (
        <div className={`row ${
        userId === message.user_id ? "justify-content-end" : ""
        }`}>
            <div className="col-md-6">
		<small className="text-muted">
                    <strong>{message.user.name} | </strong>
                </small