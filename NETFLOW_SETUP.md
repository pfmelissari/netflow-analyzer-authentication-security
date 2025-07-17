# Configuración NetFlow - Guía Completa

Esta guía te ayudará a configurar tu router o switch para enviar datos NetFlow reales a la aplicación NetFlow Analyzer.

## 🚀 Pasos Rápidos

1. **Iniciar el servidor NetFlow** en la aplicación (puerto 2055 por defecto)
2. **Configurar tu router** con los comandos apropiados
3. **Verificar conectividad** y firewall
4. **Monitorear flujos** en tiempo real

## 📋 Configuración por Tipo de Dispositivo

### Cisco IOS/IOS-XE

```bash
# Habilitar NetFlow en interfaces
interface GigabitEthernet0/1
 ip route-cache flow

# Configurar exportación NetFlow
ip flow-export destination [IP_DE_LA_APP] 2055
ip flow-export version 5
ip flow-export source GigabitEthernet0/0

# Opcional: Configurar sampling para redes de alto tráfico
ip flow-export template timeout-rate 1
ip flow-cache timeout active 1
ip flow-cache timeout inactive 15
```

### MikroTik RouterOS

```bash
# Habilitar Traffic Flow
/ip traffic-flow
set enabled=yes interfaces=ether1,ether2

# Configurar destino NetFlow
/ip traffic-flow target
add address=[IP_DE_LA_APP] port=2055 version=5

# Verificar configuración
/ip traffic-flow print
/ip traffic-flow target print
```

### pfSense

1. Ir a **System → Advanced → Miscellaneous**
2. Marcar **Enable NetFlow**
3. Configurar:
   - **NetFlow Destination**: `[IP_DE_LA_APP]:2055`
   - **NetFlow Version**: `5`
   - **Interfaces**: Seleccionar interfaces a monitorear

### Ubiquiti EdgeRouter

```bash
# Configurar NetFlow
configure
set system flow-accounting interface eth0
set system flow-accounting netflow version 5
set system flow-accounting netflow server [IP_DE_LA_APP] port 2055
commit
save
```

### Juniper JunOS

```bash
# Configurar sampling y exportación
set forwarding-options sampling instance netflow-instance family inet rate 100
set forwarding-options sampling instance netflow-instance family inet output flow-server [IP_DE_LA_APP] port 2055
set forwarding-options sampling instance netflow-instance family inet output flow-server [IP_DE_LA_APP] version 5

# Aplicar a interfaces
set interfaces ge-0/0/0 unit 0 family inet sampling input
```

### Fortinet FortiGate

```bash
# Configurar NetFlow
config system netflow
    set collector-ip [IP_DE_LA_APP]
    set collector-port 2055
    set source-ip [IP_DEL_FORTIGATE]
    set active-flow-timeout 30
    set inactive-flow-timeout 15
end

# Habilitar en política de firewall
config firewall policy
    edit [POLICY_ID]
        set netflow-sampler enable
    next
end
```

## 🔧 Configuración de Red

### Firewall

Asegúrate de que el puerto UDP 2055 esté abierto:

```bash
# Linux (iptables)
iptables -A INPUT -p udp --dport 2055 -j ACCEPT

# Windows Firewall
netsh advfirewall firewall add rule name="NetFlow" dir=in action=allow protocol=UDP localport=2055

# pfSense/OPNsense
# Crear regla en Firewall → Rules → WAN/LAN
# Protocolo: UDP, Puerto destino: 2055
```

### Verificación de Conectividad

```bash
# Desde el router, verificar que puede alcanzar la aplicación
ping [IP_DE_LA_APP]

# Verificar que el puerto UDP esté abierto
nmap -sU -p 2055 [IP_DE_LA_APP]

# Monitorear tráfico NetFlow (en el servidor)
tcpdump -i any -n port 2055
```

## 📊 Datos Capturados

La aplicación captura y procesa los siguientes datos de cada flujo NetFlow:

