# Lab 01 — Análisis de Tráfico de Red con Wireshark

**Área:** Análisis de Red / Detección de Amenazas  
**Herramientas:** Wireshark, Nmap, Apache2, Kali Linux, Windows 10, VirtualBox  
**Dificultad:** ⭐⭐☆☆☆ Intermedio-Básico  
**Tipo de lab:** Ataque + Captura (Red compartida NAT)  
**Fecha de realización:** Marzo 2026  
**Estado:** Completado

---

## Objetivo

Simular un escenario real de ataque desde Kali Linux hacia una máquina Windows 10, capturando y analizando el tráfico generado con Wireshark. Se demuestra reconocimiento de red, detección de servicios, análisis de protocolos y captura de credenciales en texto claro.

---

## Topología del Laboratorio

```
┌──────────────────────────────────────────────────────────────┐
│                        VirtualBox                            │
│                                                              │
│   ┌───────────────────────┐    ┌────────────────────────┐   │
│   │      Kali Linux       │    │      Windows 10        │   │
│   │   (Atacante)          │    │      (Víctima)         │   │
│   │                       │    │                        │   │
│   │  eth0: 10.14.99.120   │    │  Eth0: 192.168.142.134 │   │
│   │  eth1: 192.168.142.135│────│  Eth0-2: 10.14.99.128  │   │
│   │                       │    │  (NAT compartido)      │   │
│   │  Wireshark (eth0)     │    │                        │   │
│   │  Nmap                 │    │  Navegador Edge        │   │
│   │  Apache2              │    │  Sin firewall activo   │   │
│   └───────────────────────┘    └────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**IPs utilizadas en el lab:**

| Máquina | Interfaz | IP |
|---------|----------|----|
| Kali Linux | eth0 (NAT) | 10.14.99.120 |
| Kali Linux | eth1 (NAT compartido) | 192.168.142.135 |
| Windows 10 | Ethernet0 (NAT) | 192.168.142.134 |
| Windows 10 | Ethernet0-2 (NAT compartido) | 10.14.99.128 / 10.14.99.129 |

---

## Verificación de Conectividad

Antes de iniciar los ejercicios se verificó conectividad bidireccional entre ambas máquinas.

**Desde Kali → Windows:**
```bash
ping 192.168.142.134 -c 4
# Resultado: 4/4 paquetes recibidos, 0% packet loss
# RTT avg: 1.525 ms

ping 10.14.99.129 -c 4
# Resultado: 4/4 paquetes recibidos, 0% packet loss
# RTT avg: 0.411 ms
```

**Desde Windows → Kali:**
```cmd
ping 192.168.142.135
# Resultado: 4/4 recibidos, 0% loss, avg 1ms

ping 10.14.99.129
# Resultado: 4/4 recibidos, 0% loss, avg 1ms
```

Conectividad confirmada en ambas direcciones.

---

## Ejercicios

---

### Ejercicio 1 — Escaneo de puertos con Nmap

**Objetivo:** Generar y detectar el patrón de tráfico de un escaneo de puertos.

#### nmap -sS (SYN Scan sigiloso)

```bash
nmap -sS 10.14.99.128
```

**Resultado obtenido:**
```
Host is up (0.0012s latency)
Not shown: 996 closed tcp ports (reset)

PORT      STATE  SERVICE
135/tcp   open   msrpc
139/tcp   open   netbios-ssn
445/tcp   open   microsoft-ds
5357/tcp  open   wsdapi

MAC Address: 00:0C:29:B3:6C:13 (VMware)
Nmap done: 1 IP address (1 host up) scanned in 9.38 seconds
```

**Análisis:** 4 puertos abiertos detectados, 996 cerrados (RST). La dirección MAC confirma que es una VM VMware. Los puertos 135, 139 y 445 son típicos de Windows (SMB/RPC).

---

#### nmap -sV (Detección de versiones)

```bash
nmap -sV 10.14.99.128
```

**Resultado obtenido:**
```
PORT      STATE  SERVICE       VERSION
135/tcp   open   msrpc         Microsoft Windows RPC
139/tcp   open   netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open   microsoft-ds? 
5357/tcp  open   http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
Nmap done in 19.71 seconds
```

**Análisis:** Se confirmó que el sistema operativo es Windows y se identificó el servicio HTTP en puerto 5357 (SSDP/UPnP, usado por Windows para descubrimiento de dispositivos en red local).

---

#### nmap -A (Escaneo agresivo)

```bash
nmap -A 10.14.99.128
```

**Resultado obtenido:**
```
Running: Microsoft Windows 10
OS details: Microsoft Windows 10 1709 - 21H2
Network Distance: 1 hop

