---
aliases: ["SQL Truncation", "Truncado SQL", "SQL Truncation Attack"]
tags:
  - sec/web
  - sec/web/sqli
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[SQL Injection]]"
  - "[[Inyección NoSQL]]"
  - "[[Abuso de APIs]]"
  - "[[Insecure Direct Object References IDOR]]"
  - "[[Ataque de Asignación Masiva]]"
fuente: "https://portswigger.net/web-security/sql-injection"
---

# Ataque de Truncado SQL

El ataque de **truncado SQL** (**SQL Truncation**) es una técnica en la que un atacante **aprovecha la truncación silenciosa de cadenas** que algunas bases de datos aplican al insertar datos en columnas de longitud fija (`CHAR`/`NCHAR`) o en configuraciones sin modo estricto, combinada con el **tratamiento de espacios finales** en comparaciones. El resultado clásico: registrar un segundo usuario cuyo nombre, una vez truncado, **colisiona con una cuenta existente privilegiada** (p. ej. `admin@x.com`), sobreescribiéndola o dándole otra password — todo sin una sola comilla de SQL.

> [!info] No es inyección SQL
> No se inyecta sintaxis: la consulta sigue siendo la misma que la aplicación escribió. El ataque explota una **discrepancia semántica** entre lo que la aplicación cree que guarda y lo que la base de datos guarda realmente. Por eso las consultas parametrizadas no lo mitigan por sí solas — aunque el resto de tu defensa contra [[SQL Injection]] sí te deja bien parado en otras vías.

## Mecánica: dos comportamientos que chocan

### 1. Truncación al insertar

Cuando una cadena excede la longitud de la columna:

| Base de datos / tipo | Comportamiento por defecto |
|---|---|
| SQL Server `CHAR(n)`/`NCHAR(n)` | Rellena con espacios a la derecha y **trunca el exceso** |
| SQL Server `VARCHAR(n)` | Trunca el exceso **en modo no estricto**; error `String or binary data would be truncated` en modo estricto (`SET ANSI_WARNINGS ON` lo fuerza) |
| MySQL `VARCHAR(n)` | Con `sql_mode` **sin** `STRICT_TRANS_TABLES` (legacy): trunca silenciosamente; con modo estricto (default desde 5.7): error `Data too long for column` |
| PostgreSQL | **Siempre error** (`value too long for type character varying(n)`) — no vulnerable a truncado |

### 2. Espacios finales en comparaciones

| Motor | `WHERE a = b` con `b = 'admin '` (espacio final) |
|---|---|
| SQL Server | Los espacios finales de `CHAR`/comparación se **ignoran**: `'admin' = 'admin  '` → TRUE |
| MySQL | Igualdad `=` ignora espacios finales en strings: `'admin' = 'admin  '` → TRUE (`PAD SPACE` collations; con `NO_PAD` collations no) |
| PostgreSQL | `'admin' = 'admin '` → **FALSE** (distinción exacta) |

Combinando ambas propiedades nace el ataque: si la columna es de longitud 17 y `admin@admin.com` mide 15, el atacante registra `admin@admin.com  a` (18 chars). La app valida contra la BD *antes* de truncar (18 ≠ 15 → "no existe, continúa"), la BD inserta `admin@admin.com  ` (17 chars), los espacios finales se ignoran en toda comparación posterior → la fila queda **identificada como** `admin@admin.com` pero con la password del atacante.

> [!warning] Impacto del Ataque
> Este ataque puede llevar a la sobrescritura de datos existentes, la toma de control de cuentas de usuario (como en el ejemplo de `admin`), o la inserción de datos maliciosos que, una vez truncados, coincidan con entradas válidas o privilegiadas. La clave es la discrepancia entre la validación de la aplicación y el comportamiento de truncado de la base de datos.

### Ejemplo paso a paso (columna `email VARCHAR(17)`)

| Paso | Cadena | Estado |
|---|---|---|
| 1 | Existe `admin@admin.com` (15 chars) + su password | Cuenta objetivo |
| 2 | Atacante envía `admin@admin.com  a` (18 chars) | App: duplicado? `18 ≠ 15` → NO → inserta |
| 3 | `INSERT` con truncación | Guarda `admin@admin.com  ` (17 chars) |
| 4 | Login con `admin@admin.com` + password del atacante | Comparador ignora espacios → **match** → admin |

## Variantes y técnicas hermanas

