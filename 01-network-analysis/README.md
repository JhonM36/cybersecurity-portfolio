# 🔬 Laboratorio 01 — Análisis de Tráfico de Red con Wireshark

![Wireshark](https://img.shields.io/badge/Tool-Wireshark-1679A7?style=for-the-badge&logo=wireshark)
![Nmap](https://img.shields.io/badge/Tool-Nmap-blue?style=for-the-badge)
![Apache](https://img.shields.io/badge/Server-Apache2-D22128?style=for-the-badge&logo=apache)
![Kali](https://img.shields.io/badge/Attacker-Kali_Linux-557C94?style=for-the-badge&logo=kalilinux)

## 🎯 Objetivo

Simular un escenario real de ataque desde Kali Linux hacia una máquina Windows 10,
capturando y analizando el tráfico generado con Wireshark. Se demuestra reconocimiento
de red, detección de servicios, análisis de protocolos y captura de credenciales
en texto claro mediante un servidor Apache con página de login falsa.

---

## 🖥️ Entorno

| Rol | Sistema | IP | Herramientas |
|-----|---------|-----|--------------|
| Atacante / Analizador | Kali Linux | 10.14.99.120 / 192.168.142.135 | Wireshark, Nmap, Apache2 |
| Víctima | Windows 10 Pro | 10.14.99.128 / 192.168.142.134 | Navegador Edge |
| Virtualización | VirtualBox | — | Red NAT compartida |

---

## 🛠️ Herramientas utilizadas

- Wireshark 4.x — Captura y análisis de paquetes
- Nmap 7.95 — Escaneo de puertos y reconocimiento
- Apache2 — Servidor HTTP para credential harvesting
- Tailwind CSS — Diseño de la página de login falsa

---

## 📋 Procedimiento

### Paso 1 — Verificación de conectividad

Antes de iniciar los ejercicios se verificó conectividad bidireccional entre ambas máquinas.

**Desde Kali → Windows:**
```bash
ping 192.168.142.134 -c 4
# Resultado: 4/4 paquetes recibidos, 0% packet loss, RTT avg: 1.525 ms

ping 10.14.99.129 -c 4
# Resultado: 4/4 paquetes recibidos, 0% packet loss, RTT avg: 0.411 ms
```

**Desde Windows → Kali:**
```cmd
ping 192.168.142.135
# Resultado: 4/4 recibidos, 0% loss, avg 1ms
```

Conectividad confirmada en ambas direcciones ✅

### Paso 2 — Iniciar captura en Wireshark

Se abrió Wireshark en Kali y se seleccionó la interfaz `eth0` para capturar
todo el tráfico entre ambas máquinas antes de ejecutar los ataques.

---

## 🧪 Ejercicios

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
Nmap done: 1 IP address scanned in 9.38 seconds
```

**Análisis:** 4 puertos abiertos, 996 cerrados (RST). La MAC confirma VM VMware.
Los puertos 135, 139 y 445 son característicos de Windows (SMB/RPC).

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

**Análisis:** SO confirmado como Windows. Puerto 5357 ejecuta HTTPAPI 2.0
(protocolo SSDP/UPnP para descubrimiento de dispositivos en red local).

#### nmap -A (Escaneo agresivo)

```bash
nmap -A 10.14.99.128
```

**Resultado obtenido:**
```
Running: Microsoft Windows 10
OS details: Microsoft Windows 10 1709 - 21H2

Host script results:
  date: 2026-03-31T03:52:45
  smb2-security-mode: Message signing enabled but not required
  nbstat: NetBIOS name: DESKTOP-SVEV486
  clock-skew: -2s

TRACEROUTE: 1 hop → 10.14.99.128
Nmap done in 25.83 seconds
```

**Análisis — datos obtenidos del objetivo:**
- Versión exacta del SO: Windows 10 1709-21H2
- Nombre del equipo: `DESKTOP-SVEV486`
- SMB signing habilitado pero **no requerido** (vector de ataque potencial)
- Fecha y hora del sistema objetivo
- Un solo salto de red — misma red local

#### Filtros Wireshark aplicados

| Filtro | Resultado |
|--------|-----------|
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | 3,518 paquetes — patrón de SYN scan (35.5%) |
| `tcp.flags.reset == 1` | 3,086 paquetes — 996 puertos cerrados (31.2%) |
| `tcp.flags.syn == 1 && tcp.flags.ack == 1` | 147 paquetes — puertos abiertos (1.5%) |
| `ip.src == 10.14.99.128` | Tráfico aislado originado desde Windows |

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

**Resultado:** 20 paquetes capturados — 8 mostrados con filtro ICMP (40%).

**Análisis del paquete:**
```
Type: 8 (Echo ping reply)   → Windows respondiendo
Code: 0
TTL: 128                    → Confirma SO Windows (Linux = TTL 64)
Response time: 1.337 ms     → Red local de baja latencia
Data: 48 bytes
```

- `Echo Request` (tipo 8): Kali enviando el ping
- `Echo Reply` (tipo 0): Windows respondiendo
- **TTL=128** confirma que el destino es Windows — Linux usa TTL=64

---

### Ejercicio 3 — Captura de tráfico HTTP vs HTTPS

**Objetivo:** Demostrar que HTTP expone el contenido en texto claro mientras HTTPS lo cifra.

**Filtro HTTP:**
```
http
```
Resultado: URLs completas, headers, métodos GET/HEAD y respuestas del servidor
visibles en texto legible. 6 paquetes de 11,905 capturados (0.1%).

**Filtro TLS:**
```
tls
```
Resultado: Solo se ve "Application Data" cifrado. Handshake TLS 1.2 visible
(Server Hello, Certificate, Key Exchange) pero el payload es inaccesible.
1,697 paquetes (13.5% del total).

**Conclusión clave:** HTTP = contenido completamente expuesto.
HTTPS = solo metadatos del handshake, payload cifrado sin clave privada.

---

### Ejercicio 4 — Servidor Apache con página de login falsa (Credential Harvesting)

**Objetivo:** Levantar un servidor web en Kali, crear una página de login
y capturar las credenciales enviadas desde Windows en texto claro.

#### Configuración del servidor

```bash
# Instalar Apache
sudo apt install apache2

# Iniciar el servicio
sudo service apache2 start

# Ir al directorio web y editar la página
cd /var/www/html
sudo nano index.html
```

Se creó una página de login con HTML + Tailwind CSS simulando un panel
corporativo ("Acceso al Sistema Corporativo" / "SEGURIDAD DE RED NIVEL 4").

#### Captura de credenciales

**Filtro Wireshark:**
```
http && ip.dst == 192.168.142.135
```

Desde Windows 10 se accedió a `http://192.168.142.135` y se ingresaron
credenciales en el formulario.

**Credenciales capturadas en texto claro:**
```
Form item: "user" = "hola"
Form item: "pass" = "mundo1"
```

El paquete HTTP POST mostró en Wireshark los campos del formulario
completamente legibles, incluyendo User-Agent, headers del navegador
(Edge en Windows 10 x64) y el body con las credenciales.

**Análisis:** Cualquier formulario que envíe datos por HTTP expone
las credenciales del usuario. Esta técnica es la base del phishing
con credential harvesting.

---

## ⚠️ Errores encontrados y soluciones

### Error 1 — Ping fallido en configuración inicial (Bridge + NAT)

Al comenzar el lab, Kali estaba en modo **Bridged** (192.168.1.44) y
Windows en modo **NAT** (192.168.142.133). El ping desde Kali hacia
Windows no respondía.

**Causa:** Kali estaba en la red física (192.168.1.x) mientras Windows
estaba en una red virtual interna de VirtualBox (192.168.142.x).
El NAT solo permite tráfico de salida — Kali no podía iniciar conexión
hacia Windows.

**Solución:** Se unificó la configuración de red usando NAT compartido,
colocando ambas VMs en el mismo segmento de red, logrando conectividad
bidireccional.

---

## 📸 Capturas de Pantalla

### 1️⃣ Nmap -sS — SYN Scan
Escaneo sigiloso detectando 4 puertos abiertos en Windows 10.

![nmap-ss](https://github.com/user-attachments/assets/ca8a735b-5ce6-43a7-bc4c-5181709942b3)

---

### 2️⃣ Nmap -sV — Detección de versiones
Identificación de servicios: RPC, NetBIOS, SMB y HTTPAPI 2.0.

![nmap-sv](https://github.com/user-attachments/assets/07bb9827-1dab-4938-acb8-ceeb3e8f4136)

---

### 3️⃣ Nmap -A — Escaneo agresivo
Revelación del SO (Windows 10 21H2), nombre del equipo y configuración SMB.

![nmap-a](https://github.com/user-attachments/assets/2a6cc26e-96f9-4eb6-9fe2-a58187c31a75)

---

### 4️⃣ Wireshark — Filtro SYN scan
Patrón de miles de paquetes SYN en milisegundos — firma clara de escaneo automatizado.

![wireshark-syn](https://github.com/user-attachments/assets/5756ba82-c4d2-4100-a175-82b114f43453)

---

### 5️⃣ Wireshark — Filtro RST (puertos cerrados)
Respuestas RST masivas de Windows confirmando los 996 puertos cerrados.

![wireshark-rst](https://github.com/user-attachments/assets/5411f626-21ec-4627-9e23-708716674e32)

---

### 6️⃣ Wireshark — ICMP request/reply
Echo Request y Echo Reply capturados con TTL=128 (Windows confirmado).

![wireshark-icmp](https://github.com/user-attachments/assets/bf2b2adf-081b-4d95-b8eb-0290419e77fc)

---

### 7️⃣ Wireshark — HTTP en texto claro
Headers, URLs y datos del formulario completamente visibles sin cifrado.

![wireshark-http](https://github.com/user-attachments/assets/859236e0-532c-433f-acab-b968a6e5249c)

---

### 8️⃣ Wireshark — TLS cifrado (comparación)
Con HTTPS solo se ve "Application Data" — contenido inaccesible sin la clave privada.

![wireshark-tls](https://github.com/user-attachments/assets/99d0d3b9-425c-49ce-874f-f246b9237b4e)

---

### 9️⃣ Servidor Apache — página de login falsa
Página de login corporativa creada con HTML + Tailwind CSS en Kali Linux.

![apache-login](https://github.com/user-attachments/assets/f27227e2-91f1-4623-b3b3-f99a0344579a)

---

### 🔟 Credenciales capturadas en texto claro
Wireshark muestra `user=hola` y `pass=mundo1` en el HTTP POST desde Windows.

![credentials](https://github.com/user-attachments/assets/aa2ba71b-5047-4751-98a7-ecbf30213809)

---

## ✅ Conclusiones

1. Se verificó conectividad bidireccional entre Kali y Windows en entorno virtualizado
   y se resolvió el problema de red Bridge + NAT.
2. Se ejecutaron los tres tipos de escaneo Nmap y se identificaron diferencias clave:
   `-sS` detecta puertos, `-sV` identifica servicios, `-A` revela SO y configuración completa.
3. Wireshark permitió visualizar patrones claros de escaneo: miles de SYN en segundos,
   RST masivos en puertos cerrados y SYN-ACK en puertos abiertos.
4. Se demostró que el TTL de los paquetes ICMP permite identificar el SO del host remoto.
5. HTTP expone completamente el contenido — URLs, headers y credenciales en texto claro.
   HTTPS cifra el payload y solo expone metadatos del handshake TLS.
6. Se configuró Apache2 con una página de login falsa y se capturaron credenciales reales
   enviadas desde Windows, demostrando la técnica de credential harvesting sobre HTTP.

---

## 🔗 Referencias

- [Wireshark Display Filters Reference](https://www.wireshark.org/docs/dfref/)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [TryHackMe — Wireshark: The Basics](https://tryhackme.com/room/wiresharkthebasics)
- [Cyber Kill Chain — Lockheed Martin](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html)
- [MITRE ATT&CK — T1046 Network Service Discovery](https://attack.mitre.org/techniques/T1046/)

---

*← [Volver al Portafolio Principal](../README.md)*
