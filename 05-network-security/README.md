# 🔥 Laboratorio 05 — Firewall, ACLs y NAT con pfSense

![pfSense](https://img.shields.io/badge/Firewall-pfSense_2.8.1-blue?style=for-the-badge)
![VMware](https://img.shields.io/badge/Platform-VMware-607078?style=for-the-badge&logo=vmware)
![Nmap](https://img.shields.io/badge/Tool-Nmap_7.95-blue?style=for-the-badge)
![IIS](https://img.shields.io/badge/Server-IIS_10.0-0078D6?style=for-the-badge&logo=windows)

---

## 🎬 Escenario

Una empresa pequeña tiene un servidor web interno que necesita ser accesible
desde internet. El administrador abre el puerto 80 directamente en el router
y apunta hacia el servidor. Problema resuelto, ¿verdad?

No exactamente. Ahora cualquier persona en internet puede ver la IP interna
del servidor, hacer reconocimiento directo sobre él, y conectarse sin ninguna
capa de control entre el atacante y el objetivo. Además, RDP, SMB y Telnet
están accesibles desde cualquier host interno — si un equipo se compromete,
el atacante puede moverse libremente por toda la red.

Este laboratorio resuelve exactamente ese problema. Instalamos pfSense
como firewall perimetral, bloqueamos ICMP para eliminar la visibilidad
externa, aplicamos ACLs para controlar el movimiento lateral interno,
y configuramos NAT para exponer IIS de forma controlada sin revelar
la IP interna del servidor. Todo verificado con Nmap desde Kali.

---

## 🎯 Objetivo

Instalar y configurar pfSense como firewall perimetral en VMware, implementar
reglas de firewall para bloquear ICMP, ACLs por puerto para controlar acceso
a servicios críticos, y NAT con port forwarding para exponer servicios internos
de forma controlada. Verificar cada control con Nmap y curl desde Kali Linux.

---

## 🖥️ Entorno

| Rol | Sistema | IP | Interfaz |
|-----|---------|-----|----------|
| Firewall / Router | pfSense 2.8.1 | WAN: 192.168.142.138 / LAN: 192.168.1.1 | em0 (NAT) / em1 (LAN) |
| Cliente interno | Windows 10 Pro | 192.168.1.100 | LAN — IIS 10.0 activo |
| Atacante / Verificador | Kali Linux | 192.168.142.135 | NAT (misma red WAN) |
| Virtualización | VMware | — | NAT + LAN |

> **Nota:** La IP WAN de pfSense cambió de `192.168.142.130` a `192.168.142.138`
> entre sesiones por DHCP de VMware. En producción se usaría IP estática.

---

## 🛠️ Herramientas utilizadas

- **pfSense 2.8.1-RELEASE** — Firewall, ACLs y NAT
- **Nmap 7.95** — Verificación de reglas desde Kali
- **curl** — Prueba de conectividad HTTP a través del NAT
- **IIS 10.0** — Servidor web en Windows 10 como destino del port forwarding
- **VMware** — Virtualización con interfaces NAT y LAN

---

## 🗺️ Topología del Laboratorio

```
[Internet / VMware NAT - 192.168.142.x]
         │
[pfSense WAN - 192.168.142.138]   ← em0
[pfSense LAN - 192.168.1.1    ]   ← em1
         │
[Red LAN 192.168.1.x]
         │
[Windows 10 - 192.168.1.100]  ← IIS 10.0 puerto 80

[Kali - 192.168.142.135]  ← misma red WAN
```

---

## 📋 Procedimiento

### Instalación y configuración inicial

Se instaló pfSense 2.8.1 en VMware con dos interfaces (em0/em1).
Wizard completado con `pfSense.lab.local`, DNS `8.8.8.8` / `8.8.4.4`.
Acceso al dashboard desde Windows 10 en `https://192.168.1.1` ✅

---

## 🧪 Ejercicios

---

### Ejercicio 1 — Reglas de Firewall (Block ICMP)

`Firewall → Rules → WAN → Add`

```
Action:      Block
Interface:   WAN
Protocol:    ICMP
Source:      Any
Destination: WAN address
Description: Block ICMP from WAN
```

**Verificación desde Kali:**
```bash
ping 192.168.142.130 -c 4
```
```
4 packets transmitted, 0 received, +4 errors, 100% packet loss
Destination Host Unreachable
```
**ICMP bloqueado** ✅ — pfSense descarta silenciosamente (Block vs Reject)

---

### Ejercicio 2 — ACLs por Puerto

`Firewall → Rules → LAN → Add` (3 reglas)

| Regla | Puerto | Protocolo | Acción |
|---|---|---|---|
| Block Telnet from LAN | 23 | TCP/UDP | Block |
| Block RDP from LAN | 3389 | TCP | Block |
| Block SMB from LAN | 445 | TCP | Block |

**Verificación desde Kali:**
```bash
nmap -p 23,445,3389 192.168.1.100
```
```
23/tcp    filtered  telnet
445/tcp   filtered  microsoft-ds
3389/tcp  filtered  ms-wbt-server
```
**3 puertos bloqueados** ✅ — `filtered` = pfSense descartando activamente

---

### Ejercicio 3 — NAT (Port Forwarding)

**Preparación — IIS en Windows 10:**
```powershell
curl http://localhost  # StatusCode: 200 ✅
netsh advfirewall firewall add rule name="Allow HTTP" protocol=TCP dir=in localport=80 action=allow
```

`Firewall → NAT → Port Forward → Add`

```
Interface:            WAN
Protocol:             TCP
Destination:          WAN address
Destination Port:     8080
Redirect target IP:   192.168.1.100
Redirect target port: HTTP (80)
Description:          NAT port 8080 to Windows 80
```

**Verificación desde Kali:**
```bash
curl http://192.168.142.138:8080 -v
```
```
* Connected to 192.168.142.138 port 8080
< HTTP/1.1 200 OK
< Server: Microsoft-IIS/10.0
```
**NAT funcionando** ✅ — Kali accedió a IIS sin conocer la IP interna

---

### Ejercicio 4 — Verificación final con Nmap

```bash
nmap -p 23,80,445,3389,8080 -Pn 192.168.142.138 -sV
```
```
PORT      STATE     SERVICE   VERSION
23/tcp    filtered  telnet
80/tcp    filtered  http
445/tcp   filtered  microsoft-ds
3389/tcp  filtered  ms-wbt-server
8080/tcp  open      http      Microsoft IIS httpd 10.0

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
Nmap done in 14.75 seconds
```

**Resultado perfecto:** 4 puertos bloqueados + NAT exponiendo IIS controladamente ✅

---

## 📸 Capturas de Pantalla

### Instalación y configuración

### 1️⃣ pfSense — Consola con WAN y LAN asignadas
<img width="624" height="296" alt="01-pfsense-consola" src="https://github.com/user-attachments/assets/78ff6fe0-2d16-4e0e-bd8e-f699a6fe855b" />

### 2️⃣ pfSense — Wizard de configuración inicial
<img width="624" height="313" alt="02-wizard-config" src="https://github.com/user-attachments/assets/8ba7dcc2-3648-4016-a8e7-080e0d74cffe" />

### 3️⃣ pfSense — Dashboard principal
<img width="624" height="367" alt="03-dashboard" src="https://github.com/user-attachments/assets/9f85f046-2609-4f0d-a870-76bbb491792e" />


### Ejercicio 1 — Firewall ICMP

### 4️⃣ pfSense — Regla Block ICMP from WAN
<img width="624" height="449" alt="04-rule-icmp" src="https://github.com/user-attachments/assets/49a07930-5710-4253-a5b5-dec742e80418" />

### 5️⃣ Kali — Ping bloqueado (100% packet loss)
<img width="624" height="232" alt="05-ping-blocked" src="https://github.com/user-attachments/assets/d350e087-40e1-493c-bc6a-0fb828699f25" />

### 6️⃣ pfSense — Regla Block Telnet from LAN
<img width="624" height="536" alt="06-rule-telnet" src="https://github.com/user-attachments/assets/c4e3ec19-da51-4e23-aca1-ecbddd5c49c8" />


### Ejercicio 2 — ACLs por puerto

### 7️⃣ pfSense — Regla Block RDP (3389)
<img width="624" height="548" alt="07-rule-rdp" src="https://github.com/user-attachments/assets/d2737a4f-2341-43f6-a7a0-22934b916588" />

### 8️⃣ pfSense — Regla Block SMB (445)
<img width="624" height="543" alt="08-rule-smb" src="https://github.com/user-attachments/assets/233a2d30-d6f8-40ff-b102-95d528c0e0d0" />

### 9️⃣ Kali — Nmap mostrando 3 puertos filtered
<img width="555" height="277" alt="09-nmap-filtered" src="https://github.com/user-attachments/assets/b82ea7c5-2d64-4eb0-b328-768bf161aa31" />


### Ejercicio 3 — NAT

### 🔟 pfSense — Configuración NAT Port Forward
<img width="624" height="589" alt="10-nat-config" src="https://github.com/user-attachments/assets/ce9b8e45-a8f5-4349-8659-b8e77aad52f7" />

### 1️⃣1️⃣ Windows 10 — IIS respondiendo en localhost
<img width="624" height="285" alt="11-iis-localhost" src="https://github.com/user-attachments/assets/df8872e2-14bb-44d8-8162-e2078423f932" />

### 1️⃣2️⃣ Kali — curl HTTP/1.1 200 OK a través del NAT
<img width="624" height="394" alt="12-curl-nat" src="https://github.com/user-attachments/assets/85acb373-1bc9-4b1b-9343-ffed82644331" />


### Ejercicio 4 — Verificación final

### 1️⃣3️⃣ Kali — Nmap final: 4 filtered + 8080 open (IIS)
<img width="624" height="241" alt="13-nmap-final" src="https://github.com/user-attachments/assets/3367bf39-1077-412b-844a-ba9fc21c91fa" />

---

## 🔍 Findings (Hallazgos)

- ICMP bloqueado desde WAN — `100% packet loss` confirmado
- ACLs bloquearon Telnet (23), SMB (445) y RDP (3389) — todos `filtered`
- NAT redirigió `WAN:8080 → Windows10:80` — `HTTP 200 OK — IIS 10.0` confirmado
- Nmap identificó `Microsoft IIS httpd 10.0` a través del NAT — port forwarding transparente
- Puerto 80 directo permanece `filtered` — solo accesible via NAT controlado en 8080
- IP interna `192.168.1.100` nunca expuesta directamente a Kali

---

## ⚠️ Impacto

- **Sin regla ICMP:** cualquier host externo puede descubrir pfSense — primer paso del reconocimiento
- **Sin ACLs en LAN:** un host comprometido puede conectarse a RDP, SMB y Telnet libremente
- **SMB sin filtrar** en LAN habilita NTLM Relay entre hosts internos sin detección
- **Sin NAT controlado:** exponer directamente el puerto 80 revela la IP interna del servidor
- **RDP sin ACL:** permite fuerza bruta directa contra escritorios remotos de la LAN

---

## 🚨 Detección (SOC)

- **Alerta pfSense Logs:** Paquetes ICMP bloqueados — visible en `Status → Firewall Logs`
- **Alerta pfSense Logs:** Intentos de conexión a 23, 445, 3389 bloqueados desde LAN
- **Alerta de red:** Múltiples intentos a Telnet desde LAN — posible malware
- **Alerta SIEM:** Intentos SMB bloqueados — posible movimiento lateral
- **Log NAT:** Conexiones entrantes a 8080 redirigidas — trazabilidad completa

---

## 🛡️ Mitigación aplicada

```
WAN → Block ICMP → WAN address          ✅
LAN → Block TCP/UDP → any:23            ✅
LAN → Block TCP → any:445               ✅
LAN → Block TCP → any:3389              ✅
WAN → NAT 8080 → 192.168.1.100:80       ✅
```

**Recomendaciones adicionales para producción:**
- Regla `Default Deny` en WAN — bloquear todo excepto lo explícitamente permitido
- Habilitar Snort o Suricata como paquete de pfSense para IDS/IPS
- Implementar GeoIP blocking
- Habilitar pfBlockerNG para bloquear IPs maliciosas conocidas

---

## 💼 ¿Por qué importa esto en el mundo real?

La mayoría de las empresas pequeñas y medianas en la región tienen un router
doméstico o un switch gestionado básico como "firewall." No tienen ACLs,
no tienen NAT controlado, y definitivamente no tienen reglas que limiten
el movimiento lateral dentro de su propia red.

El resultado más revelador de este laboratorio no es técnico — es conceptual.
Cuando Kali hizo `curl http://192.168.142.138:8080` y recibió una respuesta
de IIS, Kali no sabía que Windows 10 existía. Solo veía pfSense. Eso es
exactamente lo que hace un firewall bien configurado en producción:
crea una capa de abstracción que protege los activos internos.

En la DGA, donde trabajo actualmente, esta arquitectura (pfSense como
perimetral con reglas explícitas de ACL y NAT) es más robusta que lo
que tienen muchas instituciones públicas de la región. Implementar esto
correctamente — y poder verificarlo con Nmap — es una habilidad directamente
aplicable en el primer día de trabajo como analista de seguridad de redes.

**La lección más importante:** un firewall sin reglas verificadas no es
un firewall — es una ilusión de seguridad. La diferencia entre `filtered`
y `open` en un Nmap scan es la diferencia entre estar protegido y creer
que estás protegido.

---

## ✅ Conclusiones

1. pfSense funciona como firewall perimetral completo — instalación y configuración
   exitosas en VMware con interfaces WAN y LAN separadas
2. Block ICMP eliminó visibilidad del firewall desde exterior — primer control perimetral
3. ACLs en LAN demostraron que el firewall protege también del movimiento lateral interno
4. NAT expuso IIS sin revelar la IP interna — arquitectura correcta para servicios públicos
5. Nmap confirmó: `filtered` en bloqueados y `open` en NAT — evidencia verificable

---

## 🔗 Referencias

- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [pfSense Firewall Rules](https://docs.netgate.com/pfsense/en/latest/firewall/index.html)
- [pfSense NAT Port Forward](https://docs.netgate.com/pfsense/en/latest/nat/port-forwards.html)
- [MITRE ATT&CK — T1046 Network Service Discovery](https://attack.mitre.org/techniques/T1046/)
- [MITRE ATT&CK — T1021.001 RDP](https://attack.mitre.org/techniques/T1021/001/)
- [MITRE ATT&CK — T1021.002 SMB](https://attack.mitre.org/techniques/T1021/002/)

---

*← [Volver al Portafolio Principal](../README.md)*
