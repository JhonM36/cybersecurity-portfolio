# 🛡️ Laboratorio 03 — Monitoreo con SIEM: Wazuh

![Wazuh](https://img.shields.io/badge/SIEM-Wazuh_4.7.5-blue?style=for-the-badge)
![Ubuntu](https://img.shields.io/badge/Server-Ubuntu_22.04-orange?style=for-the-badge&logo=ubuntu)
![Windows](https://img.shields.io/badge/Agent-Windows_10-0078D6?style=for-the-badge&logo=windows)
![MITRE](https://img.shields.io/badge/Framework-MITRE_ATT%26CK-red?style=for-the-badge)

---

## 🎬 Escenario

Un empleado con acceso administrativo crea una cuenta de usuario en un servidor
Windows un domingo a las 11 PM. Nadie lo autorizó. La cuenta se elimina 20 minutos
después — como si nunca hubiera existido.

Sin un SIEM, ese evento desaparece en los logs de Windows que nadie revisa.
Con Wazuh, genera una alerta nivel 8 en segundos, correlacionada automáticamente
con MITRE ATT&CK T1098 (Account Manipulation) y T1531 (Account Access Removal).

Este laboratorio instala Wazuh desde cero, conecta un agente en Windows 10,
y reproduce exactamente ese escenario para demostrar qué ve un analista SOC
cuando algo sale mal — y por qué el tiempo de detección importa.

---

## 🎯 Objetivo

Instalar y configurar Wazuh como plataforma SIEM para monitorear eventos de seguridad
en tiempo real, conectar un agente en Windows 10 y analizar alertas correlacionadas
con el framework MITRE ATT&CK. Se simulan actividades maliciosas reales para verificar
la capacidad de detección del SIEM.

---

## 🖥️ Entorno

| Rol | Sistema | IP | Versión |
|-----|---------|----|---------|
| Wazuh Server | Ubuntu | 192.168.1.46 | 22.04 LTS |
| Wazuh Agent | Windows 10 Pro | 192.168.1.45 | 10.0.19045 |
| Wazuh | — | — | 4.7.5 |

---

## 🛠️ Herramientas utilizadas

- **Wazuh Manager** — Motor de correlación de eventos y alertas
- **Wazuh Indexer** (OpenSearch) — Almacenamiento y búsqueda de logs
- **Wazuh Dashboard** — Interfaz web de visualización y análisis
- **Filebeat** — Transporte de logs entre componentes
- **MITRE ATT&CK Framework** — Correlación de técnicas de ataque

---

## 📋 Procedimiento

### Paso 1 — Preparación del servidor Ubuntu

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml
```

Se editó `config.yml` con la IP `192.168.1.46` en los nodos de indexer,
server y dashboard.

### Paso 2 — Instalación de componentes

```bash
sudo bash wazuh-install.sh --generate-config-files -i
sudo bash wazuh-install.sh --wazuh-indexer node-1 -i
sudo bash wazuh-install.sh --wazuh-server wazuh-1 -i
sudo bash wazuh-install.sh --start-cluster -i
sudo bash wazuh-install.sh --wazuh-dashboard dashboard -i
```

### Paso 3 — Verificación de servicios

```bash
sudo systemctl status wazuh-manager    # active (running) ✅
sudo systemctl status wazuh-indexer    # active (running) ✅
sudo systemctl status wazuh-dashboard  # active (running) ✅
```

### Paso 4 — Acceso al Dashboard

`https://192.168.1.46` → advertencia SSL normal en laboratorio → login `admin` ✅

### Paso 5 — Instalación del agente en Windows 10

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi `
  -OutFile $env:tmp\wazuh-agent.msi

msiexec.exe /i $env:tmp\wazuh-agent.msi /q `
  WAZUH_MANAGER='192.168.1.46' WAZUH_AGENT_NAME='W10'

NET START WazuhSvc
# The Wazuh service was started successfully ✅
```

### Paso 6 — Simulación de actividad maliciosa

```cmd
net user usuarioprueba Password123 /add
net user usuarioprueba /delete
```

Wazuh detectó y correlacionó con MITRE ATT&CK automáticamente.

---

## ⚠️ Errores encontrados y soluciones

### Error 1 — Sistema operativo no compatible

```
ERROR: The recommended systems are: Red Hat Enterprise Linux 7, 8, 9;
CentOS 7, 8; Amazon Linux 2; Ubuntu 16.04, 18.04, 20.04, 22.04.
```

**Solución:** Flag `-i` en todos los comandos para omitir verificación de SO.

---

### Error 2 — Dashboard no pudo conectarse al Indexer

```
ERROR: Wazuh indexer security settings not initialized.
INFO: Wazuh dashboard removed.
```

**Causa:** El cluster del Indexer debe iniciarse antes de instalar el dashboard.

**Solución:**
```bash
sudo bash wazuh-install.sh --start-cluster -i
# INFO: Wazuh indexer cluster started. ✅
sudo bash wazuh-install.sh --wazuh-dashboard dashboard -i
```

---

## 📸 Capturas de Pantalla

### 1️⃣ Error durante la instalación
![wazuh-error](capturas/01-wazuh-error.png)

### 2️⃣ Solución — Inicialización del cluster
![wazuh-solucion](capturas/02-wazuh-solucion.png)

### 3️⃣ Wazuh Manager — active (running)
![wazuh-manager](capturas/03-wazuh-manager.png)

### 4️⃣ Wazuh Indexer — active (running)
![wazuh-indexer](capturas/04-wazuh-indexer.png)

### 5️⃣ Wazuh Dashboard — active (running)
![wazuh-dashboard](capturas/05-wazuh-dashboard.png)

### 6️⃣ Login de Wazuh
![login](capturas/06-login.png)

### 7️⃣ Advertencia SSL del navegador
![ssl-warning](capturas/07-ssl-warning.png)

### 8️⃣ Dashboard principal
![dashboard](capturas/08-dashboard.png)

### 9️⃣ Instalación del agente en Windows 10
![agente-instalacion](capturas/09-agente-instalacion.png)

### 🔟 Agente W10 activo en el dashboard
![agente-activo](capturas/10-agente-activo.png)

### 1️⃣1️⃣ Alertas con MITRE ATT&CK
![alertas](capturas/11-alertas-mitre.png)

---

## 🔍 Findings (Hallazgos)

- Wazuh detectó en tiempo real la **creación y eliminación** del usuario `usuarioprueba`
- **5 técnicas MITRE ATT&CK** correlacionadas automáticamente
- Nivel de alerta **8** para operaciones de usuario — priorización correcta

| Técnica | Táctica | Descripción | Nivel |
|---------|---------|-------------|-------|
| T1098, T1531 | Persistence, Impact | User account disabled or deleted | 🔴 8 |
| T1098 | Persistence | User account enabled or created | 🔴 8 |
| T1098 | Persistence | User account changed | 🔴 8 |
| T1484 | Defense Evasion, Privilege Escalation | Domain users group changed | 🟡 5 |
| T1484 | Defense Evasion, Privilege Escalation | Users group changed | 🟡 5 |

---

## ⚠️ Impacto

- **Creación de usuarios no autorizados** (T1098) es persistencia — el atacante mantiene acceso aunque su vector inicial sea cerrado
- **Eliminación de usuarios** (T1531) puede destruir evidencia forense de cuentas comprometidas
- **Cambios en grupos de dominio** (T1484) permiten escalada de privilegios silenciosa
- Sin SIEM, estas actividades pasan desapercibidas en logs dispersos que nadie revisa

---

## 🚨 Detección (SOC)

- **Alerta Wazuh nivel 8:** `net user /add` — creación de cuenta local sospechosa
- **Alerta Wazuh nivel 8:** `net user /delete` — eliminación post-actividad
- **Alerta Wazuh nivel 5:** Modificación de grupos — posible escalada de privilegios
- **Event ID 4720:** User account created
- **Event ID 4726:** User account deleted
- **Correlación MITRE:** T1098 mapeado automáticamente

---

## 🛡️ Mitigación

```
Group Policy → User Rights Assignment →
"Add users to local groups" → Solo Administradores autorizados

auditpol → Account Management → Success and Failure

Wazuh → Alertas nivel ≥ 7 → Notificación por email/Slack
```

```powershell
# Deshabilitar cuenta Administrator local
Disable-LocalUser -Name "Administrator"
```

---

## 💼 ¿Por qué importa esto en el mundo real?

La pregunta que más escucho en entornos sin SIEM es: "¿para qué necesitamos eso
si nunca nos han atacado?" Este laboratorio responde esa pregunta con evidencia.

El comando `net user /add` seguido de `net user /delete` es una técnica real
documentada en el Cyber Kill Chain — un atacante crea una cuenta de backdoor,
la usa para moverse lateralmente, y la elimina antes de que alguien lo note.
Sin Wazuh, ese movimiento es invisible. Con Wazuh, genera 5 alertas MITRE
en menos de 30 segundos.

En la DGA, donde trabajo actualmente, el volumen de eventos de seguridad
en un servidor con Active Directory puede superar los 10,000 logs por hora.
Ningún humano puede revisar eso manualmente. Un SIEM no es un lujo —
es la diferencia entre detectar un incidente en minutos o en meses.

**Lo más importante que aprendí en este lab:** configurar el SIEM es la parte
fácil. La parte difícil — y la que más valor tiene para un empleador —
es saber interpretar las alertas, reducir falsos positivos, y entender
qué técnica MITRE está detrás de cada evento. Eso es lo que practica este laboratorio.

---

## ✅ Conclusiones

1. Wazuh 4.7.5 instalado exitosamente en Ubuntu resolviendo errores reales de instalación
2. Agente conectado en Windows 10 con cobertura completa del host
3. Detección en tiempo real de creación/eliminación de usuarios con 5 técnicas MITRE
4. Nivel 8 confirma que Wazuh prioriza correctamente eventos de alto riesgo
5. Sin SIEM, actividades de persistencia como T1098 pasan desapercibidas

---

## 🔗 Referencias

- [Documentación oficial de Wazuh](https://documentation.wazuh.com)
- [MITRE ATT&CK — T1098 Account Manipulation](https://attack.mitre.org/techniques/T1098/)
- [MITRE ATT&CK — T1531 Account Access Removal](https://attack.mitre.org/techniques/T1531/)
- [MITRE ATT&CK — T1484 Domain Policy Modification](https://attack.mitre.org/techniques/T1484/)
- [Windows Security Event IDs Reference](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/advanced-security-audit-policy-settings)

---

*← [Volver al Portafolio Principal](../README.md)*
