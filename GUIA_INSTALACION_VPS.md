# Guía Definitiva: Instalar OpenClaw en VPS con Docker, Dominio y HTTPS

Esta guía detalla el proceso para instalar OpenClaw en un VPS (DigitalOcean) usando Docker, exponerlo de forma segura a través de tu propio dominio con HTTPS, y conectar los canales de Telegram, Slack y WhatsApp.

## Fase 1: Preparación del VPS y Configuración Inicial

### Paso 1: Conectar al VPS y Actualizar el Sistema
Conéctate a tu VPS via SSH y actualiza la lista de paquetes del sistema.

```bash
# Conéctate a tu VPS
ssh -i tu-clave-ssh root@TU_IP_VPS

# Actualiza el sistema
sudo apt-get update && sudo apt-get upgrade -y
```

### Paso 2: Instalar Docker y Docker Compose
Instala Docker y el plugin de Docker Compose. Usaremos la versión con espacio (`docker compose`), que es la moderna.

```bash
# Instalar Docker
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install -y docker-ce

# Instalar el plugin de Docker Compose
sudo apt-get install -y docker-compose-plugin
```

### Paso 3: Configurar tu Dominio
> **Nota:** Este paso se realiza fuera del VPS, en el panel de control de tu registrador de dominios (GoDaddy, Namecheap, etc.).

1.  Ve a la configuración de DNS de tu dominio.
2.  Crea un **Registro A** con la siguiente información:
    *   **Tipo/Host:** `@` (o `tu-dominio.com`)
    *   **Valor/Destino:** `TU_IP_VPS` 
3.  Espera a que el DNS se propague. Puedes verificarlo con `ping tu-dominio.com` desde tu ordenador local; debería responder con la IP de tu VPS.

## Fase 2: Instalación Oficial de OpenClaw con Docker

### Paso 4: Clonar el Repositorio y Preparar Permisos
> **¡Paso Crítico!** Esto evita el error `permission denied, mkdir`.

```bash
# Navega al directorio home y clona el repositorio
cd /home
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# CREA Y DA PERMISOS AL DIRECTORIO DE CONFIGURACIÓN ANTES DE EJECUTAR EL SCRIPT
sudo mkdir -p /root/.openclaw
sudo chown -R 1000:1000 /root/.openclaw
```

### Paso 5: Ejecutar el Script de Instalación Oficial
El script `docker-setup.sh` construirá la imagen y te guiará a través de la configuración inicial.

```bash
./docker-setup.sh
```

Durante el asistente de configuración (onboarding), elige estas opciones **importantes**:
*   `I understand this is powerful and inherently risky. Continue?` -> `Yes` 
*   `Onboarding mode` -> `Manual` 
*   `What do you want to set up?` -> `Local gateway (this machine)` 
*   `Model/auth provider` -> Elige tu proveedor (ej. Z.AI, Claude) y proporciona tu API Key.
*   `Gateway bind` -> **`lan`** (Esto es crucial para que sea accesible por el proxy).
*   `Gateway auth` -> `Token` 
*   `Configure chat channels now?` -> `No` (lo haremos de forma controlada más tarde)
*   `Configure skills now?` -> `No` 

Al finalizar, el script habrá creado un archivo `.env` en `/home/openclaw`.

## Fase 3: Configuración para Dominio y HTTPS (Proxy Inverso)

### Paso 6: Detener Servicios y Limpiar
Detén los servicios actuales y deshabilita cualquier servidor web que pueda estar usando los puertos 80/443.

```bash
# Detener los contenedores de OpenClaw
sudo docker compose down

# Verificar si hay otro servicio usando el puerto 80
sudo ss -tulpn | grep :80

# Si ves un servicio (ej. nginx, apache2), deténlo y desactívalo:
# sudo systemctl stop nginx
# sudo systemctl disable nginx
```

### Paso 7: Modificar `docker-compose.yml` 
Reemplaza todo el contenido de tu archivo `docker-compose.yml` con este. Este añade el proxy Nginx y configura los servicios para la red interna.

```bash
nano docker-compose.yml
```

