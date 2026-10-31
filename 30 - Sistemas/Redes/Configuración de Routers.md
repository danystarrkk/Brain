---
tags:
  - sistemas
  - redes
  - routers
tipo: concepto
estado: terminado
---

# Configuración de Routers

Esta nota cubre la configuración de routers Cisco IOS basándose en el laboratorio real de la clase (R1 con Router-on-a-Stick y enlace punto a punto a un Multilayer Switch). Cada sección explica **primero la teoría completa** —por qué existe la tecnología, cuándo se usa y cómo funciona por dentro— y recién al final aparecen los comandos como ejemplos aplicados.

> [!info] Requisito previo
> Se asume que entendés qué es una tabla de enrutamiento, la diferencia entre capa 2 y capa 3, y qué hace una máscara de subred. Si no, empezá por [[Conceptos Fundamentales de Redes]] y [[Tipos de IP y Dirección MAC]].

---

## El sistema operativo del router: IOS y sus modos

Cisco IOS (Internetwork Operating System) es un sistema operativo monolítico que corre en flash y trabaja por **sesiones por niveles de privilegio**. Entender la jerarquía de modos es lo primero, porque todo lo demás son comandos anidados dentro de ellos:

| Modo | Prompt | Qué permite | Cómo se entra |
|------|--------|-------------|---------------|
| **User EXEC** | `R1>` | Verificación básica, comundos limitados (`show`, `ping`) | Al conectar por consola/SSH |
| **Privileged EXEC** | `R1#` | Todo el espectro de `show` y `debug`, guardar/restaurar | `enable` desde User EXEC |
| **Global Configuration** | `R1(config)#` | Cambiar la configuración del dispositivo | `configure terminal` desde Privileged EXEC |
| **Interface Configuration** | `R1(config-if)#` | Configurar una interfaz específica | `interface <nombre>` desde Global |
| **Sub-interfaz** | `R1(config-subif)#` | Configurar una interfaz virtual 802.1Q | `interface <física>.<id>` |
| **Router Configuration** | `R1(config-router)#` | Configurar un proceso de routing | `router <protocolo> <proceso>` |

**Flujo típico**: `enable` → `configure terminal` → entrar al submodo → configurar → `end` o `exit` hasta volver → `copy running-config startup-config` (o `write memory`).

### Comandos básicos del entorno

```ios
enable                          ! Sube a Privileged EXEC
configure terminal              ! Entra a Global Configuration
hostname R1-ROAS                ! Nombre del dispositivo (aparece en el prompt)
banner motd #Acceso restringido#  ! Mensaje antes del login (Message Of The Day)
enable secret <contraseña>      ! Hashea con MD5 y cifra la contraseña de enable
do write memory                 ! "do" ejecuta un comando de otro modo sin salir
```

> [!warning] `enable secret` vs `enable password`
> `enable secret` guarda el password con hash MD5 y lo muestra cifrado. `enable password` lo guarda en claro. Usá **siempre** `enable secret`. En IOS moderno se recomienda `enable algorithm-type scrypt secret <pass>` para hashing más fuerte todavía.

> [!tip] `do` — el comodín de IOS
> Desde cualquier modo de configuración, `do <comando>` ejecuta un comando de Privileged EXEC sin tener que salir. Ejemplo: `do show running-config` o `do wr`.

---

## Router-on-a-Stick (ROAS): teoría completa

### El problema que resuelve

Un router con interfaces físicas separadas asigna **una VLAN por puerto físico**: 10 VLANs necesitarían 10 puertos gigabit del router, lo cual es caro e insuficiente. **Router-on-a-Stick (ROAS)** resuelve esto usando **un solo enlace físico (trunk) hacia el switch** y **subinterfaces lógicas**, una por VLAN que necesite gateway.

El nombre viene de la analogía: el enlace trunk es un "palo" (stick) y todas las VLANs cuelgan de él como banderas.

