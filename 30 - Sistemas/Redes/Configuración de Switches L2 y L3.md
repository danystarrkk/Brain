---
tags:
  - sistemas
  - redes
  - switches
tipo: concepto
estado: terminado
---

# Configuración de Switches L2 y L3

Esta nota cubre la configuración de switches Cisco — desde un switch L2 puro (S1, S2, S3 del laboratorio) hasta un **Multilayer Switch (MLS)** que enruta entre VLANs — con base en los scripts reales de la clase. Cada sección explica **primero la teoría completa** —por qué existe la tecnología, cuándo se usa y cómo funciona por dentro— y recién al final aparecen los comandos como ejemplos aplicados.

> [!info] Requisito previo
> Ver [[Conceptos Fundamentales de Redes]] (dominios de broadcast, OSI) y [[Tipos de IP y Dirección MAC]]. La punta router del lab (ROAS + OSPF) está en [[Configuración de Routers]].

---

## Switch de capa 2: teoría base

### ¿Qué hace un switch y cómo decide?

Un switch L2 opera en la capa de enlace de datos y su trabajo es **conmutar frames** usando direcciones MAC. Antes de configurar nada, hay que entender su mecánica interna:

1. **Aprendizaje de MAC**: cuando un frame llega por un puerto, el switch anota la **source MAC** + puerto de entrada en su **tabla CAM/MAC** (Content-Addressable Memory).
2. **Búsqueda**: busca la **destination MAC** en la tabla. Si la encuentra → **forwarding** directo por ese puerto. Si no → **flooding** (envía el frame por todos los puertos excepto el de entrada).
3. **Envejecimiento**: las entradas de la tabla expiran (por defecto 300 segundos en IOS) si la MAC no vuelve a verse.

### Almacenamiento y retardo: cut-through vs store-and-forward

| Modo | Cómo procesa | Retardo | Detección de errores |
|------|--------------|---------|----------------------|
| **Store-and-forward** | Recibe el frame **completo**, verifica el FCS/CRC, y solo entonces reenvía | Mayor (depende del tamaño del frame) | **Sí** — descarta frames corruptos |
| **Cut-through** | Lee la destination MAC (los primeros 6 bytes) y reenvía de inmediato | Mínimo y constante | **No** — propaga frames corruptos |
| **Fragment-free** (híbrido) | Espera 64 bytes (donde están los errores de colisión) y reenvía | Intermedio | Parcial |

Los switches Cisco modernos trabajan en **store-and-forward** por defecto (verifican CRC completo). Cut-through se usa en trading/low-latency donde milisegundos valen más que integridad (el switch moderno puede hacer cut-through con checksum del final — "cut-through con tail checksum").

### Dominios de colisión y de broadcast

- **Dominio de colisión**: en switches, **cada puerto es su propio dominio de colisión** (full-duplex por defecto). Esto es lo que mató los hubs, donde todos compartían un dominio.
- **Dominio de broadcast**: **un switch L2 es un solo dominio de broadcast por defecto** — todos los puertos ven los broadcasts (ARP requests, DHCP DISCOVER). Con 500 hosts eso es un problema de rendimiento y seguridad. **La solución son las VLANs.**

---

## VLANs: teoría completa

### ¿Qué es una VLAN?

Una **VLAN (Virtual LAN, IEEE 802.1Q)** es una **partición lógica de un switch o de varios switches** en dominios de broadcast independientes, **sin importar la topología física**. Dos puertos en el mismo switch físico pueden estar en VLANs distintas y no se ven jamás; dos puertos en switches distintos pueden estar en la misma VLAN y comportarse como si fueran el mismo switch.

**Problema que resuelve**: sin VLANs, segmentar la red = un switch físico por departamento (y cada uno con su router). Con VLANs, un solo switch físico aloja GERENCIA (VLAN 10), TI (VLAN 20), etc., y el tráfico entre ellas **obligatoriamente pasa por un router (L3)**.

### Tipos de puertos

| Tipo | Función | Tags 802.1Q |
|------|---------|-------------|
| **Access (acceso)** | Conecta **hosts finales** (PC, teléfono, impresora). Pertenece a **una sola VLAN** | **Nunca etiqueta** — el host recibe frames crudos |
| **Trunk (troncal)** | Conecta **switch↔switch** o **switch↔router**. Trasporta **múltiples VLANs** | **Sí etiqueta** cada frame con su VID |