**Pega este contenido:**
```yaml
services:
  # Servicio Nginx como proxy inverso
  nginx-proxy:
    image: nginx:alpine
    container_name: openclaw-nginx
    ports:
      - "80:80"   # Para redirección a HTTPS
      - "443:443" # Puerto principal HTTPS
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - /var/lib/letsencrypt:/var/lib/letsencrypt:ro
    depends_on:
      - openclaw-gateway
    restart: unless-stopped

  # Servicio Gateway de OpenClaw
  openclaw-gateway:
    image: ${OPENCLAW_IMAGE:-openclaw:local}
    environment:
      HOME: /home/node
      TERM: xterm-256color
      # El archivo .env creado por el script poblará estas variables
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      ZAI_API_KEY: ${ZAI_API_KEY}
      # Añade aquí otras claves de API que configuraste (CLAUDE_AI_SESSION_KEY, etc.)
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
    # Expone el puerto solo a la red interna de Docker
    expose:
      - "18789"
    init: true
    restart: unless-stopped

  # Servicio CLI de OpenClaw
  openclaw-cli:
    image: ${OPENCLAW_IMAGE:-openclaw:local}
    environment:
      HOME: /home/node
      TERM: xterm-256color
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      BROWSER: echo
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
    stdin_open: true
    tty: true
    init: true
    entrypoint: ["node", "dist/index.js"]
```
Guarda y sal del editor (`Ctrl+X`, `Y`, `Enter`).

### Paso 8: Crear la Configuración de Nginx
Crea el archivo de configuración para Nginx, que incluye soporte para WebSockets.

```bash
mkdir nginx
nano nginx/default.conf
```

**Pega este contenido**, reemplazando `tu-dominio.com`:
```nginx
# Redirigir todo el tráfico HTTP a HTTPS
server {
    listen 80;
    server_name tu-dominio.com;

    location / {
        return 301 https://$host$request_uri;
    }
}

# Bloque principal para la conexión HTTPS
server {
    listen 443 ssl http2;
    server_name tu-dominio.com;

    # Certificados SSL (los obtendremos en el siguiente paso)
    ssl_certificate /etc/letsencrypt/live/tu-dominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tu-dominio.com/privkey.pem;

    # Mejoras de seguridad SSL
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    location / {
        # Reenviar tráfico al servicio OpenClaw gateway
        proxy_pass http://openclaw-gateway:18789;
        
        # Cabeceras específicas para WebSockets (¡MUY IMPORTANTE!)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Cabeceras estándar
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts para conexiones de larga duración
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }
}
```
Guarda y sal.

### Paso 9: Obtener el Certificado SSL
Usa Certbot para obtener un certificado gratuito de Let's Encrypt.

```bash
# Instalar Certbot
sudo apt-get install -y certbot python3-certbot-nginx

# Obtener el certificado para tu dominio
sudo certbot certonly --nginx -d tu-dominio.com
```
Sigue las instrucciones en pantalla.

### Paso 10: Configurar el Firewall
En tu panel de control de DigitalOcean, ve a **Networking -> Firewalls** y añade una regla de entrada (Inbound Rule):
*   **Protocolo:** `TCP` 
*   **Rango de Puertos:** `80, 443` 
*   **Origen:** `Anywhere` 

### Paso 11: Iniciar Todos los Servicios
Con todo configurado, inicia la pila completa.

```bash
sudo docker compose up -d
```

Verifica que los tres contenedores (`openclaw-nginx`, `openclaw-openclaw-gateway-1`, `openclaw-openclaw-cli-1`) estén en estado `Up` con `sudo docker compose ps`.

## Fase 4: Finalización y Conexión de Dispositivos y Canales

### Paso 12: Acceder y Emparejar tu Navegador
1.  Abre tu navegador y ve a `https://tu-dominio.com`.
2.  Verás un error `disconnected (1008): pairing required`. Esto es normal.
3.  Obtén tu `OPENCLAW_GATEWAY_TOKEN` del archivo `.env`:
    ```bash
    cat .env | grep OPENCLAW_GATEWAY_TOKEN
    ```
