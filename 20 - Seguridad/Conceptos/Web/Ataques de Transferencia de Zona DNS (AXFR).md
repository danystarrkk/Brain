---
aliases: ["AXFR Attack", "DNS Zone Transfer", "Transferencia de Zona DNS", "AXFR"]
tags:
  - sec/web
  - sec/recon
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
  - "[[Nmap Guía Completa]]"
  - "[[HTTP Host header attacks]]"
  - "[[Information disclosure]]"
  - "[[Enumeración y Explotación de SQUID Proxies]]"
fuente: "https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/01-Information_Gathering/05-Testing_for_DNS_Zone_Transfers"
---

# Ataques de Transferencia de Zona DNS (AXFR)

Una **transferencia de zona AXFR** (*Authoritative Transfer*) es el mecanismo estándar del protocolo DNS por el cual un servidor **secundario** obtiene del servidor **primario** una **copia completa de la base de datos de una zona**. Cuando un servidor autoritativo acepta una transferencia a **cualquier consultante anónimo**, esa misma función legítima de replicación se convierte en el reconocimiento más rentable que existe: una sola consulta devuelve **el inventario completo de nombres, hosts, servicios y políticas** de la organización — sin ruido, sin logs de IDS, sin fuerza bruta.

> [!info] Por qué este ataque es tan goloso
> El descubrimiento de subdominios por fuerza bruta (wordlists de millones de entradas) genera tráfico masivo y sigue siendo incompleto. Un AXFR exitoso devuelve **la zona autoritativa exacta, incluidos registros que ninguna wordlist adivinará**: `dev-interno.corp.local`, `vpn.partner-legacy.com`, SRV de servidores internos, TXT con políticas y metadatos. Es reconocimiento perfecto: consultas legítimas del protocolo, respuesta legítima del servidor.

## Mecánica: cómo funciona DNS y qué es exactamente una zona

Para entender el ataque hay que entender el protocolo, no memorizar el comando.

| Concepto | Definición operativa |
|---|---|
| **Zona** (*zone*) | Fragmento del árbol DNS administrado por un conjunto concreto de servidores autoritativos (p. ej. `ejemplo.com`, no el TLD entero) |
| **Servidor autoritativo** | Responde con datos **finales** (AA bit) para una zona; posee el archivo/DB de la zona |
| **Servidor de resolución** | Recursa en nombre del cliente: consulta raíz → TLD → autoritativo |
| **Primario (master)** | Tiene la fuente de verdad de la zona; acepta/refuta transferencias |
| **Secundario (slave)** | Copia la zona vía AXFR/IXFR y la sirve como autoritativo (redundancia) |
| **Notificación (NOTIFY)** | El primario avisa a los secundarios de cambios para que sincronicen sin esperar al SOA refresh |
| **Transferencia** | Copia completa (AXFR) o incremental (IXFR, RFC 1995) de los registros de la zona |

**Flujo del ataque**: el atactante no necesita "romper" nada. Solo debe actuar como un secundario legítimo. La transferencia ocurre así:

1. El consultante envía una consulta `AXFR` para la zona. **Va por TCP puerto 53** (el único flujo DNS que requiere TCP de forma nativa: la zona puede superar los 512 bytes de UDP y necesita la confiabilidad de TCP para una transacción larga).
2. El servidor responde con **una serie de mensajes** (no uno solo): abre y cierra la transferencia con un registro **SOA** idéntico, y en medio emite **todos los registros** de la zona — A, AAAA, MX, NS, TXT, SRV, CAA, PTR, y cualquier registro interno.
3. Si el servidor no está restringido, devuelve la zona a **quienquiera** que pregunte. No hay autenticación por defecto en el protocolo: la "autorización" depende 100% de la configuración del servidor.

> [!tip] Propósito legítimo de AXFR
> La transferencia de zona es una **característica del protocolo**, no un error de implementación: permite la replicación primario→secundario que da redundancia y coherencia DNS. El problema nace cuando la función no se restringe a los servidores secundarios autorizados — configuración históricamente permisiva por defecto en versiones antiguas de BIND y en despliegues mal documentados.

### AXFR vs IXFR vs NSEC walking