El host no necesita saber de VLANs: para él es una red normal. El tagging es un diálogo exclusivo entre dispositivos de infraestructura.

### IEEE 802.1Q: el tag

802.1Q inserta un **tag de 4 bytes** en el header del frame Ethernet (entre source MAC y payload):

```
┌──────────────┬─────┬─────┬──────┬──────────────────────────┐
│   TPID       │ PCP │ DEI │ VID  │   Frame original...      │
│  16 bits     │ 3b  │ 1b  │ 12b  │                          │
│  0x8100      │prio │drop │VLAN  │                          │
└──────────────┴─────┴─────┴──────┴──────────────────────────┘
```

- **TPID** `0x8100`: firma que dice "soy 802.1Q" (sin él, el EtherType original `0x0800` IPv4 ocuparía ese lugar).
- **PCP** (3 bits): prioridad 802.1p para QoS (voz = 6, etc.).
- **DEI** (1 bit): indicador de descarte en congestión.
- **VID** (12 bits): número de VLAN → **4096 posibles, VLAN 0 y 4095 reservadas → 4094 utilizables**.

### Rangos de VLAN

| Rango | IDs | Estado |
|-------|-----|--------|
| Reservadas (legacy Token Ring) | 1002–1005 | **No se pueden usar ni borrar** |
| Normal | 1–1001 | Uso estándar |
| Extended | 1006–4094 | Disponibles (algunos modelos con limitaciones en VTP transparente) |

**VLAN 1** es la default: todos los puertos nacen en ella, pero **nunca se usa como VLAN de producción** porque todos los switches la hablan por defecto (es el vector de ataques como VLAN hopping y DTP). Se la usa solo para gestión si nada más.

### Naming: buenas prácticas

Nombrar las VLANs no es cosmético — en `show vlan brief` la diferencia entre `VLAN0010` y `GERENCIA` es la diferencia entre entender y no entender la red:

```ios
vlan 10
 name GERENCIA
vlan 20
 name TI
```

---

## Trunks en profundidad: teoría completa

### ¿Por qué existe un trunk?

Si cada enlace entre dos switches solo pudiera transportar una VLAN, conectar 10 VLANs entre dos switches exigiría 10 cables. El **trunk** resuelve esto: **un solo enlace físico multiplexa lógicamente todas las VLANs**, distinguiéndolas con el tag 802.1Q.

### Flujo completo de un frame a través de un trunk

1. PC en VLAN 10 (puerto access Fa0/10) envía un frame crudo (sin tag).
2. El switch recibe en Fa0/10, ve por su tabla MAC que esa VLAN es la 10, y al reenviar por el trunk **inserta el tag con VID 10**.
3. El frame cruza el trunk con tag visible.
4. El switch vecino recibe el trunk, **lee el VID (10)**, y reenvía el frame **por los puertos access de la VLAN 10** (quitando el tag — el host no lo entiende).

### La VLAN nativa

La **VLAN nativa** es la VLAN cuyo tráfico cruza el trunk **sin tag**. Nació por compatibilidad: dispositivos antiguos que no implementan 802.1Q reciben el tráfico de una VLAN "pelado". Reglas y riesgos:

- **Default: VLAN 1** en todos los switches.
- **Ambos extremos del trunk deben coincidir**. Si no → syslog `%CDP-4-NATIVE_VLAN_MISMATCH`: el switch A envía sin tag la VLAN 10 (su nativa), el switch B la recibe y la atribuye a **su** nativa (VLAN 1) → tráfico cruzado entre VLANs sin pasar por router = fuga de tráfico.
> [!danger] VLAN hopping por native VLAN
> Un atacante que inyecta frames **sin tag** en un trunk los hace pasar por la VLAN nativa. Y con **DTP** (ver abajo) puede negociar un trunk y etiquetar a voluntad. Defensas: (1) native VLAN inexistente en ambos extremos, (2) `switchport nonegotiate` + DTP off, (3) prunning de VLANs que no se usan.

- **Buena práctica actual**: native VLAN = una VLAN **inexistente** (el lab usa la 555/777), y esa misma VLAN se configura en el otro extremo — ya sea el trunk del switch vecino o la subinterfaz del router (`encapsulation dot1q 555 native`).

### Allowed VLANs: el prunning