Host script results:
  date: 2026-03-31T03:52:45
  smb2-security-mode: Message signing enabled but not required
  nbstat: NetBIOS name: DESKTOP-SVEV486
  clock-skew: -2s

TRACEROUTE: 1 hop → 10.14.99.128
Nmap done in 25.83 seconds
```

**Análisis:** El escaneo agresivo reveló:
- Versión exacta del SO: Windows 10 1709-21H2
- Nombre del equipo: `DESKTOP-SVEV486`
- SMB signing habilitado pero no requerido (vector de ataque potencial)
- Fecha y hora del sistema objetivo
- Un solo salto de red (misma red local)

---

#### Filtros Wireshark aplicados

**Filtro 1 — Paquetes SYN (escaneo activo):**
```
tcp.flags.syn == 1 && tcp.flags.ack == 0
```
Resultado: Se visualizaron múltiples paquetes SYN desde Kali hacia todos los puertos de Windows en milisegundos — patrón claro de escaneo automatizado. Wireshark mostró 3,518 paquetes filtrados (35.5% del total).

**Filtro 2 — Puertos cerrados (RST):**
```
tcp.flags.reset == 1
```
Resultado: Respuestas RST masivas de Windows confirmando los 996 puertos cerrados. 3,086 paquetes (31.2%).

**Filtro 3 — Puertos abiertos (SYN-ACK):**
```
tcp.flags.syn == 1 && tcp.flags.ack == 1
```
Resultado: Solo 147 paquetes (1.5%) — correspondientes a los puertos abiertos respondiendo al handshake.

**Filtro 4 — Tráfico originado desde Windows:**
```
ip.src == 10.14.99.128
```
Resultado: Se aislaron todas las respuestas del host víctima, incluyendo tráfico HTTP al puerto 5357.

---

### Ejercicio 2 — Ping Sweep (ICMP)

**Objetivo:** Capturar y analizar tráfico ICMP generado por ping.

```bash
ping 192.168.142.134
```

**Filtro Wireshark:**
```
icmp
```

**Resultado obtenido:** 20 paquetes capturados — 8 mostrados con el filtro ICMP (40%).

**Análisis del paquete capturado:**
```
Type: 8 (Echo ping reply)
Code: 0
Checksum: Good
Identifier (BE): 2048 / Identifier (LE): 8
Sequence Number (BE): 1024 / (LE): 4
Response time: 1.337 ms
Data: 48 bytes
```

- `Echo Request` (tipo 8): Kali enviando el ping
- `Echo Reply` (tipo 0): Windows respondiendo
- TTL=128: confirma que el destino es Windows (Linux usa TTL=64)
- Tiempo de respuesta: 1.337 ms — red local de baja latencia

---

### Ejercicio 3 — Captura de tráfico HTTP vs HTTPS

**Objetivo:** Demostrar que HTTP expone el contenido en texto claro mientras HTTPS lo cifra.

**Filtro HTTP:**
```
http
```
Resultado: Se capturó tráfico HTTP mostrando URLs completas, headers, métodos GET/HEAD y respuestas del servidor en texto legible. 6 paquetes mostrados de 11,905 capturados (0.1%).

**Filtro TLS (HTTPS para comparar):**
```
tls
```
Resultado: Solo se ve "Application Data" cifrado. El contenido es completamente ilegible. Headers del handshake TLS 1.2 visibles (Server Hello, Certificate, Key Exchange) pero el payload está cifrado. 1,697 paquetes (13.5% del total).

**Conclusión clave:** HTTP permite ver URLs, headers y datos completos. HTTPS solo expone metadatos del handshake — el contenido es inaccesible sin la clave privada.

---

### Ejercicio 4 — Servidor Apache con página de login falsa (Credential Harvesting)

**Objetivo:** Levantar un servidor web en Kali, crear una página de login y capturar las credenciales enviadas desde Windows en texto claro.

#### Configuración del servidor

```bash
# Instalar Apache
sudo apt install apache2

