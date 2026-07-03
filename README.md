# VPN Client-to-Site — SSL-VPN Tunnel Mode con FortiGate

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [¿Qué es SSL-VPN en FortiGate?](#21-qué-es-ssl-vpn-en-fortigate)
   - [Tunnel Mode vs. Web Mode](#22-tunnel-mode-vs-web-mode)
   - [Arquitectura de la conexión](#23-arquitectura-de-la-conexión)
   - [Flujo de establecimiento de la conexión](#24-flujo-de-establecimiento-de-la-conexión)
   - [Client-to-Site vs. Site-to-Site](#25-client-to-site-vs-site-to-site)
   - [Parámetros del laboratorio](#26-parámetros-del-laboratorio)
3. [Documentación de la Red](#3-documentación-de-la-red)
   - [Topología](#31-topología)
   - [Tabla de Dispositivos y Direccionamiento IP](#32-tabla-de-dispositivos-y-direccionamiento-ip)
4. [Scripts de Configuración](#4-scripts-de-configuración)
   - [FortiGate — Servidor SSL-VPN](#41-fortigate--servidor-ssl-vpn)
   - [Configuración del Cliente Linux](#42-configuración-del-cliente-linux)
   - [Configuración del Cliente Windows](#43-configuración-del-cliente-windows)
   - [Configuración de PC1 (Host)](#44-configuración-de-pc1-host)
5. [Verificación de la Conexión (GUI)](#5-verificación-de-la-conexión-gui)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Consideraciones de Seguridad](#7-consideraciones-de-seguridad)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo del Laboratorio

El objetivo de este laboratorio es **implementar y verificar una VPN Client-to-Site utilizando SSL-VPN en modo túnel (Tunnel Mode) sobre un FortiGate**. A través de esta práctica se busca demostrar:

* La configuración de un servidor SSL-VPN en FortiOS mediante `vpn ssl settings` y `vpn ssl web portal`, incluyendo la autenticación de usuarios contra la base de datos local del propio FortiGate (`user local` / `user group`).
* La diferencia práctica entre **Web Mode** (proxy HTTP reescrito, sin interfaz de red real) y **Tunnel Mode** (interfaz PPP virtual con IP real, necesaria para tráfico IP genérico como `ping` y `traceroute`).
* La asignación dinámica de una IP de túnel al cliente remoto desde un pool local (`10.212.134.200 – 10.212.134.210`), permitiendo que el cliente acceda a la red interna del FortiGate como si estuviera conectado localmente.
* La verificación de conectividad desde el cliente remoto hacia la red interna (`20.25.37.128/25`) a través del túnel SSL-VPN, incluyendo la particularidad del `traceroute` sobre un túnel punto a punto.
* La verificación completa del laboratorio **exclusivamente desde la interfaz gráfica (GUI)** de FortiOS.

---

## 2. Marco Teórico

### 2.1 ¿Qué es SSL-VPN en FortiGate?

SSL-VPN es la solución de acceso remoto nativa de Fortinet, basada en **TLS/SSL** en lugar de IPSec/IKE. A diferencia de L2TP/IPSec o IKEv2, no requiere abrir puertos UDP especiales (500, 4500, 1701) ni negociar una Fase 1/Fase 2 tipo ISAKMP — todo el canal viaja sobre un único puerto TCP (por defecto 443, en este laboratorio remapeado a **10443**), lo cual la hace atravesar firewalls corporativos y NATs restrictivos con mucha más facilidad.

El cliente se autentica con **usuario y contraseña** (`user local`) contra la base de datos del propio FortiGate, y el servidor se identifica ante el cliente mediante un **certificado TLS** — en este laboratorio, el certificado autofirmado de fábrica `Fortinet_Factory`.

### 2.2 Tunnel Mode vs. Web Mode

FortiOS ofrece dos modos de operación para SSL-VPN, y elegir el correcto es crítico para este laboratorio:

| Característica | Web Mode | Tunnel Mode (este lab) |
|---|---|---|
| **Qué crea en el cliente** | Nada — el FortiGate actúa como proxy HTTP inverso | Una interfaz de red virtual real (`ppp0` en Linux, adaptador virtual en Windows) |
| **IP asignada al cliente** | Ninguna — no hay stack IP nuevo | Sí — IP del pool `tunnel-ip-pools` (`10.212.134.200-210`) |
| **Protocolos soportados** | Solo HTTP/HTTPS reescrito (acceso a portales web internos) | Cualquier protocolo IP — ICMP, TCP, UDP, lo que sea |
| **`ping` / `traceroute` desde el cliente** | ❌ Imposible — no hay interfaz de red que generar el paquete | ✅ Funciona — el tráfico sale por la interfaz de túnel real |
| **Comando de activación en FortiOS** | Comportamiento por defecto del portal | `set tunnel-mode enable` dentro de `vpn ssl web portal` |
| **Cliente necesario** | Solo navegador web | Navegador (para autenticar) + motor de túnel (FortiClient, o `openfortivpn` en Linux) |

> **Lección clave de este laboratorio:** intentar hacer `ping`/`traceroute` desde el portal Web Mode es intentar algo que **no existe a nivel de red** — el navegador nunca crea una interfaz IP, solo muestra páginas reescritas. El Tunnel Mode es obligatorio para cualquier prueba de conectividad de capa 3.

### 2.3 Arquitectura de la conexión

```
CLIENTE (Linux/Windows)                           FORTIGATE
───────────────────────────────────────────────────────────
  │                                                     │
  │  ┌───────────────────────────────────────────────┐  │
  │  │           TLS/SSL (TCP 10443)                 │  │
  │  │  ┌─────────────────────────────────────────┐  │  │
  │  │  │   Autenticación: vpnuser / MiClave123   │  │  │
  │  │  │  ┌───────────────────────────────────┐  │  │  │
  │  │  │  │  Interfaz de túnel (ppp0)         │  │  │  │
  │  │  │  │  IP: 10.212.134.200               │  │  │  │
  │  │  │  │  Peer: 169.254.2.1 (link-local)   │  │  │  │
  │  │  │  └───────────────────────────────────┘  │  │  │
  │  │  └─────────────────────────────────────────┘  │  │
  │  └───────────────────────────────────────────────┘  │
  │                                                     │

Puertos involucrados:
  TCP 10443 → Portal SSL-VPN (autenticación + establecimiento del túnel)
  Todo el tráfico IP subsiguiente viaja encapsulado dentro de esa sesión TLS
```

### 2.4 Flujo de establecimiento de la conexión

| Paso | Descripción |
|---|---|
| 1 | El cliente inicia una sesión TLS hacia `https://<IP-WAN-FortiGate>:10443` |
| 2 | El FortiGate presenta su certificado (`Fortinet_Factory`, autofirmado) — el cliente debe aceptarlo o confiar explícitamente en su huella digital |
| 3 | El cliente se autentica con usuario/contraseña (`vpnuser` / `MiClave123`), validado contra `user local` y el grupo `sslvpngroup` |
| 4 | El FortiGate asigna una IP del pool (`tunnel-ip-pools`) y negocia los parámetros PPP |
| 5 | Se crea la interfaz de túnel en el cliente (`ppp0`) con la IP asignada |
| 6 | La política de firewall `SSLVPN-to-LAN` (`ssl.root → port2`) permite el tráfico hacia la LAN interna; al ser una política *stateful* con NAT habilitado, el retorno se maneja automáticamente sin necesitar una segunda política en sentido inverso |

### 2.5 Client-to-Site vs. Site-to-Site

| Característica | Client-to-Site SSL-VPN (este lab) | Site-to-Site IPSec (labs anteriores) |
|---|---|---|
| **Extremos** | Un usuario individual + un FortiGate | Dos FortiGate/routers fijos |
| **IP del cliente** | Dinámica — asignada por el pool del servidor | Fija — configurada estáticamente en cada sitio |
| **Autenticación** | Usuario/contraseña (`user local`) | Solo autenticación de equipo (PSK) |
| **Protocolo de transporte** | TLS/SSL sobre TCP (un solo puerto) | IPSec/IKE sobre UDP 500/4500 + ESP |
| **Políticas de firewall necesarias** | Una sola (`ssl.root → port2`, stateful) | Dos, una por cada sentido del tráfico |
| **Transparencia para el host final** | Ninguna — el usuario inicia la VPN manualmente | Total — los hosts de ambas LAN no saben que existe un túnel |

### 2.6 Parámetros del laboratorio

| Parámetro | Valor | Descripción |
|---|---|---|
| **Tipo de VPN** | SSL-VPN, Tunnel Mode | Túnel TLS con interfaz IP real en el cliente |
| **Puerto del portal** | TCP 10443 | Puerto remapeado del portal SSL-VPN |
| **Certificado del servidor** | `Fortinet_Factory` | Certificado autofirmado de fábrica |
| **Usuario VPN** | `vpnuser` | Usuario local configurado en el FortiGate |
| **Contraseña VPN** | `MiClave123` | Contraseña del usuario |
| **Grupo VPN** | `sslvpngroup` | Grupo referenciado en la política de firewall |
| **Pool de IPs de túnel** | `10.212.134.200 – 10.212.134.210` | IPs asignadas dinámicamente a los clientes conectados |
| **Portal** | `full-access` | Portal con `tunnel-mode enable` y `split-tunneling disable` |
| **NAT en la política** | Habilitado (`set nat enable`) | El tráfico hacia la LAN sale con la IP de `port2`, evitando depender de rutas adicionales en los hosts finales |
| **Interfaz WAN (port1)** | `192.168.1.10/24` | Extremo público del túnel |
| **Interfaz LAN (port2)** | `20.25.37.129/25` | Gateway de la red interna |

> **Nota de seguridad:** Este laboratorio requirió forzar en el cliente `--min-tls=1.0` y `--cipher-list "DEFAULT@SECLEVEL=0"` para poder completar el handshake TLS contra el certificado y las suites de cifrado por defecto del build de evaluación de FortiOS usado. Esto es **intencionalmente débil** y solo aceptable en un entorno de laboratorio aislado — ver sección [7. Consideraciones de Seguridad](#7-consideraciones-de-seguridad).

> **Nota sobre la GUI en FortiOS 7.6.x:** a partir de FortiOS 7.4.1, y reforzado en 7.6.x, el menú `VPN → SSL-VPN` viene **oculto por defecto** en la interfaz gráfica, aunque la configuración por CLI funcione perfectamente. Es necesario habilitar explícitamente la visibilidad de la función antes de poder verificar nada por GUI:
>
> ```
> config system settings
>     set gui-sslvpn enable
> end
> ```
>
> o, de forma equivalente, desde `System → Feature Visibility → Core Features → SSL-VPN → Apply`. Una vez habilitado, el menú `VPN` expone tres nuevas entradas: `SSL-VPN Portals`, `SSL-VPN Settings` y `SSL-VPN Clients` (esta última es el monitor de sesiones activas, reemplazando al antiguo "SSL-VPN Monitor" de versiones previas).

---

## 3. Documentación de la Red

### 3.1 Topología

El laboratorio implementa una VPN Client-to-Site donde un cliente externo (Linux o Windows) se conecta al portal SSL-VPN del FortiGate. Una vez autenticado, el cliente recibe una IP del pool de túnel y puede acceder a la red interna del FortiGate (`20.25.37.128/25`).

```
      Cliente Linux/Windows                                 FortiGate
      (fuera de la red interna)                       ┌───────────────────┐
              │                                       │                   │
              │  port1: 192.168.1.10/24               │                   │
              └───────────────────────────────────────┤ port1             │
                    TLS/SSL — TCP 10443               │                   │
                    (SSL-VPN Tunnel Mode)             │  port2            │
                                                      │ 20.25.37.129/25   │
                                                      └─────────┬─────────┘
                                                                │
                                                        ┌───────┴───────┐
                                                        │      SW1      │
                                                        └───────┬───────┘
                                                                │
                                                        ┌───────┴───────┐
                                                        │      PC1      │
                                                        │ 20.25.37.130  │
                                                        └───────────────┘
                                                         20.25.37.128/25

  ══════════════════════════════════════════════════════════════════
  Flujo de la conexión SSL-VPN Client-to-Site:
    1. El cliente abre sesión TLS hacia 192.168.1.10:10443.
    2. Se autentica con usuario/contraseña: vpnuser / MiClave123.
    3. El FortiGate asigna una IP del pool: 10.212.134.200.
    4. Se crea la interfaz de túnel (ppp0) en el cliente.
    5. El cliente hace ping/traceroute hacia 20.25.37.129 (gateway
       LAN del FortiGate) y hacia 20.25.37.130 (PC1) a través
       del túnel — el tráfico sale cifrado por TLS y es entregado
       en texto plano dentro de la red interna vía la política
       SSLVPN-to-LAN con NAT habilitado.
  ══════════════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Rol |
|---|---|---|---|---|---|
| **FortiGate** | Fortinet FortiGate-VM (KVM) | port1 | 192.168.1.10 | /24 | WAN — extremo del portal SSL-VPN |
| | | port2 | 20.25.37.129 | /25 | Gateway LAN — red interna |
| | | ssl.root (virtual) | — | — | Interfaz lógica del túnel SSL-VPN |
| **SW1** | Switch L2 | — | — | — | Conmutación LAN interna |
| **PC1** | Host Linux / VPC | eth0 | 20.25.37.130 | /25 | Host en la red interna del FortiGate |
| **Cliente** | Linux (Ubuntu) / Windows | eth0 (física) | Dinámica (según red externa) | — | Cliente SSL-VPN |
| | | ppp0 / adaptador virtual (VPN activa) | 10.212.134.200 | /32 | IP asignada por el pool de túnel |

---

## 4. Scripts de Configuración

### 4.1 FortiGate — Servidor SSL-VPN

```fortios
# ══════════════════════════════════════════════════════
# FortiGate | VPN Client-to-Site — SSL-VPN Tunnel Mode
# Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
# ══════════════════════════════════════════════════════

# ─── PASO 1: Interfaces físicas ────────────────────────
config system interface
    edit "port1"
        set mode static
        set ip 192.168.1.10 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "port2"
        set mode static
        set ip 20.25.37.129 255.255.255.128
        set allowaccess ping http https ssh
        set role lan
        unset device-identification
    next
end

# ══════════════════════════════════════════════════════
# PASO 2: Pool de IPs para los clientes SSL-VPN
# ══════════════════════════════════════════════════════
config firewall address
    edit "SSLVPN_TUNNEL_ADDR1"
        set type iprange
        set start-ip 10.212.134.200
        set end-ip 10.212.134.210
    next
end

# ══════════════════════════════════════════════════════
# PASO 3: Usuario y grupo VPN — autenticación local
# ══════════════════════════════════════════════════════
config user local
    edit "vpnuser"
        set type password
        set passwd MiClave123
    next
end

config user group
    edit "sslvpngroup"
        set member "vpnuser"
    next
end

# ══════════════════════════════════════════════════════
# PASO 4: Configuración global del portal SSL-VPN
# ══════════════════════════════════════════════════════
config vpn ssl settings
    set servercert "Fortinet_Factory"
    set tunnel-ip-pools "SSLVPN_TUNNEL_ADDR1"
    set port 10443
    set source-interface "port1"
    set source-address "all"
    set default-portal "full-access"
end

# ══════════════════════════════════════════════════════
# PASO 5: Portal — habilita Tunnel Mode (paso crítico)
# ══════════════════════════════════════════════════════
# tunnel-mode enable es lo que crea la interfaz de red real
# en el cliente — sin esto, el portal opera solo en Web Mode
# y ping/traceroute nunca van a funcionar.
config vpn ssl web portal
    edit "full-access"
        set tunnel-mode enable
        set ip-pools "SSLVPN_TUNNEL_ADDR1"
        set split-tunneling disable
    next
end

# ══════════════════════════════════════════════════════
# PASO 6: Política de firewall — única, stateful
# ══════════════════════════════════════════════════════
# nat enable hace que el tráfico hacia la LAN salga con la
# IP de port2 (20.25.37.129) en vez de la IP del túnel
# (10.212.134.200) — evita depender de rutas adicionales
# en hosts finales sin soporte de ruteo estático (ej. VPCS).
config firewall policy
    edit 3
        set name "SSLVPN-to-LAN"
        set srcintf "ssl.root"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set groups "sslvpngroup"
        set nat enable
        set logtraffic all
    next
end

# ─── PASO 7: Logging — necesario para Forward Traffic ──
config log memory setting
    set status enable
end
```

### 4.2 Configuración del Cliente Linux

En distribuciones basadas en Debian/Ubuntu, el cliente recomendado es `openfortivpn` (ligero, estándar de facto para SSL-VPN de Fortinet en Linux, no requiere FortiClient completo):

```bash
sudo apt update
sudo apt install openfortivpn
```

**Primer intento — obtener la huella digital del certificado:**

```bash
sudo openfortivpn 192.168.1.10:10443 -u vpnuser -p MiClave123
```

Al ser un certificado autofirmado, el cliente rechaza la conexión y muestra el `sha256 digest` del certificado — se copia ese valor para el siguiente paso.

**Conexión definitiva, confiando en el certificado y forzando compatibilidad TLS con el build de evaluación:**

```bash
sudo openfortivpn 192.168.1.10:10443 -u vpnuser \
  --trusted-cert 286efc05555bc05560acbbb14928b35b93b7da0e6a47e1a6701bc983704f1190 \
  --min-tls=1.0 --cipher-list "DEFAULT@SECLEVEL=0"
```

**Verificar la interfaz de túnel una vez conectado:**

```bash
ip addr show ppp0
```

Debe mostrar una IP del rango `10.212.134.200 – 10.212.134.210`.

**Probar conectividad:**

```bash
ping -c 4 20.25.37.129
ping -c 4 20.25.37.130
```

**Traceroute — usando ICMP explícitamente:**

```bash
traceroute -I 20.25.37.130
```

> **Nota importante:** el `traceroute` por defecto en Linux usa paquetes UDP, los cuales **VPCS (el simulador de hosts de PNETLab) no responde correctamente**, generando un traceroute vacío o incompleto. Usar la bandera `-I` fuerza el uso de ICMP (el mismo protocolo del `ping`), que sí es soportado. Además, es normal y esperado que el primer salto del traceroute aparezca como `* * *` — corresponde a la IP link-local del extremo remoto de la interfaz PPP (`169.254.2.1`), la cual no genera respuesta ICMP "Time Exceeded" en ningún cliente SSL-VPN, sea `openfortivpn` o FortiClient.

### 4.3 Configuración del Cliente Windows

Windows no trae soporte nativo para SSL-VPN de Fortinet — se requiere **FortiClient VPN** (la versión standalone gratuita, sin necesidad de licencia EMS):

**Paso 1 — Instalar FortiClient VPN** desde el sitio oficial de Fortinet.

**Paso 2 — Agregar la conexión VPN:**

| Campo | Valor |
|---|---|
| **Tipo de conexión** | SSL-VPN |
| **Nombre de la conexión** | `VPN-ITLA-SSL` |
| **Descripción** | Cualquiera |
| **Remote Gateway** | `192.168.1.10` |
| **Puerto personalizado** | `10443` |
| **Nombre de usuario** | `vpnuser` |

**Paso 3 — Conectar** e introducir la contraseña `MiClave123`. Si aparece advertencia de certificado autofirmado, aceptar/confiar (esperado en un entorno de laboratorio).

**Verificar IP asignada (cmd):**

```cmd
ipconfig /all
```

Buscar el adaptador virtual de FortiClient — debe mostrar una IP del rango `10.212.134.200 – 10.212.134.210`.

**Probar conectividad:**

```cmd
ping 20.25.37.130
tracert 20.25.37.130
```

> En Windows, `tracert` usa ICMP por defecto, por lo que no se necesita ninguna bandera adicional como en Linux.

### 4.4 Configuración de PC1 (Host)

**PC1 — LAN interna del FortiGate (`20.25.37.128/25`)**

```bash
ip 20.25.37.130 255.255.255.128 20.25.37.129
```

> **Nota:** Si el host es una VPC de PNETLab se usa el comando `ip` como se muestra. Si es una máquina Linux real se configura con `ip addr add` e `ip route add default via`.

---

## 5. Verificación de la Conexión (GUI)

Toda la verificación de este laboratorio se realiza desde la **interfaz gráfica (GUI)** de FortiOS. En FortiOS 7.6.x, el menú `VPN → SSL-VPN` está oculto por defecto — antes de continuar, habilitar su visibilidad con `config system settings / set gui-sslvpn enable` (ver nota en la sección 2.6).

### 5.1 VPN → SSL-VPN Clients

Este es el monitor de sesiones activas (reemplaza al antiguo "SSL-VPN Monitor" de versiones previas de FortiOS). Muestra el usuario `vpnuser` conectado en tiempo real, con:

* La IP de túnel asignada (`10.212.134.200`).
* El tiempo de conexión activo.
* Los bytes/paquetes enviados y recibidos — deben subir mientras hay tráfico cruzando el túnel.

> Esta es la prueba más directa de que la autenticación y la negociación del túnel se completaron correctamente. Si esta tabla aparece con **"No results"**, significa que no hay ningún cliente conectado en este momento — es el estado normal cuando el túnel `openfortivpn` no está corriendo, no un error de configuración.

### 5.2 VPN → SSL-VPN Settings

Confirma visualmente la configuración activa: el pool de IPs asignado, el puerto del portal (`10443`), el certificado en uso (`Fortinet_Factory`) y el portal por defecto (`full-access`).

### 5.2.1 VPN → SSL-VPN Portals

Confirma visualmente que el portal `full-access` tiene `Tunnel Mode` habilitado y `Split Tunneling` deshabilitado, tal como se configuró por CLI.

### 5.3 Log & Report → Forward Traffic

Después de generar tráfico (ping o traceroute desde el cliente), se deben observar entradas con:

* `srcintf: ssl.root` → `dstintf: port2`.
* IP origen `10.212.134.200` (o `20.25.37.129` si el NAT ya tradujo, según el punto de captura).
* IP destino `20.25.37.130`.
* Acción **Accept**.

> Esto confirma que la política `SSLVPN-to-LAN` está dejando pasar el tráfico específico del túnel, no solo que el usuario está autenticado.

### 5.4 Log & Report → VPN Events

Muestra los eventos de login/logout de SSL-VPN — útil para confirmar que la autenticación de `vpnuser` fue exitosa y detectar intentos fallidos si los hubiera.

### 5.5 Dashboard → Network (widget de VPN)

Vista rápida que resume los túneles/usuarios SSL-VPN activos, sin tener que navegar a `Monitor → SSL-VPN Monitor` cada vez.

### 5.6 Tabla de comprobaciones GUI

| Sección GUI | Dónde | Qué confirma |
|---|---|---|
| `VPN → SSL-VPN Clients` | FortiGate | Usuario conectado, IP de túnel asignada, contadores de tráfico activos (vacío si no hay sesión activa) |
| `VPN → SSL-VPN Settings` | FortiGate | Configuración del portal, pool y certificado en uso |
| `VPN → SSL-VPN Portals` | FortiGate | Confirma `Tunnel Mode` habilitado y `Split Tunneling` deshabilitado en el portal `full-access` |
| `Log & Report → Forward Traffic` | FortiGate | Tráfico real cruzando `ssl.root → port2` con acción `Accept` |
| `Log & Report → VPN Events` | FortiGate | Eventos de autenticación exitosa/fallida del usuario SSL-VPN |
| `Dashboard → Network` (widget VPN) | FortiGate | Vista rápida del estado general de los túneles activos |
| `ip addr show ppp0` / `ipconfig /all` | Cliente | Confirma la IP asignada del pool de túnel |
| `ping` / `traceroute -I` (Linux) / `tracert` (Windows) | Cliente | Conectividad extremo a extremo real hacia la LAN interna |

---

## 6. Capturas de Pantalla

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, FortiGate, switch y PC1 encendidos. |
| 2 | [`02_ssl_vpn_settings.png`](screenshots/02_ssl_vpn_settings.png) | GUI del FortiGate → `VPN → SSL-VPN Settings` mostrando el puerto `10443`, el pool de IPs y el certificado `Fortinet_Factory`. |
| 3 | [`03_ssl_web_portal.png`](screenshots/03_ssl_web_portal.png) | GUI del FortiGate → portal `full-access` mostrando `Tunnel Mode` habilitado y `Split Tunneling` deshabilitado. |
| 4 | [`04_firewall_policy.png`](screenshots/04_firewall_policy.png) | GUI del FortiGate → `Policy & Objects → Firewall Policy` mostrando la política `SSLVPN-to-LAN` con NAT habilitado. |
| 5 | [`05_cliente_conexion_tls.png`](screenshots/05_cliente_conexion_tls.png) | Terminal Linux mostrando `openfortivpn` con `Tunnel is up and running` y la IP asignada. |
| 6 | [`06_ssl_vpn_clients.png`](screenshots/06_ssl_vpn_clients.png) | `VPN → SSL-VPN Clients` mostrando `vpnuser` conectado con IP de túnel `10.212.134.200`. |
| 7 | [`07_forward_traffic_accept.png`](screenshots/07_forward_traffic_accept.png) | `Log & Report → Forward Traffic` mostrando el tráfico del cliente hacia `20.25.37.130` con acción `Accept`. |
| 8 | [`08_ping_exitoso.png`](screenshots/08_ping_exitoso.png) | Ping exitoso desde el cliente (Linux/Windows) hacia PC1 (`20.25.37.130`) con el túnel activo. |
| 9 | [`09_traceroute_exitoso.png`](screenshots/09_traceroute_exitoso.png) | Traceroute desde el cliente hacia PC1 mostrando el salto a través de la interfaz de túnel. |
| 10 | [`10_vpn_events_login.png`](screenshots/10_vpn_events_login.png) | `Log & Report → VPN Events` mostrando el evento de login exitoso de `vpnuser`. |

---

## 7. Consideraciones de Seguridad

### 7.1 Debilidades de esta configuración

* **Certificado autofirmado (`Fortinet_Factory`):** el cliente no puede verificar la identidad real del servidor mediante una cadena de confianza pública — solo confía porque se le indicó explícitamente el hash del certificado (`--trusted-cert`). En producción, esto expone a un atacante en posición de MITM a suplantar el portal si logra interceptar la primera conexión.
* **Compatibilidad TLS forzada a la baja (`--min-tls=1.0`, `--cipher-list "DEFAULT@SECLEVEL=0"`):** fue necesario relajar deliberadamente la seguridad del canal TLS para completar el handshake contra las suites de cifrado del build de evaluación de FortiOS. TLS 1.0 y las suites permitidas bajo `SECLEVEL=0` incluyen algoritmos considerados obsoletos e inseguros (por ejemplo, RC4 o cifrados sin autenticación fuerte), vulnerables a ataques como BEAST o POODLE.
* **Contraseña en texto plano en el comando (`-p MiClave123`):** expone la credencial en el historial de shell y en la lista de procesos (`ps aux`) mientras el comando corre. `openfortivpn` ya lo advierte explícitamente al ejecutarse.
* **`split-tunneling disable`:** todo el tráfico del cliente, incluyendo el que no va dirigido a la red interna, se enruta por el túnel — esto es intencional en este laboratorio para forzar todas las pruebas por la VPN, pero en producción aumenta la carga del servidor VPN y expone más tráfico del usuario al inspeccionamiento del lado corporativo.

### 7.2 Migración recomendada para producción

```fortios
# Configuración SSL-VPN endurecida
config vpn ssl settings
    set servercert "CA_Certificada_Real"     ! Certificado firmado por una CA de confianza pública o corporativa
    set tunnel-ip-pools "SSLVPN_TUNNEL_ADDR1"
    set port 443
    set source-interface "port1"
    set source-address "all"
    set default-portal "full-access"
end

config vpn ssl web portal
    edit "full-access"
        set tunnel-mode enable
        set ip-pools "SSLVPN_TUNNEL_ADDR1"
        set split-tunneling enable          ! Solo el tráfico hacia la LAN corporativa pasa por el túnel
    next
end
```

* **Certificado firmado por una CA real** (pública o PKI corporativa interna) — elimina la necesidad de que el cliente "confíe a ciegas" en un hash.
* **TLS 1.2 o 1.3 únicamente**, con suites de cifrado modernas (AES-GCM, ChaCha20-Poly1305) — no forzar downgrades de seguridad como se hizo en este laboratorio.
* **Autenticación multifactor (MFA)** además de usuario/contraseña, disponible de forma nativa en FortiOS mediante FortiToken.
* **Split tunneling habilitado**, limitando explícitamente qué subredes viajan por el túnel — reduce superficie de exposición y mejora el rendimiento del cliente.

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](https://youtu.be/9yItletRNKg)**

**Duración:** 7:10

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Configuración del FortiGate mostrando los bloques de interfaces, SSL-VPN settings, portal y política de firewall.
* ✅ Conexión desde el cliente (Linux u openfortivpn / Windows con FortiClient) mostrando el túnel establecido.
* ✅ `Monitor → SSL-VPN Monitor` mostrando el usuario conectado con su IP de túnel.
* ✅ `Log & Report → Forward Traffic` mostrando el tráfico permitido hacia la LAN interna.
* ✅ Ping exitoso desde el cliente hacia PC1.
* ✅ Traceroute exitoso desde el cliente hacia PC1 (con la bandera `-I` en Linux, si aplica).

---

## 9. Referencias

* Fortinet Inc. (2024). *FortiOS Administration Guide — SSL VPN*.
* Fortinet Inc. (2024). *FortiOS Handbook — SSL VPN Tunnel Mode vs. Web Mode*.
* Rescorla, E. (2018). *RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3*. IETF.
* Dierks, T. & Rescorla, E. (2008). *RFC 5246 — The Transport Layer Security (TLS) Protocol Version 1.2*. IETF.
* openfortivpn Project. (2024). *openfortivpn — Documentation and Man Pages*. GitHub.
* Frankel, S. et al. (2005). *NIST SP 800-77 — Guide to IPsec VPNs*. National Institute of Standards and Technology.
