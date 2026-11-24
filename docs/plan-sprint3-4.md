# Routopia — Plan de arquitectura y tareas: Sprint 3 (desde US#28) y Sprint 4

Análisis de brechas entre lo pedido en `Sprint 3.pptx` (diapositivas 16-20) y `Sprint 4.pptx` (completo) contra el estado real del código a día de hoy. No incluye implementación, solo diagnóstico, diseño propuesto y lista de tareas.

Repos: `routopia_back` (Spring Boot) y `tour_booking` (React/Vite). Ver `/CLAUDE.md` en la raíz para convenciones generales.

---

## 0. Hallazgo transversal que bloquea varias historias

**El** `id` **del usuario logueado nunca llega al frontend.**

- `AuthResponse` (`routopia_back/src/main/java/com/back/routopia/dto/AuthResponse.java`) no tiene campo `id`. Ni `/api/auth/login`, ni `/api/auth/register`, ni `/api/auth/me` lo devuelven.
- Como consecuencia, `AuthContext.jsx` (frontend) nunca guarda `user.id`.
- Eso obligó a un workaround: `useTourBookingForm.jsx:91` manda `userId: 1` **hardcodeado**, con el comentario `// TODO: Cambiar por el id del usuario logueado`.
- `BookingService.createBooking()` (backend) confía ciegamente en el `userId` que manda el cliente — no lo valida contra el usuario autenticado (a diferencia de `ReviewController`, que sí usa `Authentication` para resolver el usuario real).

**Por qué importa:** ahora mismo, en el código actual, **toda reserva que se cree queda asociada al usuario #1**, sin importar quién esté logueado. Esto rompe en cascada:

- US#28 (puntuar producto): el chequeo "reserva finalizada" (`ReviewService.hasFinishedBooking`) nunca va a dar `true` para nadie que no sea el usuario #1, aunque la lógica esté bien escrita.
- US#33 (historial de reservas, Sprint 4): un historial "por usuario" es imposible de construir correctamente si todas las reservas pertenecen al usuario #1.
- US#32 (realizar reserva): la reserva que ve el usuario como "confirmada" no es realmente suya en la base de datos.

**Esto no es parte de ninguna user story puntual — es un defecto de la base sobre la que se apoyan tres historias.** Recomiendo resolverlo primero, antes de tocar cualquiera de las historias de abajo que dependen de "quién reservó qué".

### Diseño propuesto

1. Agregar `id` (Long) a `AuthResponse` y poblarlo en los tres métodos de `AuthController` (`register`, `login`, `me`).
2. Frontend: `AuthContext.jsx` guarda `id` en el objeto `user` (login, register y en la carga desde `/auth/me`).
3. `useTourBookingForm.jsx`: reemplazar `userId: 1` por `user.id` desde `useAuth()`.
4. Backend, opcional pero recomendable: `BookingService.createBooking` debería tomar el usuario desde `Authentication` (como ya hace `ReviewController.requireUser`) en vez de confiar en `request.getUserId()`. Así se elimina la clase de bug por completo (nadie puede crear una reserva "a nombre de otro" manipulando el body) y el DTO ya no necesita `userId` como input del cliente.

---

## 1. Sprint 3 — US#28: Puntuar producto (prioridad Baja)

### Estado actual

El código está **completo** en ambos repos, pero sin commitear (`git status` muestra todos los archivos de reviews como `??` o modificados). Ya lo revisamos en detalle en esta conversación:

- Backend: `Review` entity, `ReviewController`, `ReviewService`, `ReviewRepository`, DTOs — cumplen los 6 criterios de aceptación (estrellas 1-5, nombre + fecha + comentario, promedio recalculado en la misma transacción, `reviewCount` expuesto en `DestinoDTO`).
- Frontend: `StarRating.jsx`, `TourReviews.jsx`, `useProductReviews.jsx` — sección de reseñas en `TourDetail.jsx`, promedio visible también en `RecomendationCard.jsx` (usado en Home, Category, Favorites y SearchResults).

### La única condición que no se puede probar hoy

`ReviewService.hasFinishedBooking()` exige `Booking.status == "CONFIRMED"` y `bookingDate <= hoy`. La lógica en sí es razonable — no hace falta un flujo extra de "finalizar reserva", alcanza con que la fecha de la reserva ya haya pasado. El problema real es el del punto 0: como todas las reservas se crean a nombre del usuario #1, ningún otro usuario puede probar el flujo de reseñas de punta a punta hoy.

### Tu pregunta: ¿sacar la condición de "reserva finalizada"?