> [!info] Flujo completo de un paquete ROAS
> 1. Host de VLAN 10 envía un paquete al gateway (la IP de la subinterfaz `.10`).
> 2. El switch recibe el frame en un puerto access de VLAN 10, lo re-etiqueta con tag 802.1Q (VID 10) al salir por el trunk.
> 3. El router recibe el frame, lee el tag, y lo entrega a la subinterfaz `g0/0/1.10`.
> 4. El router hace routing (busca en su tabla, decrementa TTL).
> 5. Si la destino es VLAN 20, sale por la subinterfaz `g0/0/1.20` y el router **re-etiqueta** el frame con VID 20.
> 6. El switch recibe el frame ya etiquetado para VLAN 20 y lo conmuta por los puertos access de esa VLAN.

### IEEE 802.1Q: la encapsulación

802.1Q es el estándar IEEE (no Cisco; Cisco también soporta el propietario ISL, obsoleto) que inserta un **tag de 4 bytes** en el frame Ethernet:

| Campo del tag | Bits | Función |
|---------------|------|---------|
| **TPID** | 16 | `0x8100` — identifica "este frame tiene tag 802.1Q" |
| **PCP** | 3 | Priority Code Point — 802.1p, prioridad QoS (0-7) |
| **DEI** | 1 | Drop Eligible Indicator (antes CFI) — descartable en congestión |
| **VID** | 12 | VLAN ID — el número de VLAN (1-4094 útiles) |

12 bits de VID = 4096 valores, pero **VLAN 0 y VLAN 4095 están reservadas**, quedando **4094 VLANs utilizables**. El tag se inserta entre el header Ethernet source MAC y el payload (cuando no hay encapsulación extendida).

### Subinterfaces

Una subinterfaz es una interfaz **virtual y lógica**, no existe en el cable: `GigabitEthernet0/0/1.10` significa "subinterfaz 10 de la interfaz g0/0/1". Cada subinterfaz:

- Lleva un **número de VLAN asociado vía `encapsulation dot1q <vlan>`** — ese número es solo una etiqueta de control; **el VID real viajando en el tag es el que se pasa en ese comando**.
- Tiene **su propia IP/máscara**, que actúa como gateway de esa VLAN.
- El tráfico entre subinterfaces pasa por el proceso de routing del router (CPU), no por el switch fabric.

### La VLAN nativa

La **VLAN nativa** es la VLAN cuyo tráfico cruza el trunk **sin tag** (frame crudo). Existe por compatibilidad con dispositivos que no entienden 802.1Q. Reglas críticas:

- **Por defecto es la VLAN 1** en ambos extremos.
- **Los dos extremos del trunk deben coincidir**: el trunk del switch (`switchport trunk native vlan X`) y la subinterfaz del router (`encapsulation dot1q X native`).
- Si no coinciden → syslog `%CDP-4-NATIVE_VLAN_MISMATCH` y todo el tráfico de la VLAN nativa del switch llega "perdido" al router.
> [!danger] Seguridad: VLAN hopping
> Si la VLAN nativa queda en VLAN 1 (default) y un atacante inyecta frames sin tag, los interpreta la VLAN nativa. **Buena práctica**: mover la nativa a una VLAN inexistente (en este lab se usa la 555) y usar esa misma VLAN inexistente en ambos extremos, o usar DTP off + `switchport nonegotiate`. Ver la teoría completa en [[Configuración de Switches L2 y L3]].

### Limitaciones de ROAS

| Limitación | Detalle |
|------------|---------|
| **Ancho de banda compartido** | Todas las VLANs comparten el mismo enlace físico (full-duplex en la práctica: subida y bajada separadas, pero un solo pipe) |
| **CPU del router** | El tagging/des-tagging lo hace el procesador del router, no hardware ASIC → con tráfico intenso no escala |
| **Sin soporte de EtherChannel** | ROAS no hace LACP sobre el enlace trunk del router |
| **Ventaja** | Cero costo adicional de hardware, ideal para labs y redes chicas |

Cuando ROAS se queda corto, la alternativa es el **enrutamiento entre VLANs en un Multilayer Switch** (SVIs + `ip routing`) — ver [[Configuración de Switches L2 y L3]] — o routers con múltiples interfaces físicas.

### Configuración ROAS — ejemplo del lab (R1)

