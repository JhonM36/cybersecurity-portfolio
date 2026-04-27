# 🔥 Laboratorio 05 — Firewall, ACLs y NAT con pfSense

![pfSense](https://img.shields.io/badge/Firewall-pfSense_2.8.1-blue?style=for-the-badge)
![VMware](https://img.shields.io/badge/Platform-VMware-607078?style=for-the-badge&logo=vmware)
![Nmap](https://img.shields.io/badge/Tool-Nmap_7.95-blue?style=for-the-badge)
![IIS](https://img.shields.io/badge/Server-IIS_10.0-0078D6?style=for-the-badge&logo=windows)

## 🎯 Objetivo

Instalar y configurar pfSense como firewall perimetral en VMware, implementando
reglas de firewall para bloquear tráfico ICMP, ACLs por puerto para controlar
acceso a servicios críticos (Telnet, SMB, RDP) y NAT con port forwarding para
exponer servicios internos de forma controlada. Se verifica cada control con
Nmap y curl desde Kali Linux.

---

## 🖥️ Entorno

| Rol | Sistema | IP | Interfaz |
|-----|---------|-----|----------|
| Firewall / Router | pfSense 2.8.1 | WAN: 192.168.142.138 / LAN: 192.168.1.1 | em0 (NAT) / em1 (LAN) |
| Cliente interno | Windows 10 Pro | 192.168.1.100 | LAN — IIS 10.0 activo |
| Atacante / Verificador | Kali Linux | 192.168.142.135 | NAT (misma red WAN) |
| Virtualización | VMware | — | NAT + LAN |

> **Nota:** La IP WAN de pfSense cambió de `192.168.142.130` a `192.168.142.138`
> entre sesiones por asignación DHCP de VMware. En producción se usaría IP estática.

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
┌─────────────────────────────────────────────────────────────┐
│                         VMware                              │
│                                                             │
│  [Internet / VMware NAT]                                    │
│         │ 192.168.142.x                                     │
│         │                                                   │
│  ┌──────┴──────────────────┐                                │
│  │     pfSense 2.8.1       │                                │
│  │  WAN: 192.168.142.138   │ ← em0 (NAT)                   │
│  │  LAN: 192.168.1.1       │ ← em1 (LAN)                   │
│  │                         │                                │
│  │  Reglas activas:        │                                │
│  │  - Block ICMP (WAN)     │                                │
│  │  - Block Telnet 23      │                                │
│  │  - Block SMB 445        │                                │
│  │  - Block RDP 3389       │                                │
│  │  - NAT 8080→80          │                                │
│  └──────┬──────────────────┘                                │
│         │ 192.168.1.x (LAN)                                 │
│         │                                                   │
│  ┌──────┴──────────┐                                        │
│  │  Windows 10     │                                        │
│  │  192.168.1.100  │ ← IIS 10.0 en puerto 80               │
│  └─────────────────┘                                        │
│                                                             │
│  [Kali Linux 192.168.142.135] ← misma red WAN que pfSense  │
└─────────────────────────────────────────────────────────────┘
```

---

## 📋 Procedimiento

### Instalación y configuración inicial de pfSense

Se instaló pfSense 2.8.1-RELEASE en VMware con dos interfaces:
- **em0:** Adaptador NAT → WAN (internet)
- **em1:** Adaptador LAN → red interna 192.168.1.x

Se completó el wizard de configuración inicial:
```
Hostname:           pfSense
Domain:             lab.local
Primary DNS:        8.8.8.8
Secondary DNS:      8.8.4.4
LAN IP:             192.168.1.1/24
Admin password:     cambiado desde valor por defecto
```

Se accedió al dashboard desde Windows 10 en `https://192.168.1.1` ✅

---

## 🧪 Ejercicios

---

### Ejercicio 1 — Reglas de Firewall (Block ICMP)

**Objetivo:** Bloquear ping (ICMP) desde la WAN hacia pfSense para evitar
reconocimiento de red desde el exterior.

#### Configuración en pfSense

`Firewall → Rules → WAN → Add`

```
Action:          Block
Interface:       WAN
Address Family:  IPv4
Protocol:        ICMP
ICMP Subtypes:   any
Source:          Any
Destination:     WAN address
Description:     Block ICMP from WAN
```

#### Regla adicional — Bloquear Telnet desde LAN

`Firewall → Rules → LAN → Add`

```
Action:                Block
Interface:             LAN
Protocol:              TCP/UDP
Source:                Any
Destination:           Any
Destination Port Range: Telnet (23)
Description:           Block Telnet from LAN
```

#### Verificación desde Kali

```bash
ping 192.168.142.130 -c 4
```

**Resultado:**
```
PING 192.168.142.130 56(84) bytes of data.
From 192.168.142.135 icmp_seq=1 Destination Host Unreachable
From 192.168.142.135 icmp_seq=2 Destination Host Unreachable
From 192.168.142.135 icmp_seq=3 Destination Host Unreachable
From 192.168.142.135 icmp_seq=4 Destination Host Unreachable

4 packets transmitted, 0 received, +4 errors, 100% packet loss
```

**Análisis:** `Destination Host Unreachable` con `100% packet loss` confirma
que pfSense descarta silenciosamente los paquetes ICMP — comportamiento
correcto de una regla `Block` (drop silencioso vs `Reject` que envía RST).

---

### Ejercicio 2 — ACLs por Puerto

**Objetivo:** Bloquear acceso a servicios críticos desde la red LAN
para limitar el movimiento lateral en caso de compromiso interno.

#### Regla 1 — Bloquear RDP (3389)

```
Action:                Block
Interface:             LAN
Protocol:              TCP
Source:                Any
Destination:           Any
Destination Port Range: MS RDP (3389)
Description:           Block RDP from LAN
```

#### Regla 2 — Bloquear SMB (445)

```
Action:                Block
Interface:             LAN
Protocol:              TCP
Source:                Any
Destination:           Any
Destination Port Range: MS DS (445)
Description:           Block SMB from LAN
```

#### Verificación desde Kali

```bash
nmap -p 23,445,3389 192.168.1.100
```

**Resultado:**
```
PORT      STATE     SERVICE
23/tcp    filtered  telnet
445/tcp   filtered  microsoft-ds
3389/tcp  filtered  ms-wbt-server

Nmap done: 1 IP address scanned in 7.90 seconds
```

**Análisis:** Los 3 puertos muestran `filtered` — pfSense está bloqueando
activamente el tráfico. La diferencia entre `filtered` y `closed`:
- `filtered` → firewall descarta el paquete (no hay respuesta)
- `closed` → puerto rechazado por el host (RST enviado)
- `open` → servicio accesible

---

### Ejercicio 3 — NAT (Port Forwarding)

**Objetivo:** Redirigir el puerto 8080 de la WAN de pfSense hacia el
puerto 80 de Windows 10, exponiendo IIS de forma controlada sin
revelar la IP interna del servidor.

#### Preparación — Habilitar IIS en Windows 10

Se verificó que IIS estaba activo respondiendo en puerto 80:
```powershell
curl http://localhost
# StatusCode: 200 — IIS Windows funcionando ✅
```

Se habilitó el tráfico entrante al puerto 80 en el firewall de Windows:
```powershell
netsh advfirewall firewall add rule name="Allow HTTP" `
  protocol=TCP dir=in localport=80 action=allow
# Ok. ✅
```

#### Configuración NAT en pfSense

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

pfSense creó automáticamente la regla de firewall asociada:
```
Action:      Pass
Interface:   WAN
Protocol:    TCP
Source:      Any
Destination: 192.168.1.100:80
Description: NAT NAT port 8080 to Windows 80
```

#### Verificación desde Kali

```bash
curl http://192.168.142.138:8080 -v
```

**Resultado:**
```
* Trying 192.168.142.138:8080...
* Connected to 192.168.142.138 port 8080
* using HTTP/1.x
> GET / HTTP/1.1
> Host: 192.168.142.138:8080

< HTTP/1.1 200 OK
< Content-Type: text/html
< Server: Microsoft-IIS/10.0
< Date: Mon, 27 Apr 2026 14:41:11 GMT
< Content-Length: 696

<!DOCTYPE html PUBLIC ...>
<title>IIS Windows</title>
```

**Análisis:** Kali se conectó a pfSense WAN en puerto 8080 y recibió
la página de IIS de Windows 10 — el NAT redirigió transparentemente
la conexión hacia `192.168.1.100:80` sin que Kali conozca la IP interna.

---

### Ejercicio 4 — Verificación final con Nmap

Escaneo completo para documentar el estado de todas las reglas:

```bash
nmap -p 23,80,445,3389,8080 -Pn 192.168.142.138 -sV
```

**Resultado:**
```
PORT      STATE     SERVICE   VERSION
23/tcp    filtered  telnet
80/tcp    filtered  http
445/tcp   filtered  microsoft-ds
3389/tcp  filtered  ms-wbt-server
8080/tcp  open      http      Microsoft IIS httpd 10.0

MAC Address: 00:0C:29:B9:ED:7F (VMware)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap done: 1 IP address scanned in 14.75 seconds
```

**Análisis del resultado:**
- `23, 80, 445, 3389` → `filtered` — ACLs bloqueando correctamente
- `8080` → `open` — NAT funcionando, IIS accesible a través de pfSense
- Nmap identificó `Microsoft IIS httpd 10.0` a través del NAT — el
  port forwarding opera de forma transparente
- `80/tcp filtered` — el puerto 80 directo está bloqueado pero accesible
  solo a través del NAT en 8080 — control de acceso correcto

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

- pfSense bloqueó correctamente ICMP desde WAN — `100% packet loss` confirmado desde Kali
- Las ACLs bloquearon los 3 puertos críticos: Telnet (23), SMB (445) y RDP (3389) — todos `filtered`
- El NAT redirigió transparentemente `WAN:8080 → Windows10:80` — IIS respondió con `HTTP 200 OK`
- Nmap identificó `Microsoft IIS httpd 10.0` a través del NAT — port forwarding completamente funcional
- El puerto 80 directo permanece `filtered` — solo accesible a través del NAT controlado en 8080
- La IP interna de Windows 10 (`192.168.1.100`) nunca fue expuesta directamente a Kali

---

## ⚠️ Impacto

- **Sin regla ICMP:** cualquier host externo puede hacer ping sweep para descubrir que pfSense existe y está activo — primer paso del reconocimiento
- **Sin ACLs en LAN:** un host comprometido en la red interna puede conectarse libremente a RDP (movimiento lateral), SMB (robo de archivos/hashes) y Telnet (credenciales en claro)
- **Sin NAT controlado:** exponer directamente el puerto 80 de Windows 10 revela la IP interna y elimina la capa de abstracción del firewall
- **SMB sin filtrar en LAN** habilitaría ataques de NTLM Relay entre hosts internos sin que el firewall perimetral lo detecte
- **RDP sin filtrar** permitiría ataques de fuerza bruta directamente contra el escritorio remoto de cualquier host en la LAN

---

## 🚨 Detección (SOC)

- **Alerta IDS:** Intentos de ping hacia la WAN del firewall — posible reconocimiento perimetral
- **Alerta pfSense Firewall Logs:** Paquetes bloqueados por las reglas ICMP, Telnet, SMB y RDP — visibles en `Status → System Logs → Firewall`
- **Alerta de red:** Múltiples intentos de conexión a puerto 23 (Telnet) desde LAN — posible malware o usuario malintencionado
- **Alerta SIEM:** Intentos de conexión a SMB (445) bloqueados — posible intento de movimiento lateral
- **Log NAT:** Conexiones entrantes a puerto 8080 redirigidas a servidor interno — trazabilidad completa del tráfico
- **Alerta Nmap:** Escaneo de puertos desde `192.168.142.135` hacia pfSense WAN detectado en logs

---

## 🛡️ Mitigación

**Reglas aplicadas en este lab:**

```
WAN → Block ICMP any → WAN address          ✅ Reconocimiento bloqueado
LAN → Block TCP/UDP any → any:23            ✅ Telnet bloqueado
LAN → Block TCP any → any:445              ✅ SMB bloqueado
LAN → Block TCP any → any:3389             ✅ RDP bloqueado
WAN → NAT 8080 → 192.168.1.100:80          ✅ Acceso controlado a IIS
```

**Recomendaciones adicionales para producción:**
- Implementar regla `Default Deny` en WAN — bloquear todo excepto lo explícitamente permitido
- Habilitar logging en todas las reglas de bloqueo para alimentar el SIEM
- Configurar IDS/IPS (Snort o Suricata) como paquete de pfSense para detección de amenazas
- Implementar GeoIP blocking para bloquear países de origen no esperados
- Usar aliases en pfSense para agrupar IPs y puertos — facilita la gestión de reglas
- Habilitar pfSense `pfBlockerNG` para bloquear IPs maliciosas conocidas automáticamente

---

## ✅ Conclusiones

1. pfSense demostró ser un firewall perimetral completamente funcional — instalación,
   configuración y verificación completadas en un entorno virtualizado con VMware
2. Las reglas de firewall Block ICMP eliminaron la visibilidad del firewall desde
   la red externa — primer control de seguridad perimetral esencial
3. Las ACLs por puerto (Telnet, SMB, RDP) en la interfaz LAN demostraron que el
   firewall protege no solo del exterior sino también del movimiento lateral interno
4. El NAT con port forwarding permitió exponer IIS de Windows 10 de forma controlada —
   Kali accedió al servicio sin conocer la IP interna del servidor
5. Nmap confirmó el estado de todas las reglas — `filtered` para bloqueados y
   `open` para el NAT — evidencia técnica verificable del funcionamiento
6. La combinación Firewall + ACLs + NAT cubre los tres pilares de seguridad
   perimetral: bloqueo de reconocimiento, control de acceso interno y
   exposición controlada de servicios

---

## 🔗 Referencias

- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [pfSense Firewall Rules](https://docs.netgate.com/pfsense/en/latest/firewall/index.html)
- [pfSense NAT Port Forward](https://docs.netgate.com/pfsense/en/latest/nat/port-forwards.html)
- [MITRE ATT&CK — T1046 Network Service Discovery](https://attack.mitre.org/techniques/T1046/)
- [MITRE ATT&CK — T1021.001 RDP](https://attack.mitre.org/techniques/T1021/001/)
- [MITRE ATT&CK — T1021.002 SMB](https://attack.mitre.org/techniques/T1021/002/)
- [CIS Benchmark — pfSense](https://www.cisecurity.org/benchmark/cisco)

---

*← [Volver al Portafolio Principal](../README.md)*