4.  **Usa el método correcto para listar y aprobar dispositivos** (ejecutando el comando dentro del contenedor gateway):
    ```bash
    # Lista los dispositivos pendientes (reemplaza TU_TOKEN_DE_GATEWAY)
    docker compose exec -it openclaw-gateway node dist/index.js devices list --token=TU_TOKEN_DE_GATEWAY

    # Aprueba el dispositivo usando el Request ID (reemplaza REQUEST_ID y TU_TOKEN_DE_GATEWAY)
    docker compose exec -it openclaw-gateway node dist/index.js devices approve REQUEST_ID --token=TU_TOKEN_DE_GATEWAY
    ```
5.  Recarga la página de tu navegador. ¡Debería conectarse correctamente!

### Paso 13: Conectar Canales de Chat (Telegram, Slack, WhatsApp)

> **Prerrequisito General:** Necesitarás el `OPENCLAW_GATEWAY_TOKEN` que generaste durante la instalación. Puedes verlo en cualquier momento con `cat .env | grep OPENCLAW_GATEWAY_TOKEN`.

#### A) Conectar Telegram

1.  **Crear un Bot en Telegram:**
    *   Abre Telegram y busca al usuario `@BotFather`.
    *   Envía el comando `/newbot` y sigue las instrucciones para darle un nombre y un nombre de usuario a tu bot.
    *   BotFather te dará un **HTTP API Token**. Cópialo.

2.  **Conectar el Bot a OpenClaw:**
    ```bash
    # Reemplaza TU_TOKEN_DE_TELEGRAM y TU_TOKEN_DE_GATEWAY
    docker compose exec -it openclaw-gateway node dist/index.js channels add --channel telegram --token "TU_TOKEN_DE_TELEGRAM" --token=TU_TOKEN_DE_GATEWAY
    ```

#### B) Conectar Slack

1.  **Crear una App de Slack:**
    *   Ve a [api.slack.com/apps](https://api.slack.com/apps) y haz clic en "Create New App" -> "From scratch".
    *   Dale un nombre y elige el Workspace de Slack donde quieres instalar el bot.
    *   En la sección "OAuth & Permissions", añade los siguientes "Bot Token Scopes": `app_mentions:read`, `channels:history`, `chat:write`, `im:history`, `im:read`, `im:write`.
    *   Instala la app en tu workspace. Se te proporcionará un **Bot User OAuth Token** (empieza con `xoxb-`). Cópialo.

2.  **Conectar el Bot a OpenClaw:**
    ```bash
    # Reemplaza TU_TOKEN_DE_SLACK y TU_TOKEN_DE_GATEWAY
    docker compose exec -it openclaw-gateway node dist/index.js channels add --channel slack --token "TU_TOKEN_DE_SLACK" --token=TU_TOKEN_DE_GATEWAY
    ```

#### C) Conectar WhatsApp (con Allowlist de Seguridad)

> **¡Paso de Seguridad Crítico!** Configura una lista de permitidos (`allowlist`) para que solo los números que tú autorices puedan interactuar con tu OpenClaw.

1.  **Configurar la Allowlist en `openclaw.json`:**
    ```bash
    nano /root/.openclaw/openclaw.json
    ```
    Añade el siguiente bloque dentro del objeto JSON principal, asegurándote de que la sintaxis sea correcta (comas incluidas).
    ```json
    {
      // ... otras configuraciones existentes ...
    
      "channels": {
        "whatsapp": {
          "dmPolicy": "allowlist",
          "allowFrom": ["+15551234567", "+5491112345678"]
        }
      },
    
      // ... otras configuraciones existentes ...
    }
    ```
    Reemplaza los números de teléfono con los tuyos en formato internacional y guarda el archivo.

2.  **Iniciar el Proceso de Login en WhatsApp:**
    ```bash
    # Reemplaza TU_TOKEN_DE_GATEWAY
    docker compose exec -it openclaw-gateway node dist/index.js channels login --channel whatsapp --token=TU_TOKEN_DE_GATEWAY
    ```
    *   El comando mostrará un **código QR** en tu terminal.
    *   Abre WhatsApp en tu teléfono, ve a `Ajustes` > `Aparatos vinculados` > `Vincular dispositivo` y escanea el código QR.

3.  **Reiniciar los Servicios de OpenClaw:**
    ```bash
    sudo docker compose down
    sudo docker compose up -d
    ```

¡Con esto, tu OpenClaw estará completamente operativo, seguro y conectado a tus canales de comunicación preferidos!