En el laboratorio, R1 usa `g0/0/1` como enlace trunk hacia el switch, con VLANs 10 (GERENCIA), 20 (TI) y la nativa 555:

```ios
enable
config t
hostname R1-ROAS

! Subinterfaz VLAN 10 — gateway de GERENCIA
int g0/0/1.10
  encapsulation dot1q 10
  ip address 192.168.10.1 255.255.255.192

! Subinterfaz VLAN 20 — gateway de TI
int g0/0/1.20
  encapsulation dot1q 20
  ip address 192.168.20.1 255.255.255.224

! Subinterfaz de la VLAN nativa (sin IP: no enruta, solo acepta tráfico sin tag)
int g0/0/1.555
  encapsulation dot1q 555 native

! La interfaz física SIEMPRE debe estar activa — sin esto las subinterfaz no funcionan
int g0/0/1
  no shutdown

do write memory
```

> [!error] Corrección al script original
> En el script de la clase, la subinterfaz `.20` tenía `ip address 192.168.10.1/26 255.255.255.224` — tres errores: (1) la IP `192.168.10.1` ya está usada por la subinterfaz `.10` (conflicto), (2) la notación `/26` no existe en el comando de IOS (acepta solo `ip address <ip> <mascara>`), y (3) la máscara debía ser `/27` (`255.255.255.224`) para coincidir con la red `192.168.20.0/27` declarada en OSPF. La versión corregida es la de arriba.

**Comandos de verificación en el router:**

```ios
show ip interface brief      ! Estado (up/down) e IP de todas las interfaces
show interfaces g0/0/1.10    ! Detalle: MTU, encapsulación, contador de errores
show ip route                ! Tabla de enrutamiento — buscar la "C" de connected
```

---

## Enlaces punto a punto: la lógica de /30 y /31

Entre dos dispositivos L3 (router ↔ router, o router ↔ MLS) se usa un enlace **punto a punto** con una subred mínima:

| Máscara | IPs utilizables | Uso |
|---------|-----------------|-----|
| `/30` (`255.255.255.252`) | 2 (una por extremo) | Clásico, el estándar en CCNA |
| `/31` (`255.255.255.254`) | 2 (ambas) | RFC 3021 para enlaces punto a punto — ahorra addresses |
| `/29` o más grandes | sobran IPs | Desperdicio innecesario en un link directo |

En el lab: `10.10.10.0/30` → R1 usa `.1`, el MLS usa `.2`. La red `10.10.10.0/30` tiene wildcard `0.0.0.3` para OSPF (ver la sección de OSPF).

```ios
int g0/0/0
  description ENLACE R1 -> MLS      ! Texto descriptivo, se ve en show interfaces
  ip address 10.10.10.1 255.255.255.252
  no shutdown
```

> [!tip] `description` siempre
> Un enlace sin descripción es un enlace imposible de documentar a las 3 AM. Describí origen y destino: `description ENLACE R1 -> MLS`.

---

## OSPF: teoría completa

### ¿Qué es OSPF?

**OSPF (Open Shortest Path First)** es un protocolo de enrutamiento **dinámico** del tipo **link-state** (estado de enlace), definido en **RFC 2328 (OSPFv2, para IPv4)**. A diferencia de los protocolos distance-vector (como RIPv2) que "rumorean" tablas de rutas, OSPF:

1. **Descubre vecinos** — envía paquetes **Hello** por las interfaces (multicast `224.0.0.5`) para detectar vecinos OSPF.
2. **Construye una base de datos de enlaces (LSDB)** idéntica en todos los routers del área — cada router anuncia el estado de sus interfaces (LSA, Link-State Advertisement).
3. **Ejecuta el algoritmo Dijkstra (SPF)** sobre la LSDB calculando el camino más corto (menor costo) hacia cada red.
4. **Instala las rutas en la tabla de enrutamiento** y las mantiene vivas con actualizaciones incrementales (solo cuando hay cambios, no periódicas).

> [!info] Costo OSPF
> Costo por defecto = `10^8 / ancho de banda en bps`. Un enlace Gigabit → costo 1, un enlace FastEthernet → costo 10. OSPF siempre elige el camino de **menor costo acumulado**.

### Áreas OSPF

