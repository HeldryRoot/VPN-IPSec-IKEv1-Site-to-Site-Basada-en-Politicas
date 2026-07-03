# VPN IPSec IKEv1 — Site-to-Site Basada en Políticas

<img width="654" height="276" alt="image" src="https://github.com/user-attachments/assets/edfcc30b-66cc-4465-95dc-13ea7e5fda80" />

**LABORATORIO DE NETWORKING**

**VPN IPSec IKEv1 — Site-to-Site Basada en Políticas**

_Documentación Técnica Profesional_

**Heldry Terrero**

Matrícula: 2025-0719

Materia: Seguridad de Redes

Fecha: Junio 2026

|   |
|---|
|⚠ AVISO IMPORTANTE — Este proyecto fue desarrollado únicamente con fines educativos y de laboratorio controlado, en el marco de la asignatura Seguridad de Redes del ITLA. Los scripts y técnicas documentadas SOLO deben ejecutarse en simuladores autorizados (PNetLab, GNS3, EVE-NG). QUEDA ESTRICTAMENTE PROHIBIDO usar este material en redes públicas o de terceros sin autorización explícita. El uso indebido puede constituir un delito conforme a las leyes de ciberseguridad de la República Dominicana.|

  

  

# 1. Objetivo del Laboratorio

Este laboratorio implementa y verifica una VPN Site-to-Site (S2S) basada en políticas utilizando IPSec con IKE versión 1 (IKEv1). Una VPN Policy-Based define el tráfico a cifrar mediante Access Control Lists (ACLs): solo el tráfico que coincide con la política es encapsulado dentro del túnel IPSec, mientras que el resto se enruta normalmente. Este modelo es ampliamente utilizado en entornos corporativos para proteger comunicaciones entre sucursales sobre redes públicas.

•        Configurar y verificar la negociación IKEv1 en Fase 1 (ISAKMP SA) y Fase 2 (IPSec SA).

•        Implementar cifrado AES-256 con hash SHA-256 y grupo Diffie-Hellman 14 (2048 bits).

•        Definir políticas de tráfico mediante ACLs simétricas para seleccionar el tráfico a proteger.

•        Verificar el establecimiento del túnel con comandos show crypto isakmp sa y show crypto ipsec sa.

•        Comprobar conectividad cifrada extremo a extremo entre las redes LAN de ambos sitios.

# 2. Marco Teórico

## 2.1 IPSec — Internet Protocol Security (RFC 4301)

IPSec es un conjunto de protocolos que opera en la capa 3 del modelo OSI. Proporciona confidencialidad (AES-256), integridad (SHA-256) y autenticación del origen. En modo túnel encapsula el paquete IP original completo dentro de un nuevo paquete cuyas IPs corresponden a las interfaces WAN de los routers, haciendo invisible el tráfico LAN privado en la red pública.

## 2.2 IKEv1 — Fases de Negociación

|**Fase**|**Nombre**|**Mensajes**|**Estado Esperado**|**Función**|
|---|---|---|---|---|
|Fase 1|ISAKMP SA|6 (Main Mode)|QM_IDLE|Canal seguro de control entre peers|
|Fase 2|IPSec SA|3 (Quick Mode)|ACTIVE|SA de datos para cifrar el tráfico|

Fase 1 negocia: algoritmo de cifrado (AES-256), hash (SHA-256), autenticación (PSK), grupo DH (14/2048 bits) y lifetime (86400 s). Fase 2 negocia el Transform Set (ESP-AES256/SHA256) y las redes de interés definidas por las ACLs.

## 2.3 Componentes de una Policy-Based VPN

|**Componente**|**Comando**|**Función**|
|---|---|---|
|ISAKMP Policy|crypto isakmp policy 10|Parámetros Fase 1|
|Pre-Shared Key|crypto isakmp key KEY address IP|Autenticación entre peers|
|Transform Set|crypto ipsec transform-set|Algoritmos de cifrado Fase 2|
|ACL de Cripto|access-list 100 permit ip|Define el tráfico a cifrar|
|Crypto Map|crypto map MAP 10 ipsec-isakmp|Vincula peer + TS + ACL|
|Aplicar en WAN|crypto map MAP (en interfaz)|Activa el mapa en la WAN|

## 2.4 Diffie-Hellman Grupo 14

