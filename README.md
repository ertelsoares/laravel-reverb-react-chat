Pré-requisitos
Você precisará das seguintes ferramentas para o aplicativo que criaremos neste artigo:

PHP : versão 8.2 ou superior (executephp -vpara verificar a versão)
Composer (executecomposerpara verificar se existe)
Node.js : versão 20 ou superior (executenode -vpara verificar a versão)
MySQL : versão 5.7 ou superior (executemysql --versionpara verificar se existe ou siga a documentação para instalá-lo)
Etapas Gerais
As principais etapas deste artigo serão:

Instalando o Laravel 11.
Adicionando fluxo de autenticação a ele (andaime de autenticação). O Laravel fornece um ponto de partida básico para isso usando Bootstrap com React/Vue.
Instalando Reverberação.
Componentes React.js e escuta de eventos no frontend.
Como instalar o Laravel
Para começar, instale o Laravel 11 usando o comando compositor:

composer create-project laravel/laravel:^11.0 laravel-reverb-react-chat && cd laravel-reverb-react-chat/
Neste ponto, você pode verificar o aplicativo executando o servecomando:

php artisan serve
Como Criar o Modelo e Migração
Você pode gerar um modelo e uma migração para as mensagens usando este único comando:

php artisan make:model -m Message
Então você precisará configurar o modelo da Mensagem com o seguinte código:

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
app/Modelos/Mensagem.php
Como você pode ver, há um getTimeAttribute()acessador que formatará o carimbo de data/hora de criação da mensagem em um formato de data e hora legível. Ele será exibido no topo de cada mensagem na caixa de bate-papo.

A seguir, configure a migração para a messagestabela do banco de dados com este código:

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
banco de dados/migrações/2024_03_25_000831_create_messages_table.php
Esta migração cria uma messagestabela no banco de dados. A tabela contém colunas para uma chave primária de incremento automático ( id), uma chave estrangeira ( user_id) referenciando a idcoluna da userstabela, uma textcoluna para armazenar o conteúdo da mensagem e timestampspara rastrear automaticamente os tempos de criação e modificação de cada registro.

A migração também inclui um método de reversão ( down()) para eliminar a messagestabela, se necessário.

Neste artigo, usaremos o banco de dados MySQL, mas você pode usar o SQLite como padrão, se preferir. Apenas certifique-se de configurar .envcorretamente as credenciais do seu banco de dados no arquivo:

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=database_name
DB_USERNAME=username
DB_PASSWORD=password
.env
Após configurar as variáveis ​​de ambiente, otimize o cache:

php artisan optimize
Execute migrações para recriar as tabelas do banco de dados e também para adicionar a messagestabela:

php artisan migrate:fresh
Como adicionar autenticação
Agora, você pode adicionar estruturas de autenticação ao seu aplicativo. Você pode usar o pacote UI do Laravel para importar alguns arquivos de ativos. Primeiro você precisará instalar o pacote apropriado:

composer require laravel/ui
Em seguida, importe os ativos relacionados ao React para o aplicativo:

php artisan ui react --auth
Pode ser necessário substituir o app/Http/Controllers/Controller.php, e você pode prosseguir e permitir:

The [Controller.php] file already exists. Do you want to replace it? (yes/no) [no]
Isso fará com que todo o andaime de autenticação seja compilado e instalado, incluindo rotas, controladores, visualizações, configurações de vite e um exemplo simples específico do React.
Neste ponto, você está a apenas um passo de o aplicativo estar pronto para funcionar.

NOTA: Certifique-se de ter o Node.js (com npm ) versão 20 ou superior instalado. Você pode verificar isso executando o node -vcomando. Caso contrário, vá em frente e instale-o usando a página oficial .

npm install && npm run build
O comando acima irá instalar pacotes NPM e construir ativos de frontend. Agora você pode iniciar o aplicativo Laravel e verificar seu exemplo de aplicativo totalmente pronto:

php artisan optimize && php artisan serve
artigo-imagem-1
Uma captura de tela da página de registro
Também é importante observar que você pode executar o devcomando separadamente em vez de usá-lo buildsempre que fizer alterações nos arquivos de frontend:

npm run dev
Veja os detalhes no package.jsonarquivo, em scriptscampo.

Como configurar rotas
Neste aplicativo de bate-papo em tempo real, você precisará de algumas rotas:

homepara a página inicial (já deve ser adicionado)
messagepara adicionar uma nova mensagem
messagespara obter todas as mensagens existentes
Você terá este tipo de rotas no web.phparquivo:

<?php

use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\HomeController;

Route::get('/', function () { return view('welcome'); });

Auth::routes();

