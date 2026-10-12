---
tags:
  - ciberseguridad
  - web
  - asignacion-masiva
tipo: concepto
estado: terminado
---

# Ataque de Asignación Masiva (Mass Assignment Attack)

El ataque de asignación masiva (**Mass Assignment Attack**, también llamado **over-posting**) se basa en la manipulación de parámetros de entrada de una solicitud HTTP para **crear o modificar campos en un objeto de modelo de datos** en la aplicación web. En lugar de agregar nuevos parámetros inventados, los atacantes explotan los parámetros que el *framework* ya sabe asignar: el servidor mapea ciegamente los datos de la petición sobre los atributos del objeto de negocio, incluidos aquellos que jamás debieron ser controlables por el cliente (rol, saldo, verificación, descuento, flags internos).

> [!info] Mecanismo de Asignación Masiva
> Muchos frameworks web (Ruby on Rails, Laravel, ASP.NET MVC, Spring, Django, Express + ORMs) ofrecen "model binding": convierten los parámetros de la petición directamente en propiedades de un objeto de modelo. Esto simplifica el desarrollo, pero sin listas blancas ("fillable" en Laravel, "strong parameters" en Rails, `[Bind(Include=...)]` en ASP.NET) el atacante controla **toda** la superficie de escritura del objeto.

Ejemplo clásico: un formulario de registro solo muestra nombre de usuario, correo y contraseña, pero el modelo `User` tiene además `role`, `is_verified` y `credit`. Si el backend hace `User.create(params[:user])` sin filtrar, el atacante envía `role=admin` en el mismo JSON/formulario y se registra como administrador. No necesita *inyección* alguna: usa el mecanismo legítimo de binding por encima de lo previsto.

## Mecánica: cómo el binding convierte un formulario en RCE o admin

El flujo vulnerable es idéntico en todos los frameworks:

```
Petición (JSON/form/querystring)
        │  parseo
        ▼
Diccionario de parámetros  ──►  Instancia de clase/modelo
        │  asignación campo a campo (sin allowlist)
        ▼
Objeto con propiedades inyectadas  ──►  persistencia / lógica de negocio
```

La gravedad depende de **qué atributos existan** en la clase:

| Atributo expuesto | Impacto del ataque |
|---|---|
| `role`, `is_admin`, `permissions` | Escalada de privilegios directa |
| `price`, `discount`, `credit`, `balance` | Manipulación de lógica financiera |
| `verified`, `email_confirmed` | Bypass de flujos de verificación |
| `id`, `owner_id`, `user_id` | Se convierte en [[Insecure Direct Object References IDOR]] / takeover de objeto |
| `password_reset_token`, `secret` | Account takeover |
| Referencias a objetos serializados/clases | Escalada a [[Ataques de Deserialización]] / RCE (Jackson `enableDefaultTyping`, ASP.NET TypeNameHandling) |

### Protecciones por framework (lo que debes conocer a ambos lados)

| Framework | Binding por defecto | Mecanismo seguro | Ejemplo |
|---|---|---|---|
| Ruby on Rails | Asignaba todo salvo `attr_accessible` (≤3.2) | **Strong parameters** | `params.require(:user).permit(:name, :email)` |
| Laravel (PHP) | `$fillable` (allowlist) o `$guarded` (blocklist) | `$fillable` siempre | `protected $fillable = ['name','email'];` |
| ASP.NET MVC | `UpdateModel`/`ModelState` sobre propiedades públicas | `[Bind(Include="...")]` | `[Bind(Include="Name,Email")] User u` |
| Spring (Java) | `@ModelAttribute` bindea todos los campos | `@InitBinder` + `setAllowedFields` | `binder.setAllowedFields("name","email")` |
| Django | `ModelForm` con `fields`/`exclude` explícitos | Declarar `fields` | `class F(forms.ModelForm): fields = ['name']` |
| Node (Mongoose etc.) | `Objet.assign(doc, req.body)` a mano | Allowlist explícita | copiar solo campos conocidos |