| Mecanismo | Qué devuelve | Requisito | Equivalente ofensivo |
|---|---|---|---|
| **AXFR** | Zona completa | El servidor acepta la transferencia | `dig @ns zona AXFR` |
| **IXFR** | Solo deltas desde un serial | Consultante conoce un serial antiguo | Tras un AXFR denegado probar IXFR: algunos servidores responden incremental aunque denieguen completo |
| **NSEC walking** | Lista de nombres existentes (DNSSEC, modo NSEC clásico) | Zona firmada con NSEC (no NSEC3 opt-out) | `dnsrecon -z` / `nsec3walker` — no es AXFR pero da el mismo resultado: enumeración completa |
| **Catálogo de zonas** (BIND catalog zones, RFC 9432) | El primario publica qué zonas sirve a qué secundarios | Zona de catálogo legítima expuesta | Leer la zona de catálogo revela el mapa completo de zonas del proveedor DNS |

> [!warning] Información sensible expuesta por un AXFR exitoso
> Una transferencia exitosa entrega al atacante, en una sola transacción:
> *   **Subdominios completos**: `dev.`, `staging.`, `vpn.`, `gitlab.`, `backup.` — superficie de ataque sin fuerza bruta.
> *   **Direcciones IP internas**: registros A/AAAA de recursos internos publicados en la zona → plano para pivoteo y reconocimiento interno (combinable con proxies abiertos como en [[Enumeración y Explotación de SQUID Proxies]]).
> *   **Servidores de correo (MX)**: targets para phishing, spoofing, pruebas de SPF/DMARC.
> *   **Servidores de nombres (NS)**: mapa de la infraestructura DNS, otros dominios administrados por el mismo equipo.
> *   **TXT / SPF / DKIM / DMARC**: políticas de correo y a veces datos arbitrarios con pistas internas (proveedores, rangos IP autorizados, tokens).
> *   **SRV / CAA / registros internos**: servicios (LDAP, SIP, `_autodiscover`), autoridades de certificados, nombres que ninguna wordlist contiene.

## Metodología de ataque: workflow de exploración DNS

### Paso 1 — Descubrir los servidores autoritativos de la zona

```bash
# Servidores NS delegados para el dominio (respuesta recursiva normal)
dig +short NS ejemplo.com

# Autoridad exacta que responde la zona (quién responde con AA=1)
dig +short NS ejemplo.com @$(dig +short A ns1.ejemplo.com | head -1)

# Registro de inicio de zona (SOA) — confirma autoridad y serial
dig +short SOA ejemplo.com
```

### Paso 2 — Intentar la transferencia contra **cada** NS

```bash
# Forma canónica: servidor autoritativo + tipo AXFR (TCP 53 implícito)
dig @<DNS-server> <domain-name> AXFR

# Ejemplo concreto
dig @ns1.ejemplo.com ejemplo.com AXFR

# Corto: solo los valores de los registros (parseo rápido)
dig @ns1.ejemplo.com ejemplo.com AXFR +short

# Alternativa clásica: lista completa de la zona con host
host -l -a ejemplo.com ns1.ejemplo.com
```

> [!tip] Prueba TODOS los NS
> Cada delegado es una configuración independiente. El patrón de compromiso realista: `ns1` (gestionado por el equipo de red, correcto) deniega, pero `ns2` (apuntado a un proveedor de hosting viejo, o el servidor heredado de una adquisición) acepta. Automatizar el bucle sobre la salida de `dig NS` es la diferencia entre "AXFR denegado" y "zona completa".

**Criterios de lectura de la respuesta:**

| Salida de dig | Significado |
|---|---|
| Registros + `XFR size reached` / mensajes múltiples | **AXFR aceptado** — procesar y guardar la zona |
| `Transfer failed.` / `refused` | Denegado por ACL (`allow-transfer`) — seguir con wordlists o NSEC walking |
| Timeout en TCP 53 | Firewall o servidor no implementa AXFR; probar otro NS |
| `SERVFAIL` | Zona mal referenciada o DNSSEC roto — reintentar contra otros NS |

### Paso 3 — Herramientas automatizadas

| Herramienta | Qué hace | Comando típico |
|---|---|---|
| **dnsenum** | Transferencia de zona, registros, brute force de subdominios, búsqueda de emails en la zona | `dnsenum --enum -f /usr/share/seclists/.../subdomains-top1million-5000.txt ejemplo.com` |
| **dnsrecon** | Standard/brute/zone walking (NSEC), exportación | `dnsrecon -d ejemplo.com -t std` |
| **fierce** | Enumeración DNS con IP walking y detección de wildcard | `fierce --domain ejemplo.com` |
| **axfr-cli / dig en bucle** | Prueba mínima sin dependencias | script propio sobre `dig NS` |

