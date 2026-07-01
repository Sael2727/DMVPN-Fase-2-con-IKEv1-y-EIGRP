# 🔐 DMVPN Fase 2 — IPSec IKEv1 + EIGRP

<div align="center">

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![DMVPN](https://img.shields.io/badge/DMVPN-Fase%202-green?style=for-the-badge)
![IKEv1](https://img.shields.io/badge/IPSec-IKEv1-purple?style=for-the-badge)
![EIGRP](https://img.shields.io/badge/Routing-EIGRP-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-red?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-gray?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Configuración y validación de una **VPN DMVPN Fase 2 con IPSec IKEv1 y enrutamiento dinámico EIGRP**. La solución permite que dos sucursales (SPOKE1 y SPOKE2) se registren contra un HUB central mediante **NHRP** y puedan comunicarse dinámicamente. En **Fase 2**, el tráfico entre spokes puede tomar un camino directo spoke-to-spoke después de la resolución NHRP, sin depender del HUB como salto permanente.

> 💡 **Claves técnicas de Fase 2:**
> - El HUB usa `no ip split-horizon eigrp 25` y `no ip next-hop-self eigrp 25` en Tunnel0 para que los spokes aprendan rutas con el next-hop original del spoke remoto (no el HUB).
> - La clave ISAKMP wildcard `0.0.0.0 0.0.0.0` en todos los routers permite que SPOKE1 y SPOKE2 formen IPSec directo entre ellos sin pasar por el HUB.
> - IPSec opera en **modo transport** (no tunnel), ya que GRE multipoint provee el encapsulado del túnel.

---

## 🗺️ Topología de Red

Un HUB central con dos SPOKES y un ISP que simula la red pública. Cada sitio tiene una LAN con una VPC. El HUB también cuenta con su propia LAN (VPCH).

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara | Función |
|:-----------:|:--------:|:-------------:|---------|
| HUB | Ethernet0/0 | 10.7.25.1/30 | WAN hacia ISP |
| ISP | Ethernet0/0 | 10.7.25.2/30 | Enlace hacia HUB |
| ISP | Ethernet0/2 | 10.7.25.6/30 | Enlace hacia SPOKE1 |
| ISP | Ethernet0/1 | 10.7.25.10/30 | Enlace hacia SPOKE2 |
| SPOKE1 | Ethernet0/0 | 10.7.25.5/30 | WAN hacia ISP |
| SPOKE2 | Ethernet0/0 | 10.7.25.9/30 | WAN hacia ISP |
| HUB | Ethernet0/1 | 10.7.25.65/27 | Gateway LAN HUB |
| VPCH | eth0 | 10.7.25.66/27 | Cliente LAN HUB |
| SPOKE1 | Ethernet0/1 | 10.7.25.97/27 | Gateway LAN SPOKE1 |
| VPC1 | eth0 | 10.7.25.98/27 | Cliente LAN SPOKE1 |
| SPOKE2 | Ethernet0/1 | 10.7.25.129/27 | Gateway LAN SPOKE2 |
| VPC2 | eth0 | 10.7.25.130/27 | Cliente LAN SPOKE2 |
| **HUB** | **Tunnel0** | **10.7.25.161/27** | **DMVPN NHS** |
| **SPOKE1** | **Tunnel0** | **10.7.25.162/27** | **DMVPN Spoke** |
| **SPOKE2** | **Tunnel0** | **10.7.25.163/27** | **DMVPN Spoke** |

### 📡 Segmentos

| Segmento | Red | Descripción |
|:--------:|:---:|-------------|
| HUB - ISP | 10.7.25.0/30 | Enlace WAN HUB |
| SPOKE1 - ISP | 10.7.25.4/30 | Enlace WAN SPOKE1 |
| SPOKE2 - ISP | 10.7.25.8/30 | Enlace WAN SPOKE2 |
| LAN HUB | 10.7.25.64/27 | Red local del HUB |
| LAN SPOKE1 | 10.7.25.96/27 | Red local de SPOKE1 |
| LAN SPOKE2 | 10.7.25.128/27 | Red local de SPOKE2 |
| Red DMVPN | 10.7.25.160/27 | Red multipoint del túnel |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Tipo de VPN | DMVPN Fase 2 Hub-and-Spoke |
| Peers | 1 HUB y 2 SPOKES |
| Túnel | GRE multipoint sobre Tunnel0 |
| Resolución de peers | NHRP, network-id 725, autenticación DMVPN123 |
| Cifrado IPSec | IKEv1, AES-256, SHA, grupo 5 |
| Transform-set | TS-DMVPN: esp-aes 256 esp-sha-hmac |
| Modo IPSec | Transport mode |
| Clave precompartida | VPN12345 (wildcard 0.0.0.0 0.0.0.0) |
| Enrutamiento dinámico | EIGRP AS 25 |
| Objetivo Fase 2 | Spoke-to-spoke directo por NHRP |

---

## 🔍 Funcionamiento

DMVPN combina cuatro componentes trabajando juntos:

**GRE Multipoint** — Tunnel0 en cada router opera en modo `gre multipoint`, lo que permite que un único túnel lógico soporte múltiples peers dinámicamente, sin necesidad de configurar un túnel por cada par.

**NHRP** — Los spokes se registran en el HUB (NHS) al arrancar, informando su dirección NBMA (WAN real). En Fase 2, cuando SPOKE1 quiere comunicarse con SPOKE2, consulta al HUB la dirección NBMA de SPOKE2 y luego establece un túnel directo.

**IPSec IKEv1** — Protege el tráfico GRE con la clave wildcard `0.0.0.0 0.0.0.0` para que cualquier peer pueda negociar IPSec con cualquier otro, no solo con el HUB.

**EIGRP AS 25** — Distribuye las redes LAN entre todos los sitios. El HUB usa `no ip next-hop-self eigrp 25` para anunciar las rutas con el next-hop original del spoke, lo que permite que el tráfico spoke-to-spoke no pase por el HUB.

---

## 🚀 Scripts de Configuración

### ISP
```cisco
hostname ISP
ip cef
interface Ethernet0/0
 description WAN_HACIA_HUB
 ip address 10.7.25.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description WAN_HACIA_SPOKE2
 ip address 10.7.25.10 255.255.255.252
 no shutdown
interface Ethernet0/2
 description WAN_HACIA_SPOKE1
 ip address 10.7.25.6 255.255.255.252
 no shutdown
```

### HUB — Configuración Principal
```cisco
hostname HUB
ip cef
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_HUB
 ip address 10.7.25.65 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.2

crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
crypto isakmp key VPN12345 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha-hmac
 mode transport

crypto ipsec profile DMVPN-PROFILE
 set transform-set TS-DMVPN
 set pfs group5

interface Tunnel0
 description DMVPN_PHASE2_HUB
 ip address 10.7.25.161 255.255.255.224
 no ip redirects
 ip nhrp authentication DMVPN123
 ip nhrp map multicast dynamic
 ip nhrp network-id 725
 no ip split-horizon eigrp 25
 no ip next-hop-self eigrp 25
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 725
 tunnel protection ipsec profile DMVPN-PROFILE
 no shutdown

router eigrp 25
 no auto-summary
 passive-interface default
 no passive-interface Tunnel0
 network 10.7.25.64 0.0.0.31
 network 10.7.25.160 0.0.0.31
```

### SPOKE1 — Configuración Principal
```cisco
hostname SPOKE1
ip cef
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.5 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_SPOKE1
 ip address 10.7.25.97 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.6

crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
crypto isakmp key VPN12345 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha-hmac
 mode transport

crypto ipsec profile DMVPN-PROFILE
 set transform-set TS-DMVPN
 set pfs group5

interface Tunnel0
 description DMVPN_PHASE2_SPOKE1
 ip address 10.7.25.162 255.255.255.224
 no ip redirects
 ip nhrp authentication DMVPN123
 ip nhrp map 10.7.25.161 10.7.25.1
 ip nhrp map multicast 10.7.25.1
 ip nhrp nhs 10.7.25.161
 ip nhrp network-id 725
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 725
 tunnel protection ipsec profile DMVPN-PROFILE
 no shutdown

router eigrp 25
 no auto-summary
 passive-interface default
 no passive-interface Tunnel0
 network 10.7.25.96 0.0.0.31
 network 10.7.25.160 0.0.0.31
```

### SPOKE2 — Configuración Principal
```cisco
hostname SPOKE2
ip cef
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.9 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_SPOKE2
 ip address 10.7.25.129 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.10

crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
crypto isakmp key VPN12345 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha-hmac
 mode transport

crypto ipsec profile DMVPN-PROFILE
 set transform-set TS-DMVPN
 set pfs group5

interface Tunnel0
 description DMVPN_PHASE2_SPOKE2
 ip address 10.7.25.163 255.255.255.224
 no ip redirects
 ip nhrp authentication DMVPN123
 ip nhrp map 10.7.25.161 10.7.25.1
 ip nhrp map multicast 10.7.25.1
 ip nhrp nhs 10.7.25.161
 ip nhrp network-id 725
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 725
 tunnel protection ipsec profile DMVPN-PROFILE
 no shutdown

router eigrp 25
 no auto-summary
 passive-interface default
 no passive-interface Tunnel0
 network 10.7.25.128 0.0.0.31
 network 10.7.25.160 0.0.0.31
```

### Configuración de VPCs
```bash
# VPCH (LAN HUB)
ip 10.7.25.66 255.255.255.224 10.7.25.65

# VPC1 (LAN SPOKE1)
ip 10.7.25.98 255.255.255.224 10.7.25.97

# VPC2 (LAN SPOKE2)
ip 10.7.25.130 255.255.255.224 10.7.25.129
```

---

## ✅ Verificación

```cisco
show dmvpn
show ip nhrp
show crypto isakmp sa
show crypto ipsec sa
show crypto session
show ip eigrp neighbors
show ip route eigrp
```

| Comando | Estado esperado |
|:-------:|------------------|
| `show dmvpn` en HUB | 2 peers NHRP registrados (SPOKE1 y SPOKE2) |
| `show ip nhrp` en HUB | Entradas dinámicas de ambos spokes |
| `show crypto isakmp sa` | QM_IDLE entre HUB↔SPOKE1, HUB↔SPOKE2 y SPOKE1↔SPOKE2 |
| `show crypto ipsec sa` | encaps/decaps activos |
| `show crypto session` | UP-ACTIVE hacia todos los peers |
| `show ip route eigrp` | LANs remotas aprendidas por EIGRP |
| `show ip nhrp` en SPOKE1 | Entrada directa hacia SPOKE2 (NBMA 10.7.25.9) |
| `show ip nhrp` en SPOKE2 | Entrada directa hacia SPOKE1 (NBMA 10.7.25.5) |

> 💡 **Evidencia clave de Fase 2:** El trace VPC1 → VPC2 debe mostrar el camino directo `10.7.25.97 → 10.7.25.163 → 10.7.25.130`, sin pasar por la IP del HUB. Si el HUB aparece como salto intermedio, Fase 2 aún no estableció el túnel spoke-to-spoke directo; genera tráfico y repite el trace.

> 💡 **Nota sobre traceroute:** El mensaje "Destination port unreachable" al final del trace es normal en VPCS — usa paquetes UDP y el destino responde que el puerto no está disponible. Confirma que el host final fue alcanzado.

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Interfaces WAN, LAN y Tunnel0 en todos los routers | ✅ up/up |
| NHRP: SPOKE1 y SPOKE2 registrados en HUB | ✅ 2 peers en `show dmvpn` |
| IKEv1 SA (HUB↔SPOKE1, HUB↔SPOKE2, SPOKE1↔SPOKE2) | ✅ QM_IDLE |
| IPSec SA | ✅ encaps/decaps activos |
| Sesiones crypto | ✅ UP-ACTIVE |
| Vecinos EIGRP por Tunnel0 | ✅ Activos |
| Rutas LAN remotas aprendidas por EIGRP | ✅ Confirmadas |
| NHRP SPOKE1 resolviendo SPOKE2 directamente | ✅ NBMA 10.7.25.9 |
| NHRP SPOKE2 resolviendo SPOKE1 directamente | ✅ NBMA 10.7.25.5 |
| Ping / Trace VPC1 → VPC2 (spoke-to-spoke directo) | ✅ Exitoso |
| Ping / Trace VPC2 → VPC1 (spoke-to-spoke directo) | ✅ Exitoso |
| Ping VPC1 → VPCH | ✅ Exitoso |
| Ping VPC2 → VPCH | ✅ Exitoso |

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`SaelGerman_2025-0725_Script_DMVPN_Fase2_IKEV1_EIGRP.txt`](SaelGerman_2025-0725_Script_DMVPN_Fase2_IKEV1_EIGRP.txt) | Scripts de configuración Cisco IOS |
| [`SaelGerman_2025-0725_DMVPN-Fase2-IKEV1_P2.pdf`](SaelGerman_2025-0725_DMVPN-Fase2-IKEV1_P2.pdf) | Documentación técnica completa |

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología DMVPN Fase 2](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/01_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 2 — Interfaces del ISP](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/02_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 3 — Interfaces del HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/03_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 4 — Configuración Tunnel0 en HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/04_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 5 — Parámetros crypto IKEv1/IPSec en HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/05_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 6 — Configuración EIGRP AS 25 en HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/06_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 7 — Interfaces de SPOKE1](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/07_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 8 — Configuración Tunnel0 en SPOKE1](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/08_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 9 — Parámetros crypto IKEv1/IPSec en SPOKE1](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/09_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 10 — Interfaces de SPOKE2](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/10_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 11 — Configuración Tunnel0 en SPOKE2](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/11_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 12 — Parámetros crypto IKEv1/IPSec en SPOKE2](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/12_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 13 — Configuración EIGRP AS 25 en SPOKE2](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/13_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 14 — show dmvpn en HUB (2 peers registrados)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/14_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 15 — Tabla NHRP en HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/15_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 16 — IKEv1 SA en HUB (QM_IDLE)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/16_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 17 — IPSec SA en HUB (encaps/decaps)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/17_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 18 — Sesiones crypto activas en HUB (UP-ACTIVE)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/18_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 19 — (Evidencia adicional)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/19_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 20 — (Evidencia adicional)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/20_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 21 — (Evidencia adicional)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/21_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 22 — (Evidencia adicional)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/22_evidencia_extraida_fotos_docx.png)
- 📸 [Figura 23 — Ruta EIGRP en SPOKE1 hacia LAN de SPOKE2](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/ruta_spoke1_hacia_lan_spoke2.png)
- 📸 [Figura 24 — Ruta EIGRP en SPOKE2 hacia LAN de SPOKE1](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/ruta_spoke2_hacia_lan_spoke1.png)
- 📸 [Figura 25 — NHRP en SPOKE1 resolviendo SPOKE2 directamente](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/nhrp_spoke1_resuelve_spoke2_directo.png)
- 📸 [Figura 26 — NHRP en SPOKE2 resolviendo SPOKE1 directamente](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/nhrp_spoke2_resuelve_spoke1_directo.png)
- 📸 [Figura 27 — Trace VPC1 → VPC2 (spoke-to-spoke directo)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/trace_vpc1_hacia_vpc2_spoke_to_spoke.png)
- 📸 [Figura 28 — Trace VPC2 → VPC1 (spoke-to-spoke directo)](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/trace_vpc2_hacia_vpc1_spoke_to_spoke.png)
- 📸 [Figura 29 — Prueba VPC1 hacia VPCH](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/prueba_vpc1_hacia_vpch.png)
- 📸 [Figura 30 — Prueba VPC2 hacia VPCH](SaelGerman_2025-0725_Capturas_DMVPN_Fase2_IKEV1_EIGRP_GitHub/prueba_vpc2_hacia_vpch.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_DMVPN-Fase2-IKEV1_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/gFus08gh-IA)

---

## 📚 Referencias

1. Cisco Systems. *Configuring DMVPN Phase 2 with IPSec and EIGRP*. Documentación oficial Cisco IOS.
2. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