Route::get('/home', [HomeController::class, 'index'])
    ->name('home');
Route::get('/messages', [HomeController::class, 'messages'])
    ->name('messages');
Route::post('/message', [HomeController::class, 'message'])
    ->name('message');
Depois de configurar essas rotas, vamos usar as vantagens dos Laravel Events e Queue Jobs.

Como configurar um evento Laravel
Você precisa criar um GotMessageevento para ouvir um evento específico:

php artisan make:event GotMessage
Os eventos do Laravel fornecem uma implementação simples de padrão de observador, permitindo que você se inscreva e ouça vários eventos que ocorrem em sua aplicação. As classes de eventos normalmente são armazenadas no app/Eventsdiretório. ( Documentos )
Configure um canal WebSocket privado no broadcastOnmétodo para que todos os usuários autenticados recebam mensagens em tempo real. Neste caso, vamos chamá-lo de "channel_for_everyone", mas você também pode torná-lo dinâmico, dependendo do usuário, como "App.Models.User.{$this->message['user_id']}".

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
        // $this->message is available here
        return [
            new PrivateChannel("channel_for_everyone"),
        ];
    }
}
app/Eventos/GotMessage.php
Como você pode ver, há uma $massagepropriedade pública como argumento do construtor, para que você possa obter informações da mensagem no front-end.

Já usamos o nome do canal no arquivo de canais e também o usaremos no front end para atualizações de mensagens em tempo real.

Não esqueça de implementar a ShouldBroadcastinterface na classe do evento.

Como configurar um trabalho de fila do Laravel
Agora é hora de criar o SendMessagejob para envio de mensagens:

php artisan make:job SendMessage
O Laravel permite criar facilmente jobs enfileirados que podem ser processados ​​em segundo plano. Ao mover tarefas demoradas para uma fila, seu aplicativo pode responder às solicitações da web com incrível velocidade e fornecer uma melhor experiência de usuário aos seus clientes. ( Documentos )
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
app/Jobs/SendMessage.php
O SendMessage.phptrabalho da fila é responsável por despachar o GotMessageevento com informações sobre uma mensagem recém-enviada. Recebe um Messageobjeto na construção, representando a mensagem a ser enviada.

Em seu handle()método, ele despacha o GotMessageevento com detalhes como ID da mensagem, ID do usuário, texto e carimbo de data/hora. Este trabalho foi projetado para ser enfileirado para processamento assíncrono, permitindo o tratamento eficiente de tarefas de envio de mensagens em segundo plano.

Como você pode ver, há uma $massagepropriedade pública como argumento construtor, que usaremos para anexar informações de mensagem ao trabalho da fila.

Como escrever os métodos do controlador
Para as rotas definidas, aqui estão os métodos de controlador apropriados:

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
app/Http/Controllers/HomeController.php
No homemétodo, obteremos os dados do usuário logado no banco de dados usando o Usermodelo e enviaremos para o blade view.
No messagesmétodo, recuperaremos todas as mensagens do banco de dados usando o Messagemodelo, anexaremos os userdados de relacionamento a ele, anexaremos o timecampo (acessador) a cada item e enviaremos tudo isso para a visualização.
No messagemétodo, uma nova mensagem será criada na tabela do banco de dados usando o Messagemodelo, e o SendMessagetrabalho da fila será despachado.
Como instalar o Laravel Reverb
Agora chegamos ao momento mais importante: é hora de instalar o Reverb no seu aplicativo Laravel.

É tão fácil. Todo o empacotamento e configuração necessários podem ser feitos usando este único comando:

php artisan install:broadcasting
Ele solicitará que você instale o Laravel Reverb, bem como instale e construa as dependências do Node necessárias para a transmissão. Basta pressionar Enter para continuar.

Após a execução do comando, certifique-se de ter adicionado automaticamente variáveis ​​de ambiente específicas de reverberação ao .envarquivo, como:

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
Você também terá dois novos arquivos de configuração no configdiretório:

reverb.php
broadcasting.php
Como configurar canais WebSocket
Por último, você precisará adicionar um canal ao channels.phparquivo. Já deve ser criado após a instalação do Reverb.

<?php

use Illuminate\Support\Facades\Broadcast;

Broadcast::channel('channel_for_everyone', function ($user) {
    return true;
});
rotas/canais.php
Você terá apenas um canal. Você pode alterar o nome do canal e torná-lo dinâmico – você decide. No fechamento do canal sempre retornaremos true, mas você poderá modificá-lo posteriormente para fazer algumas restrições quanto à inscrição do canal.

Otimize os caches mais uma vez:

php artisan optimize
Como personalizar visualizações do Laravel
Agora seu back-end deve estar pronto neste ponto, para que você possa mudar para o front-end.

Antes de trabalhar no React, você desejará configurar *.blade.phpas visualizações do Laravel. Na homevisualização blade, certifique-se de ter o div raiz com um ID mainpara renderizar todos os componentes do React lá.

@extends('layouts.app')

@section('content')
    <div class="container">
        <div id="main" data-user="{{ json_encode($user) }}"></div>
    </div>
@endsection
recursos/views/home.blade.php
A div com ID de mainobtém uma propriedade data para armazenar as $userinformações enviadas do método do controlador home.

Não vou colocar todo o resources/views/welcome.blade.phpconteúdo aqui, mas você pode apenas fazer as seguintes pequenas alterações:

Substituir url('/dashboard')com url('/home');
Substituir Dashboardcom Home;
Remova maine footerseções.
Vamos trabalhar no front-end
No Reverb, a transmissão de eventos é feita por um driver de transmissão do lado do servidor que transmite seus eventos do Laravel para que o front-end possa recebê-los dentro do cliente do navegador.

No front-end, o Laravel Echo faz esse trabalho nos bastidores. Faça eco de uma biblioteca JavaScript que facilita a assinatura de canais e a escuta de eventos transmitidos pelo driver de transmissão do servidor.

Você pode encontrar as configurações do WebSocket com Echo no rources/js/echo.jsarquivo, mas não precisa fazer nada lá para este projeto.

Vamos criar alguns componentes React para termos um projeto refatorado e mais legível.

Crie um Main.jsxcomponente na nova componentspasta:

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
recursos/js/componentes/Main.jsx
Aqui verificaremos se existe um elemento com o id 'main'. Se existir, ele prossegue com a renderização do aplicativo React.

Como você pode ver, há um ChatBoxcomponente. Aprenderemos mais sobre isso em breve.

Remova o resources/js/components/Example.jsxarquivo e importe o Main.jsxcomponente no app.js:

import './bootstrap';
import './components/Main.jsx';
Crie Message.jsxe MessageInput.jsxarquivos para que você possa usá-los no ChatBoxcomponente.

O Messagecomponente receberá userIdargumentos message(campos) para mostrar cada mensagem na caixa de chat.

import React from "react";

const Message = ({ userId, message }) => {
    return (
        <div className={`row ${
        userId === message.user_id ? "justify-content-end" : ""
        }`}>
            <div className="col-md-6">
		<small className="text-muted">
                    <strong>{message.user.name} | </strong>
                </small>
                <small className="text-muted float-right">
                    {message.time}
                </small>
                <div className={`alert alert-${
                userId === message.user_id ? "primary" : "secondary"
                }`} role="alert">
                    {message.text}
                </div>
            </div>
        </div>
    );
};

export default Message;
recursos/js/componentes/Message.jsx
O Message.jsxcomponente renderiza mensagens individuais na interface de chat. Ele recebe os adereços userIde message. Com base na correspondência do remetente da mensagem com o usuário atual, ele alinha a mensagem ao lado apropriado da tela.

Cada mensagem inclui o nome do remetente, o carimbo de data/hora e o próprio conteúdo da mensagem, com estilo diferente dependendo se a mensagem é enviada pelo usuário atual ou por outro usuário.

O MessageInputcomponente se preocupará em criar uma nova mensagem:

import React, { useState } from "react";

const MessageInput = ({ rootUrl }) => {
    const [message, setMessage] = useState("");

    const messageRequest = async (text) => {
        try {
            await axios.post(`${rootUrl}/message`, {
                text,
            });
        } catch (err) {
            console.log(err.message);
        }
    };

    const sendMessage = (e) => {
        e.preventDefault();
        if (message.trim() === "") {
            alert("Please enter a message!");
            return;
        }

        messageRequest(message);
        setMessage("");
    };

    return (
        <div className="input-group">
            <input onChange={(e) => setMessage(e.target.value)}
                   autoComplete="off"
                   type="text"
                   className="form-control"
                   placeholder="Message..."
                   value={message}
            />
            <div className="input-group-append">
                <button onClick={(e) => sendMessage(e)}
                        className="btn btn-primary"
                        type="button">Send</button>
            </div>
        </div>
    );
};

export default MessageInput;
recursos/js/componentes/MessageInput.jsx
O MessageInputcomponente fornece um campo de entrada de formulário para os usuários digitarem mensagens e enviá-las na interface de chat. Ao clicar no botão, ele aciona uma função para enviar a mensagem ao servidor por meio de uma solicitação Axios POST para o especificado rootUrlobtido do ChatBoxcomponente pai. Ele também lida com a validação para garantir que os usuários não possam enviar mensagens vazias. Você pode personalizá-lo mais tarde, se desejar.