Por defecto un trunk **transporta todas las VLANs** (1–1005). Si entre dos switches solo pasan la 10 y la 20, el resto es ancho de banda y flooding desperdiciado (broadcast de VLANs ajenas viaja por el trunk). `switchport trunk allowed vlan` **restringe la lista**:

```ios
switchport trunk allowed vlan 10,20        ! Solo VLANs 10 y 20
switchport trunk allowed vlan 10,20,30     ! Agregar con lista explícita
switchport trunk allowed vlan remove 20    ! Quitar una
switchport trunk allowed vlan all          ! Todas (default)
```

> [!warning] Gotcha de `allowed vlan`
> Agregar VLANs **sobreescribe** la lista previa: `switchport trunk allowed vlan 30` **borra** las 10 y 20 y deja solo la 30. Para agregar sin perder, usá `add`: `switchport trunk allowed vlan add 30`.

### DTP: Dynamic Trunking Protocol

**DTP** es un protocolo **proprietario Cisco** que negocia automáticamente si un enlace se vuelve trunk:

| Comando (modo) | Comportamiento |
|----------------|----------------|
| `switchport mode dynamic desirable` | **Inicia** negociación trunk (PAgP-like, proactivo) |
| `switchport mode dynamic auto` | **Espera** a que el otro negocie (pasivo — dos `auto` = NUNCA trunk) |
| `switchport mode trunk` | Fuerza trunk, sigue mandando DTP |
| `switchport mode access` | Fuerza acceso, sigue mandando DTP |
| `switchport nonegotiate` | **Corta DTP** — debe ir con trunk/access forzado |

> [!danger] Regla de seguridad
> En puertos hacia **hosts**: `switchport mode access` + `switchport nonegotiate`. En puertos hacia switches vecinos de confianza: `switchport mode trunk` + `switchport nonegotiate`. **Nunca dejar `auto`/`desirable` en producción** — un atacante con una laptop puede negociar un trunk y etiquetar frames para saltarse segmentos (VLAN hopping).

---

## Puertos de acceso: teoría

Un puerto access es la conexión al **mundo host**. Reglas:

- Pertenece a **exactamente una VLAN** (`switchport access vlan X`).
- **No etiqueta jamás** — el tráfico sale crudo.
- `switchport mode access` fija el modo (sin esto, si el vecino habla DTP podría intentar convertirlo en trunk).
- Si el `switchport access vlan` apunta a una VLAN **inexistente**, IOS la crea automáticamente como VLAN **sin nombre** (vlan 99 → `VLAN0099`) — sorpresa frecuente al auditar.

Aplicado por rangos en el lab (S1: GERENCIA en Fa0/10-19, TI en Fa0/20-24):

```ios
int range fa0/10 - 19
  switchport mode access
  switchport access vlan 10
int range fa0/20 - 24
  switchport mode access
  switchport access vlan 20
```

`int range` permite configurar varios puertos de una vez separando con coma (`int range fa0/1, fa0/4`) o rangos con guion (`fa0/10 - 19`). Sin `range`, cada interface requiere su propio `interface` — con 24 puertos es insoportable.

---

## EtherChannel: teoría completa

### ¿Qué es y por qué existe?

**EtherChannel** (también llamado Port Trunking o Link Aggregation) **agrupa 2–8 enlaces físicos paralelos en un solo enlace lógico (Port-channel)**. Motivos:

1. **Ancho de banda acumulado**: 4 × 1 Gbps = 4 Gbps lógicos.
2. **Redundancia**: si un link físico muere, el bundle sigue vivo con el resto.
3. **Sin loops STP**: sin EtherChannel, STP **bloquearía** los enlaces extra (solo dejaría 1 activo = desperdicio); con EtherChannel, STP ve **un solo enlace** y no bloquea nada.

> [!info] Balanceo, no suma
> El tráfico de un host **nunca se reparte por todos los links** (eso rompería el orden de llegada de los paquetes). Un flujo (par src-dst) se asigna a **un solo link** según un algoritmo hash (src-mac, dst-ip, src-dst-ip...). La "suma" se logre distribuyendo **flujos distintos** por links distintos. Un solo flujo gigante NUNCA supera 1 link.

### Protocolos de negociación: LACP vs PAgP vs on