El Grupo 14 usa un módulo de 2048 bits, equivalente a ~112 bits de seguridad simétrica. Según NIST SP 800-57 es adecuado hasta el año 2030. Permite establecer claves compartidas sobre un canal inseguro sin transmitir la clave directamente. En IKEv1 se usa durante la Fase 1 para derivar el material de clave de la ISAKMP SA.

# 3. Topología de Red

La topología consta de dos routers Cisco peers (R1-PEER-A y R2-PEER-B) interconectados a través de un R-ISP que simula Internet. Cada peer tiene una red LAN /28 con un host para verificar la conectividad cifrada extremo a extremo.

| <img width="1195" height="152" alt="1-6" src="https://github.com/user-attachments/assets/46a2053a-0f5f-4cba-8100-8fb78d5817c5" />
 |

## 3.1 Tabla de Direccionamiento IP

|**Dispositivo**|**Interfaz**|**Dirección IP**|**Máscara**|**Descripción**|
|---|---|---|---|---|
|R1-PEER-A|Ethernet0/0|20.25.1.1|255.255.255.252|WAN → ISP|
|R1-PEER-A|Ethernet0/1|20.25.7.1|255.255.255.240|LAN Sitio A|
|R-ISP|Ethernet0/1|20.25.1.2|255.255.255.252|Enlace → R1|
|R-ISP|Ethernet0/0|20.25.2.1|255.255.255.252|Enlace → R2|
|R2-PEER-B|Ethernet0/0|20.25.2.2|255.255.255.252|WAN → ISP|
|R2-PEER-B|Ethernet0/1|20.25.9.1|255.255.255.240|LAN Sitio B|
|PC-A|eth0|20.25.7.2|255.255.255.240|Host LAN Sitio A|
|PC-B|eth0|20.25.9.2|255.255.255.240|Host LAN Sitio B|

## 3.2 Redes LAN y Subredes WAN

|**Sitio**|**Red**|**Prefijo**|**Rango Hosts**|**Gateway**|
|---|---|---|---|---|
|Sitio A|20.25.7.0/28|255.255.255.240|20.25.7.1–20.25.7.14|20.25.7.1|
|Sitio B|20.25.9.0/28|255.255.255.240|20.25.9.1–20.25.9.14|20.25.9.1|
|WAN R1–ISP|20.25.1.0/30|255.255.255.252|20.25.1.1–20.25.1.2|—|
|WAN ISP–R2|20.25.2.0/30|255.255.255.252|20.25.2.1–20.25.2.2|—|

# 4. Parámetros IPSec Configurados

## 4.1 Fase 1 — ISAKMP Policy 10

|**Parámetro**|**Valor**|**Descripción**|
|---|---|---|
|Cifrado|AES-256|Advanced Encryption Standard, clave 256 bits|
|Hash / Integridad|SHA-256|HMAC-SHA256, salida de 256 bits|
|Autenticación|Pre-Share (PSK)|Clave precompartida: cisco123 (solo lab)|
|Grupo DH|14 (2048 bits)|Intercambio de claves NIST-válido hasta 2030|
|Lifetime Fase 1|86400 segundos|24 horas — vida de la ISAKMP SA|
|Modo de negociación|Main Mode|6 mensajes, identidad protegida|

## 4.2 Fase 2 — Transform Set TS_VPN

|**Parámetro**|**Valor**|**Descripción**|
|---|---|---|
|Protocolo|ESP (protocolo 50)|Encapsulating Security Payload|
|Cifrado ESP|esp-aes 256|AES-256 para confidencialidad del payload|
|Integridad ESP|esp-sha256-hmac|HMAC-SHA256 para autenticación e integridad|
|Modo|Tunnel|Encapsula el paquete IP completo|
|Lifetime Fase 2|3600 s (default)|1 hora — vida de la IPSec SA|

## 4.3 ACLs de Criptografía — Tráfico Interesante

|**Router**|**ACL**|**Tráfico Protegido**|
|---|---|---|
|R1-PEER-A|permit ip 20.25.7.0 0.0.0.15  20.25.9.0 0.0.0.15|LAN-A → LAN-B|
|R2-PEER-B|permit ip 20.25.9.0 0.0.0.15  20.25.7.0 0.0.0.15|LAN-B → LAN-A|
|IMPORTANTE: Las ACLs deben ser simétricas (espejo) en ambos routers. La wildcard 0.0.0.15 corresponde a /28 (255.255.255.240). Si no coinciden exactamente, el túnel no se establecerá.|   |   |

  
  
  
  
  

# 5. Scripts de Configuración

## 5.1 R1-PEER-A

