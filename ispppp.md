¡Oído, Luími! Aquí la data fresca de lo que están haciendo otros ISPs con tu mismo dolor de cabeza. Vamos al grano, en pasos:

### 1. Ratio CG-NAT: No te pases de 128 clientes por IP pública
- Nfware y otros ISPs recomiendan **máximo 128 suscriptores por IPv4 pública** para evitar que Meta/Google te marquen como "spam" por tráfico compartido [[5]].
- Con 15 IPs públicas y 2000 clientes, estás en ~133/cliente por IP. **Baja a 100-120 máximo** si puedes, o separa tráfico crítico (Facebook, WhatsApp) a IPs menos cargadas.

### 2. Asignación determinística de puertos (clave para logging y rate limits)
- En MikroTik, usa el script de `addNatRules` que asigna rangos fijos de puertos por cliente (ej: 10000-19999 para 100.64.0.1) [[72]].
- **Ventaja**: Si Meta bloquea una IP:puerto, puedes rastrear exactamente qué cliente fue, sin tener que loguear cada conexión (ahorra CPU y disco).
- Configura `to-ports` por protocolo (TCP/UDP separado) y evita la regla "genérica" sin puertos si no es estrictamente necesario.

### 3. IPv6 nativo YA para tráfico de Meta
- Tienes 65k+ IPv6 públicas: **prioriza IPv6 en tu DNS local** para dominios de Meta (`facebook.com`, `instagram.com`, `fbcdn.net`).
- Muchos servicios de Meta ya soportan IPv6 nativo y evitan el CG-NAT IPv4, lo que reduce drásticamente los rate limits compartidos [[50]][[66]].
- En MikroTik, asegúrate de que el firewall permita salida IPv6 sin NAT y que los clientes reciban /64 o /56 via DHCPv6-PD.

### 4. DNS cache: Ajuste fino para dominios de Meta
- Tu cache de 100M peticiones/72h está brutal, pero **no caches agresivamente los TTLs de dominios dinámicos de Meta** (ej: `edge-mqtt.facebook.com`).
- Usa reglas en tu DNS local para:
  - TTL largo (24h) solo para dominios estáticos (`static.xx.fbcdn.net`).
  - TTL corto (5-15 min) para dominios de API/CDN dinámicos.
- Esto evita que clientes reciban IPs obsoletas que Meta rota frecuentemente para balanceo/seguridad [[80]].

### 5. Workaround inmediato para "Facebook se friza"
- Agrega en MikroTik una regla de mangle para marcar tráfico a puertos 443/80 de dominios de Meta y enrutarlo por las IPs públicas menos usadas:
```bash
/ip firewall mangle
add chain=prerouting dst-address-list=meta_domains action=mark-routing new-routing-mark=meta_route
/ip route
add dst-address=0.0.0.0/0 gateway=<IP_publica_1> routing-mark=meta_route
```
- Lista `meta_domains` con los dominios clave de Meta (puedes generarla desde tu DNS cache).

### 6. Logging mínimo pero útil (para cumplir con la ley sin matar el router)
- No loguees cada conexión de CG-NAT. En su lugar, usa el método determinístico + logging solo para eventos de error o puertos específicos [[72]].
- Configura syslog-ng en un servidor externo para guardar solo: `timestamp, src_ip_private, dst_ip_public, src_port, dst_port, public_ip_used`.

### Bonus: Si el problema persiste, prueba esto
- **Endpoint-independent NAT** en MikroTik para UDP (mejora conexiones P2P y juegos) [[20]].
- **Hairpin NAT** si tienes servidores locales que los clientes acceden vía IP pública.
- Considera **NAT64+DNS64** si quieres forzar tráfico IPv6-only para clientes modernos (pero prueba primero en un grupo pequeño).

Si después de esto Meta sigue jodiendo, el problema puede estar en su lado (rate limit global por ASN). En ese caso, contacta a Meta for Developers y reporta el rango de IPs públicas que usas para CG-NAT.

¡Éxito con esa red, campeón! Si necesitas el script completo de CG-NAT para MikroTik, avísame y te lo paso en un toque. 🔧🚀