| Protocolo | Estándar | Modos | Soporte |
|-----------|----------|-------|---------|
| **LACP** (Link Aggregation Control Protocol) | **IEEE 802.3ad / 802.1AX** | `active` (inicia) / `passive` (espera) | **Multi-vendor** — el estándar |
| **PAgP** (Port Aggregation Protocol) | Cisco propietario | `desirable` (inicia) / `auto` (espera) | Solo Cisco |
| **on** | Estático (sin negociación) | — | Todos, pero cero detección de errores |

Combinaciones válidas:

```
LACP :  active + active  ✅   active + passive  ✅   passive + passive  ❌ (nadie inicia)
PAgP :  desirable + desirable ✅  desirable + auto ✅  auto + auto ❌
on   :  on + on ✅ (pero ver warning)
```

> [!warning] `mode on` y el peligro
> `on` no negocia nada: fuerza el bundle. Si el otro extremo no lo tiene configurado idéntico, **no hay error visible — hay loops o tráfico perdido**. Además, si el switch vecino es de otro fabricante, PAgP no funciona (usá LACP).

### Reglas para que el bundle levante

Todos los puertos de un Port-channel deben ser **idénticos** en:

- Speed y duplex
- VLAN access/trunk y allowed VLANs
- Native VLAN
- MTU y QoS

Si alguno difiere, ese puerto queda en **suspended/hot-standby** y no aporta al bundle (ver `show etherchannel summary` con el flag `s`).

### Configuración — ejemplo del lab

**S1 ↔ S2 con LACP** (puertos f0/1-3, bundle Po2):

```ios
! S1 — lado activo
int range f0/1 - 3
  channel-group 2 mode active
! S2 — lado pasivo
int range f0/1 - 3
  channel-group 2 mode passive
```

**S3 ↔ MLS con PAgP** (puertos f0/1 y f0/4, bundle Po1):

```ios
int range f0/1, f0/4
  channel-group 1 mode desirable
```

`channel-group N mode X` crea automáticamente la interfaz lógica `Port-channel N`, a la que después se le aplica la configuración de trunk (ver sección MLS).

---

## Switch de Capa 3 (MLS): teoría completa

### Del L2 al L3: por qué un MLS

Un switch L2 **no enruta**: las VLANs son islas. Para que VLAN 10 hable con VLAN 20 hacen falta un router (ROAS — ver [[Configuración de Routers]]) o un **Multilayer Switch**. El MLS es **un switch con ASICs de conmutación que además hacen switching de rutas** — la misma caja hace L2 y L3 a velocidad de hardware. Ventajas sobre ROAS:

| | ROAS (router + switch L2) | MLS |
|--|---------------------------|-----|
| Velocidad entre VLANs | Limitada por el enlace trunk + CPU del router | **Wire-speed** (switching de rutas en hardware) |
| Puntos de falla | Router es SPOF del tráfico inter-VLAN | Caja única (o VSS/stack) |
| Ancho de banda | Trunk compartido | Backplane interno del switch |

### `ip routing`: el interruptor

Un MLS **arranca como L2** — hasta que ejecutás `ip routing`, que habilita el plano L3 (tabla de rutas, protocolos, reenvío entre VLANs):

```ios
config t
ip routing
```

Sin esto, las SVIs tienen IP pero **no enrutan nada** — el síntoma clásico es "la SVI hace ping pero las VLANs no se hablan".

### SVI (Switch Virtual Interface): el gateway dentro del switch

Una **SVI** es una interfaz **virtual L3** asociada a una VLAN. Funciona como el **gateway de esa VLAN** — el equivalente MLS de la subinterfaz ROAS:

- Tiene IP/máscara — es el default gateway de los hosts de esa VLAN.
- Para que esté **up**, la VLAN debe existir y tener **al menos un puerto access up** en ella (o estar up vía trunk con un neighbor activo).
- Su tráfico hacia otras VLANs sale por el plano L3 del MLS (una vez con `ip routing`).

```ios
int vlan 30
  ip address 192.168.30.1 255.255.255.240
  no shutdown
int vlan 40
  ip address 192.168.40.1 255.255.255.248
  no shutdown
```

Cada SVI es el gateway de su VLAN; los hosts de VLAN 30 apuntan su default gateway a `192.168.30.1`.

### Puerto enrutado (routed port): interfaz física L3

Un **routed port** es un puerto físico del MLS al que se le **quita la función switch** para comportarse como una interfaz de router:

```ios
int g0/2
  no switchport              ! Quota de switch → ahora es L3 puro
  ip address 10.10.10.2 255.255.255.252
  no shutdown
```

| | Puerto access | SVI | Routed port |
|--|---------------|-----|-------------|
| Opera en | L2 (una VLAN) | L3 (gateway de una VLAN) | L3 (interfaz física) |
| Se configura con | `switchport access vlan` | `int vlan X` + IP | `no switchport` + IP |
| Caso de uso | Hosts | Gateway interno de VLAN | Enlace dedicado a router/MLS vecino (punto a punto) |

En el lab, `g0/2` es el routed port que enfrenta al `g0/0/0` de R1 por el enlace `10.10.10.0/30` — el MLS enruta OSPF por ahí hacia el router.

### Inter-VLAN routing en el MLS: el flujo completo

1. Host de VLAN 30 (`192.168.30.50`, gw `192.168.30.1`) quiere hablar a VLAN 40 (`192.168.40.10`).
2. ARP → llega a la SVI `Vlan30`.
3. El MLS (con `ip routing`) ve en su tabla que `192.168.40.0/29` está conectada vía SVI `Vlan40`.
4. Reenvía **en hardware** — sin salir del chasis, sin trunk, sin router externo.

---

## Configuración completa del MLS del lab

Juntando las piezas (el script "MLS" del laboratorio):

```ios
enable
config t
hostname MLS

! Plano L3 activado — sin esto nada enruta
ip routing

! VLANs
vlan 30
 name DC1
vlan 40
 name DC2

! EtherChannel con S3 (PAgP)
int range f0/1, f0/4
  channel-group 1 mode desirable

! El Port-channel como trunk (misma nativa que S3: 777)
int Po1
  switchport mode trunk
  switchport native vlan 777
  switchport trunk allowed vlan 30,40

! SVIs = gateways de cada VLAN
int vlan 30
  ip address 192.168.30.1 255.255.255.240
  no shutdown
int vlan 40
  ip address 192.168.40.1 255.255.255.248
  no shutdown

! Routed port hacia R1
int g0/2
  no switchport
  ip address 10.10.10.2 255.255.255.252
  no shutdown

! OSPF hacia R1 (teoría completa en la nota de routers)
router ospf 2
  router-id 2.2.2.2
  network 192.168.30.0 0.0.0.15 area 0
  network 192.168.40.0 0.0.0.7 area 0
  network 10.10.10.0 0.0.0.3 area 0

do write memory
```

---

## Hardening básico del switch (del script S2)

El script `switch2` muestra las medidas de higiene mínima de un switch de acceso:

```ios
hostname S2
banner motd $
-----------------------------------------------------
                        Switch
              No Ingresar sin autorización
-----------------------------------------------------$
enable secret stark1234

! Puertos NO usados: shutdown + VLAN de reserva (no la 1)
int range fa0/1 - 10, fa0/16 - 24
  shutdown
  switchport mode access
  switchport access vlan 150
```

Por qué cada pieza:

- **`banner motd`**: obligatorio por ley en muchas jurisdicciones (aviso de acceso no autorizado). Se muestra antes del login.
- **`enable secret`**: hash MD5 de la contraseña de privilegio — sin ella, cualquiera con acceso físico escala a config mode.
- **Puertos shutdown**: puertos sin uso = puertos de ataque físico. Apagados + en una VLAN sin gateway → doble contención (no negocian, no enrutan).

> [!warning] Nota sobre el script
> El script original usaba `switchport mode acces` (falta la "s") y asignaba la VLAN 150 a puertos shutdown — si la VLAN 150 **no existe** en la base de datos, IOS la crea automáticamente sin nombre. Creala explícita con `vlan 150 / name RESERVA`.

---

## Verificación: comandos `show` del switch

```ios
show vlan brief                    ! VLANs, puertos por VLAN, estado
show interfaces trunk               ! Trunks activos: VLANs allowed, native, estado
show interfaces status              ! Tabla resumen: puerto, VLAN, duplex, speed, estado
show interfaces switchport          ! Detalle L2 de una interfaz (modo, acceso, trunking)
show mac address-table              ! Tabla CAM — aprendizaje de MACs
show etherchannel summary           ! Bundles: puertos, protocolo (P=PAgP, L=LACP), flags
show ip interface brief             ! (MLS) Interfaces L3 y SVIs con estado
show ip route                       ! (MLS) Tabla de rutas
show running-config                 ! Configuración activa
```

