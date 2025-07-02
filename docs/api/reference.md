# 📡 API Reference

## Introducción

La documentación completa de la API está disponible de forma interactiva a través de **Swagger UI** y **ReDoc**, generada automáticamente por FastAPI.

!!! info "Acceso a la Documentación"
    Una vez que el sistema esté desplegado, puedes acceder a la documentación interactiva de la API en:
    
    - **Swagger UI**: [https://alertasfail2ban.xmakuno.com/docs](https://alertasfail2ban.xmakuno.com/docs)
    - **ReDoc**: [https://alertasfail2ban.xmakuno.com/redoc](https://alertasfail2ban.xmakuno.com/redoc)
    - **OpenAPI Schema**: [https://alertasfail2ban.xmakuno.com/openapi.json](https://alertasfail2ban.xmakuno.com/openapi.json)

## 🔗 Enlaces de Documentación

### Swagger UI (Interactiva)
[**https://alertasfail2ban.xmakuno.com/docs**](https://alertasfail2ban.xmakuno.com/docs)

!!! success "Características de Swagger UI"
    - **🧪 Pruebas en vivo**: Ejecuta requests directamente desde el navegador
    - **📋 Esquemas detallados**: Visualiza modelos de datos con ejemplos
    - **🔍 Exploración interactiva**: Navega por todos los endpoints disponibles
    - **📝 Documentación automática**: Generada desde el código fuente

### ReDoc (Documentación)
[**https://alertasfail2ban.xmakuno.com/redoc**](https://alertasfail2ban.xmakuno.com/redoc)

!!! info "Características de ReDoc"
    - **📖 Vista optimizada para lectura**: Layout diseñado para documentación
    - **🔍 Búsqueda avanzada**: Encuentra endpoints y modelos rápidamente
    - **📱 Responsive design**: Funciona perfectamente en móviles
    - **📊 Ejemplos de código**: Snippets en múltiples lenguajes

## 🚀 Endpoints Principales

### Resumen de la API

| Categoría | Endpoint | Descripción |
|-----------|----------|-------------|
| **Health** | `GET /health` | Estado del sistema y servicios |
| **Jails** | `GET /api/jails` | Lista de jails de Fail2ban |
| **Jails** | `GET /api/jails/{jail_name}` | Detalles de un jail específico |
| **Logs** | `GET /api/logs` | Consulta de logs desde Loki |
| **Logs** | `GET /api/logs/stream` | Stream de logs en tiempo real |
| **WebSocket** | `WS /ws/logs` | Conexión WebSocket para logs |

### Ejemplos Rápidos

=== "Health Check"
    ```bash
    curl https://alertasfail2ban.xmakuno.com/health
    ```
    
    ```json
    {
        "status": "healthy",
        "timestamp": "2025-01-21T10:30:00Z",
        "services": {
            "loki": "connected",
            "fail2ban": "connected"
        }
    }
    ```

=== "Lista de Jails"
    ```bash
    curl https://alertasfail2ban.xmakuno.com/api/jails
    ```
    
    ```json
    [
        {
            "name": "sshd",
            "status": "active",
            "banned_ips": ["192.168.1.100"],
            "total_banned": 5,
            "total_failed": 25
        }
    ]
    ```

=== "Logs Recientes"
    ```bash
    curl "https://alertasfail2ban.xmakuno.com/api/logs?limit=10"
    ```

## 🛠️ Desarrollo y Testing

### Entorno Local

Si estás ejecutando el sistema localmente para desarrollo:

```bash
# Documentación local
http://localhost:8000/docs      # Swagger UI
http://localhost:8000/redoc     # ReDoc
http://localhost:8000/openapi.json  # Schema
```

### Pruebas de API

```python
import requests

# Configuración base
BASE_URL = "https://alertasfail2ban.xmakuno.com"

# Test health endpoint
response = requests.get(f"{BASE_URL}/health")
print(response.json())

# Test jails endpoint
response = requests.get(f"{BASE_URL}/api/jails")
print(response.json())
```

### Cliente WebSocket

```javascript
// Conexión WebSocket para logs en tiempo real
const ws = new WebSocket('wss://alertasfail2ban.xmakuno.com/ws/logs');

ws.onmessage = function(event) {
    const logData = JSON.parse(event.data);
    console.log('Nuevo log:', logData);
};
```

## 📚 Modelos de Datos

### Principales Schemas

Los schemas completos están disponibles en la documentación interactiva, pero aquí tienes un resumen:

=== "JailStatus"
    ```json
    {
        "name": "string",
        "status": "active | inactive",
        "banned_ips": ["string"],
        "total_banned": 0,
        "total_failed": 0,
        "filter_name": "string",
        "actions": ["string"]
    }
    ```

=== "LogEntry"
    ```json
    {
        "timestamp": "2025-01-21T10:30:00Z",
        "level": "INFO | WARNING | ERROR",
        "message": "string",
        "jail": "string",
        "action": "string",
        "ip": "string"
    }
    ```

=== "HealthStatus"
    ```json
    {
        "status": "healthy | unhealthy",
        "timestamp": "2025-01-21T10:30:00Z",
        "services": {
            "loki": "connected | disconnected",
            "fail2ban": "connected | disconnected"
        },
        "version": "string"
    }
    ```

## 🔧 Configuración de Cliente

### Headers Recomendados

```bash
# Headers para requests
Content-Type: application/json
Accept: application/json
User-Agent: YourApp/1.0
```

### Rate Limiting

!!! warning "Límites de Rate"
    - **Requests por minuto**: 60 por IP
    - **Burst limit**: 10 requests
    - **WebSocket connections**: 5 por IP
    
    Si necesitas límites más altos, contacta al administrador.

### Error Handling

```json
// Formato de errores estándar
{
    "detail": "Error description",
    "status_code": 400,
    "timestamp": "2025-01-21T10:30:00Z"
}
```

## 📖 Documentación Adicional

### Enlaces Útiles

- **[Código Fuente](https://github.com/tu-usuario/aca-fail2ban-dashboard)**: Repositorio completo del proyecto
- **[FastAPI Docs](https://fastapi.tiangolo.com/)**: Documentación oficial de FastAPI
- **[Loki API](https://grafana.com/docs/loki/latest/api/)**: Documentación de Loki para consultas avanzadas

### Soporte

Si encuentras problemas con la API:

1. **Verifica el status**: `GET /health`
2. **Consulta los logs**: Documentación de [troubleshooting](../deployment/troubleshooting.md)
3. **Revisa ejemplos**: En la documentación interactiva
4. **Contacta soporte**: Issues en el repositorio de GitHub

!!! tip "Mejores Prácticas"
    - **Usa la documentación interactiva** para explorar y probar
    - **Implementa retry logic** para requests críticos
    - **Monitorea rate limits** en aplicaciones de producción
    - **Usa WebSockets** para datos en tiempo real

!!! success "¡Explora la API!"
    Visita [https://alertasfail2ban.xmakuno.com/docs](https://alertasfail2ban.xmakuno.com/docs) para comenzar a explorar todos los endpoints disponibles de forma interactiva.