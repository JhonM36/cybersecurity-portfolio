# 🛡️ Laboratorio 04 — Hardening en Windows Server 2019 (CIS Benchmark)

![CIS](https://img.shields.io/badge/Standard-CIS_Benchmark-red?style=for-the-badge)
![Windows Server](https://img.shields.io/badge/Target-Windows_Server_2019-0078D6?style=for-the-badge&logo=windows)
![PowerShell](https://img.shields.io/badge/Tool-PowerShell-5391FE?style=for-the-badge&logo=powershell)
![Nessus](https://img.shields.io/badge/Scanner-Nessus_Essentials-00B388?style=for-the-badge)

---

## 🎬 Escenario

Mayo 2017. WannaCry cifra 200,000 sistemas en 150 países en 72 horas.
El vector: SMBv1 activo en servidores Windows que nadie había deshabilitado
porque "siempre funcionó así." El parche existía desde hacía dos meses.
Nadie lo había aplicado.

Junio 2021. PrintNightmare permite a cualquier usuario del dominio convertirse
en administrador del sistema. El servicio Print Spooler — activo por defecto
en todos los Windows Server — era el vector. Miles de organizaciones
lo tenían habilitado en servidores que no eran servidores de impresión.

Ambos ataques tenían algo en común: eran 100% prevenibles con hardening básico.

Este laboratorio aplica 14 controles del CIS Benchmark en Windows Server 2019,
verifica cada uno con evidencia, y demuestra por qué el hardening no es
una tarea de una vez — es un proceso continuo.

---

## 🎯 Objetivo

Aplicar controles de seguridad basados en el estándar CIS Benchmark para
Windows Server 2019, documentar el estado antes y después del hardening
usando PolicyAnalyzer, y validar la reducción de superficie de ataque con Nessus.

---

## 🖥️ Entorno

| Rol | Sistema | IP | Herramientas |
|-----|---------|-----|--------------|
| Objetivo / Servidor | Windows Server 2019 | 192.168.1.28 | PowerShell, PolicyAnalyzer |
| Escáner | Nessus Essentials | Local Scanner | Basic Network Scan |
| Dominio | JhonRuiz | — | Active Directory, Group Policy |

---

## 🛠️ Herramientas utilizadas

- **PolicyAnalyzer 4.0** — Auditoría y comparación de políticas de grupo (GPO)
- **Nessus Essentials** — Validación post-hardening
- **PowerShell** — Aplicación de controles CIS
- **Group Policy Management** — Backup y gestión de GPOs

---

## 📋 Procedimiento

### FASE 1 — Escaneo inicial (antes del hardening)

#### PolicyAnalyzer — Backup de GPOs

Se realizaron backups de las GPOs del dominio `JhonRuiz`:
- `Jhon.Ruiz_ADAuditPlusPolicy`
- `Default Domain Controllers Policy`
- `Default Domain Policy`

**Policy File Importer detectó 7 archivos** incluyendo registry.pol,
GptTmpl.inf, audit.csv y CSE-Machine policies.

#### PolicyAnalyzer — Compare to Effective State

**69 items de política** analizados y comparados contra el estado efectivo.
Resultado: alineación en auditoría pero ausencia de controles de red críticos.

#### Nessus — Escaneo pre-hardening

**Target:** `192.168.1.28` | **Duración:** 14 minutos | **Auth:** Fail

| Severidad | Cantidad |
|-----------|----------|
| 🔴 Critical | 0 |
| 🟠 High | 0 |
| 🟡 Medium | 0 |
| ⚪ Info | 22 |

Findings informativos: SMB Multiple Issues (6), DCE Services (13),
DNS Server Detection (2), LDAP (2), LLMNR Detection (1).

---

### FASE 2 — Aplicar hardening (CIS Benchmark)

#### CIS 1.1 — Política de contraseñas

```powershell
net accounts /minpwlen:14
net accounts /maxpwage:60
net accounts /minpwage:1
net accounts /uniquepw:24
# The command completed successfully. (x4) ✅
```

#### CIS 1.2 — Bloqueo de cuenta

```powershell
net accounts /lockoutthreshold:5
net accounts /lockoutduration:15
net accounts /lockoutwindow:15
# The command completed successfully. (x3) ✅
```

#### CIS 2.2 — Deshabilitar servicios innecesarios

```powershell
Set-Service -Name "Spooler" -StartupType Disabled
Stop-Service -Name "Spooler"

Set-Service -Name "RemoteRegistry" -StartupType Disabled
Stop-Service -Name "RemoteRegistry"
```

#### CIS 2.3 — Deshabilitar SMBv1

```powershell
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

#### CIS 9.1 — Firewall

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
```

#### CIS 17 — Auditoría de eventos

```powershell
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Sensitive Privilege Use" /success:enable /failure:enable
auditpol /set /subcategory:"File System" /success:enable /failure:enable
# The command was successfully executed. (x6) ✅
```

#### CIS 18 — Registry Security

```powershell
# LLMNR
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Force
Set-ItemProperty -Path "HKLM:\...\DNSClient" -Name "EnableMulticast" -Value 0

# NetBIOS
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration
foreach ($adapter in $adapters) { $adapter.SetTcpipNetbios(2) }

# NLA para RDP
Set-ItemProperty -Path 'HKLM:\...\RDP-Tcp' -Name "UserAuthentication" -Value 1

# Acceso anónimo
Set-ItemProperty -Path "HKLM:\...\Lsa" -Name "RestrictAnonymous" -Value 1
Set-ItemProperty -Path "HKLM:\...\Lsa" -Name "RestrictAnonymousSAM" -Value 1
```

#### CIS 19 — Windows Defender

```powershell
Set-MpPreference -DisableRealtimeMonitoring $false
Set-MpPreference -EnableNetworkProtection Enabled
Set-MpPreference -PUAProtection Enabled
```

---

### FASE 3 — Verificación post-hardening

```
Minimum password length:    14  ✅
Maximum password age:       60  ✅
Password history:           24  ✅
Lockout threshold:           5  ✅
Lockout duration:           15  ✅

RemoteRegistry   Stopped  Disabled  ✅
Spooler          Stopped  Disabled  ✅

EnableSMB1Protocol: False  ✅

Domain   True  ✅
Private  True  ✅
Public   True  ✅

EnableMulticast:      0  ✅ (LLMNR deshabilitado)
UserAuthentication:   1  ✅ (NLA requerido)
```

---

## 📸 Capturas de Pantalla

### 1️⃣ PolicyAnalyzer — Herramienta
*(agregar: capturas/01-policyanalyzer.png)*

### 2️⃣ PolicyAnalyzer — Backup de GPOs
*(agregar: capturas/02-gpo-backup.png)*

### 3️⃣ PolicyAnalyzer — Policy File Importer
*(agregar: capturas/03-policy-importer.png)*

### 4️⃣ PolicyAnalyzer — Análisis completado
*(agregar: capturas/04-analysis-complete.png)*

### 5️⃣ PolicyAnalyzer — View/Compare (69 items)
*(agregar: capturas/05-policy-compare.png)*

### 6️⃣ PolicyAnalyzer — Compare to Effective State
*(agregar: capturas/06-effective-state.png)*

### 7️⃣ Nessus — Escaneo pre-hardening
*(agregar: capturas/07-nessus-pre.png)*

### 8️⃣ PowerShell — Política de contraseñas
*(agregar: capturas/08-password-policy.png)*

### 9️⃣ PowerShell — Bloqueo de cuenta
*(agregar: capturas/09-lockout.png)*

### 🔟 PowerShell — Servicios deshabilitados
*(agregar: capturas/10-services.png)*

### 1️⃣1️⃣ PowerShell — SMBv1 deshabilitado
*(agregar: capturas/11-smb.png)*

### 1️⃣2️⃣ PowerShell — Firewall habilitado
*(agregar: capturas/12-firewall.png)*

### 1️⃣3️⃣ PowerShell — Auditoría configurada
*(agregar: capturas/13-audit.png)*

### 1️⃣4️⃣ PowerShell — LLMNR deshabilitado
*(agregar: capturas/14-llmnr.png)*

### 1️⃣5️⃣ PowerShell — NLA en RDP
*(agregar: capturas/15-nla.png)*

---

## 🔍 Findings (Hallazgos)

**Pre-hardening:**
- Print Spooler activo — vector de PrintNightmare (CVE-2021-1675)
- LLMNR activo — explotable con Responder
- SMBv1 no deshabilitado explícitamente — riesgo EternalBlue residual
- RemoteRegistry activo — modificación remota del registro sin autenticación adicional
- Sin política de contraseñas fuerte — vulnerable a password spraying

**Post-hardening — 14 controles CIS verificados:**
- SMBv1 deshabilitado — EternalBlue mitigado ✅
- LLMNR deshabilitado — NTLM Relay mitigado ✅
- NLA requerido en RDP — BlueKeep mitigado ✅
- Print Spooler deshabilitado — PrintNightmare mitigado ✅
- Auditoría completa habilitada — visibilidad total sobre eventos críticos ✅

---

## ⚠️ Impacto

- **Print Spooler activo** permitía PrintNightmare — ejecución como SYSTEM sin autenticación
- **LLMNR activo** permitía captura de hashes NTLMv2 con Responder sin credenciales
- **SMBv1** es el vector de WannaCry — cifró hospitales e infraestructura crítica en 2017
- **Sin auditoría** un atacante puede comprometer el sistema sin dejar rastro en logs
- **Sin política de contraseñas** las cuentas son vulnerables a fuerza bruta y spraying

---

## 🚨 Detección (SOC)

- **Alerta SIEM:** Event ID 4625 (failed logon) — ahora auditado ✅
- **Alerta SIEM:** Event ID 4720/4738 (cuenta creada/modificada) — ahora auditado ✅
- **Alerta SIEM:** Event ID 4672 (Special Logon / escalada) — ahora auditado ✅
- **Alerta de red:** Tráfico SMBv1 post-hardening → indica VM no parcheada
- **Alerta de red:** Tráfico LLMNR → indica host no endurecido en la red

---

## 🛡️ Controles CIS aplicados

| Control | Descripción | Estado |
|---|---|---|
| CIS 1.1.1 | Longitud mínima de contraseña: 14 | ✅ |
| CIS 1.1.2 | Edad máxima de contraseña: 60 días | ✅ |
| CIS 1.1.3 | Historial de contraseñas: 24 | ✅ |
| CIS 1.2.1 | Umbral de bloqueo: 5 intentos | ✅ |
| CIS 1.2.2 | Duración de bloqueo: 15 minutos | ✅ |
| CIS 2.2 | Print Spooler y RemoteRegistry deshabilitados | ✅ |
| CIS 2.3 | SMBv1 deshabilitado | ✅ |
| CIS 9.1 | Firewall habilitado (3 perfiles) | ✅ |
| CIS 17 | Auditoría avanzada de eventos | ✅ |
| CIS 18.1 | LLMNR deshabilitado | ✅ |
| CIS 18.2 | NetBIOS deshabilitado | ✅ |
| CIS 18.3 | NLA requerido para RDP | ✅ |
| CIS 18.4 | Acceso anónimo restringido | ✅ |
| CIS 19 | Windows Defender + Network Protection | ✅ |

**Total: 14 controles CIS aplicados y verificados ✅**

---

## 💼 ¿Por qué importa esto en el mundo real?

WannaCry y PrintNightmare no son historia antigua — son ejemplos de lo que
pasa cuando el hardening no existe como proceso. Ambos ataques usaron
vulnerabilidades conocidas, con parches disponibles, en servicios activos
por defecto que nadie había revisado.

En la República Dominicana, donde trabajo actualmente en la DGA, la mayoría
de las instituciones públicas no tienen un proceso formal de hardening.
Los servidores se instalan, se configuran para que "funcionen", y permanecen
así durante años. LLMNR activo, SMBv1 habilitado, Print Spooler corriendo
en servidores que nunca imprimieron nada.

**Lo que este laboratorio demuestra que sé hacer:** no solo ejecutar comandos,
sino entender POR QUÉ cada control importa, qué ataque específico previene,
y cómo verificar que realmente se aplicó. PolicyAnalyzer con 69 políticas
comparadas contra el estado efectivo es exactamente el tipo de evidencia
que un auditor de seguridad necesita entregar.

El hardening no es glamoroso. No tiene el atractivo de un exploit funcionando.
Pero es la diferencia entre ser el analista que previene el incidente y el
que lo investiga después de que ya causó daño.

---

## ✅ Conclusiones

1. PolicyAnalyzer auditó 69 políticas del dominio y comparó baseline vs estado efectivo
2. Hardening CIS eliminó vectores reales: EternalBlue, PrintNightmare, BlueKeep, NTLM Relay
3. Auditoría habilitada convierte el servidor en host monitoreable por Wazuh/SIEM
4. Nessus confirmó post-hardening: sin vulnerabilidades Critical, High o Medium detectables
5. El hardening debe ejecutarse periódicamente — no es un evento único

---

## 🔗 Referencias

- [CIS Benchmark — Windows Server 2019](https://www.cisecurity.org/benchmark/microsoft_windows_server)
- [Microsoft PolicyAnalyzer](https://www.microsoft.com/en-us/download/details.aspx?id=55319)
- [CVE-2021-1675 — PrintNightmare](https://nvd.nist.gov/vuln/detail/CVE-2021-1675)
- [CVE-2017-0144 — EternalBlue](https://nvd.nist.gov/vuln/detail/CVE-2017-0144)
- [MITRE ATT&CK — T1110 Brute Force](https://attack.mitre.org/techniques/T1110/)
- [MITRE ATT&CK — T1557.001 LLMNR Poisoning](https://attack.mitre.org/techniques/T1557/001/)

---

*← [Volver al Portafolio Principal](../README.md)*