Además de `dig`, existen alternativas para consultas y transferencia: **`host`** (más simple), **`nslookup`** (menos preciso en resultados), **`fierce`** (Perl, histórico) y **`dnsenum`** (el más completo para automatizar AXFR + brute force en un solo paso).

### Paso 4 — De la zona al descubrimiento de subdominios

La zona obtenida es tu wordlist perfecta. Workflow posterior:

1. **Filtrar registros A/AAAA/CNAME** → lista de hosts vivos; validar resolución y alcance (¿público o RFC1918?).
2. **Detectar wildcard DNS**: consultar un nombre aleatorio — si resuelve, el DNS no valida existencia y la enumeración por resolución es engañosa (comparar IPs, TTL y respuestas byte a byte). El AXFR, en cambio, devuelve los registros **reales**.
3. **Enriquecer**: pasar los nombres a crt.sh (certificados CT), a fuzzing dirigido solo con esos nombres como raíz, y correlacionar con servicios expuestos vía [[Nmap Guía Completa]].
4. **Reconocimiento web**: cada subdominio descubierto es un target de virtual host — Host header routing (ver [[HTTP Host header attacks]]), apps de staging/dev suelen ser el eslabón débil.

## Detección (defensor)

- **Logs del servidor DNS**: BIND escribe `client <IP>#: query (AXFR ...) denied` cuando rechaza; una tasa inusual de consultas AXFR (especialmente desde IPs externas o hacia muchas zonas) es señal de enumeración.
- **NetFlow/pcap**: volumen anómalo en TCP/53 desde un mismo origen — transferencias legítimas son pocas y entre IPs fijas.
- **SIEM con reglas**: alertar ante cualquier AXFR cuyo origen no esté en la allowlist de secundarios (el evento es inequívoco: no hay usos legítimos externos).
- **Honeypot de zona**: zona trampa (`canary.ejemplo.com` con un registro único) transferible solo a secundarios — su aparición fuera expone el filtrado.

## Impacto

1. **Reconocimiento completo sin ruido** — punto de partida de casi cualquier breach: el atacante planifica ataques dirigidos (fuerza bruta contra `vpn.`, aplicaciones viejas en `test.`, paneles administrativos) sobre un mapa verificado.
2. **Movimiento lateral**: IPs internas y nombres de infraestructura reducen la fase de descubrimiento interno a cero.
3. **Exposición de políticas de correo** → phishing de precisión (dominios aliados, rutas de reenvío en SPF).
4. **Pivote a otros activos**: subdominios apuntando a proveedores-cloud mal configurados (buckets, instancias de staging) son el vector real de breach; el AXFR solo te dijo dónde mirar.

## Prevención (con configuración)

La medida crítica es simple: **restringir transferencias a los secundarios autorizados, sin excepción**.

```bash
# BIND 9: transferencia solo al secundario autenticado (mejor: con TSIG)
zone "ejemplo.com" {
    type master;
    file "db.ejemplo.com";
    allow-transfer { key "slave-key."; };      # requiere TSIG correcto
    # Variante por IP (menos fuerte, spoofeable en redes controladas):
    # allow-transfer { 192.168.1.10; 192.168.1.11; };
};

# Si NO hay secundarios: denegar explícitamente
zone "ejemplo.com" {
    type master;
    file "db.ejemplo.com";
    allow-transfer { none; };
};
```

```bash
# Generación de clave TSIG para autenticar la transferencia (primario y secundario)
tsig-keygen -a hmac-sha256 slave-key > /etc/bind/keys/slave-key.conf
# En el primario: allow-transfer { key "slave-key."; };
# En el secundario: server { 10.0.0.1; keys { "slave-key."; }; };
```

Para otros servidores:

| Software | Restricción |
|---|---|
| **Knot DNS** | `acl.allow-transfer: [ "slave-keys" ]` en `knot.conf` |
| **NSD** | `provide-xfr: 10.0.0.10 NO` / lista de IPs con `NO`=solo notificaciones |
| **PowerDNS** | `allow-axfr-ips=10.0.0.10` y `only-notify` |
| **Windows DNS** | Propiedades de zona → *Transferencia de zona a servidores* → solo IPs listadas |
| **Cloud Route53 / DNS gestionado** | Transferencias solo entre zonas del mismo cuenta; denegar por defecto |