| Variante | Mecanismo |
|---|---|
| **Truncado en registro** | El ejemplo de arriba: segunda fila que colisiona con la existente |
| **Truncado de password reset** | El email de reset se envía/valida sobre la forma truncada → el token llega al buzón del atacante |
| **Colisión de nombre reservado (reserved-name collision)** | Windows/SQL Server: objetos `sp_*`/`xp_*` están reservados. Si puedes crear un stored procedure/trigger con nombre reservado en una BD no-master, las resoluciones de nombre del motor pueden priorizar tu objeto (ataque documentado: `sp_` + autenticación integrada → elevación a sysadmin en instalaciones mal configuradas). MySQL: para crear triggers necesitas `DELIMITER` + `BEGIN...END` porque el nombre `TRIGGER` es palabra reservada |
| **Truncado + unique index parcial** | El índice único se evalúa sobre el valor **pre o post truncación** según motor: en MySQL con sql_mode laxo, dos emails distintos que truncan al mismo valor pueden colisionar o duplicar según el momento de la validación |
| **Truncado en MD5/otros hash cortos** | Campos `CHAR(32)` con hashes: cadenas que truncan al mismo hash |
| **Null-byte como "truncado" del cliente** | `admin%00@x.com` en PHP/C antiguos — prima hermana, ver el patrón en [[Abuso de Subida de Archivos]] (null byte en extensiones) |

### Catálogo de payloads

| Contexto | Payload |
|---|---|
| Email 17 chars | `admin@admin.com  a` (espacios dobles + comodín final) |
| Email genérico (n = longitud columna) | `admin@site.com` + espacios hasta `n-1` + `a` |
| Username corto | `admin` + espacios + `x` |
| Variante numérica | `admin1 ` truncado a `admin` (si el validador no exige `@`) |
| Reset flow | `victim@x.com  attacker@x.com` (la parte visible del buzón depende del cliente) |

Truco de construcción: en Burp, rellenar con **espacios reales** (no `+` en el body salvo que el form sea urlencoded, donde `+` = espacio). Verificar siempre la longitud exacta que acepta el cliente mirando el atributo `maxlength` del HTML.

## Metodología de ataque

1. **Descubrir límites de longitud**: en el formulario aparece `maxlength="13"` (controlado por el cliente). Inspeccionar el HTML (ver [[Abuso de APIs]] para el flujo de trabajo con proxy).
2. **Revelar la longitud real de la columna**: enviar `maxlength` bypassed con cadenas de longitud creciente (14, 15, 16… 30) con Burp/ffuf y observar cuándo la app **deja de rechazar por duplicado** o cambia de error → longitud exacta del esquema.
3. **Construir la colisión**: `víctima` + espacios hasta `len(col)-1` + carácter comodín.
4. **Registrar / actualizar** y verificar: login con la cuenta de la víctima y tu password.
5. En APIs: mismo patrón con JSON (`{"email":"admin@x.com   a"}`) — los límites de longitud del cliente no existen allí.

## Detección

- Formularios con `maxlength` del cliente y validación de duplicados **previa** al INSERT (el flujo "usuario no disponible" solo se dispara con la cadena exacta).
- Errores que desaparecen al añadir espacios finales.
- Filas duplicadas aparentes en BD: `SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*)>1` vs `SELECT COUNT(*) FROM users` — la diferencia revela colisiones por PAD SPACE.
- Auditoría: `sql_mode` sin `STRICT_TRANS_TABLES` en MySQL; columnas `CHAR(n)` usadas para emails; validación de unicidad solo en aplicación.

## Prevención (con código)

```python
# 1. Validar longitud ANTES de tocar la BD, contra el contrato del esquema
EMAIL_MAX = 17
if len(email) != len(email.rstrip()) or len(email) > EMAIL_MAX:
    raise ValidationError("email inválido")
```

```sql
-- 2. MySQL: modo estricto (el truncado silencioso desaparece)
SET GLOBAL sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';

-- 3. Unicidad real en la BD: la última línea de defensa
ALTER TABLE users ADD CONSTRAINT uq_email UNIQUE (email);

-- 4. CHECK constraint que impida el "relleno" con espacios (PostgreSQL / MySQL 8.0.16+)
ALTER TABLE users ADD CONSTRAINT ck_email_no_pad CHECK (email = RTRIM(email));
```

```php
// 5. Normalizar en la aplicación: recortar y validar formato completo
$email = trim($_POST['email']);
if (!filter_var($email, FILTER_VALIDATE_EMAIL) || strlen($email) > 17) {
    die("email inválido");
}
// 6. Y AUN ASÍ usar consultas parametrizadas (defiende de [[SQL Injection]])
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = ? AND password = ?");
$stmt->execute([$email, $hash]);
```

Reglas de oro:

- **La BD manda la unicidad**: constraint UNIQUE + normalización `RTRIM`; la validación de la app es UX, no seguridad.
- Comparar identidades sin depender de collations con PAD SPACE: normaliza siempre (`TRIM`) antes de comparar y guarda formas canónicas.
- Nunca confíes en `maxlength` del cliente: es decoración.

## Errores comunes

**Atacante:**
1. Enviar los espacios sin decodificarlos cuando el formulario los colapsa — capturar con Burp y editar en crudo.
2. Asumir la longitud del `maxlength` del cliente = longitud de columna: puede ser menor. Bisectar con peticiones reales.
3. Olvidar que PostgreSQL no trunca: perder tiempo en Postgres con esta técnica.
4. Producir la colisión pero loguearse con la password vieja: la fila nueva puede quedar como segundo registro inaccesible según motor; verificar qué cuenta acepta qué password.

**Defensor (falsas soluciones):**
1. "Ponemos `maxlength` en el HTML" — no existe servidor; se elimina con Burp.
2. "Usamos consultas parametrizadas" — parametrización no evita truncado; necesitas validación de longitud + constraints.
3. "Validamos el formato con regex" sin longitud: `a@b.c` con 5000 caracteres sigue siendo "formato válido" hasta que trunca.
4. Confiar en el `sql_mode` legacy "por compatibilidad".

## Cheatsheet

| Paso | Acción |
|---|---|
| 1 | Detectar `maxlength` en el formulario (inspección) |
| 2 | Bypass en Burp → medir longitud real de la columna por bisect |
| 3 | Payload: víctima + espacios hasta `n-1` + `x` |
| 4 | Registrar → loguear con credenciales propias sobre la cuenta de la víctima |
| 5 | MySQL: comprobar `SELECT @@sql_mode` (¿STRICT?) |

| Señal | Significado |
|---|---|
| `maxlength` + unicidad solo en app | Superficie clásica |
| MySQL sin STRICT_TRANS_TABLES | Truncado silencioso activo |
| `'a' = 'a  '` → 1 | Collation PAD SPACE |
| PostgreSQL | Esta técnica no aplica |

## Notas prácticas

![[Pasted image 20260306190213.png]]

Mientras se hacen pruebas vamos a ver que no se nos permite poner mas de cierta cantidad de caracteres:

![[Pasted image 20260306190422.png]]

como podemos ver no podemos escribir mas y si revisamos el código fuente podremos encontrar lo siguiente:

![[Pasted image 20260306190507.png]]

como podemos observar tenemos una longitud máxima y esto ya nos puede dar indicios de un truncado SQL (en este caso 13 dígitos, controlados solo por el cliente).

Nosotros en la web podemos registrarnos y vamos a hacerlo de la siguiente manera:

![[Pasted image 20260306190724.png]]
![[{3EC1B9B2-E6DF-4D5C-A6F7-BE5DF12710FC}.png]]

podemos observar que ya nos encontramos registrados y además si intentamos volver hacerlo con nuestro usuario no podemos, pero si podemos intentar ingresar con las credenciales que nos registramos:

![[Pasted image 20260306213654.png]]

Dejando de lado esto, la vulnerabilidad sucede por la restricción hecha en el lado del cliente: recordemos que los 13 dígitos los limitan de nuestro lado, por lo tanto podemos modificar la petición con Burp y enviar más. Es allí el problema: si nos limitan es porque la base de datos solo admite ese número de caracteres, y podemos sobrescribir usuarios usando esto — definimos uno nuestro, ponemos espacios hasta el límite y terminamos con un carácter comodín para que, tras el truncado, la fila colisione con la cuenta existente.

## Laboratorios y referencias

- PortSwigger — SQL truncation lab (dentro de SQL Injection): https://portswigger.net/web-security/sql-injection
- MySQL 8.0 — Server SQL Modes: https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html
- SQL Server — char/varchar padding semantics: https://learn.microsoft.com/en-us/sql/t-sql/data-types/char-varchar-text-transact-sql
- Reserved namespace abuse (sp_/xp_): https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-via-sql-server

## Relacionadas

- [[SQL Injection]] — la familia principal de inyección; truncado es la prima "sin comillas"
- [[Inyección NoSQL]] — misma filosofía de discrepancia de semántica en MongoDB (`$ne`, `$gt` en el lab de crAPI)
- [[Abuso de APIs]] — los límites de longitud solo existen en el cliente; las APIs REST exponen el campo sin `maxlength`
- [[Abuso de Subida de Archivos]] — el null-byte truncation (`%00`) es el primo del ataque en el mundo de ficheros
- [[Ataque de Asignación Masiva]] — otra discrepancia entre modelo de app y modelo persistido