enable  
configure terminal  
hostname R1-PEER-A  
!  
interface ethernet0/0  
 description WAN_ISP  
 ip address 20.25.1.1 255.255.255.252  
 no shutdown  
!  
interface ethernet0/1  
 description LAN_A  
 ip address 20.25.7.1 255.255.255.240  
 no shutdown  
!  
ip route 0.0.0.0 0.0.0.0 20.25.1.2  
!  
! === FASE 1: ISAKMP Policy ===  
crypto isakmp policy 10  
 encryption aes 256  
 hash sha256  
 authentication pre-share  
 group 14  
 lifetime 86400  
!  
crypto isakmp key cisco123 address 20.25.2.2  
!  
! === FASE 2: Transform Set ===  
crypto ipsec transform-set TS_VPN esp-aes 256 esp-sha256-hmac  
 mode tunnel  
!  
! === ACL de trafico interesante ===  
access-list 100 permit ip 20.25.7.0 0.0.0.15 20.25.9.0 0.0.0.15  
!  
! === Crypto Map ===  
crypto map MAP_VPN 10 ipsec-isakmp  
 set peer 20.25.2.2  
 set transform-set TS_VPN  
 match address 100  
!  
interface ethernet0/0  
 crypto map MAP_VPN  
!  
do write memory

## 5.2 R-ISP

enable  
configure terminal  
hostname R-ISP  
!  
interface ethernet0/1  
 description Hacia_R1-PEER-A  
 ip address 20.25.1.2 255.255.255.252  
 no shutdown  
!  
interface ethernet0/0  
 description Hacia_R2-PEER-B  
 ip address 20.25.2.1 255.255.255.252  
 no shutdown  
!  
do write memory

## 5.3 R2-PEER-B

enable  
configure terminal  
hostname R2-PEER-B  
!  
interface ethernet0/0  
 description WAN_ISP  
 ip address 20.25.2.2 255.255.255.252  
 no shutdown  
!  
interface ethernet0/1  
 description LAN_B  
 ip address 20.25.9.1 255.255.255.240  
 no shutdown  
!  
ip route 0.0.0.0 0.0.0.0 20.25.2.1  
!  
crypto isakmp policy 10  
 encryption aes 256  
 hash sha256  
 authentication pre-share  
 group 14  
 lifetime 86400  
!  
crypto isakmp key cisco123 address 20.25.1.1  
!  
crypto ipsec transform-set TS_VPN esp-aes 256 esp-sha256-hmac  
 mode tunnel  
!  
access-list 100 permit ip 20.25.9.0 0.0.0.15 20.25.7.0 0.0.0.15  
!  
crypto map MAP_VPN 10 ipsec-isakmp  
 set peer 20.25.1.1  
 set transform-set TS_VPN  
 match address 100  
!  
interface ethernet0/0  
 crypto map MAP_VPN  
!  
do write memory

## 5.4 Hosts VPCS

PC-A> ip 20.25.7.2/28 20.25.7.1  
PC-B> ip 20.25.9.2/28 20.25.9.1

# 6. Flujo de Negociación IKEv1

|**Paso**|**Fase**|**Acción**|**Estado**|
|---|---|---|---|
|1|Fase 1|R1 inicia Main Mode hacia R2 (20.25.2.2)|MM_NO_STATE|
|2|Fase 1|Intercambio proposals ISAKMP (AES256/SHA256/DH14)|MM_SA_SETUP|
|3|Fase 1|Intercambio DH — generación material de clave|MM_KEY_EXCH|
|4|Fase 1|Autenticación mutua con PSK (cisco123)|MM_KEY_AUTH|
|5|Fase 1|ISAKMP SA establecida — canal seguro activo|QM_IDLE|
|6|Fase 2|Negociación Quick Mode — IPSec SA (TS_VPN)|QM_IDLE|
|7|Fase 2|Túnel activo — tráfico cifrado con ESP-AES256|ACTIVE|

# 7. Demostración y Verificación

## 7.1 Ping de PC-A a PC-B — Disparar el túnel

PC-A> ping 20.25.9.2

| <img width="675" height="190" alt="Ping" src="https://github.com/user-attachments/assets/481adb83-7c43-485c-8459-9b6312254aa3" />
 |



## 7.2 Verificar Fase 1 — show crypto isakmp sa

El estado QM_IDLE confirma que la Fase 1 fue negociada exitosamente:

R1-PEER-A# show crypto isakmp sa  
   