> [!warning] `$guarded = []` es mass assignment habilitado
> En Laravel, vaciar `$guarded` (y no definir `$fillable`) deja **todos** los atributos, incluido `id`, `role` y `password`, escribibles desde la petición. Es el "no protection" por defecto de hecho.

## Metodología de ataque

1. **Mapear el modelo**: recolectar campos observando (a) los formularios reales, (b) las respuestas JSON de la API (devuelven objetos completos con atributos que el formulario no muestra), (c) el JS del cliente y endpoints espejo (registro vs. perfil vs. checkout).
2. **Interceptar** la petición legítima con Burp Suite (ver [[Burp Suite Escaneo Dirigido y Estructuras de Datos No Estándar]]) o Postman.
3. **Inyectar atributos sospechosos** en el cuerpo JSON o en los campos del formulario, con valores plausibles (`role: administrator`, `discount: 100`, `price: 0`, `is_verified: true`).
4. **Probar también variantes de nombre**: mayúsculas (`Role`), prefijos de framework (`user[role]`, `user.role`), anidación (`{"user":{"role":"admin"}}`), arrays y tipos no string (`"role": true`, `"credit": -1000`).
5. **Verificar el efecto** releyendo el recurso (perfil, saldo, panel admin) — la ausencia de error 400 con parámetros desconocidos ya es un indicio fuerte.
6. En APIs REST, encadenar con fuzzing de métodos (ver [[Abuso de APIs]]): un `PATCH`/`PUT` casi siempre es el vector de mass assignment por excelencia.

### Catálogo de payloads por contexto

| Contexto | Payload |
|---|---|
| JSON API | `{"username":"x","password":"x","role":"administrator"}` |
| JSON API (flag booleana) | `{"name":"x","is_admin":true}` (probar también `1`, `"true"`) |
| Formulario PHP | `username=x&password=x&role=admin` |
| Formulario Rails/Laravel anidado | `user[role]=admin` / `user.role=admin` |
| Financiero | `{"price":0}` `{"discount":100}` `{"credit":-1000}` |
| ID de objeto | `{"user_id":1}` o `{"owner":1}` (cambio de pertenencia) |
| Reset de token | `{"password_reset_token":"aaaa"}` (forzar valor propio) |
| Jackson/Java (peligro: deserialización) | `{"@class":"com.example.Admin"}` — ver [[Ataques de Deserialización]] |

## Detección (lado defensor y auditor)

- Las respuestas de la API contienen **más campos** de los que el formulario envía (el objeto completo se serializa de vuelta): señal de que el modelo tiene superficie oculta.
- El endpoint acepta parámetros que nunca muestra la UI (`role` en el JS no aparece, pero el binding lo procesa).
- No hay diferencia entre enviar un campo inexistente (debería ignorarse o rechazarse) y uno existente sensible.
- Código fuente: búsquedas de `params.require(...).permit(` ausentes, `Model.create(request.body)`, `Object.assign(user, req.body)`, `@ModelAttribute` sin `setAllowedFields`.
- Automatización: Intruder/ffuf sobre el JSON body con una wordlist de atributos de modelo (p. ej. los nombres de columnas que aparecen en respuestas o en errores de SQL).

## Impacto real: cómo se encadena

1. **Privilege escalation**: registro o actualización a `admin` → panel de administración → [[Insecure Direct Object References IDOR]] masivo, gestión de usuarios, RCE si el panel tiene features peligrosas.
2. **Manipulación económica** (lab de PortSwigger y crAPI): `discount: 100` en el objeto de compra → comprar artículos gratis. En crAPI, además, un producto con precio negativo restado dos veces **suma** saldo: `70 - (-1000)` → crédito masivo (ver la nota [[Abuso de APIs]] para el walkthrough completo).
3. **Takeover de cuenta**: sobreescribir `email` de otro usuario + flujo de reset, o forzar `password_reset_token`.
4. **RCE**: mass assignment sobre atributos de configuración que controlan rutas de plantillas, clases deserializadas o comandos (histórico famoso: Rails ≤3.2 `attr_accessible` + `attr_accessor` de sistema).