Mi recomendación es **no sacarla**. Es un criterio de aceptación explícito ("Los usuarios que hayan finalizado una reserva, pueden puntuar un producto") — quitarlo haría que la historia no cumpla lo pedido. La solución no es debilitar la regla, es:

1. Resolver el punto 0 (userId real), y
2. Para poder demostrar el flujo sin esperar a que pase una fecha real, sembrar en `DataInitializer.java` una reserva de ejemplo con `bookingDate` en el pasado para el usuario demo (`user@routopia.com`) sobre algún destino ya sembrado. Es dato de demo, no un flujo nuevo de producto.

### Tareas

- [x] Commitear el trabajo ya hecho de reviews (backend + frontend) — está terminado, solo falta guardarlo.
- [x] Resolver el hallazgo transversal (punto 0).
- [x] Sembrar en `DataInitializer` una reserva `CONFIRMED` con fecha pasada para el usuario demo, para poder mostrar el flujo de reseñas sin esperar.
- [x] Reiniciar el backend local (el proceso corriendo tiene bytecode viejo, sin la regla nueva de `SecurityConfig` para `/api/reviews/**`) y probar el flujo end-to-end.

---

## 2. Sprint 3 — US#29: Eliminar categoría (prioridad Baja)

### Estado actual

- Backend: **completo**. `DELETE /api/categories/{id}` existe (`CategoryController.delete_category`) y `deleteCategory()` ya está en `category.service.js` del frontend — pero no se usa en ningún lado.
- Frontend: **no implementado**. `CategoriesForm.jsx` no tiene botón de eliminar, `useCategoriesForm.jsx` no tiene lógica de borrado, y `CategoriesList.jsx` tampoco. El componente `ModalConfirm.jsx` ya existe y ya se usa para borrar destinos en `Admin.jsx` — es el patrón a reutilizar.

Nota: `Traits` sí tiene botón de eliminar (`TraitsForm.jsx` → `handleDeleteTrait`), pero **sin modal de confirmación** (llama al borrado directo). Como referencia de patrón a copiar, usar `Admin.jsx` (destinos), no `TraitsForm` — es el único lugar que ya cumple el criterio de "alerta con Confirmar/Cancelar".

### Diseño propuesto

Replicar exactamente el patrón de `Admin.jsx`:

1. `useCategoriesForm.jsx` (o el hook contenedor `useCategoriesContainer.jsx`): agregar estado `showDeleteModal` + `handleDeleteCategory` que llame a `deleteCategory(id)` (ya existe en el service) y luego dispare `onDeleteCategory`/`onSaved` para refrescar la lista.
2. `CategoriesForm.jsx`: agregar botón "Eliminar categoría" (visible solo si `selectedCategory` existe, igual que en `TraitsForm`) que abra el `ModalConfirm`.
3. Mensaje del modal: debe indicar el nombre de la categoría y advertir sobre los destinos asociados (el criterio de aceptación lo pide explícitamente — "cuáles podrían ser las consecuencias, por ejemplo la eliminación de todos los productos asociados").
4. **Punto a decidir antes de implementar** (ver nota en la diapositiva: _"¿Comportamiento ante usuarios no registrados?"_): hoy `Destino.category` es `@ManyToOne` sin `orphanRemoval` ni `CascadeType`. Hay que decidir qué pasa con los destinos que usan esa categoría al borrarla — ¿se bloquea el borrado si tiene destinos asociados, o se borran en cascada, o se les pone categoría null? Esto es una decisión de negocio, no solo técnica, y falta definirla.

### Tareas

- [x] Backend: decidir y ajustar la relación `Destino → Category` para el caso de borrado con destinos asociados (bloquear con 409, o permitir con reasignación).
- [x] Frontend: wire de `deleteCategory` + `ModalConfirm` en el flujo de categorías, siguiendo el patrón de `Admin.jsx`.

---

## 3. Sprint 4 — completo (Meta del sprint: "Desarrollar la funcionalidad de Reserva de Productos")

### Estado actual general

Existe un flujo de reserva **básico e inline**: `TourDescription.jsx` (dentro de `TourDetail.jsx`) tiene un formulario embebido con date picker (`react-datepicker`, ya excluye fechas ocupadas vía `GET /api/bookings/availability`) y cantidad de personas, que al enviar llama a `POST /api/bookings` y muestra un toast de éxito/error. Esto cubre una fracción de lo pedido en el sprint, pero como flujo de una sola pantalla — no como las 3 pantallas/pasos que describen las historias.

### US#30 — Reservas: Seleccionar fecha (Alta)

**Pedido:** acceder a la reserva desde el detalle del producto → chequear login al hacer clic en "Reservar" → si no está logueado, redirigir a login con un aviso explícito de que hace falta registrarse/loguearse → si está logueado, ir a una página de reserva → buscar disponibilidad → seleccionar rango de fechas → fecha se refleja en el formulario.

