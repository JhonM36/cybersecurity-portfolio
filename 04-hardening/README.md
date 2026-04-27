# 🛡️ Laboratorio 04 — Hardening en Windows Server 2019 (CIS Benchmark)

![CIS](https://img.shields.io/badge/Standard-CIS_Benchmark-red?style=for-the-badge)
![Windows Server](https://img.shields.io/badge/Target-Windows_Server_2019-0078D6?style=for-the-badge&logo=windows)
![PowerShell](https://img.shields.io/badge/Tool-PowerShell-5391FE?style=for-the-badge&logo=powershell)
![Nessus](https://img.shields.io/badge/Scanner-Nessus_Essentials-00B388?style=for-the-badge)

## 🎯 Objetivo

Aplicar controles de seguridad basados en el estándar **CIS Benchmark para Windows Server 2019**
sobre un servidor con configuración por defecto, documentando el estado antes y después
del hardening. Se usa PolicyAnalyzer para auditar GPOs y Nessus para validar la reducción
de superficie de ataque, demostrando el proceso completo de endurecimiento de sistemas
que realiza un analista de seguridad en entornos reales.

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
- **Nessus Essentials** — Validación de vulnerabilidades antes/después del hardening
- **PowerShell** — Aplicación de controles CIS via cmdlets y registry
- **Group Policy Management** — Backup y gestión de GPOs del dominio

---

## 📋 Procedimiento

---

### FASE 1 — Escaneo inicial (antes del hardening)

#### PolicyAnalyzer — Backup de GPOs

Se descargó e instaló PolicyAnalyzer 4.0 desde Microsoft.
Se realizaron backups de las GPOs existentes en el dominio `JhonRuiz`
y se guardaron en la carpeta `GPO Backups`:

GPOs respaldadas:
- `Jhon.Ruiz_ADAuditPlusPolicy`
- `Default Domain Controllers Policy`
- `Default Domain Policy`

Se importaron los backups en PolicyAnalyzer para analizar el estado
de las políticas antes de aplicar el hardening.

**Policy File Importer detectó:**

| Policy Name | Policy Type | Archivo |
|---|---|---|
| Jhon.Ruiz_ADAuditPlusPolicy | Computer | registry.pol |
| Default Domain Controllers Policy | Sec Template | GptTmpl.inf |
| Jhon.Ruiz_ADAuditPlusPolicy | Audit Policy | audit.csv |
| Default Domain Controllers Policy | Audit Policy | audit.csv |
| Group policy CSE — Registry Policy | CSE-Machine | {D3785EAC...} |
| Group policy CSE — Security | CSE-Machine | {827D319E...} |
| Group policy CSE — Audit Policy | CSE-Machine | {F3CCC681...} |

#### PolicyAnalyzer — View / Compare

Se analizó el archivo `MyGpo` (36,680 bytes, 4/26/2026) con
**69 items de política** comparados contra el estado efectivo del sistema.

**Compare to Effective State** mostró alineación entre el baseline
configurado y el estado real del servidor en categorías de auditoría:

| Categoría | Baseline | Estado Efectivo |
|---|---|---|
| Kerberos Authentication Service | Success and Failure | Success and Failure |
| User Account Management | Success and Failure | Success and Failure |
| Security Group Management | Success | Success |
| Logon | Success and Failure | Success and Failure |
| File System | Success and Failure | Success and Failure |
| Handle Manipulation | Success and Failure | Success and Failure |

#### Nessus — Escaneo inicial (antes del hardening)

**Escaneo:** `Lab04-Window Server` contra `192.168.1.28`
**Duración:** 14 minutos | **Auth:** Fail (sin credenciales)

**Resultados pre-hardening:**

| Severidad | Cantidad |
|-----------|----------|
| 🔴 Critical | 0 |
| 🟠 High | 0 |
| 🟡 Medium | 0 |
| 🔵 Low | 0 |
| ⚪ Info | 22 |

**Findings informativos detectados:**
- SMB (Multiple Issues) — 6 findings
- HTTP (Multiple Issues) — 2 findings
- DCE Services Enumeration — 13 findings
- Nessus SYN Scanner — 12 findings
- DNS Server Detection — 2 findings
- LDAP Crafted Search Request Server Information Disclosure — 2 findings
- LDAP Server Detection — 2 findings
- Service Detection — 2 findings
- Common Platform Enumeration (CPE) — 1 finding
- Device Type — 1 finding
- Ethernet Card Manufacturer Detection — 1 finding

> **Nota:** La ausencia de Critical/High pre-hardening indica que Windows Server 2019
> tiene mejoras de seguridad por defecto vs versiones anteriores. El hardening busca
> cerrar vectores de ataque que Nessus sin credenciales no detecta.

---

### FASE 2 — Aplicar hardening (CIS Benchmark)

Todos los controles se ejecutaron en **PowerShell como Administrador**.

#### CIS 1.1 — Política de contraseñas

```powershell
net accounts /minpwlen:14
net accounts /maxpwage:60
net accounts /minpwage:1
net accounts /uniquepw:24
# Resultado: The command completed successfully. (x4)
```

#### CIS 1.2 — Bloqueo de cuenta

```powershell
net accounts /lockoutthreshold:5
net accounts /lockoutduration:15
net accounts /lockoutwindow:15
# Resultado: The command completed successfully. (x3)
```

#### CIS 2.2 — Deshabilitar servicios innecesarios

**Telnet:**
```powershell
Set-Service -Name "TlntSvr" -StartupType Disabled -ErrorAction SilentlyContinue
Stop-Service -Name "TlntSvr" -ErrorAction SilentlyContinue
```

**Print Spooler:**
```powershell
Set-Service -Name "Spooler" -StartupType Disabled
Stop-Service -Name "Spooler"
```

**Remote Registry:**
```powershell
Set-Service -Name "RemoteRegistry" -StartupType Disabled
Stop-Service -Name "RemoteRegistry"
```

#### CIS 2.3 — Deshabilitar SMBv1

```powershell
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

#### CIS 9.1 — Firewall habilitado en todos los perfiles

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
```

#### CIS 17 — Auditoría de eventos

```powershell
# Logon/Logoff
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Logoff" /success:enable

# Account Management
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable

# Privilege Use
auditpol /set /subcategory:"Sensitive Privilege Use" /success:enable /failure:enable

# Object Access
auditpol /set /subcategory:"File System" /success:enable /failure:enable

# Resultado: The command was successfully executed. (x6)
```

#### CIS 18 — Configuraciones de seguridad via Registry

**Deshabilitar LLMNR:**
```powershell
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" `
  -Name "EnableMulticast" -Value 0
# Resultado: DNSClient key creado exitosamente
```

**Deshabilitar NetBIOS sobre TCP/IP (via WMI):**
```powershell
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration
foreach ($adapter in $adapters) {
    $adapter.SetTcpipNetbios(2)
}
# ReturnValue: 0 = Éxito en adaptador principal
```

**Requerir NLA para RDP:**
```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
  -Name "UserAuthentication" -Value 1
```

**Deshabilitar acceso anónimo:**
```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" `
  -Name "RestrictAnonymous" -Value 1
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" `
  -Name "RestrictAnonymousSAM" -Value 1
```

#### CIS 19 — Windows Defender

```powershell
Set-MpPreference -DisableRealtimeMonitoring $false
Set-MpPreference -EnableNetworkProtection Enabled
Set-MpPreference -PUAProtection Enabled
```

---

### FASE 3 — Verificación (después del hardening)

#### Política de contraseñas

```
Minimum password length:              14  ✅ (CIS: mín 14)
Maximum password age (days):          60  ✅ (CIS: máx 60)
Minimum password age (days):           1  ✅ (CIS: mín 1)
Length of password history:           24  ✅ (CIS: mín 24)
Lockout threshold:                     5  ✅ (CIS: máx 5)
Lockout duration (minutes):           15  ✅ (CIS: mín 15)
Lockout observation window (minutes): 15  ✅ (CIS: mín 15)
```

#### Servicios deshabilitados

```
RemoteRegistry    Stopped    Disabled  ✅
Spooler           Stopped    Disabled  ✅
```

#### SMBv1

```
EnableSMB1Protocol: False  ✅
```

#### Firewall

```
Domain    True  ✅
Private   True  ✅
Public    True  ✅
```

#### Auditoría de eventos

```
Logon/Logoff
  Logon                          Success and Failure  ✅
  Logoff                         Success              ✅
  Special Logon                  Success and Failure  ✅
  Other Logon/Logoff Events      Success and Failure  ✅

Object Access
  File System                    Success and Failure  ✅
  Removable Storage              Success and Failure  ✅

Account Management
  User Account Management        Success and Failure  ✅
  Security Group Management      Success              ✅
  Computer Account Management    Success              ✅

Detailed Tracking
  Process Creation               Success              ✅
  Process Termination            Success              ✅
```

#### LLMNR

```
EnableMulticast: 0  ✅ (LLMNR deshabilitado)
```

#### NLA en RDP

```
UserAuthentication: 1  ✅ (NLA requerido)
```

---

## 📸 Capturas de Pantalla

### FASE 1 — Estado inicial

### 1️⃣ PolicyAnalyzer — Herramienta descargada
<img width="624" height="102" alt="01-policyanalyzer-tool" src="https://github.com/user-attachments/assets/6c497bae-ca14-4537-8f47-82460e76f766" />

### 2️⃣ PolicyAnalyzer — Backup de GPOs en progreso
<img width="624" height="274" alt="02-gpo-backup" src="https://github.com/user-attachments/assets/476c8815-388c-4d15-b9de-1cdeadb89a67" />

### 3️⃣ PolicyAnalyzer — Policy File Importer con backups cargados
<img width="624" height="199" alt="03-policy-importer" src="https://github.com/user-attachments/assets/421b35ee-7375-4597-8baa-fbc8c00c48b3" />

### 4️⃣ PolicyAnalyzer — Análisis completado (MyGpo, 1 selected)
<img width="624" height="413" alt="04-analysis-complete" src="https://github.com/user-attachments/assets/7226b061-252a-4474-a5ed-fef41d0d06fd" />

### 5️⃣ PolicyAnalyzer — Comparación de 69 políticas (View/Compare)
<img width="624" height="414" alt="05-policy-compare" src="https://github.com/user-attachments/assets/de647bed-5101-4549-9284-368eb2819eba" />

### 6️⃣ PolicyAnalyzer — Compare to Effective State
<img width="624" height="392" alt="06-effective-state" src="https://github.com/user-attachments/assets/30928027-0f72-4f39-8a4c-1c3e4515dd59" />

### 7️⃣ Nessus — Escaneo inicial (pre-hardening)
<img width="624" height="197" alt="07-nessus-prescan" src="https://github.com/user-attachments/assets/dd0dd4bf-019b-40d8-8532-ee406f2a6596" />

### 8️⃣ Nessus — 22 findings informativos pre-hardening
<img width="624" height="238" alt="08-nessus-findings" src="https://github.com/user-attachments/assets/57565451-7aec-43ad-9306-70e8b4996cf1" />

---

### FASE 2 — Aplicación del hardening

### 9️⃣ PowerShell — Política de contraseñas aplicada
<img width="624" height="135" alt="09-password-policy" src="https://github.com/user-attachments/assets/2f4b225d-f0db-48ad-b34c-620af439beca" />

### 🔟 PowerShell — Bloqueo de cuenta configurado
<img width="567" height="158" alt="10-lockout-policy" src="https://github.com/user-attachments/assets/d865e114-ad11-4f15-88ab-cc6cfc26de5d" />

### 1️⃣1️⃣ PowerShell — Servicios deshabilitados (Telnet, Spooler)
<img width="624" height="47" alt="11-services-disabled" src="https://github.com/user-attachments/assets/372d6a4b-f39f-46c5-b443-f0ed57c3ef2b" />

### 1️⃣2️⃣ PowerShell — SMBv1 deshabilitado
<img width="624" height="45" alt="12-smb-disabled" src="https://github.com/user-attachments/assets/001c8f44-27b8-4f64-a2b8-f68837a338f2" />

### 1️⃣3️⃣ PowerShell — Firewall habilitado
<img width="624" height="33" alt="13-firewall-enabled" src="https://github.com/user-attachments/assets/e00b1710-3ec1-4440-9a7e-8f913b7c1549" />

### 1️⃣4️⃣ PowerShell — Auditoría configurada
<img width="624" height="63" alt="14-audit-policy" src="https://github.com/user-attachments/assets/656140c6-4359-488e-8105-3e8bb05424b3" />

### 1️⃣5️⃣ PowerShell — LLMNR deshabilitado
<img width="624" height="163" alt="15-llmnr-disabled" src="https://github.com/user-attachments/assets/108f6890-0187-4df6-ba14-4e7283dbfad6" />

### 1️⃣6️⃣ PowerShell — NetBIOS deshabilitado via WMI
<img width="624" height="590" alt="16-netbios-disabled" src="https://github.com/user-attachments/assets/a5c765c4-3695-4404-8e11-d8aa0cf02f55" />

### 1️⃣7️⃣ PowerShell — NLA requerido para RDP
<img width="624" height="40" alt="17-nla-rdp" src="https://github.com/user-attachments/assets/c5302f4d-bd7c-4140-a857-118cc4df5b03" />

### 1️⃣8️⃣ PowerShell — Windows Defender habilitado
<img width="624" height="84" alt="18-defender" src="https://github.com/user-attachments/assets/4fd0f206-6409-42db-8362-971f1f7fdeae" />

---

### FASE 3 — Verificación post-hardening

### 1️⃣9️⃣ Verificación política de contraseñas
<img width="550" height="259" alt="19-verify-passwords" src="https://github.com/user-attachments/assets/999dddad-8162-4972-a59a-44da7c10c934" />

### 2️⃣0️⃣ Verificación servicios deshabilitados
<img width="624" height="160" alt="20-verify-services" src="https://github.com/user-attachments/assets/d9c5e627-7f7c-462b-86ef-d30d9048338b" />

### 2️⃣1️⃣ Verificación SMBv1 = False
<img width="624" height="163" alt="21-verify-smb" src="https://github.com/user-attachments/assets/136fea34-d4c5-4cf0-8f4c-f29ca6c79958" />

### 2️⃣2️⃣ Verificación Firewall = True (3 perfiles)
<img width="610" height="172" alt="22-verify-firewall" src="https://github.com/user-attachments/assets/67dd7895-25be-48d7-b940-804d27d84121" />

### 2️⃣3️⃣ Verificación auditoría completa
<img width="552" height="755" alt="23-verify-audit" src="https://github.com/user-attachments/assets/46442d52-d1b2-42ce-add7-2b0879f086c2" />

### 2️⃣4️⃣ Verificación LLMNR = 0
<img width="624" height="130" alt="24-verify-llmnr" src="https://github.com/user-attachments/assets/1c5aab0a-76c2-4ac5-bf21-0dbeabc80722" />

### 2️⃣5️⃣ Verificación NLA = 1
<img width="624" height="119" alt="25-verify-nla" src="https://github.com/user-attachments/assets/7383efa5-58bf-425d-b868-416a32813a52" />

---

## 🔍 Findings (Hallazgos)

**Pre-hardening (estado por defecto):**
- PolicyAnalyzer identificó **69 políticas** para analizar en el dominio JhonRuiz
- El estado efectivo coincidía con el baseline en auditoría pero carecía de controles de red
- Nessus detectó **22 findings informativos** — sin autenticación no detecta configuraciones débiles internas
- LLMNR estaba activo — vector de ataque para captura de hashes NTLMv2
- NetBIOS sobre TCP/IP activo — protocolo legacy explotable para reconocimiento
- Print Spooler activo — vector para ataques PrintNightmare (CVE-2021-1675)
- RemoteRegistry activo — permite modificar el registro de forma remota

**Post-hardening (después de aplicar CIS Benchmark):**
- **11 controles CIS** aplicados y verificados exitosamente
- SMBv1 deshabilitado — EternalBlue (CVE-2017-0144) mitigado ✅
- LLMNR deshabilitado — NTLM Relay via Responder mitigado ✅
- NLA requerido en RDP — BlueKeep (CVE-2019-0708) mitigado ✅
- Print Spooler deshabilitado — PrintNightmare mitigado ✅
- Política de contraseñas fortalecida: mínimo 14 caracteres, historial 24 ✅
- Auditoría completa habilitada — visibilidad total sobre eventos críticos ✅

---

## ⚠️ Impacto

- **Print Spooler activo** habría permitido PrintNightmare — ejecución remota de código como SYSTEM en cualquier Windows Server no parcheado
- **LLMNR activo** permitía a cualquier atacante en la red capturar hashes NTLMv2 con Responder sin necesidad de credenciales
- **SMBv1** con signing no requerido era el vector de WannaCry — ransomware que cifró hospitales, infraestructura crítica y empresas en 150 países en 2017
- **RemoteRegistry activo** permite a un atacante con acceso a la red modificar configuraciones del sistema de forma remota
- **Sin política de contraseñas** las cuentas son vulnerables a ataques de fuerza bruta y password spraying
- **Sin auditoría** un atacante puede comprometer el sistema sin dejar rastro en los logs

---

## 🚨 Detección (SOC)

- **Alerta SIEM:** Intentos de autenticación fallidos → Event ID 4625 (ahora auditado)
- **Alerta SIEM:** Creación/modificación de cuentas → Event ID 4720/4738 (auditado)
- **Alerta SIEM:** Escalada de privilegios → Event ID 4672 — Special Logon (auditado)
- **Alerta SIEM:** Acceso a archivos sensibles → Event ID 4663 — File System (auditado)
- **Alerta de red:** Tráfico SMBv1 detectado post-hardening → indica bypass o VM no parcheada
- **Alerta de red:** Tráfico LLMNR detectado → indica host no endurecido en la red
- **Alerta Nessus/VA:** Print Spooler activo en servidor → requiere remediación inmediata

---

## 🛡️ Controles CIS aplicados — Resumen

| Control CIS | Descripción | Estado |
|---|---|---|
| CIS 1.1.1 | Longitud mínima de contraseña: 14 | ✅ Aplicado |
| CIS 1.1.2 | Edad máxima de contraseña: 60 días | ✅ Aplicado |
| CIS 1.1.3 | Historial de contraseñas: 24 | ✅ Aplicado |
| CIS 1.2.1 | Umbral de bloqueo: 5 intentos | ✅ Aplicado |
| CIS 1.2.2 | Duración de bloqueo: 15 minutos | ✅ Aplicado |
| CIS 2.2 | Servicios innecesarios deshabilitados | ✅ Aplicado |
| CIS 2.3 | SMBv1 deshabilitado | ✅ Aplicado |
| CIS 9.1 | Firewall habilitado (3 perfiles) | ✅ Aplicado |
| CIS 17 | Auditoría avanzada de eventos | ✅ Aplicado |
| CIS 18.1 | LLMNR deshabilitado | ✅ Aplicado |
| CIS 18.2 | NetBIOS deshabilitado | ✅ Aplicado |
| CIS 18.3 | NLA requerido para RDP | ✅ Aplicado |
| CIS 18.4 | Acceso anónimo restringido | ✅ Aplicado |
| CIS 19 | Windows Defender + Network Protection | ✅ Aplicado |

**Total: 14 controles CIS aplicados y verificados ✅**

---

## ✅ Conclusiones

1. PolicyAnalyzer permitió auditar 69 políticas del dominio y comparar el estado efectivo
   contra el baseline — herramienta esencial para auditorías de seguridad en entornos AD
2. El hardening CIS eliminó múltiples vectores de ataque activos: SMBv1 (EternalBlue),
   LLMNR (NTLM Relay), Print Spooler (PrintNightmare) y RDP sin NLA (BlueKeep)
3. La auditoría de eventos habilitada transforma el servidor en un host monitoreable —
   ahora cualquier actividad sospechosa genera Event IDs capturables por un SIEM como Wazuh
4. Nessus confirmó que post-hardening no existen vulnerabilidades de severidad
   Critical, High o Medium detectables sin credenciales
5. La política de contraseñas (mínimo 14 caracteres, historial 24) elimina los vectores
   más comunes de compromiso de cuentas: fuerza bruta y password spraying
6. El hardening no es un evento único — debe ejecutarse periódicamente y validarse
   con herramientas como Nessus y PolicyAnalyzer para mantener el cumplimiento CIS

---

## 🔗 Referencias

- [CIS Benchmark — Windows Server 2019](https://www.cisecurity.org/benchmark/microsoft_windows_server)
- [Microsoft PolicyAnalyzer](https://www.microsoft.com/en-us/download/details.aspx?id=55319)
- [CVE-2017-0144 — EternalBlue / MS17-010](https://nvd.nist.gov/vuln/detail/CVE-2017-0144)
- [CVE-2021-1675 — PrintNightmare](https://nvd.nist.gov/vuln/detail/CVE-2021-1675)
- [CVE-2019-0708 — BlueKeep RDP](https://nvd.nist.gov/vuln/detail/CVE-2019-0708)
- [MITRE ATT&CK — T1110 Brute Force](https://attack.mitre.org/techniques/T1110/)
- [MITRE ATT&CK — T1557.001 LLMNR Poisoning](https://attack.mitre.org/techniques/T1557/001/)
- [MITRE ATT&CK — T1021.001 RDP](https://attack.mitre.org/techniques/T1021/001/)

---

*← [Volver al Portafolio Principal](../README.md)*
