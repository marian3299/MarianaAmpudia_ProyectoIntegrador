# Routopia

Routopia es una plataforma de reserva de tours y destinos turísticos: los usuarios pueden explorar destinos, filtrarlos por categoría o fecha, reservarlos, dejar valoraciones, marcarlos como favoritos y consultar al proveedor por WhatsApp. Los administradores gestionan el catálogo de destinos, categorías, características, usuarios y la configuración general del sitio desde un panel propio.

El proyecto está dividido en dos aplicaciones independientes:

- **`routopia_back/`** — API REST en Spring Boot (Java 17).
- **`tour_booking/`** — SPA en React 19 + Vite que consume esa API.

## Requisitos previos

- **Java 17** y Maven (el proyecto incluye el wrapper `./mvnw`, no hace falta instalar Maven aparte).
- **Node.js** (v18 o superior) y npm.
- Una cuenta de **AWS S3** (para subir imágenes de destinos, categorías y características).
- Opcional para probar el envío de emails de confirmación: una **cuenta de Gmail** con verificación en dos pasos activada (ver más abajo).

## Cómo levantar el backend

```bash
cd routopia_back
cp src/main/resources/application.properties.example src/main/resources/application.properties
```

Editá `application.properties` y completá:

- `aws.accessKey`, `aws.secretKey`, `aws.region`, `aws.s3.bucketName` — credenciales de tu bucket de S3.
- `jwt.secret` — cualquier cadena larga y aleatoria (se usa para firmar los tokens de sesión).

Luego:

```bash
./mvnw spring-boot:run
```

El backend queda escuchando en `http://localhost:8080`.

- Swagger (documentación de la API): `http://localhost:8080/swagger-ui/index.html`
- Consola H2 (base de datos local, se guarda en `routopia_back/data/routopia`): `http://localhost:8080/h2-console`
  - JDBC URL: `jdbc:h2:file:./data/routopia`
  - Usuario: `sa` — Contraseña: `sa`

Al arrancar por primera vez, el backend siembra automáticamente:

- Un usuario administrador: `admin@routopia.com` / `admin123`
- Un usuario de prueba: `user@routopia.com` / `user123`
- 10 destinos de ejemplo con sus categorías, características y políticas.

## Cómo levantar el frontend

```bash
cd tour_booking
npm install
npm run dev
```

La app queda disponible en `http://localhost:5173`. El `vite.config` ya tiene configurado un proxy de `/api/*` hacia `http://localhost:8080`, así que **el backend tiene que estar corriendo** para que el frontend funcione.

## Configurar el envío de emails (panel de administrador)

Las reservas disparan automáticamente un email de confirmación al usuario, pero las credenciales de envío **no se configuran en un archivo** — se cargan en tiempo real desde el panel de administración, y quedan guardadas en la base de datos (tabla `app_settings`). Mientras no se configuren, el envío de emails simplemente se omite (no rompe la creación de la reserva).

### 1. Generar una contraseña de aplicación de Gmail

Gmail no permite usar tu contraseña normal para que una app externa envíe correos — hace falta una "contraseña de aplicación":

1. Entrá a tu cuenta de Google → **Seguridad**.
2. Activá la **Verificación en dos pasos** si todavía no la tenés activada (es un requisito para poder generar contraseñas de aplicación).
3. Buscá **"Contraseñas de aplicaciones"** (podés escribirlo directo en el buscador de ajustes de tu cuenta de Google).
4. Creá una nueva, elegí cualquier nombre descriptivo (por ejemplo "Routopia"), y Google te va a dar un código de 16 caracteres. Copialo — es la **contraseña de aplicación**, no tu contraseña normal.

### 2. Cargarla en Routopia

1. Logueate en el frontend como administrador (`admin@routopia.com` / `admin123` si usás el usuario sembrado por defecto).
2. En el menú superior, hacé clic en **"Configuración"**.
3. En la sección **"Email de confirmación"**, completá:
   - **Cuenta de Gmail**: la dirección de correo completa (ej. `tucuenta@gmail.com`).
   - **Contraseña de aplicación**: el código de 16 caracteres que generaste en el paso anterior.
4. Guardá. A partir de ese momento, cada reserva confirmada va a disparar un email automático al usuario que reservó.

> Si más adelante solo necesitás cambiar el email remitente y no la contraseña, podés dejar el campo de contraseña en blanco al guardar — Routopia mantiene la contraseña que ya tenía cargada.

### 3. (Opcional) Configurar WhatsApp

En esa misma pantalla de **"Configuración"** también se define el número de WhatsApp y el mensaje por defecto que usa el botón flotante de contacto (visible para cualquier visitante, sin necesidad de estar logueado). El número debe ir en formato internacional, solo dígitos, sin "+" ni espacios (por ejemplo `5491112345678`).
