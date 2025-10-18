# Cliente Android DTProto

Este guia resume a integração do artefato `dtunnel.aar` a um aplicativo Android, mostra como construir o `VpnService` mínimo e explica os eventos emitidos pelo cliente DTProto.

## Pré-requisitos

- Projeto Android API 23 ou superior.
- Permissão `android.permission.INTERNET` no `AndroidManifest.xml`.
- Serviço que estende `VpnService` (obrigatório para interfaces TUN).
- Artefato `dtunnel.aar` gerado pelo workflow de release ou via `./build_android.sh`.

## Instalação do artefato

1. Copie `dtunnel.aar` (e opcionalmente `dtunnel-sources.jar`) para `app/libs/`.
2. No `build.gradle.kts` (ou `build.gradle` Groovy) do módulo:

```kotlin
dependencies {
    implementation(files("libs/dtunnel.aar"))
}
```

3. Sincronize o projeto no Android Studio.

## Eventos de status

O cliente publica atualizações em tempo real através de `StatusListener`. Os valores possíveis são:

| Status              | Significado                                             |
|---------------------|---------------------------------------------------------|
| `CONNECTING`        | Abertura do canal com o servidor DTProto                |
| `AUTHENTICATING`    | Envio de credenciais                                    |
| `AUTHENTICATION_FAILED` | Falha na validação das credenciais                 |
| `HANDSHAKING`       | Troca de chaves e parâmetros de sessão                  |
| `OPENING_TUN`       | Configuração da interface TUN                           |
| `CONNECTED`         | Túnel ativo; tráfego pode ser roteado                   |
| `DISCONNECTED`      | Túnel encerrado voluntariamente                         |
| `ERROR`             | Erro fatal (verifique o parâmetro `error`)              |

## Serviço mínimo

```kotlin
class ProtoVpnService : VpnService() {
    private var startThread: Thread? = null
    private var stopThread: Thread? = null

    private val statusListener = StatusListener { status, error ->
        Log.i(TAG, "status=$status error=$error")
        updateNotification(status ?: "desconhecido")
        if (status == "ERROR" || status == "AUTHENTICATION_FAILED") {
            error?.let { Log.e(TAG, "Falha no cliente", it) }
        }
    }

    private val logHandler = LogHandler { level, _, message ->
        Log.println(toAndroidLevel(level.toInt()), TAG, message.orEmpty())
    }

    private val socketOpener = SocketOpener {
        val socket = Socket()
        if (!protect(socket)) {
            socket.close()
            throw IllegalStateException("VpnService.protect falhou")
        }
        ParcelFileDescriptor.fromSocket(socket).detachFd().toLong()
    }

    private val tunBuilder = TunInterfaceBuilder { ip ->
        val builder = Builder()
            .setSession("DTProto")
            .setMtu(1500)
            .addAddress(ip ?: "10.8.0.2", 32)
            .addDnsServer("1.1.1.1")
            .addRoute("0.0.0.0", 0)

        val fd = builder.establish()
            ?: throw IllegalStateException("Falha ao criar TUN")

        fd.detachFd().toLong()
    }

    private val client by lazy {
        val cfg = DTProtoClientConfig().apply {
            username = "user@example"
            password = "secret"
            keepAliveInterval = 120
            keepAliveMaxRetry = 5
            reconnectDelay = 3
        }
        LibDTProto.new_(cfg, tunBuilder, socketOpener, statusListener, logHandler)
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startForeground(NOTIFICATION_ID, buildNotification("Conectando..."))

        startThread?.interrupt()
        startThread = thread(name = "dtproto-start") {
            runCatching { client.start() }
                .onFailure { err ->
                    Log.e(TAG, "Erro ao iniciar DTProto", err)
                    stopForeground(STOP_FOREGROUND_REMOVE)
                    stopSelf()
                }
        }

        return START_STICKY
    }

    override fun onDestroy() {
        stopThread?.interrupt()
        stopThread = thread(name = "dtproto-stop") {
            runCatching { client.stop() }
        }
        super.onDestroy()
    }

    private fun buildNotification(content: String) =
        NotificationCompat.Builder(this, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_vpn)
            .setContentTitle("DTProto VPN")
            .setContentText(content)
            .setOngoing(true)
            .build()

    private fun updateNotification(status: String) {
        NotificationManagerCompat.from(this).notify(
            NOTIFICATION_ID,
            buildNotification("Estado: $status")
        )
    }

    private fun toAndroidLevel(level: Int) = when (level) {
        0 -> Log.DEBUG   // logging.LevelDebug
        1 -> Log.INFO    // logging.LevelInfo
        2 -> Log.WARN    // logging.LevelWarn
        3 -> Log.ERROR   // logging.LevelError
        else -> Log.ASSERT // logging.LevelFatal
    }

    companion object {
        private const val TAG = "ProtoVpnService"
        private const val CHANNEL_ID = "dtproto_vpn"
        private const val NOTIFICATION_ID = 1001
    }
}
```

> 💡 Crie o canal de notificação `dtproto_vpn` antes de chamar `startForeground` (por exemplo, no `Application`).

## Boas práticas

- Ajuste `username` e `password` com as credenciais reais recebidas do backend.
- Quando `ip` vier preenchido do handshake, utilize o valor informado em vez do IP padrão.
- Garanta que apenas uma instância de `LibDTProto` esteja ativa por sessão; finalize sempre com `client.stop()` ao encerrar o serviço.
- Proteja quaisquer sockets criados pelo app usando `VpnService.protect()` antes de estabelecer conexões.
- Durante o desenvolvimento, monitore `adb logcat | grep ProtoVpnService` para depurar rapidamente status e logs.

Seguindo estes passos, o cliente Android DTProto ficará pronto para distribuição e integração direta no seu aplicativo.