- **IPs origen y destino**
- **Puertos origen y destino**
- **Protocolo** (TCP, UDP, ICMP)
- **Bytes y paquetes** transferidos
- **Timestamps** de inicio y fin de flujo
- **Flags TCP** y Type of Service (ToS)
- **Duración** del flujo

### Detección Automática

La aplicación automáticamente:

- **Identifica aplicaciones** basándose en puertos conocidos
- **Categoriza tráfico** (web, sistema, actualizaciones, media)
- **Detecta actualizaciones** de Windows, Chrome, Edge, Office
- **Genera información HTTP** para tráfico web (puertos 80/443)

## 🔍 Troubleshooting

### No se reciben flujos

1. **Verificar servidor NetFlow**
   - ¿Está iniciado el servidor en la aplicación?
   - ¿Está escuchando en el puerto correcto?

2. **Verificar configuración del router**
   - ¿Está habilitado NetFlow en las interfaces correctas?
   - ¿Está configurada la IP de destino correcta?
   - ¿Está usando NetFlow versión 5?

3. **Verificar conectividad**
   - ¿Puede el router hacer ping a la aplicación?
   - ¿Está abierto el puerto UDP 2055 en el firewall?
   - ¿Hay algún NAT o proxy en el medio?

### Flujos incompletos o erróneos

1. **Verificar versión NetFlow**
   - La aplicación solo soporta NetFlow v5
   - Verificar configuración: `show ip flow export`

2. **Verificar interfaces**
   - ¿Están configuradas todas las interfaces necesarias?
   - ¿Hay tráfico pasando por esas interfaces?

3. **Verificar logs**
   - Revisar logs del router: `show logging`
   - Revisar consola de la aplicación para errores

### Rendimiento

Para redes de alto tráfico:

```bash
# Cisco: Configurar sampling
ip flow-cache entries 65536
ip flow-cache timeout active 1
ip flow-cache timeout inactive 15

# MikroTik: Limitar interfaces
/ip traffic-flow
set cache-entries=65536 interfaces=ether1
```

## 📈 Optimización

### Para Redes Pequeñas (< 100 Mbps)
- Usar todas las interfaces sin sampling
- Timeout activo: 30 segundos
- Timeout inactivo: 15 segundos

### Para Redes Medianas (100 Mbps - 1 Gbps)
- Sampling 1:100 en interfaces principales
- Timeout activo: 5 segundos
- Timeout inactivo: 10 segundos

### Para Redes Grandes (> 1 Gbps)
- Sampling 1:1000 o mayor
- Timeout activo: 1 segundo
- Timeout inactivo: 5 segundos
- Considerar usar sFlow en lugar de NetFlow

## 🔒 Seguridad

### Recomendaciones

1. **Usar VLAN dedicada** para tráfico NetFlow
2. **Configurar ACLs** para limitar acceso al puerto NetFlow
3. **Monitorear logs** para detectar anomalías
4. **Usar autenticación** si está disponible (NetFlow v9/IPFIX)

### Ejemplo de ACL (Cisco)

```bash
# Crear ACL para NetFlow
ip access-list extended NETFLOW-ACL
 permit udp host [IP_DEL_ROUTER] host [IP_DE_LA_APP] eq 2055
 deny ip any any log

# Aplicar a interface
interface GigabitEthernet0/0
 ip access-group NETFLOW-ACL out
```

## 📞 Soporte

Si tienes problemas:

1. **Verificar logs** en la consola del navegador
2. **Revisar configuración** del router paso a paso
3. **Probar conectividad** básica (ping, telnet)
4. **Capturar tráfico** con tcpdump/Wireshark

### Comandos de Diagnóstico

```bash
# Cisco
show ip flow export
show ip cache flow

# MikroTik
/ip traffic-flow print
/ip traffic-flow target print

# Linux (en el servidor)
netstat -ulnp | grep 2055
tcpdump -i any -n port 2055 -v
```

---

**¡Listo!** Una vez configurado correctamente, deberías ver flujos NetFlow reales apareciendo en la aplicación en tiempo real.