# Iniciar el servicio
sudo service apache2 start

# Ir al directorio web
cd /var/www/html

# Editar la página principal
sudo nano index.html
```

Se creó una página de login con HTML + Tailwind CSS con el diseño de un panel corporativo ("Acceso al Sistema Corporativo" / "SEGURIDAD DE RED NIVEL 4").

#### Captura de credenciales

**Filtro Wireshark:**
```
http && ip.dst == 192.168.142.135
```

**Desde Windows 10:** Se accedió a `http://192.168.142.135` y se ingresaron credenciales en el formulario.

**Credenciales capturadas en texto claro:**
```
Form item: "user" = "hola"
Form item: "pass" = "mundo1"
```

El paquete HTTP POST mostró en el panel inferior de Wireshark los campos del formulario completamente legibles, incluyendo User-Agent, headers del navegador (Edge en Windows 10 x64), y el body con las credenciales.

**Análisis:** Esto demuestra que cualquier formulario que envíe datos por HTTP (sin HTTPS) expone las credenciales del usuario. Esta técnica es la base del phishing con credential harvesting.

---

## Tabla de Filtros Utilizados

| Filtro | Resultado en el lab |
|--------|---------------------|
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | 3,518 paquetes — patrón de SYN scan |
| `tcp.flags.reset == 1` | 3,086 paquetes — puertos cerrados |
| `tcp.flags.syn == 1 && tcp.flags.ack == 1` | 147 paquetes — puertos abiertos |
| `ip.src == 10.14.99.128` | Tráfico originado desde Windows |
| `icmp` | 8 paquetes — Echo request/reply |
| `http` | Headers y URLs en texto claro |
| `tls` | Solo Application Data cifrado |
| `http && ip.dst == 192.168.142.135` | POST con credenciales visibles |

---

## Perspectiva Blue Team

| Actividad ejecutada | Alerta en SOC real |
|---|---|
| SYN scan (nmap -sS) | "Port scan detected — 996 ports probed in 9s" |
| Version scan (nmap -sV) | "Service enumeration attempt detected" |
| Aggressive scan (nmap -A) | "OS fingerprinting + SMB enumeration" |
| Ping sweep | "ICMP echo sweep from single host" |
| HTTP credential capture | "Credentials transmitted in cleartext (HTTP POST)" |
| Apache phishing page | "Suspicious HTTP server on non-standard host" |

---

## Lecciones Aprendidas

- [x] Configuré conectividad entre Kali y Windows en entorno virtualizado
- [x] Ejecuté los tres tipos de escaneo Nmap y analicé sus diferencias
- [x] Identifiqué el patrón de SYN scan en Wireshark (miles de SYN en segundos)
- [x] Distinguí paquetes RST (cerrado) vs SYN-ACK (abierto) en los filtros
- [x] Capturé tráfico ICMP y analicé TTL para identificar el SO del host
- [x] Comparé visibilidad de HTTP (texto claro) vs HTTPS (cifrado)
- [x] Configuré Apache2 y creé una página de login falsa con Tailwind CSS
- [x] Capturé credenciales reales enviadas por HTTP POST desde Windows

---

## Herramientas Utilizadas

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Wireshark | 4.x | Captura y análisis de paquetes |
| Nmap | 7.95 | Escaneo de puertos y reconocimiento |
| Apache2 | 2.x | Servidor HTTP para credential harvesting |
| Tailwind CSS | CDN | Diseño de la página de login falsa |
| VirtualBox | 7.x | Virtualización del entorno |

---

## Recursos

- [Wireshark Display Filters Reference](https://www.wireshark.org/docs/dfref/)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [TryHackMe — Wireshark: The Basics](https://tryhackme.com/room/wiresharkthebasics)
- [Cyber Kill Chain — Lockheed Martin](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html)

---

*← [Volver al Portafolio Principal](../README.md)*
