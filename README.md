# Kristyco Studio · Backend

API REST de **Kristyco Studio**, una tienda online de **invitaciones de boda y papelería artesanal**, desarrollada como Trabajo de Fin de Grado del Ciclo Formativo de Grado Superior en **Desarrollo de Aplicaciones Web (DAW)**.

Está construida con **Spring Boot 3.4** y **Java 21**, persiste en **PostgreSQL** y da servicio a la aplicación Angular del repositorio [TFG_Frontend](https://github.com/Daniel0611CN/TFG_Frontend).

🔗 **Frontend desplegado:** [kristyco-studio.vercel.app](https://kristyco-studio.vercel.app) · también en [integrated-proyect-fp.vercel.app](https://integrated-proyect-fp.vercel.app/home)

![Java](https://img.shields.io/badge/Java-21-ED8B00?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.3-6DB33F?logo=springboot&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Maven](https://img.shields.io/badge/Maven-C71A36?logo=apachemaven&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)

---

## Tabla de contenidos

1. [Descripción](#descripción)
2. [Funcionalidades](#funcionalidades)
3. [Tecnologías](#tecnologías)
4. [Arquitectura](#arquitectura)
5. [Modelo de datos](#modelo-de-datos)
6. [Seguridad](#seguridad)
7. [Endpoints](#endpoints)
8. [Requisitos previos](#requisitos-previos)
9. [Instalación y ejecución](#instalación-y-ejecución)
10. [Configuración](#configuración)
11. [Documentación de la API](#documentación-de-la-api)
12. [Pruebas](#pruebas)
13. [Docker](#docker)
14. [Despliegue](#despliegue)
15. [Autor](#autor)

---

***Cabe destacar que a día de hoy el backend del proyecto no funcionará en web por finalización del servicio de base de datos y backend***

## Descripción

El backend de Kristyco Studio gestiona el catálogo (productos agrupados en categorías y asociados a proveedores), los usuarios y sus roles, los pedidos y los pagos. Se encarga también de la autenticación, del envío de correos transaccionales (confirmación de cuenta y recuperación de contraseña) y de la integración con **PayPal**.

En el frontend, las **categorías** se muestran como *colecciones* y los **productos** como *invitaciones*.

## Funcionalidades

- **CRUD** de categorías, productos, proveedores, usuarios, pedidos y pagos.
- **Registro** con validaciones y **confirmación de cuenta por email** (el usuario nace deshabilitado hasta confirmar).
- **Login** con token de sesión y control de acceso por roles (`ROL_ADMIN`, `ROL_USER`).
- **Recuperación de contraseña** por email, cambio de contraseña y revalidación de tokens caducados.
- **Envío de correos** (texto, con adjunto y HTML) con plantillas Thymeleaf.
- **Pagos con PayPal** (creación de orden y captura), actualmente en entorno *sandbox*.
- **Consultas de pedidos**: por usuario, por categoría, posteriores a una fecha, por coste mínimo, por estado y búsqueda combinada.
- **Filtrado de productos** por id, nombre, proveedor o categoría.
- **Gestión centralizada de errores** (`GeneralControllerAdvice`) y mensajes de validación en `messages.properties`.
- **Documentación interactiva** con Swagger UI y exploración HAL con Spring Data REST.

## Tecnologías

| Categoría | Tecnología |
|---|---|
| Lenguaje | Java 21 |
| Framework | Spring Boot 3.4.3 |
| API | Spring Web (MVC), Spring Data REST + HAL Explorer, WebFlux (incluido como dependencia) |
| Persistencia | Spring Data JPA (Hibernate), Spring JDBC |
| Base de datos | PostgreSQL |
| Seguridad | Spring Security (stateless) + BCrypt |
| Validación | Spring Validation |
| Correo | Spring Mail + Thymeleaf |
| Mapeo | MapStruct 1.5.5 |
| Productividad | Lombok |
| Documentación | springdoc-openapi 2.8.5 (Swagger UI), colección Postman |
| Testing | JUnit 5, Spring Boot Test, Spring Security Test |
| Build | Maven (con Maven Wrapper), CycloneDX (SBOM) |
| Contenedores | Docker (multi-stage, Amazon Corretto 21) |

## Arquitectura

Aplicación por capas, con el paquete base `org.iesvdm.proyecto_servidor`:

```
src/main/java/org/iesvdm/proyecto_servidor/
├── config/        # WebSecurityConfig, MailConfig, PaypalConfig
├── controller/    # Controladores REST (/api/v1/...)
├── service/       # Lógica de negocio (Pedido, Producto, Mail, Paypal, Password...)
├── repository/    # Repositorios Spring Data JPA
├── model/
│   ├── domain/    # Entidades JPA
│   ├── enums/     # EstadoPedido, EstadoPago, MetodoPago, TipoRol, TokenType
│   └── record/    # Records para tokens, correo y PayPal
├── dto/           # DTOs (login, registro, respuestas)
├── mapper/        # MapStruct
├── security/      # Filtro de token, utilidades de cifrado, entry point
└── exception/     # Excepciones propias y @ControllerAdvice
```

Recursos relevantes en `src/main/resources/`:

| Archivo | Descripción |
|---|---|
| `application.properties` | Configuración de la aplicación |
| `messages.properties` | Mensajes de validación del registro |
| `templates/` | Plantillas de correo (`mail.html`, `resend-mail.html`, `reset-password.html`) |
| `kristyco_studio.postman.json` | Colección de Postman con las peticiones del proyecto |
| `static/img/logo.png` | Logo usado en los correos |

## Modelo de datos

```
Usuario ──< Pedido >──< Producto >── Categoria
   │           │            └──────── Proveedor
   │           └── Pago (1:1)
   └──< Rol (N:M)
```

| Entidad | Campos principales |
|---|---|
| `Usuario` | nombre, apellido1, apellido2, email, teléfono, dirección, password, roles, `enabled`, `locked` |
| `Rol` | `TipoRol`: `ROL_ADMIN`, `ROL_USER` |
| `Producto` | nombre, descripción, precio, stock, imagen, categoría, proveedor |
| `Categoria` | nombre, descripción |
| `Proveedor` | teléfono, email, dirección, fecha de creación |
| `Pedido` | fecha, estado, dirección, total, coste de envío, usuario, productos, pago |
| `Pago` | fecha, monto, método, estado, pedido |
| `ConfirmationToken` | token de confirmación/restablecimiento asociado a un usuario |

Enums: `EstadoPedido` (PENDIENTE, EN_CAMINO, ENTREGADO, CANCELADO), `EstadoPago` (PENDIENTE, CANCELADO, COMPLETADO), `MetodoPago` (TARJETA, BIZUM, EFECTIVO) y `TokenType` (REGISTER_CONFIRMATION, REVALIDATE_TOKEN, RESET_PASSWORD).

El esquema se genera y actualiza automáticamente con `spring.jpa.hibernate.ddl-auto=update`.

## Seguridad

- API **stateless** (sin sesiones) con **CSRF desactivado** y **CORS** activado.
- Contraseñas almacenadas con **BCrypt**.
- Tras el login se devuelve un **token propio**: contiene la marca de tiempo y el nombre de usuario, cifrado con **AES/CBC** (`EncryptionUtil`). Se envía en cada petición como `Authorization: Bearer <token>` y es validado por `AuthTokenFilter`. Duración por defecto: **1 hora** (`time.token`).
- Los tokens de confirmación de registro y de restablecimiento de contraseña se guardan en base de datos y caducan (30 minutos en la confirmación de registro).
- Autorización por rol con reglas centralizadas en `WebSecurityConfig`:

| Petición | Acceso |
|---|---|
| `/api/v1/auth/**` | Público |
| `GET /api/v1/productos/**` | Público |
| `GET` y `PUT` de `/api/v1/confirmation_token/**` | Público |
| `POST /api/v1/confirmation_token/**` | `ROL_ADMIN` o `ROL_USER` |
| `POST /api/v1/pedidos/**` | `ROL_USER` |
| `PUT /api/v1/usuarios/password` y `DELETE /api/v1/usuarios/**` | `ROL_USER` |
| Resto de `GET /api/v1/**` | `ROL_ADMIN` o `ROL_USER` |
| Resto de `POST`, `PUT`, `DELETE /api/v1/**` | `ROL_ADMIN` |
| Cualquier otra ruta | Autenticado |

- **CORS**: se define con `@CrossOrigin` en los controladores y permite el origen `https://kristyco-studio.vercel.app`. Para desarrollo local, cambia a la línea comentada de `http://localhost:4200`.

## Endpoints

Todos cuelgan de `/api/v1`, salvo los de PayPal y Spring Data REST.

| Recurso | Ruta | Operaciones |
|---|---|---|
| Autenticación | `/auth` | `POST /login`, `POST /register` |
| Tokens | `/confirmation_token` | `GET /confirm-register`, `PUT /revalidate-token/{username}`, `PUT /request-reset-password`, `GET /validate-reset-token`, `PUT /reset-password`, `POST /validate-old-password` |
| Categorías | `/categorias` | CRUD |
| Productos | `/productos` | CRUD, `GET /list`, `GET /filterByIdOrNombreOrProvIdOrCatId` |
| Proveedores | `/proveedores` | CRUD |
| Usuarios | `/usuarios` | CRUD, `PUT /password` |
| Pedidos | `/pedidos` | CRUD, `GET /list`, `GET /{id}/pedidosPorUsuario`, `GET /{id}/pedidosByCategoria`, `GET /pedidosAfterFecha`, `GET /getPedidosHigherThanCoste`, `GET /getPedidosByEstado`, `GET /findByIdOrDireccionOrEstadoOrFechaOrCosteEnvio` |
| Pagos | `/pagos` | CRUD |
| Correo | `/mail` | `POST /send-mail`, `POST /send-mail-file`, `POST /send-html-mail` |
| PayPal | `/payment` (sin prefijo `/api/v1`) | `POST /payment/create`, `GET /payment/success` |
| Spring Data REST | `/data-api` | Exploración HAL de las entidades |

Puedes importar `src/main/resources/kristyco_studio.postman.json` en Postman para probar las peticiones.

## Requisitos previos

- **JDK 21**
- **PostgreSQL** accesible (local o en la nube)
- **Maven** (opcional: el repositorio incluye `mvnw` y `mvnw.cmd`)
- Una cuenta de **Gmail** como remitente de los emails (el servidor SMTP configurado es `smtp.gmail.com`)
- Credenciales de **PayPal Developer** (sandbox) para probar pagos
- **Docker** (opcional)

## Instalación y ejecución

```bash
git clone https://github.com/Daniel0611CN/TFG_Backend.git
cd TFG_Backend

# Linux / macOS
./mvnw spring-boot:run

# Windows
mvnw.cmd spring-boot:run
```

La API arranca en <http://localhost:8080>.

Para generar el JAR:

```bash
./mvnw clean package
java -jar target/proyectoServidor-0.0.1-SNAPSHOT.jar
```

## Configuración

Toda la configuración está en `src/main/resources/application.properties`. Cualquier propiedad puede **sobrescribirse con variables de entorno** (Spring Boot las convierte automáticamente), lo que permite no guardar secretos en el repositorio.

| Propiedad | Variable de entorno | Descripción |
|---|---|---|
| `spring.datasource.url` | `SPRING_DATASOURCE_URL` | URL JDBC de PostgreSQL |
| `spring.datasource.username` | `SPRING_DATASOURCE_USERNAME` | Usuario de la base de datos |
| `spring.datasource.password` | `SPRING_DATASOURCE_PASSWORD` | Contraseña de la base de datos |
| `time.token` | `TIME_TOKEN` | Duración del token de sesión en ms (3600000 = 1 h) |
| `encryption.key` | `ENCRYPTION_KEY` | Clave AES (16 caracteres) |
| `encryption.initVector` | `ENCRYPTION_INITVECTOR` | Vector de inicialización AES (16 caracteres) |
| `encryption.algorithm` | `ENCRYPTION_ALGORITHM` | Algoritmo (`AES/CBC/PKCS5PADDING`) |
| `email.sender` | `EMAIL_SENDER` | Correo remitente |
| `email.sender.pswd` | `EMAIL_SENDER_PSWD` | Contraseña del remitente (en Gmail, una contraseña de aplicación) |
| `paypal.client-id` | `PAYPAL_CLIENT_ID` | Client ID de PayPal |
| `paypal.client-secret` | `PAYPAL_CLIENT_SECRET` | Client Secret de PayPal |
| `paypal.base-url` | `PAYPAL_BASE_URL` | `https://api-m.sandbox.paypal.com` en pruebas |
| `spring.data.rest.basePath` | — | Ruta base de Spring Data REST (`/data-api`) |
| `spring.servlet.multipart.max-file-size` | — | Tamaño máximo de archivo (10 MB) |

> ⚠️ **Nunca subas credenciales reales** (base de datos, correo, PayPal, claves de cifrado) a un repositorio público. Usa variables de entorno o un archivo local ignorado por Git.

Reglas de validación del registro (definidas en `messages.properties`): nombre de 3 a 15 caracteres, primer apellido obligatorio (máx. 20), email de `gmail.com`, teléfono de 9 dígitos, dirección obligatoria y contraseña de hasta 24 caracteres con mayúsculas, minúsculas, número y carácter especial.

## Documentación de la API

Con la aplicación en marcha:

| Recurso | URL |
|---|---|
| Swagger UI | <http://localhost:8080/swagger-ui.html> |
| OpenAPI (JSON) | <http://localhost:8080/v3/api-docs> |
| Spring Data REST | <http://localhost:8080/data-api> |

> Con la configuración de seguridad actual, estas rutas requieren autenticación (`anyRequest().authenticated()`). Si quieres consultar Swagger sin token, añade `/swagger-ui/**` y `/v3/api-docs/**` a `permitAll()` en `WebSecurityConfig`.

## Pruebas

```bash
./mvnw test
```

El proyecto incluye `ProyectoServidorApplicationTests` y pruebas específicas de pedidos, proveedores, roles y totales por usuario. Se ejecutan con JUnit 5 y Spring Boot Test, por lo que necesitan acceso a la base de datos configurada.

## Docker

El `Dockerfile` usa un build multi-stage (Maven + Amazon Corretto 21) y expone el puerto **8080**.

```bash
docker build -t kristyco-backend .

docker run -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://HOST:5432/kristyco_studio \
  -e SPRING_DATASOURCE_USERNAME=usuario \
  -e SPRING_DATASOURCE_PASSWORD=contraseña \
  -e ENCRYPTION_KEY=clave16caracteres \
  -e ENCRYPTION_INITVECTOR=vector16caracter \
  -e EMAIL_SENDER=correo@ejemplo.com \
  -e EMAIL_SENDER_PSWD=contraseña_app \
  -e PAYPAL_CLIENT_ID=... \
  -e PAYPAL_CLIENT_SECRET=... \
  kristyco-backend
```

## Despliegue

- **API:** desplegada en [Render](https://render.com) (`kristyco-studio-proyect.onrender.com`).
- **Base de datos:** PostgreSQL gestionado en Render.
- **Frontend:** Vercel, repositorio [TFG_Frontend](https://github.com/Daniel0611CN/TFG_Frontend).

> En el plan gratuito de Render el servicio puede "dormirse" tras un periodo de inactividad, por lo que la primera petición puede tardar unos segundos.

## Autor

**Daniel** — [@Daniel0611CN](https://github.com/Daniel0611CN)

Proyecto realizado como TFG del Ciclo Formativo de Grado Superior en Desarrollo de Aplicaciones Web.