**Estado real:**

- ✅ Hay selector de fecha con exclusión de fechas ocupadas.
- ⚠️ No hay chequeo de login _específico del botón "Reservar"_ — hoy toda la página `/tour/:id` está detrás de `ProtectedRoute`, así que un usuario no logueado ni siquiera puede ver el detalle del producto (criterio pide que sí pueda verlo, y que el gate sea sobre la acción de reservar).
- ❌ No hay mensaje de login diciendo "necesitás iniciar sesión para reservar" — `Login.jsx` no soporta ningún mensaje contextual.
- ❌ No es una "página de reserva" separada — es un formulario embebido en el detalle.
- ❌ Es fecha única (`bookingDate`), no un **rango** de fechas como pide el criterio ("seleccionar un rango de fechas para la reserva").

### US#31 — Reservas: Visualizar detalles (Alta)

**Pedido:** una página de reserva que muestre datos del producto (título, ubicación, imágenes, descripción, destacados) + datos del usuario que reserva (nombre, apellido, email, ampliable) + el rango de fechas elegido + botón submit.

**Estado real:** ❌ No existe. No hay una pantalla intermedia de revisión antes de confirmar — el submit del formulario embebido en `TourDescription.jsx` crea la reserva directamente.

### US#32 — Realizar reserva (Alta)

**Pedido:** seleccionar producto, elegir rango de fechas, ver datos no obligatorios del formulario, y tras reservar mostrar una **página de confirmación de éxito**; en caso de error, mensaje específico.

**Estado real:**

- ✅ Ya se puede crear la reserva (`POST /api/bookings`) y ya hay manejo de error con mensaje (`notify` con `error.response?.data?.message`).
- ❌ No hay página de confirmación — solo un toast que desaparece.
- ⚠️ Depende del hallazgo del punto 0 (userId hardcodeado).

### US#33 — Acceder a historial (Media)

**Pedido:** sección donde el usuario ve sus reservas anteriores, ordenadas, con nombre del producto, fecha de reserva y fecha de uso.

**Estado real:** ❌ No existe ni en backend ni en frontend.

- Backend: `BookingRepository` no tiene ningún método `findByUserId(...)`; no hay endpoint `GET /api/bookings/me` ni similar.
- Frontend: `Profile.jsx` solo muestra nombre/email/logout — no hay lista de reservas.

### US#34 — WhatsApp: Iniciar chat (Baja)

**Pedido:** botón/ícono flotante fijo (abajo a la derecha, en toda resolución), sin necesidad de estar logueado, que abra un chat de WhatsApp con el proveedor; confirmación de envío; manejo de error si el número es inválido o no hay conexión.

**Estado real:** ❌ No existe como tal. Lo único relacionado es la opción "compartir por WhatsApp" en `ShareProductModal.jsx` (comparte el link del producto en redes, incluyendo WhatsApp) — es una funcionalidad distinta (compartir vs. contactar al proveedor) y no es un botón flotante global.

Nota de la diapositiva (comentario de la profesora/DB): _"¿Otro panel de backoffice o dentro del admin de propiedades?"_ — falta definir de dónde sale el número de WhatsApp del proveedor (¿es fijo/global de Routopia, o por destino/proveedor? Hoy `Destino` no tiene ningún campo de contacto).

### US#35 — Notificación: Confirmar reserva por correo (Baja)

**Pedido:** tras confirmar una reserva, enviar un correo automático al usuario con los datos de la reserva (producto, fecha/hora, contacto del proveedor), dentro de un plazo razonable (~24hs), con diseño responsive.

**Estado real:** ❌ No existe ninguna infraestructura de correo. `pom.xml` no tiene `spring-boot-starter-mail` ni ninguna dependencia de envío de emails. No hay `EmailService` ni configuración SMTP en `application.properties`/`application.properties.example`.

---

## 4. Diseño propuesto para Sprint 4

### Backend

