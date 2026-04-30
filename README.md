# Jhon M. Ruiz Melenciano — Cybersecurity Portfolio

![Banner](https://img.shields.io/badge/Cybersecurity-Portfolio-blue?style=for-the-badge&logo=github)
![CCNA](https://img.shields.io/badge/Certified-CCNA-green?style=for-the-badge&logo=cisco)
![Google](https://img.shields.io/badge/Google-Cybersecurity-red?style=for-the-badge&logo=google)
![Blue Team](https://img.shields.io/badge/Focus-Blue_Team_/_SOC-informational?style=for-the-badge)
![Labs](https://img.shields.io/badge/Labs_Completados-6_/_15-success?style=for-the-badge)

> Analista de Ciberseguridad orientado a Blue Team con experiencia en monitoreo SIEM, análisis de tráfico y hardening de sistemas.
> Enfocado en detección de amenazas, análisis de vulnerabilidades y seguridad perimetral.

---

## 👤 Sobre mí

- 📍 Santo Domingo, República Dominicana
- 🎓 Tecnólogo en Ciberseguridad — ITLA (en curso)
- 🔍 Orientado a Blue Team, SOC Tier 1 y Seguridad de Redes
- 📧 maicolruiz560@gmail.com

---

## 💼 Experiencia

**Pasante — Seguridad Electrónica | Dirección General de Aduanas (DGA)**
`Noviembre 2025 – Febrero 2026`
Instalación y soporte de sistemas CCTV (IP y analógica), control de acceso biométrico,
sistemas de alarma e intrusión y detección de incendios.

**Soporte Técnico | Consultoría Tecnológica Avanzada — Grupos CTA**
`Enero 2025 – Julio 2025`
Administración de redes LAN empresariales, soporte a VPN, configuración de switches
y routers, gestión de tickets con Freshdesk según SLA, administración de centrales VoIP.

---

## 🛠️ Habilidades Técnicas

| Área | Herramientas / Tecnologías |
|------|---------------------------|
| Análisis de tráfico | Wireshark |
| Escaneo y vulnerabilidades | Nmap, Nessus |
| SIEM / Monitoreo | Wazuh, Splunk (laboratorio) |
| Firewall / Seguridad perimetral | pfSense, ACLs, NAT, VPN |
| Hardening | CIS Benchmark, PolicyAnalyzer, Group Policy |
| Offensive Security | Metasploit, msfvenom, Meterpreter |
| Redes | Switches, Routers, LAN/WAN, VoIP, Cableado estructurado |
| Seguridad electrónica | CCTV, Control de acceso biométrico, Alarmas |
| Sistemas | Windows 10/11, Windows Server 2019, Linux, macOS |
| Cloud | Microsoft Azure (básico) |
| Programación | Python, JavaScript, SQL Server |

---

## 🗂️ Laboratorios

Todos los laboratorios incluyen formato SOC:
**Findings → Impacto → Detección → Mitigación**

| # | Laboratorio | Herramientas | Técnicas MITRE | Estado |
|---|-------------|--------------|----------------|--------|
| 01 | [Análisis de Tráfico + Credential Harvesting](./01-network-analysis/) | Wireshark, Nmap, Apache2 | T1040, T1557 | ✅ Completado |
| 02 | [Escaneo de Vulnerabilidades](./02-vulnerability-scanning/) | Nmap NSE, Nessus Essentials | T1046, T1557.001 | ✅ Completado |
| 03 | [Monitoreo con SIEM — Wazuh](./03-siem-monitoring/) | Wazuh 4.7.5, MITRE ATT&CK | T1098, T1531, T1484 | ✅ Completado |
| 04 | [Hardening en Windows Server 2019](./04-hardening/) | CIS Benchmark, PolicyAnalyzer, PowerShell | T1110, T1557.001 | ✅ Completado |
| 05 | [Firewall, ACLs y NAT con pfSense](./05-network-security/) | pfSense 2.8.1, Nmap, IIS | T1046, T1021.001/002 | ✅ Completado |
| 06 | [Reverse Shell Detection](./06-reverse-shell-detection/) | Metasploit, msfvenom, Wireshark | T1059, T1071, T1105, T1572 | ✅ Completado |

---

## 🔬 Resumen de hallazgos por laboratorio

**Lab 01 — Wireshark:** Captura de credenciales en texto claro (`user=hola`, `pass=mundo1`)
via HTTP POST desde servidor Apache falso. Identificación de SYN scan y TTL fingerprinting.

**Lab 02 — Nmap + Nessus:** SMB Signing not required (CVSS 5.3), LLMNR activo,
SMBv1 habilitado deliberadamente. Comparación antes/después con 15 → 28+ findings.

**Lab 03 — Wazuh SIEM:** Detección en tiempo real de creación/eliminación de usuarios.
5 técnicas MITRE ATT&CK correlacionadas automáticamente. Alertas nivel 8.

**Lab 04 — Hardening CIS:** 14 controles CIS aplicados y verificados. Eliminación de
vectores: EternalBlue, PrintNightmare, BlueKeep, NTLM Relay, LLMNR Poisoning.

**Lab 05 — pfSense:** ICMP bloqueado (100% packet loss), ACLs en puertos 23/445/3389
(`filtered`), NAT port forwarding 8080→80 verificado con `HTTP 200 OK — IIS 10.0`.

**Lab 06 — Reverse Shell:** Sesión Meterpreter abierta como `JHON1499\Administrator`.
Payload de 7,680 bytes — conexión TCP `ESTABLISHED` en puerto 4444 capturada en Wireshark.
IOCs identificados: proceso desde `Downloads`, puerto no estándar, stage de 230,982 bytes.

---

## 📜 Certificaciones

- ✅ CCNA: Introduction to Networks
- ✅ CCNA: Switching, Routing, and Wireless Essentials
- ✅ CCNA: Enterprise Networking, Security, and Automation
- ✅ IT Essentials
- ✅ Google Cybersecurity

---

## 📊 GitHub Stats

![Stats](https://github-readme-stats.vercel.app/api?username=JhonM36&show_icons=true&theme=dark&hide_border=true)
![Top Langs](https://github-readme-stats.vercel.app/api/top-langs/?username=JhonM36&theme=dark&hide_border=true&layout=compact)

---

> *"La ciberseguridad no es un destino, es un proceso continuo."*
