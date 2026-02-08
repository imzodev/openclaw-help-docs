> **Nota:** Todos los comandos se deben ejecutar desde el directorio del proyecto: `cd /home/openclaw` o donde hayas configurado el proyecto.

---

## 🐳 Gestión General de Docker Compose

### Verificar el estado de los contenedores
```bash
sudo docker compose ps
```
*Muestra si los contenedores (`openclaw-nginx`, `openclaw-gateway`, `openclaw-cli`) están en ejecución (`Up`).*

### Ver los logs en tiempo real
```bash
# Ver todos los logs
sudo docker compose logs -f

# Ver los logs de un servicio específico (muy útil para troubleshooting)
sudo docker compose logs -f openclaw-gateway
sudo docker compose logs -f nginx-proxy
```
*El `-f` sigue mostrando los logs nuevos que aparezcan. Presiona `Ctrl+C` para salir.*

### Iniciar los servicios
```bash
sudo docker compose up -d
```
*Inicia todos los contenedores en segundo plano (`-d`).*

### Detener los servicios
```bash
sudo docker compose down
```
*Detiene y elimina los contenedores. Los datos persisten en los volúmenes.*

### Reiniciar un servicio específico
```bash
sudo docker compose restart openclaw-gateway
sudo docker compose restart nginx-proxy
```
*Útil si solo necesitas reiniciar un componente sin afectar a los demás.*

### Reconstruir la imagen Docker (si actualizas el código)
```bash
sudo docker compose build --no-cache
sudo docker compose up -d
```
*Usa esto después de hacer un `git pull` para actualizar OpenClaw a una nueva versión.*

---

## 📱 Gestión de Canales de Chat

> **Prerrequisito:** Necesitas tu `OPENCLAW_GATEWAY_TOKEN`. Puedes verlo con `cat .env | grep OPENCLAW_GATEWAY_TOKEN`.

### Añadir canal de Telegram
```bash
# Reemplaza "TU_TOKEN_DE_TELEGRAM" y "TU_TOKEN_DE_GATEWAY"
docker compose exec -it openclaw-gateway node dist/index.js channels add --channel telegram --token "TU_TOKEN_DE_TELEGRAM" --token=TU_TOKEN_DE_GATEWAY
```

### Añadir canal de Slack
```bash
# Reemplaza "TU_TOKEN_DE_SLACK" y "TU_TOKEN_DE_GATEWAY"
docker compose exec -it openclaw-gateway node dist/index.js channels add --channel slack --token "TU_TOKEN_DE_SLACK" --token=TU_TOKEN_DE_GATEWAY
```

### Iniciar sesión de WhatsApp (mostrará un código QR)
```bash
# Reemplaza "TU_TOKEN_DE_GATEWAY"
# Asegúrate de haber configurado la allowlist en /root/.openclaw/openclaw.json primero
docker compose exec -it openclaw-gateway node dist/index.js channels login --channel whatsapp --token=TU_TOKEN_DE_GATEWAY
```

---

## 🔐 Gestión de Dispositivos (Emparejamiento)

### Obtener tu Token del Gateway
```bash
cat .env | grep OPENCLAW_GATEWAY_TOKEN
```

### Listar dispositivos (pendientes y aprobados)
```bash
# Reemplaza "TU_TOKEN_DE_GATEWAY"
docker compose exec -it openclaw-gateway node dist/index.js devices list --token=TU_TOKEN_DE_GATEWAY
```

### Aprobar un nuevo dispositivo (navegador, app, etc.)
```bash
# Reemplaza "REQUEST_ID" y "TU_TOKEN_DE_GATEWAY"
docker compose exec -it openclaw-gateway node dist/index.js devices approve REQUEST_ID --token=TU_TOKEN_DE_GATEWAY
```

---

## 💻 Acceso a la CLI Interactiva

### Ejecutar cualquier comando de la CLI de OpenClaw
El formato general es:
```bash
docker compose exec -it openclaw-gateway node dist/index.js [COMANDO] [ARGUMENTOS] --token=TU_TOKEN_DE_GATEWAY
```

**Ejemplos:**
*   Ver el estado de salud del gateway:
    ```bash
    docker compose exec -it openclaw-gateway node dist/index.js health --token=TU_TOKEN_DE_GATEWAY
    ```
*   Ver la configuración actual:
    ```bash
    docker compose exec -it openclaw-gateway node dist/index.js config --token=TU_TOKEN_DE_GATEWAY
    ```
*   Obtener un enlace al dashboard con token:
    ```bash
    docker compose exec -it openclaw-gateway node dist/index.js dashboard --no-open --token=TU_TOKEN_DE_GATEWAY
    ```