Agora crie um ChatBox.jsxcomponente para ter o front end pronto:

import React, { useEffect, useRef, useState } from "react";
import Message from "./Message.jsx";
import MessageInput from "./MessageInput.jsx";

const ChatBox = ({ rootUrl }) => {
    const userData = document.getElementById('main')
        .getAttribute('data-user');

    const user = JSON.parse(userData);
    // `App.Models.User.${user.id}`;
    const webSocketChannel = `channel_for_everyone`;

    const [messages, setMessages] = useState([]);
    const scroll = useRef();

    const scrollToBottom = () => {
        scroll.current.scrollIntoView({ behavior: "smooth" });
    };

    const connectWebSocket = () => {
        window.Echo.private(webSocketChannel)
            .listen('GotMessage', async (e) => {
                // e.message
                await getMessages();
            });
    }

    const getMessages = async () => {
        try {
            const m = await axios.get(`${rootUrl}/messages`);
            setMessages(m.data);
            setTimeout(scrollToBottom, 0);
        } catch (err) {
            console.log(err.message);
        }
    };

    useEffect(() => {
        getMessages();
        connectWebSocket();

        return () => {
            window.Echo.leave(webSocketChannel);
        }
    }, []);

    return (
        <div className="row justify-content-center">
            <div className="col-md-8">
                <div className="card">
                    <div className="card-header">Chat Box</div>
                    <div className="card-body"
                         style={{height: "500px", overflowY: "auto"}}>
                        {
                            messages?.map((message) => (
                                <Message key={message.id}
                                         userId={user.id}
                                         message={message}
                                />
                            ))
                        }
                        <span ref={scroll}></span>
                    </div>
                    <div className="card-footer">
                        <MessageInput rootUrl={rootUrl} />
                    </div>
                </div>
            </div>
        </div>
    );
};

export default ChatBox;
recursos/js/componentes/ChatBox.jsx
O ChatBoxcomponente gerencia uma interface de chat dentro do aplicativo. Ele busca e exibe mensagens de um servidor usando solicitações WebSocket e HTTP.

O componente renderiza uma lista de mensagens, um campo de entrada de mensagem e rola automaticamente para o final quando novas mensagens chegam.

Ele define um canal WebSocket para atualizações de mensagens em tempo real. Você precisa configurar esse canal usando o mesmo nome que foi escrito no routes/hannels.phptrabalho da app/Events/GotMessage.phpfila.

Além disso, a leave()função é chamada na useEffectfunção de limpeza para cancelar a assinatura do canal WebSocket quando o componente é desmontado. Isso evita vazamentos de memória e conexões de rede desnecessárias, impedindo que o componente escute atualizações no canal WebSocket quando ele não for mais necessário.

Executando o aplicativo
Agora está tudo pronto e é hora de conferir o aplicativo. Siga estas instruções:

artigo-imagem-2
Uma captura de tela do terminal com todos os comandos necessários
Crie ativos de front-end (este não é um comando de execução "para sempre"):
npm run build
Comece a ouvir os eventos do Laravel:
php artisan queue:listen
Inicie o servidor WebSocket:
php artisan reverb:start
Inicie o servidor (você pode usar uma alternativa para seu aplicativo, como um servidor local em execução):
php artisan serve
Após a execução de todos os comandos necessários, você pode conferir o aplicativo visitando a URL padrão: http://127.0.0.1:8000.

Para teste, você pode registrar dois usuários diferentes, fazer com que esses usuários façam login, enviem mensagens de cada um deles e vejam a caixa de bate-papo.

Recursos úteis de reverberação
Agora que chegamos ao final deste artigo, vale a pena listar alguns recursos úteis sobre Reverb:

Laravel Broadcasting (documentação oficial)
Taylor Otwel - Atualização do Laravel (palestra sobre Laracon EU 2024)
Joe Dixon no X (criador do Reverb)
Episódio Laracast (exemplo prático com Reverb)
Conclusão
Agora você sabe como construir aplicações em tempo real com Laravel Reverb na nova versão do Laravel. Com isso, você pode implementar comunicações WebSocket em seu aplicativo full-stack e evitar o uso de serviços adicionais de terceiros (como Pusher e Socket.io).

Se você quiser ter uma ideia clara de como integrar React.js em seu aplicativo Laravel sem usar nenhuma ferramenta Laravel adicional (como Inertia), você pode ler meu artigo anterior do freeCodeCamp , onde você pode construir um arquivo completo de página única. empilhar o aplicativo Tasklist.