1. **Rango de fechas en reservas**: hoy `Booking` tiene una sola `bookingDate` (con constraint único `destino_id + booking_date`, pensado para una reserva = un día). El criterio de US#30 pide poder elegir un _rango_. Hay que decidir: ¿una reserva ocupa varios días consecutivos (agregar `endDate` a `Booking` y ajustar la lógica de disponibilidad para bloquear todo el rango), o "rango" en el mockup se refiere solo a la UI del calendario pero la reserva sigue siendo de un día? Esto cambia el modelo de datos — conviene revisar el prototipo de la diapositiva 17 (Sprint 3) y el de Sprint 4 antes de tocar la entidad.
2. **Historial de reservas** (US#33): agregar `findByUserIdOrderByBookingDateDesc(Long userId)` a `BookingRepository`, y un endpoint `GET /api/bookings/me` en `BookingController` que resuelva el usuario desde `Authentication` (mismo patrón que `ReviewController`) y devuelva `List<BookingDTO>` — conviene enriquecer `BookingDTO` con el nombre/imagen del destino para no obligar al frontend a hacer un fetch por reserva.
3. **Email de confirmación** (US#35): agregar `spring-boot-starter-mail`, configurar SMTP (dev: se puede usar Mailtrap/Ethereal o similar para no depender de credenciales reales de un proveedor), crear `EmailService` con un método `sendBookingConfirmation(Booking booking)`, invocarlo desde `BookingService.createBooking()` al final (idealmente de forma asíncrona con `@Async`, para no bloquear la respuesta HTTP si el envío de correo tarda o falla).
4. **Datos de contacto del proveedor** (para US#34 y US#35): definir si es un dato global (una sola config `app.whatsapp-number` / `app.provider-email`) o por destino (agregar campo a `Destino`). Recomiendo empezar global y simple, ya que la historia es de prioridad Baja y no hay entidad "proveedor" en el modelo actual.

### Frontend

1. **Gate de login solo en la acción de reservar** (US#30): sacar `/tour/:id` de `ProtectedRoute` (debe ser visible sin login, según el criterio) y en su lugar, en `TourDescription.jsx`, chequear `user` de `useAuth()` al hacer clic en "Reservar": si no hay usuario, `navigate("/login", { state: { message: "Iniciá sesión para reservar este producto" } })`, y que `Login.jsx` lea ese `state` y lo muestre arriba del formulario.
2. **Página de reserva separada** (US#31): nueva ruta, ej. `/tour/:id/reservar`, con un nuevo componente que muestre resumen del producto + datos del usuario logueado (ya disponibles en `AuthContext`) + selector de rango de fechas + botón submit. El formulario embebido actual en `TourDescription.jsx` pasaría a ser solo el punto de entrada ("Reservar" → navega a esta página), no el formulario completo.
3. **Página de confirmación** (US#32): tras el submit exitoso, `navigate('/tour/:id/reserva-confirmada', { state: { booking } })` (o similar) en vez de solo mostrar un toast.
4. **Historial de reservas** (US#33): nueva ruta `/mis-reservas` (o sección dentro de `Profile.jsx`), hook `useBookingHistory.jsx` que llame al nuevo `GET /api/bookings/me`, listado ordenado por fecha con nombre de producto + fecha de reserva + fecha de uso.
5. **Botón flotante de WhatsApp** (US#34): componente `WhatsAppFloatButton.jsx` montado a nivel de `App.jsx` (visible en toda la app, no depende de login), `position: fixed; bottom; right`, con `href="https://wa.me/<numero>?text=..."`, `target="_blank"`. No requiere backend.

---

## 5. Backlog priorizado (orden sugerido)

1. **Fix transversal de** `userId` (punto 0) — desbloquea US#28, US#32, US#33. Es chico y de bajo riesgo.
2. **Commitear + probar US#28** (reviews) — el trabajo ya está hecho, solo falta cerrarlo.
3. **US#29** (eliminar categoría) — chica, backend ya listo, solo falta frontend + decisión de negocio sobre destinos huérfanos.
4. **US#33** (historial) — depende del fix de punto 0, es la más chica de Sprint 4 y no requiere decisiones de modelo de datos nuevas (a diferencia de US#30/31 con el tema del rango de fechas).
5. **US#30 + US#31 + US#32** (flujo de reserva completo) — conviene tratarlas juntas porque son un solo flujo de 3 pantallas; primero definir si el rango de fechas cambia el modelo de `Booking`.
6. **US#34** (WhatsApp) — independiente, se puede hacer en paralelo en cualquier momento, es solo frontend.
7. **US#35** (email) — depende de tener credenciales/servicio SMTP de prueba; es la única que requiere una decisión de infraestructura externa antes de arrancar.

## 6. Decisiones que te faltan tomar (no son técnicas, son de producto)

- ¿Qué pasa con los destinos al borrar su categoría? (US#29)
- ¿Una reserva es de un día puntual o de un rango de días consecutivos? Cambia el modelo de `Booking` y la lógica de disponibilidad. (US#30/31/32)
- ¿El contacto de WhatsApp es un número único de Routopia, o por destino/proveedor? (US#34)
- ¿Con qué proveedor de correo vas a probar el envío en desarrollo? (US#35)
