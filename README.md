# Topologia_DMVPN_Fase2_IKEv1
https://www.youtube.com/watch?v=i1W4PuQmz0s

Instituto Tecnologico
Documentacion Tecnica - Configuracion de VPN
VPN Hub and Spoke DMVPN Fase 2
IKEv1 con Enrutamiento Dinamico EIGRP
Estudiante: Darwing Manuel Pena Reyes
Matricula: 2024-2690
Asignatura: Seguridad de Redes
Fecha: Julio 2026
 1. Objetivo de la VPN
Implementar una red DMVPN (Dynamic Multipoint VPN) en topologia hub-and-spoke con un Hub y dos Spokes, permitiendo que los spokes establezcan tuneles directos entre si (spoke-to-spoke) sin necesidad de pasar todo el trafico por el Hub, usando NHRP para el descubrimiento dinamico de peers e IKEv1/IPSec para la proteccion criptografica del trafico sobre una interfaz mGRE (multipoint GRE), con enrutamiento dinamico EIGRP sobre el tunel.
2. Topologia de red
Un router Hub (R1) y dos routers Spoke (R2 y R3), cada uno con su propia LAN, conectados a un router ISP comun que solo conoce las redes publicas /30 de cada uno. Los tres routers comparten una unica interfaz Tunnel0 configurada en modo mGRE sobre la red logica 10.40.40.0/24, protegida por un IPSec profile dinamico (crypto map generado automaticamente por 'tunnel protection'), con NHRP para resolver las IPs publicas reales de cada peer y EIGRP para el intercambio de rutas.
2.1 Interfaces utilizadas
Dispositivo / Interfaz	Rol
R1 (Hub) - F0/0	WAN, tunnel source hacia ISP
R1 (Hub) - G2/0	LAN Hub
R1/R2/R3 - Tunnel0	Interfaz mGRE compartida (DMVPN)
R2 (Spoke1) - F0/0	WAN, tunnel source hacia ISP
R2 (Spoke1) - G2/0	LAN Spoke1
R3 (Spoke2) - F0/0	WAN, tunnel source hacia ISP
R3 (Spoke2) - G2/0	LAN Spoke2

2.2 Direccionamiento IP
Segmento	Red / IP
LAN Hub (R1)	20.24.40.0/24 - Gateway .90 - PC-Hub .91
LAN Spoke1 (R2)	20.24.41.0/24 - Gateway .90 - PC1 .91
LAN Spoke2 (R3)	20.24.42.0/24 - Gateway .90 - PC2 .91
WAN ISP-R1	172.16.40.0/30
WAN ISP-R2	172.16.41.0/30
WAN ISP-R3	172.16.42.0/30
Tunnel0 (mGRE)	10.40.40.0/24 (Hub .1 / Spoke1 .2 / Spoke2 .3)

[ INSERTAR CAPTURA DE PANTALLA AQUI: Topologia completa en GNS3 (con nombre y matricula visibles) ]
Explicacion: Captura general de la topologia armada en GNS3, mostrando todos los dispositivos, sus interfaces conectadas, y el recuadro con nombre y matricula del estudiante.
3. Parametros utilizados
Parametro	Valor configurado
Version IKE	IKEv1 (ISAKMP)
Metodo de autenticacion	Pre-shared key comodin (address 0.0.0.0 0.0.0.0)
Cifrado Fase 1	AES 256 / SHA256 / Grupo DH 14
Transform-set (Fase 2)	esp-aes 256 + esp-sha256-hmac, modo transport
NHRP network-id	4040
NHRP authentication	DMVPN123
Protocolo de enrutamiento	EIGRP AS 100
Fase DMVPN	Fase 2 (next-hop NO se reescribe hacia el Hub)

4. Configuracion y explicacion (capturas de pantalla)
Llave ISAKMP comodin (necesaria para multiples peers)
crypto isakmp key CiscoVPN123 address 0.0.0.0 0.0.0.0
[ INSERTAR CAPTURA DE PANTALLA AQUI: Llave ISAKMP comodin (necesaria para multiples peers) ]
Explicacion: A diferencia de las VPN punto a punto, el Hub debe aceptar conexiones IKE de cualquier IP de origen (los spokes), por lo que la llave se asocia a un comodin en vez de una IP fija. Esta misma configuracion se replica en los 3 routers.
Interfaz Tunnel0 en modo mGRE (Hub)
interface Tunnel0
 ip address 10.40.40.1 255.255.255.0
 ip nhrp authentication DMVPN123
 ip nhrp map multicast dynamic
 ip nhrp network-id 4040
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF


Explicacion: El Hub actua como servidor NHRP (NHS), aprendiendo dinamicamente las IPs publicas de los spokes cuando estos se registran. 'tunnel mode gre multipoint' es lo que distingue a DMVPN de un GRE punto a punto tradicional.
Configuracion NHRP en un Spoke
interface Tunnel0
 ip nhrp map 10.40.40.1 172.16.40.2
 ip nhrp map multicast 172.16.40.2
 ip nhrp nhs 10.40.40.1
 tunnel mode gre multipoint
<img width="490" height="471" alt="image" src="https://github.com/user-attachments/assets/ccbc69a8-743f-432e-885b-2aaf3ec07b57" />


Explicacion: Los spokes SI necesitan mapear estaticamente la IP del tunel y la IP publica del Hub, ya que es el unico punto fijo que conocen de antemano; a partir de ahi, el Hub les ayuda a descubrir dinamicamente a los demas spokes.
5. Evidencia de funcionamiento
Registro NHRP en el Hub
R1# show ip nhrp
10.40.40.2/32 via 10.40.40.2, Tunnel0 created, dynamic
10.40.40.3/32 via 10.40.40.3, Tunnel0 created, dynamic
Confirma que ambos spokes se registraron correctamente contra el Hub, requisito previo para que EIGRP pueda formar vecindad sobre el tunel.
Vecinos EIGRP formados sobre Tunnel0
R2# show ip eigrp neighbors
H   Address         Interface
0   10.40.40.1      Tu0
El Spoke1 forma vecindad EIGRP directamente con el Hub a traves de la interfaz de tunel, lo que permite el intercambio dinamico de rutas de las LANs remotas.
Traceroute Spoke-a-Spoke
PC1> trace 20.24.42.91
(evidencia de si el trafico pasa por el Hub o forma un tunel directo spoke-to-spoke)
En Fase 2, el primer paquete puede pasar por el Hub mientras NHRP resuelve la ruta directa; posteriormente puede establecerse un tunel spoke-to-spoke, verificable con entradas dinamicas adicionales en 'show ip nhrp' de ambos spokes.
6. Conclusion
La red DMVPN Fase 2 con IKEv1 y enrutamiento dinamico EIGRP quedo configurada sobre una interfaz mGRE compartida, con NHRP resolviendo dinamicamente los peers y IPSec protegiendo el trafico. Esta arquitectura permite escalar una VPN hub-and-spoke a multiples sitios sin necesidad de configurar un tunel dedicado por cada par de routers.
