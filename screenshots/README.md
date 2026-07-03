# Capturas de pantalla — VPN Client-to-Site SSL-VPN Tunnel Mode con FortiGate

Capturas del laboratorio en orden de demostración.

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](/screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, FortiGate, switch y PC1 encendidos. |
| 2 | [`02_ssl_vpn_settings.png`](/screenshots/02_ssl_vpn_settings.png) | GUI del FortiGate → `VPN → SSL-VPN Settings` mostrando el puerto `10443`, el pool de IPs y el certificado `Fortinet_Factory`. |
| 3 | [`03_ssl_web_portal.png`](/screenshots/03_ssl_web_portal.png) | GUI del FortiGate → portal `full-access` mostrando `Tunnel Mode` habilitado y `Split Tunneling` deshabilitado. |
| 4 | [`04_firewall_policy.png`](/screenshots/04_firewall_policy.png) | GUI del FortiGate → `Policy & Objects → Firewall Policy` mostrando la política `SSLVPN-to-LAN` con NAT habilitado. |
| 5 | [`05_cliente_conexion_tls.png`](/screenshots/05_cliente_conexion_tls.png) | Terminal Linux mostrando `openfortivpn` con `Tunnel is up and running` y la IP asignada. |
| 6 | [`06_ssl_vpn_clients.png`](/screenshots/06_ssl_vpn_clients.png) | `VPN → SSL-VPN Clients` mostrando `vpnuser` conectado con IP de túnel `10.212.134.200`. |
| 7 | [`07_forward_traffic_accept.png`](/screenshots/07_forward_traffic_accept.png) | `Log & Report → Forward Traffic` mostrando el tráfico del cliente hacia `20.25.37.130` con acción `Accept`. |
| 8 | [`08_ping_exitoso.png`](/screenshots/08_ping_exitoso.png) | Ping exitoso desde el cliente (Linux/Windows) hacia PC1 (`20.25.37.130`) con el túnel activo. |
| 9 | [`09_traceroute_exitoso.png`](/screenshots/09_traceroute_exitoso.png) | Traceroute desde el cliente hacia PC1 mostrando el salto a través de la interfaz de túnel. |
| 10 | [`10_vpn_events_login.png`](/screenshots/10_vpn_events_login.png) | `Log & Report → VPN Events` mostrando el evento de login exitoso de `vpnuser`. |
