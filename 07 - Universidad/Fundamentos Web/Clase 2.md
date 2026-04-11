--------

# Estructura de una URL

Las URL (Uniform Resource Locator) es la dirección única que identifica un recuro en la web, como una página, imagen, video. Un ejemplo es https://www.google.com

Protocolo -> indica el método que se usa para acceder al recurso en este caso HTTP o HTTPS
Dominio o Host -> Es el nombre del servidor donde se aloja el recurso.
Puerto -> Es el canal de comunicación utilizado. Si no se especifica se usan los predeterminados como 80 o 443.
Ruta -> Especifica la ubicación exacta del recurso.
Parámetros -> se utiliza para enviar datos al servidor, se escriben con `?` y se los separa con &.
Fragmento (anchor o hash) -> Identifica una sección especifica del documento.

## Tipos de URLs

- **Absolutas** -> contienen toda la información necesaria para acceder al recursos incluyendo el dominio, es decir toda la información viaja y se usa en la url.
- **Relativas** -> depende del contexto del sitio no incluyen el dominio.

## Buenas practicas

- Mantenerlas legibles y descriptivas
- Usar minúsculas y evitar espacio o caracteres especiales.
- Utilizar guiones en lugar de espacios.