Reglas de higiene:
- **Auditar periódicamente** (`dig AXFR` contra cada NS propio desde fuera — es un test de seguridad de 10 segundos; automatizarlo con cron).
- Nunca dejar `allow-transfer { any; };` (configuración legacy en ejemplos antiguos de tutoriales de Internet).
- Aislar TCP/53 de Internet salvo el tráfico estricto de resolución si la arquitectura lo permite.
- Considerar NSEC3 (o NSEC con respuestas minimizadas) si la zona está firmada, para cerrar el walking alternativo — con sus límites (NSEC3 opt-out sigue filtrando delegaciones).

## Errores comunes

**Atacante:**
1. Probar AXFR solo contra el NS "principal" (`ns1`) y no contra todos los delegados.
2. Olvidar que AXFR usa **TCP 53** — descartar "timeout" sin verificar si el firewall filtra TCP/53 (muchos solo permiten UDP/53).
3. No comprobar si el nombre tiene **wildcard DNS**: enumerar subdominios por resolución sobre un wildcard da nombres falsos que no existen.
4. Asumir que "Transfer failed" significa "imposible" — probar IXFR (`dig IXFR=serial`) y NSEC walking antes de rendirse.
5. Ignorar el **barrido de dominios heredados**: transferencia funciona en zonas apuntadas por registros NS antiguos o en zonas padre mal vigiladas.

**Defensor (falsas soluciones):**
1. "Nadie va a adivinar nuestro NS interno" — los NS públicos se listan en los propios delegados (`dig NS` los publica a cualquiera; `dig +trace` los confirma).
2. Confiar solo en el firewall — el propio servidor debe denegar la transferencia; si el firewall cae, la zona queda expuesta.
3. Creer que "no publicamos registros internos" — basta un solo A record apuntando a 10.x para abrir el plano interno.
4. Denegar AXFR pero dejar el walking NSEC sin mitigar.
5. Usar IPs fijas en `allow-transfer` sin TSIG en entornos donde el atacante controla una red por la que viajan los paquetes.

## Cheatsheet

```bash
# Descubrir autoridad
dig +short NS ejemplo.com && dig +short SOA ejemplo.com

# Intentar transferencia (contra cada NS)
dig @ns1.ejemplo.com ejemplo.com AXFR
dig @ns2.ejemplo.com ejemplo.com AXFR
host -l -a ejemplo.com ns1.ejemplo.com

# Automatizado
dnsenum --enum ejemplo.com
dnsrecon -d ejemplo.com -t std

# Verificar wildcard (nombre aleatorio debe NO resolver)
dig +short AAAA no-existe-$RANDOM.ejemplo.com
```

| Señal | Lectura |
|---|---|
| Zona completa en la respuesta | Compromiso total del reconocimiento |
| `Transfer failed` | ACL activa — probar IXFR/NSEC/otros NS |
| Timeout solo en TCP | Firewall — no concluyas "denegado" |
| SOA de otra zona | El NS servía otra zona; buscar NS padre/zona lateral |

## Laboratorios y referencias

- OWASP WSTG — Testing for DNS Zone Transfer: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/01-Information_Gathering/05-Testing_for_DNS_Zone_Transfers
- HackTricks — DNS (pentesting, AXFR, subdomain enumeration): https://book.hacktricks.xyz/en/network-services-pentesting/pentesting-dns
- DigitalOcean — Understanding zone transfers (contexto defensivo): https://www.digitalocean.com/community/tutorials/understanding-dns-zone-transfers
- DNSrecon (herramienta): https://github.com/darkoperator/dnsrecon
- RFC 5936 — AXFR: https://datatracker.ietf.org/doc/html/rfc5936
- RFC 1995 — IXFR incremental: https://datatracker.ietf.org/doc/html/rfc1995

## Relacionadas

- [[Reconocimiento y Escaneo de Redes y Web]] — el descubrimiento DNS como fase inicial del reconocimiento
- [[Nmap Guía Completa]] — lleva los hosts descubiertos al descubrimiento de servicios (`-sV -sC` sobre los A records)
- [[HTTP Host header attacks]] — una vez conocidos los subdominios, el routing por Host header abre vhosts internos
- [[Information disclosure]] — el AXFR es disclosure puro: el servidor "filtra" su propia base de datos por diseño mal configurado
- [[Enumeración y Explotación de SQUID Proxies]] — combinación clásica: IPs internas descubiertas + proxy abierto = escaneo interno