Lectura rápida de `show etherchannel summary`:

| Flag | Significado |
|------|-------------|
| `P` | en uso (in-bundle) |
| `s` | suspended (configuración distinta al resto) |
| `D` | down |
| `(P)` / `(L)` / `(S)` | PAgP / LACP / static (on) |

---

## Errores comunes y correcciones (detectados en los scripts del lab)

| # | Error en el script | Problema | Corrección |
|---|--------------------|----------|------------|
| 1 | `switchport mode acces` | Typo — IOS rechaza el comando | `switchport mode access` |
| 2 | `ip address 192.168.168.20.1 ...` en SVI Vlan30 | Typo — la red OSPF del lab es `192.168.30.0/28` | `ip address 192.168.30.1 255.255.255.240` |
| 3 | `ip rounting` | Typo | `ip routing` |
| 4 | SVI configurada sin verificar puerto up en la VLAN | SVI en down si la VLAN no tiene puertos activos | `show ip interface brief` / `show vlan brief` |
| 5 | EtherChannel sin `shutdown` previo de los miembros | Práctica de lab: apagar mientras se cablea para evitar loops | `shutdown` en miembros → configurar → `no shutdown` |
| 6 | Nativa de trunk sin alinear entre extremos | `%CDP-4-NATIVE_VLAN_MISMATCH` | Misma native en ambos lados (555/777 en el lab) |
| 7 | `channel-group` mixto active/desirable en el mismo bundle | LACP y PAgP no conviven — el bundle no levanta | Mismo protocolo en ambos extremos |

---

## CheatSheet — Configuración de Switches L2 y L3

```ios
! ═════════ IOS básico ═════════
enable
config t
hostname <nombre>
banner motd #<mensaje>#
enable secret <contraseña>
do write memory

! ═════════ VLANs ═════════
vlan <id>
 name <nombre>
exit

! ═════════ Puerto de acceso (hosts) ═════════
int range <ini> - <fin>
 switchport mode access
 switchport access vlan <id>

! ═════════ Trunk (switch↔switch / switch↔router) ═════════
int <interfaz>
 switchport mode trunk
 switchport trunk native vlan <id>          ! Misma en ambos extremos
 switchport trunk allowed vlan <ids>        ! add/remove para modificar sin pisar
 switchport nonegotiate                     ! Corta DTP (seguridad)

! ═════════ EtherChannel ═════════
int range <puertos>
 channel-group <n> mode active|passive      ! LACP (IEEE 802.3ad)
 channel-group <n> mode desirable|auto      ! PAgP (Cisco)
 channel-group <n> mode on                  ! Estático (sin negociación)
int Po<n>
 switchport trunk encap dot1q               ! solo en switch L3
 switchport mode trunk
 switchport trunk allowed vlan <ids>

! ═════════ Multilayer Switch (L3) ═════════
ip routing                                 ! Activa el plano L3
int vlan <id>                              ! SVI = gateway de la VLAN
 ip address <ip> <mascara>
 no shutdown
int <interfaz fisica>
 no switchport                             ! Routed port (L3 puro)
 ip address <ip> <mascara>
 no shutdown

! ═════════ Hardening ═════════
int range <puertos sin uso>
 shutdown
 switchport mode access
 switchport access vlan <reserva>

! ═════════ Verificación ═════════
show vlan brief                ! VLANs y puertos
show interfaces trunk          ! Trunks, allowed, native
show interfaces status         ! Estado resumen de puertos
show mac address-table         ! Tabla CAM
show etherchannel summary      ! Bundles y flags
show ip interface brief        ! (MLS) interfaces L3
show ip route                  ! (MLS) tabla de rutas
show running-config            ! Config completa
```

---

## Relacionadas

- [[Configuración de Routers]] — ROAS (la otra forma de inter-VLAN routing) y OSPF del lab
- [[Conceptos Fundamentales de Redes]] — OSI, dominios de broadcast, direccionamiento
- [[Tipos de Interfaces de Red]] — nomenclatura Fa/Gi, slots y puertos
- [[Tipos de IP y Dirección MAC]] — máscaras, subreding, /30 y /28 usados en el lab