Para escalar sin que la LSDB crezca sin control, OSPF divide el dominio en **áreas**:

- **Área 0 (backbone)** — es el centro de la topología. **Todo tráfico entre áreas debe pasar por el área 0.**
- **Áreas no backbone (1, 2, …)** — conectan al backbone mediante **ABR (Area Border Router)**.
- Routers internos de un área solo conocen detalladamente las rutas de su propia área; el ABR anuncia un **resumen**.

Reglas básicas:
- Todos los routers del **mismo área** deben tener la **misma LSDB**.
- Área 0 debe ser **contigua**.
- En redes pequeñas (como este lab) se usa **área 0 pura** — todo junto.

### Router-ID

Cada instancia OSPF se identifica con un **Router-ID** de 32 bits con formato de IP (`1.1.1.1`, `2.2.2.2`). Es **solo un identificador**, no tiene que ser una IP real ni alcanzable (aunque es práctica común usar una del loopback o la de una interfaz). Reglas de selección automática:

1. Router-ID configurado **manualmente** (gana siempre).
2. Si no, la IP de **loopback** más alta.
3. Si no, la IP de **interfaz física** más alta.

> [!warning] Router-ID manual siempre
> La selección automática puede cambiar si se modifica una interfaz, lo que provoca un **resync completo de OSPF** (todas las LSAs se re-generan). Configuralo a mano.

### Wildcard mask: la máscara invertida

En el comando `network`, OSPF usa **wildcard mask** (máscara de comodín), que es la **inversa bit a bit de la máscara de subred**:

| Subred | Máscara | Wildcard | Significado |
|--------|---------|----------|-------------|
| `192.168.10.0/26` | `255.255.255.192` | `0.0.0.63` | Matchea las IPs .1–.63 |
| `192.168.20.0/27` | `255.255.255.224` | `0.0.0.31` | Matchea .1–.31 |
| `10.10.10.0/30` | `255.255.255.252` | `0.0.0.3` | Matchea .1–.2 |
| `0.0.0.0` | `255.255.255.255` | `0.0.0.0` | Matchea todas las interfaces (todo en OSPF) |

Regla rápida: **`0` = el bit importa, `255` = el bit no importa**. La wildcard `0.0.0.3` dice "los dos primeros octetos y medio no importan, los últimos 2 bits sí" — exactamente las 4 IPs de un /30.

El comando `network` le dice a OSPF: *"en estas interfaces, participá en el proceso OSPF y anunciá estas redes"*. El número después de `router ospf` es solo el **número de proceso local** (1-65535); dos routers con números distintos de proceso **siguen siendo vecinos** — el proceso es solo interno al dispositivo.

### Configuración OSPF — ejemplo del lab

**R1** (proceso 1, router-id 1.1.1.1):

```ios
router ospf 1
  router-id 1.1.1.1
  network 192.168.10.0 0.0.0.63 area 0    ! VLAN 10 (GERENCIA /26)
  network 192.168.20.0 0.0.0.31 area 0    ! VLAN 20 (TI /27)
  network 10.10.10.0 0.0.0.3 area 0       ! Enlace punto a punto al MLS
```

**MLS** (proceso 2, router-id 2.2.2.2 — ver nota switches):

```ios
router ospf 2
  router-id 2.2.2.2
  network 192.168.30.0 0.0.0.15 area 0    ! VLAN 30 (DC1 /28)
  network 192.168.40.0 0.0.0.7 area 0     ! VLAN 40 (DC2 /29)
  network 10.10.10.0 0.0.0.3 area 0       ! Mismo enlace punto a punto
```

> [!warning] Coherencia entre interfaz y `network`
> El rango que cubre la wildcard del `network` **debe contener la IP real de la interfaz**. Si `g0/0/0` tiene `10.10.10.1/30`, el `network` debe cubrir `10.10.10.0/30` (wildcard `0.0.0.3`). Si no coincide, OSPF nunca activa esa interfaz y no hay vecino.

### Verificación OSPF

