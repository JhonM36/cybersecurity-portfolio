# 🛡️ Laboratorio 03 — Monitoreo con SIEM: Wazuh

![Wazuh](https://img.shields.io/badge/SIEM-Wazuh_4.7.5-blue?style=for-the-badge)
![Ubuntu](https://img.shields.io/badge/Server-Ubuntu-orange?style=for-the-badge&logo=ubuntu)
![Windows](https://img.shields.io/badge/Agent-Windows_10-0078D6?style=for-the-badge&logo=windows)
![MITRE](https://img.shields.io/badge/Framework-MITRE_ATT%26CK-red?style=for-the-badge)

## 🎯 Objetivo
Instalar y configurar Wazuh como plataforma SIEM para monitorear eventos de seguridad
en tiempo real, conectar un agente en Windows 10 y analizar alertas correlacionadas
con el framework MITRE ATT&CK.

---

## 🖥️ Entorno

| Rol | Sistema | IP | Versión |
|-----|---------|----|---------|
| Wazuh Server | Ubuntu | 192.168.1.46 | 22.04 |
| Wazuh Agent | Windows 10 Pro | 192.168.1.45 | 10.0.19045 |
| Wazuh | - | - | 4.7.5 |

---

## 🛠️ Herramientas utilizadas
- Wazuh Manager
- Wazuh Indexer (OpenSearch)
- Wazuh Dashboard
- Filebeat
- MITRE ATT&CK Framework

---

## 📋 Procedimiento

### Paso 1 — Preparación del servidor Ubuntu
Se descargó el instalador oficial y el archivo de configuración de Wazuh:
```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml
```
Se editó `config.yml` colocando la IP del servidor `192.168.1.46` en los nodos
de indexer, server y dashboard.

### Paso 2 — Instalación de componentes
Se instalaron los componentes en orden usando el flag `-i` para omitir
la verificación de sistema operativo:
```bash
sudo bash wazuh-install.sh --generate-config-files -i
sudo bash wazuh-install.sh --wazuh-indexer node-1 -i
sudo bash wazuh-install.sh --wazuh-server wazuh-1 -i
sudo bash wazuh-install.sh --wazuh-dashboard dashboard -i
# ⚠️ Error: dashboard falló — ver sección de errores
sudo bash wazuh-install.sh --start-cluster -i
sudo bash wazuh-install.sh --wazuh-dashboard dashboard -i
```

### Paso 3 — Verificación de servicios
Se verificó que los tres servicios principales estuvieran activos:
```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```
Los tres servicios mostraron estado `active (running)` ✅

### Paso 4 — Acceso al Dashboard
Se accedió a la interfaz web en `https://192.168.1.46` desde el navegador.
El navegador mostró una advertencia SSL por certificado self-signed,
lo cual es normal en entornos de laboratorio. Se aceptó el riesgo y
se inició sesión con el usuario `admin`.

### Paso 5 — Instalación del agente en Windows 10
Desde el dashboard se generó el comando de instalación del agente.
Se ejecutó en PowerShell como Administrador en la VM Windows 10:
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi `
-OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q `
WAZUH_MANAGER='192.168.1.46' WAZUH_AGENT_NAME='W10'
NET START WazuhSvc
```
Resultado: `The Wazuh service was started successfully` ✅

### Paso 6 — Generación de alertas de seguridad
Desde CMD como Administrador en Windows 10 se creó y eliminó un usuario
para simular actividad sospechosa:
```cmd
net user usuarioprueba Password123 /add
net user usuarioprueba /delete
```
Wazuh detectó y correlacionó automáticamente los eventos con MITRE ATT&CK.

---

## ⚠️ Errores encontrados y soluciones

### Error 1 — Sistema operativo no compatible
Durante la instalación Wazuh detectó que el sistema operativo no estaba
en su lista oficial de sistemas soportados, mostrando el siguiente error:
```
ERROR: The recommended systems are: Red Hat Enterprise Linux 7, 8, 9;
CentOS 7, 8; Amazon Linux 2; Ubuntu 16.04, 18.04, 20.04, 22.04.
```
**Solución:** Se agregó el flag `-i` a todos los comandos de instalación
para omitir la verificación del sistema operativo.

---

### Error 2 — Dashboard no pudo conectarse al Indexer
Al intentar instalar el dashboard, el sistema indicó que el Wazuh Indexer
no había sido inicializado correctamente, por lo que la instalación falló
y el dashboard fue removido automáticamente:
```
ERROR: Cannot connect to Wazuh dashboard.
ERROR: Wazuh indexer security settings not initialized.
Please run the installation assistant using -s|--start-cluster
in one of the wazuh indexer nodes.
INFO: --- Removing existing Wazuh installation ---
INFO: Wazuh dashboard removed.
```
**Causa:** El cluster del Indexer debe iniciarse antes de instalar
el dashboard.

**Solución:** Se ejecutó primero el comando de inicialización del cluster:
```bash
sudo bash wazuh-install.sh --start-cluster -i
```
Resultado:
```
INFO: Wazuh indexer cluster security configuration initialized.
INFO: Wazuh indexer cluster started.
```
Luego se reinstalo el dashboard exitosamente:
```bash
sudo bash wazuh-install.sh --wazuh-dashboard dashboard -i
```
---

## 📸 Evidencias

### 1️⃣ Wazuh Manager activo
El servicio `wazuh-manager` en estado `active (running)` con todos sus
procesos internos iniciados correctamente.

![wazuh-manager](capturas/01-wazuh-manager.png)

### 2️⃣ Wazuh Indexer activo
El servicio `wazuh-indexer` en estado `active (running)`, motor de
búsqueda basado en OpenSearch responsable de indexar todos los eventos.

![wazuh-indexer](capturas/02-wazuh-indexer.png)

### 3️⃣ Wazuh Dashboard activo
El servicio `wazuh-dashboard` en estado `active (running)`, confirmando
que la interfaz web está disponible.

![wazuh-dashboard](capturas/03-wazuh-dashboard.png)

### 4️⃣ Advertencia SSL del navegador
Firefox detectó el certificado self-signed generado por Wazuh durante
la instalación. Comportamiento esperado en entornos de laboratorio.

![ssl-warning](capturas/04-ssl-warning.png)

### 5️⃣ Login de Wazuh
Acceso exitoso a la interfaz web del dashboard con el usuario `admin`.

![login](capturas/05-login.png)

### 6️⃣ Dashboard principal
Panel principal de Wazuh con todos sus módulos disponibles: Security Events,
Integrity Monitoring, Policy Monitoring, System Auditing, Vulnerabilities,
MITRE ATT&CK, PCI DSS y NIST 800-53.

![dashboard](capturas/06-dashboard.png)

### 7️⃣ Instalación del agente en Windows 10
Ejecución del comando de instalación en PowerShell como Administrador.
El agente quedó instalado y el servicio inició exitosamente.

![agente-instalacion](capturas/07-agente-instalacion.png)

### 8️⃣ Agente W10 activo en el dashboard
El agente W10 aparece conectado con status `active`, IP `192.168.1.45`,
Windows 10 Pro y cobertura del 100%.

![agente-activo](capturas/08-agente-activo.png)

### 9️⃣ Alertas de seguridad con MITRE ATT&CK
Wazuh detectó y correlacionó automáticamente los eventos generados
con las siguientes técnicas del framework MITRE ATT&CK:

| Técnica | Táctica | Descripción | Nivel |
|---------|---------|-------------|-------|
| T1098, T1531 | Persistence, Impact | User account disabled or deleted | 🔴 8 |
| T1098 | Persistence | User account enabled or created | 🔴 8 |
| T1098 | Persistence | User account changed | 🔴 8 |
| T1484 | Defense Evasion, Privilege Escalation | Domain users group changed | 🟡 5 |
| T1484 | Defense Evasion, Privilege Escalation | Users group changed | 🟡 5 |

![alertas](capturas/09-alertas-mitre.png)

---

## ✅ Conclusiones

1. Se instaló exitosamente Wazuh 4.7.5 como plataforma SIEM completa
   en Ubuntu, incluyendo Manager, Indexer y Dashboard.
2. Se conectó un agente en Windows 10 Pro, logrando una cobertura del 100%.
3. Wazuh detectó automáticamente eventos de creación y eliminación de
   usuarios, correlacionándolos con técnicas del framework MITRE ATT&CK.
4. Se aprendió a resolver errores reales de instalación como la
   incompatibilidad de sistema operativo y el problema de inicialización
   del cluster.
5. El laboratorio demuestra capacidad de despliegue, configuración y
   monitoreo básico de un SIEM en un entorno virtualizado.

---

## 🔗 Referencias
- [Documentación oficial de Wazuh](https://documentation.wazuh.com)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
