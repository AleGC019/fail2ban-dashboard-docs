# 🔒 Configuración SSL/TLS

## Configuración Caddy SSL

### Configuración Automática

```caddyfile
# /etc/caddy/Caddyfile
{
    # Email para Let's Encrypt
    email admin@tudominio.com
    
    # Configuración TLS avanzada
    tls {
        protocols tls1.2 tls1.3
        ciphers TLS_AES_256_GCM_SHA384 TLS_CHACHA20_POLY1305_SHA256
    }
    
    # Configuración ACME
    acme_ca https://acme-v02.api.letsencrypt.org/directory
}

alertasfail2ban.xmakuno.com {
    reverse_proxy localhost:8000
    
    # Security headers
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Frame-Options "DENY"
        X-Content-Type-Options "nosniff"
        X-XSS-Protection "1; mode=block"
    }
}
```

## Verificación SSL

### Comandos de Verificación

```bash
# Verificar certificado
openssl s_client -servername alertasfail2ban.xmakuno.com -connect alertasfail2ban.xmakuno.com:443

# Verificar fechas de expiración
echo | openssl s_client -servername alertasfail2ban.xmakuno.com -connect alertasfail2ban.xmakuno.com:443 2>/dev/null | openssl x509 -noout -dates

# Test de SSL Labs (desde web)
# https://www.ssllabs.com/ssltest/

# Verificar headers de seguridad
curl -I https://alertasfail2ban.xmakuno.com
```

### Renovación Automática

```bash
# Caddy maneja renovación automática
# Verificar logs de renovación
sudo journalctl -u caddy | grep -i "certificate"

# Forzar renovación si es necesario
sudo caddy reload --config /etc/caddy/Caddyfile
```

!!! success "Configuración Óptima"
    - TLS 1.2 y 1.3 únicamente
    - HSTS habilitado con preload
    - Renovación automática configurada
    - Grade A+ en SSL Labs