```ios
show ip ospf process               ! Estado del proceso, router-id, área
show ip ospf neighbor              ! Vecinos: FULL = OK, INIT/2-WAY = problema
show ip protocols                 ! Protocolos activos y redes anunciadas
show ip route ospf                 ! Solo rutas aprendidas por OSPF (código "O")
show ip route                      ! Tabla completa: C=connected, O=OSPF, S=static
```

Códigos de la tabla de enrutamiento:

| Código | Significado |
|--------|-------------|
| `C` | Connected — red directamente conectada a una interfaz |
| `L` | Local — la IP de la propia interfaz (host route /32) |
| `O` | OSPF intra-área |
| `O IA` | OSPF inter-área (resumen de ABR) |
| `S` | Estática |
| `S*` | Estática por defecto (candidate default) |

---

## Comandos de la nota rápida — referencia cruzada

Estos comandos aparecen detallados arriba; quedan acá solo como índice rápido de ubicación:

| Comando | Sección |
|---------|---------|
| `enable`, `config t`, `hostname`, `banner motd`, `enable secret` | IOS y sus modos |
| `encapsulation dot1q <vlan> [native]` | ROAS |
| `ip address <ip> <mascara>` | ROAS / enlaces p2p |
| `router ospf <proceso>`, `router-id`, `network <red> <wildcard> area <área>` | OSPF |
| `show ip route`, `show ip ospf neighbor`, `show ip interface brief` | Verificación |

---

## Errores comunes y correcciones (detectados en los scripts del lab)

| # | Error en el script | Problema | Corrección |
|---|--------------------|----------|------------|
| 1 | `ip address 192.168.10.1/26 255.255.255.224` en subint .20 | IP duplicada con .10 + notación `/26` inválida en IOS + máscara inconsistente con OSPF | `ip address 192.168.20.1 255.255.255.224` |
| 2 | `swho ip route` | Typo | `show ip route` |
| 3 | Subinterfaz configurada sin `no shutdown` en la física | Las subinterfaz no levantan si la física está administrativamente down | Siempre `no shutdown` en la interfaz física |
| 4 | VLAN nativa sin alinear entre switch y router | `%CDP-4-NATIVE_VLAN_MISMATCH` | Misma VLAN nativa en ambos extremos (`555` en el lab) |
| 5 | Sin `router-id` manual | Router-ID puede cambiar y resincronizar OSPF | `router-id x.x.x.x` siempre explícito |

---

## CheatSheet — Configuración de Routers

```ios
! ═════════ IOS básico ═════════
enable
config t
hostname <nombre>
banner motd #<mensaje>#
enable secret <contraseña>
do write memory                    ! Guardar (equivale a copy run start)

! ═════════ Router-on-a-Stick ═════════
int g0/0/1.<vlan>
  encapsulation dot1q <vlan>
  ip address <ip> <mascara>        ! Gateway de esa VLAN
int g0/0/1.<vlan-nativa>
  encapsulation dot1q <vlan> native
int g0/0/1
  no shutdown                      ! La física SIEMPRE activa

! ═════════ Enlace punto a punto ═════════
int <interfaz>
  description <origen> -> <destino>
  ip address <ip> <máscara /30>
  no shutdown

! ═════════ OSPF ═════════
router ospf <proceso>
  router-id <x.x.x.x>
  network <red> <wildcard> area 0

! ═════════ Verificación ═════════
show ip interface brief             ! Estado + IP de interfaces
show ip route                       ! Tabla de enrutamiento (C, O, S)
show ip route ospf                  ! Solo rutas OSPF
show ip ospf neighbor               ! Vecinos OSPF (estado FULL)
show ip protocols                   ! Protocolos y redes anunciadas
show interfaces <intf>              ! Detalle de interfaz
copy running-config startup-config  ! Persistencia (alias: write / wr)
```

---

## Relacionadas

- [[Configuración de Switches L2 y L3]] — la otra punta del lab: VLANs, trunks, EtherChannel y MLS con SVIs
- [[Conceptos Fundamentales de Redes]] — OSI, dominios de broadcast, direccionamiento
- [[Tipos de Interfaces de Red]] — nomenclatura de interfaces (GigabitEthernet, Serial, etc.)
- [[Tipos de IP y Dirección MAC]] — clases, máscaras, subreding