! Resultado:  
IPv4 Crypto ISAKMP SA  
dst             src             state    conn-id status  
20.25.2.2       20.25.1.1       QM_IDLE     1001 ACTIVE

| <img width="842" height="194" alt="isakmp" src="https://github.com/user-attachments/assets/5a869878-c92e-435e-a813-92efd9026497" />
 |


  
  
  

## 7.3 Verificar Fase 2 — show crypto ipsec sa

Los contadores #pkts encaps y #pkts decaps deben incrementar con cada ping:

R1-PEER-A# show crypto ipsec sa  
   
! Fragmento clave:  
  local  ident: (20.25.7.0/255.255.255.240/0/0)  
  remote ident: (20.25.9.0/255.255.255.240/0/0)  
  current_peer: 20.25.2.2 port 500  
   
  #pkts encaps: 5,  #pkts encrypt: 5,  #pkts digest: 5  
  #pkts decaps: 5,  #pkts decrypt: 5,  #pkts verify: 5  
  #send errors 0,   #recv errors 0

| <img width="629" height="51" alt="encrypt" src="https://github.com/user-attachments/assets/b6f30759-fdd5-42fb-9a9d-7450326d9f11" />
 |


# 8. Troubleshooting

|**Síntoma**|**Causa Probable**|**Solución**|
|---|---|---|
|Túnel no sube|ACLs no simétricas|Verificar que ACL R1 sea espejo de ACL R2|
|MM_NO_STATE|PSK diferente entre peers|Verificar crypto isakmp key cisco123 address X|
|Fase 2 falla|Transform sets no coinciden|Verificar TS_VPN en ambos routers|
|Ping falla / SA activa|Falta ruta por defecto|Verificar ip route 0.0.0.0 0.0.0.0 next-hop|
|#pkts encaps=0|Crypto map no aplicado|Verificar crypto map MAP_VPN en Ethernet0/0|

! Debug Fase 1  
R1-PEER-A# debug crypto isakmp  
   
! Debug Fase 2  
R1-PEER-A# debug crypto ipsec  
   
! Apagar todos los debugs  
R1-PEER-A# undebug all

# 9. Buenas Prácticas y Marco Normativo

•        PSK robusta: 'cisco123' es solo para laboratorio. En producción usar mínimo 32 caracteres alfanuméricos.

•        Perfect Forward Secrecy (PFS): Agregar 'set pfs group14' en el crypto map para mayor seguridad.

•        Dead Peer Detection: Agregar 'crypto isakmp keepalive 30 5' para detectar peers caídos.

•        Deshabilitar Aggressive Mode: 'crypto isakmp aggressive-mode disable' por ser menos seguro.

•        Considerar migrar a IKEv2 para nuevas implementaciones según NIST SP 800-77r1.

|**Marco / Estándar**|**Relevancia**|
|---|---|
|NIST SP 800-77|Guía para implementaciones IPSec — valida AES-256 + SHA-256|
|NIST SP 800-57|Gestión de claves — valida DH Grupo 14 hasta 2030|
|RFC 4301|Arquitectura IPSec — define modo túnel y SAs|
|ISO/IEC 27033-5|Seguridad en comunicaciones — VPN cifradas|
|Ley 172-13 RD|Protección de datos personales — justifica uso de VPN|

# 10. Conclusión

La VPN IPSec IKEv1 Site-to-Site Policy-Based fue configurada y verificada exitosamente. El estado QM_IDLE en la ISAKMP SA confirma la Fase 1 exitosa, mientras que los contadores encaps/decaps > 0 en show crypto ipsec sa confirman que el tráfico entre las LANs está siendo cifrado con AES-256 y autenticado con SHA-256. La naturaleza Policy-Based garantiza que solo el tráfico entre 20.25.7.0/28 y 20.25.9.0/28 sea encapsulado en el túnel, siendo transparente para el resto del tráfico.

|   |
|---|
|RESULTADO: VPN IPSec IKEv1 Policy-Based activa. QM_IDLE confirmado. Conectividad cifrada PC-A (20.25.7.2) → PC-B (20.25.9.2) verificada mediante ESP-AES256/SHA256.|

Heldry Terrero — Matrícula: 2025-0719 — Seguridad de Redes — Junio 2026

Link del Video: [https://youtu.be/tYuHLEJM1w0](https://youtu.be/tYuHLEJM1w0)  
  

Link del GitHub: https://github.com/HeldryRoot/VPN-IPSec-IKEv1-Site-to-Site-Basada-en-Politicas
