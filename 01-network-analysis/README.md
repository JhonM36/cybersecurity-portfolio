# 🔬 Laboratorio 01 — Análisis de Tráfico de Red con Wireshark

![Wireshark](https://img.shields.io/badge/Tool-Wireshark-1679A7?style=for-the-badge&logo=wireshark)
![Nmap](https://img.shields.io/badge/Tool-Nmap-blue?style=for-the-badge)
![Apache](https://img.shields.io/badge/Server-Apache2-D22128?style=for-the-badge&logo=apache)
![Kali](https://img.shields.io/badge/Attacker-Kali_Linux-557C94?style=for-the-badge&logo=kalilinux)

---

## 🎬 Escenario

Son las 2:47 AM. El equipo de SOC recibe una alerta: tráfico inusual en la red interna.
Alguien está enviando miles de paquetes SYN hacia un host Windows en menos de 10 segundos.
Minutos después, ese mismo host empieza a recibir peticiones HTTP con credenciales
en texto claro hacia una IP desconocida dentro de la red.

Este laboratorio reproduce exactamente ese escenario desde la perspectiva del atacante
que genera el tráfico y del analista que lo captura y analiza con Wireshark.
Entender ambos lados es lo que convierte a un técnico en un analista SOC.

---

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

**Desde Kali → Windows:**
```bash
ping 192.168.142.134 -c 4
# Resultado: 4/4 paquetes, 0% packet loss, RTT avg: 1.525 ms

ping 10.14.99.129 -c 4
# Resultado: 4/4 paquetes, 0% packet loss, RTT avg: 0.411 ms
```

**Desde Windows → Kali:**
```cmd
ping 192.168.142.135
# Resultado: 4/4 recibidos, 0% loss, avg 1ms
```
Conectividad confirmada en ambas direcciones ✅

### Paso 2 — Iniciar captura en Wireshark

Se abrió Wireshark en Kali seleccionando la interfaz `eth0` para capturar
todo el tráfico entre ambas máquinas antes de ejecutar los ataques.

---

## 🧪 Ejercicios

---

### Ejercicio 1 — Escaneo de puertos con Nmap

#### nmap -sS (SYN Scan sigiloso)
```bash
nmap -sS 10.14.99.128
```
**Resultado:**
```
PORT      STATE  SERVICE
135/tcp   open   msrpc
139/tcp   open   netbios-ssn
445/tcp   open   microsoft-ds
5357/tcp  open   wsdapi
MAC Address: 00:0C:29:B3:6C:13 (VMware)
Nmap done: 1 IP address scanned in 9.38 seconds
```

#### nmap -sV (Detección de versiones)
```bash
nmap -sV 10.14.99.128
```
**Resultado:**
```
PORT      STATE  SERVICE       VERSION
135/tcp   open   msrpc         Microsoft Windows RPC
139/tcp   open   netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open   microsoft-ds?
5357/tcp  open   http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows
Nmap done in 19.71 seconds
```

#### nmap -A (Escaneo agresivo)
```bash
nmap -A 10.14.99.128
```
**Resultado:**
```
OS details: Microsoft Windows 10 1709 - 21H2
nbstat: NetBIOS name: DESKTOP-SVEV486
smb2-security-mode: Message signing enabled but not required
TRACEROUTE: 1 hop → 10.14.99.128
Nmap done in 25.83 seconds
```

#### Filtros Wireshark aplicados

| Filtro | Resultado |
|--------|-----------|
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | 3,518 paquetes — patrón SYN scan (35.5%) |
| `tcp.flags.reset == 1` | 3,086 paquetes — 996 puertos cerrados (31.2%) |
| `tcp.flags.syn == 1 && tcp.flags.ack == 1` | 147 paquetes — puertos abiertos (1.5%) |
| `ip.src == 10.14.99.128` | Tráfico aislado desde Windows |

---

### Ejercicio 2 — Ping Sweep (ICMP)

```bash
ping 192.168.142.134
```
Filtro: `icmp` → 8 paquetes mostrados de 20 capturados (40%)

**Análisis:**
- `Echo Request` tipo 8 — Kali enviando ping
- `Echo Reply` tipo 0 — Windows respondiendo
- TTL=128 → confirma SO Windows (Linux usa TTL=64)
- Response time: 1.337 ms

---

### Ejercicio 3 — HTTP vs HTTPS

| Protocolo | Filtro | Resultado |
|-----------|--------|-----------|
| HTTP | `http` | URLs, headers y datos en texto claro — 6 paquetes (0.1%) |
| HTTPS | `tls` | Solo "Application Data" cifrado — 1,697 paquetes (13.5%) |

**Conclusión:** HTTP expone todo. HTTPS cifra el payload inaccesible sin clave privada.

---

### Ejercicio 4 — Credential Harvesting con Apache

```bash
sudo apt install apache2
sudo service apache2 start
cd /var/www/html
sudo nano index.html
```

Se creó una página de login con HTML + Tailwind CSS simulando un panel
corporativo. Desde Windows se accedió a `http://192.168.142.135` e ingresaron credenciales.

**Filtro:**
```
http && ip.dst == 192.168.142.135
```

**Credenciales capturadas en texto claro:**
```
Form item: "user" = "hola"
Form item: "pass" = "mundo1"
```

---

## ⚠️ Errores encontrados y soluciones

### Error — Ping fallido (Bridge + NAT)

**Causa:** Kali en modo Bridged (192.168.1.x) y Windows en NAT (192.168.142.x).
Redes distintas NAT solo permite tráfico de salida.

**Solución:** Se unificó ambas VMs en NAT compartido, logrando conectividad
bidireccional en el mismo segmento de red.

---

## 📸 Capturas de Pantalla

### 1️⃣ Nmap -sS — SYN Scan
![nmap-ss](https://github.com/user-attachments/assets/ca8a735b-5ce6-43a7-bc4c-5181709942b3)

### 2️⃣ Nmap -sV — Detección de versiones
![nmap-sv](https://github.com/user-attachments/assets/07bb9827-1dab-4938-acb8-ceeb3e8f4136)

### 3️⃣ Nmap -A — Escaneo agresivo
![nmap-a](https://github.com/user-attachments/assets/2a6cc26e-96f9-4eb6-9fe2-a58187c31a75)

### 4️⃣ Wireshark — Filtro SYN scan
![wireshark-syn](https://github.com/user-attachments/assets/5756ba82-c4d2-4100-a175-82b114f43453)

### 5️⃣ Wireshark — RST puertos cerrados
![wireshark-rst](https://github.com/user-attachments/assets/5411f626-21ec-4627-9e23-708716674e32)

### 6️⃣ Wireshark — ICMP request/reply
![wireshark-icmp](https://github.com/user-attachments/assets/bf2b2adf-081b-4d95-b8eb-0290419e77fc)

### 7️⃣ Wireshark — HTTP en texto claro
![wireshark-http](https://github.com/user-attachments/assets/859236e0-532c-433f-acab-b968a6e5249c)

### 8️⃣ Wireshark — TLS cifrado
![wireshark-tls](https://github.com/user-attachments/assets/99d0d3b9-425c-49ce-874f-f246b9237b4e)

### 9️⃣ Servidor Apache — login falso
![apache-login](https://github.com/user-attachments/assets/f27227e2-91f1-4623-b3b3-f99a0344579a)

### 🔟 Credenciales capturadas
![credentials](https://github.com/user-attachments/assets/aa2ba71b-5047-4751-98a7-ecbf30213809)

---

## 🔍 Findings (Hallazgos)

- Se detectó un escaneo SYN activo desde Kali hacia Windows, 3,518 paquetes en menos de 10 segundos
- Se identificaron 4 puertos abiertos: 135 (RPC), 139 (NetBIOS), 445 (SMB), 5357 (HTTPAPI)
- El escaneo agresivo reveló nombre del equipo, versión del SO y configuración SMB
- SMB signing habilitado pero **no requerido** — configuración débil
- Credenciales capturadas en texto claro: `user=hola`, `pass=mundo1`
- TTL=128 permitió identificar el SO del host sin escaneo adicional

---

## ⚠️ Impacto

- El reconocimiento con Nmap expuso la superficie de ataque completa del host Windows
- Los puertos 139 y 445 con SMB signing no requerido habilitan ataques de NTLM Relay
- Las credenciales capturadas en HTTP pueden usarse para acceso no autorizado
- Un atacante con esta información puede planificar lateral movement y privilege escalation

---

## 🚨 Detección (SOC)

- **Alerta IDS:** 3,518 paquetes SYN desde una sola IP en menos de 10 segundos → Port scan
- **Alerta IDS:** OS fingerprinting y enumeración de versiones → Reconocimiento activo
- **Alerta DLP:** Credenciales transmitidas en texto claro por HTTP POST
- **Alerta de red:** Servidor HTTP no autorizado en host interno
- **Log de eventos:** ICMP sweep desde host interno hacia otro segmento

---

## 🛡️ Mitigación

- Implementar IDS/IPS (Snort, Suricata) con reglas de detección de port scanning
- Habilitar SMB signing como **requerido** en todos los hosts Windows
- Forzar HTTPS en todos los servicios web
- Segmentar la red con VLANs para limitar visibilidad entre hosts
- Monitorear tráfico HTTP interno con proxy que detecte credenciales en claro

---

## 💼 ¿Por qué importa esto en el mundo real?

Este laboratorio replica la **fase de reconocimiento del Cyber Kill Chain** 
el primer paso que da cualquier atacante antes de comprometer un sistema.

En un entorno corporativo real, el SYN scan que ejecuté generaría una alerta
en el SIEM en segundos. Sin embargo, el 80% de las organizaciones pequeñas y
medianas en la región no tienen un SIEM configurado lo que significa que
ese escaneo pasaría completamente desapercibido.

La captura de credenciales via HTTP no es un ataque sofisticado. Es una técnica
de 1996 que sigue funcionando en 2026 porque muchos sistemas internos
impresoras, cámaras IP, paneles de administración todavía usan HTTP.
En la DGA, durante mi pasantía, encontré exactamente este tipo de dispositivos.

**Lo que aprendí que llevaría a un trabajo real:** antes de desplegar cualquier
solución de monitoreo, lo primero es entender qué tráfico normal se ve en la red.
Un analista que nunca ha capturado tráfico real no sabe distinguir lo anómalo
de lo legítimo y eso es exactamente lo que entrena este laboratorio.

---

## ✅ Conclusiones

1. Nmap `-sS` detecta puertos, `-sV` identifica servicios y `-A` revela OS completo
2. Wireshark identifica patrones de ataque: miles de SYN en segundos = escaneo automatizado
3. TTL de paquetes ICMP permite fingerprinting pasivo del SO sin herramientas adicionales
4. HTTP expone completamente credenciales HTTPS es obligatorio en cualquier formulario
5. Un servidor Apache falso captura credenciales reales base del credential harvesting

---

## 🔗 Referencias

- [Wireshark Display Filters Reference](https://www.wireshark.org/docs/dfref/)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [MITRE ATT&CK — T1046 Network Service Discovery](https://attack.mitre.org/techniques/T1046/)
- [MITRE ATT&CK — T1557 NTLM Relay](https://attack.mitre.org/techniques/T1557/)
- [Cyber Kill Chain — Lockheed Martin](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html)

---

*← [Volver al Portafolio Principal](../README.md)*