## Notas prácticas

![[Pasted image 20260227165202.png]]

Tenemos esa web donde podemos registrarnos:

![[Pasted image 20260227165224.png]]

Tenemos que capturar y ver cómo se tramita el registro:

![[Pasted image 20260227165504.png]]

> [!tip] Interceptación de Solicitudes
> Herramientas como Burp Suite o OWASP ZAP son esenciales para interceptar y modificar solicitudes HTTP. Permiten examinar los parámetros enviados por el cliente y los recibidos por el servidor, facilitando la identificación de posibles puntos de manipulación para ataques de asignación masiva.

Como podemos ver, el servidor envía ciertos campos, entre los cuales observamos al registrar algunos campos diferentes como el de `role`. En este punto, entra en juego la cuestión del ataque: vamos a intentar enviar una petición similar. Aunque no nos dejará registrar al mismo usuario, podemos en esta nueva petición enviar `role: administrator` con el objetivo de obtener otro tipo de permisos:

![[Pasted image 20260227165815.png]]

> [!warning] Impacto de la Asignación Masiva
> La explotación exitosa de una vulnerabilidad de asignación masiva puede llevar a la escalada de privilegios (como en este ejemplo), la modificación de datos sensibles, la elusión de controles de seguridad o incluso la ejecución remota de código si se manipulan atributos que controlan la lógica de la aplicación o la deserialización de objetos.

Como podemos ver, al ingresar, el servidor confía en el parámetro enviado, lo cual es un error grave. Intentemos ingresar como el usuario, recordando que este se creó al final:

![[Pasted image 20260227170132.png]]

El servidor aceptó `role: administrator` sin validar contra una lista de roles permitidos: creamos directamente una cuenta de administrador. Lab terminado.

> [!tip] Relación con el laboratorio de PortSwigger
> El lab "Exploiting a mass assignment vulnerability" (comprar la chaqueta inyectando `discount: 100` en el carrito) está walkthrough-eado paso a paso con capturas en la nota [[Abuso de APIs]].

## Prevención (con código)

La única defensa correcta es la **lista blanca explícita de campos escribibles** — nunca filtrar por blocklist de campos peligrosos.

```php
// Laravel: $fillable es una ALLOWLIST — todo lo demás se descarta silenciosamente
class User extends Model {
    protected $fillable = ['name', 'email', 'password'];
    // role, is_verified, credit... NO se pueden asignar masivamente
}
```

```ruby
# Rails: strong parameters — permit solo lista los campos aceptados
def user_params
  params.require(:user).permit(:name, :email, :password)
end
User.create(user_params)   # params[:user][:role] se ignora
```

```java
// Spring: restringir el binding a nivel de controller
@InitBinder
public void initBinder(WebDataBinder binder) {
    binder.setAllowedFields("name", "email", "password");
}
```

```python
# Django: ModelForm con fields explícito (nunca fields = "__all__" en forms públicos)
class UserForm(forms.ModelForm):
    class Meta:
        model = User
        fields = ["name", "email", "password"]
```

```javascript
// Node/Express: asignar campo a campo, jamás Object.assign(user, req.body)
const ALLOWED = new Set(["name", "email"]);
for (const k of Object.keys(req.body)) {
  if (ALLOWED.has(k)) user[k] = req.body[k];
  else throw new Error(`unexpected field: ${k}`);   // rechazar > ignorar
}
```

Reglas de diseño que cierran la clase por completo:

- Los campos sensibles (rol, estado de verificación, saldos) **no se modifican nunca** por rutas de input de usuario: se cambian por flujos dedicados con autorización propia (p. ej. endpoint `POST /admin/promote` con chequeo de rol).
- Rechazar (400) parámetros desconocidos en APIs nuevas: convierte el fuzzing de atributos en trivialmente detectable.
- Serializar respuestas con **DTOs de salida**, no con el objeto de modelo completo: reduce la superficie de descubrimiento del ataque.
- Audit-binding: en revisiones, buscar `.create(req.body)`, `update($_POST)`, `setattr(obj, k, v)` con `k` del cliente.

## Errores comunes

**Atacante:**
1. Solo probar en el formulario visible — olvidar los endpoints `PUT`/`PATCH` equivalentes de la API (donde casi siempre está la ventana).
2. Enviar `role=admin` sin verificar antes qué campos devuelve el objeto en la respuesta: adivinar en ciego pierde tiempo.
3. Asumir que un 200 significa éxito: verificar siempre el estado **después** (releer perfil/saldo/rol).

**Defensor (falsas soluciones):**
1. Blocklist de campos (`unset($params['role'])`): se olvida un alias (`is_admin`, `rol`, `Role`) y se rompe al añadir atributos nuevos.
2. Validar solo en el cliente (JS u ocultar campos del formulario): el binding ocurre en el servidor.
3. Confiar en que "el framework protege": Rails ≤3.2 no lo hacía, Laravel solo protege si hay `$fillable` declarado.
4. Creer que mass assignment e inyección son disjoint: un atributo que llega a una consulta SQL sin parametrizar convierte mass assignment en [[SQL Injection]].

## Cheatsheet

| Paso | Acción |
|---|---|
| 1 | Capturar petición de registro/perfil/compra en Burp |
| 2 | Leer la respuesta JSON del objeto completo → lista de atributos |
| 3 | Inyectar atributos sensibles (`role`, `discount`, `price`, `is_verified`) |
| 4 | Variantes: anidado (`user[role]`), booleanos (`true`/`1`), negativos, arrays |
| 5 | Verificar efecto releyendo el recurso |
| 6 | En APIs: probar `PATCH`/`PUT`/`POST` alternativos con `-X FUZZ` (ffuf) |

| Señal | Significado |
|---|---|
| Acepta campos desconocidos sin 400 | Binding sin allowlist |
| Respuesta con campos extra | Modelo expuesto completo |
| `role`/`discount` procesado desde la petición | Mass assignment confirmado |

## Laboratorios y referencias

- PortSwigger — Mass Assignment: https://portswigger.net/web-security/mass-assignment
- PortSwigger lab "Exploiting a mass assignment vulnerability" (walkthrough propio en [[Abuso de APIs]])
- OWASP — Mass Assignment: https://owasp.org/www-community/attacks/Mass_Assignment
- OWASP crAPI (del lab propio): https://github.com/OWASP/crAPI
- Rails Guides — Strong Parameters: https://guides.rubyonrails.org/action_controller_overview.html#strong-parameters
- Laravel — Eloquent Mass Assignment: https://laravel.com/docs/eloquent#mass-assignment

## Relacionadas

- [[Abuso de APIs]] — el ecosistema completo de explotación de APIs (BOLA/BFLA, enumeración, versiones antiguas) con el que se combina este ataque
- [[Insecure Direct Object References IDOR]] — el otro lado del mismo problema: control sobre *qué objeto* se toca
- [[GraphQL Introspección y Mutaciones en Ataques IDOR]] — mass assignment aplicado a mutaciones GraphQL
- [[Ataques de Deserialización]] — cuando los atributos inyectados controlan objetos deserializados (RCE)
- [[Ataque de Type Juggling]] — confusión de tipos en comparaciones como bypass complementario (PHP)
- [[Nmap Guía Completa]] y [[Burp Suite Escaneo Dirigido y Estructuras de Datos No Estándar]] — enumeración y proxy de trabajo estándar
