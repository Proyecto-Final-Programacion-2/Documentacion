# Plan de casos de uso, tareas e issues --- Proyecto Integrador 2026

**Repositorios:** `app-kmp`, `catalogo`, `turnos`, `documentacion`\
**Objetivo:** convertir el enunciado de Programación 2 y el contrato de
integración en trabajo implementable y verificable.

> El enunciado describe tres repositorios de código (KMP y dos
> backends). Se contempla `documentacion` como cuarto repositorio
> separado porque ya lo tenés creado. Los issues deben vivir
> preferentemente en el repositorio dueño de cada cambio; en
> `documentacion` se registran tareas documentales y decisiones
> transversales.

## 1. Arquitectura y límites de responsabilidad

-   **KMP:** interfaz Android, registro/login de usuario final,
    búsqueda, filtros, selección de turno, ingreso de teléfono y
    consulta/cancelación de reservas.
-   **Catálogo:** copia local de categorías, profesionales y agendas;
    sincronización snapshot/incremental; filtros locales; API propia
    para KMP y turnos.
-   **Turnos:** disponibilidad, holds, procesos de reserva, integración
    REST/Kafka, reservas propias por usuario y cancelaciones.
-   **Documentación:** README general, diagramas, contratos internos,
    modelos, decisiones, evidencias y guía de ejecución.

El servicio central de la cátedra no es uno de tus cuatro repositorios y
no debe implementarse. Se consume mediante REST, Redis y Kafka.

## 2. Casos de uso

### CU-01 --- Registrar usuario final

**Actor:** visitante.\
**Repositorio principal:** `app-kmp` + backend que gestione identidad.\
**Flujo:** completa login, contraseña, nombre, apellido, email y
opcionalmente imageUrl; KMP envía los datos; backend valida, almacena
contraseña con hashing y activa la cuenta.\
**Alternativas:** datos inválidos o login/email duplicados; informar
error sin crear cuenta.\
**Aceptación:** el usuario puede iniciar sesión luego del registro y la
contraseña nunca se almacena ni devuelve en texto plano.

### CU-02 --- Iniciar sesión

**Actor:** usuario registrado.\
**Flujo:** ingresa credenciales; backend autentica y emite JWT; KMP
conserva la sesión de forma segura y lo envía en solicitudes
protegidas.\
**Alternativas:** credenciales incorrectas, token vencido o backend no
disponible.\
**Aceptación:** APIs protegidas rechazan solicitudes anónimas y KMP
presenta errores comprensibles.

### CU-03 --- Configurar integración técnica con cátedra

**Actor:** desarrollador/alumno (operación inicial).\
**Repositorio:** `catalogo`, `turnos`, `documentacion`.\
**Flujo:** registrar una única cuenta técnica con Postman; obtener JWT e
integración; configurar ambos backends con los mismos valores mediante
variables/secretos.\
**Aceptación:** credenciales no están en Git, logs, documentación
pública ni KMP. `groupId` identifica la integración técnica, no al
usuario final.

### CU-04 --- Sincronizar catálogo completo

**Actor:** proceso de sincronización.\
**Repositorio:** `catalogo`.\
**Flujo:** si no hay copia local o se requiere reconstrucción, obtiene
`GET /api/synchronization/snapshot`; aplica categorías, profesionales y
horarios consistentemente; actualiza la versión local solo cuando las
tres colecciones se aplicaron correctamente.\
**Alternativas:** REST falla o transacción no completa; no avanzar
versión.\
**Aceptación:** snapshot inicial persistido y versión correspondiente
consistente con los datos.

### CU-05 --- Sincronizar catálogo incremental

**Actor:** consumidor Kafka.\
**Repositorio:** `catalogo`.\
**Flujo:** recibe `CatalogUpdated`; deduplica `eventId`; consulta
metadata/versiones Redis; aplica cada versión en orden leyendo IDs
afectados y entidades vigentes; avanza versión solo tras aplicar
correctamente.\
**Alternativas:** notificación duplicada, cambio ya aplicado, versión
perdida o fuera de ventana.\
**Aceptación:** duplicados no repiten efectos y la versión local nunca
avanza antes de persistir los cambios.

### CU-06 --- Reconstruir catálogo ante discontinuidad

**Actor:** proceso de sincronización.\
**Repositorio:** `catalogo`.\
**Flujo:** detecta que la versión local es menor que
`oldestAvailableVersion`, que falta continuidad o que no puede verificar
el estado; obtiene snapshot y reemplaza la copia consistentemente.\
**Aceptación:** la versión y las tres colecciones quedan alineadas; una
falla no deja una versión nueva con datos parciales.

### CU-07 --- Buscar y filtrar profesionales

**Actor:** usuario autenticado.\
**Repositorios:** `catalogo`, `app-kmp`.\
**Flujo:** KMP solicita búsqueda al servicio de catálogo; se filtra
usando exclusivamente la copia local por categoría, nombre, habilitado y
disponibilidad.\
**Aceptación:** las búsquedas no consultan al servicio central por cada
interacción; se contemplan resultados vacíos y errores.

### CU-08 --- Consultar disponibilidad

**Actor:** usuario autenticado.\
**Repositorios:** `turnos`, `catalogo`, `app-kmp`.\
**Flujo:** usuario elige profesional y fecha; turnos obtiene agenda
vigente del catálogo y ocupaciones REST; combina horarios semanales y
ocupaciones `CONFIRMED`/`HELD`; presenta slots libres.\
**Alternativas:** profesional deshabilitado, fecha inválida,
catálogo/servicio central no disponible.\
**Aceptación:** no se ofrece un slot ocupado ni fuera de agenda; si
catálogo no está disponible, no se inicia una operación que requiera
validar vigencia.

### CU-09 --- Iniciar reserva y crear hold

**Actor:** usuario autenticado.\
**Repositorio:** `turnos`.\
**Flujo:** KMP envía selección; turnos verifica disponibilidad y crea
`POST /api/appointment-holds` con `externalPatientId` estable del
usuario; persiste `holdId`, `reservationProcessId`, vencimiento y
estado.\
**Alternativas:** slot reservado/held, inválido, hold vencido o
timeout.\
**Aceptación:** proceso local queda asociado al usuario autenticado; no
se confía en un ID de usuario enviado libremente por KMP.

### CU-10 --- Confirmar inicialmente hold

**Actor:** usuario autenticado/proceso de turnos.\
**Repositorio:** `turnos`.\
**Flujo:** turnos invoca `POST /api/appointment-holds/{holdId}/confirm`
con `externalPatientId` y nombre/apellido; persiste
`WAITING_FOR_PHONE`.\
**Aceptación:** se conserva el `expiresAt`; un hold local no extiende el
TTL externo.

### CU-11 --- Solicitar y enviar teléfono

**Actor:** usuario autenticado.\
**Repositorio:** `turnos` + `app-kmp`.\
**Flujo:** consumidor recibe `AdditionalInformationRequested`; registra
`eventId` como `requestEventId`; KMP solicita teléfono; backend publica
`AdditionalInformationSubmitted` en el topic asignado, con
`reservationProcessId` como message key.\
**Alternativas:** teléfono inválido; permitir reintento mientras no
venza el proceso.\
**Aceptación:** payload cumple contrato, `schemaVersion=1`, `eventType`
correcto e idempotencia por `eventId`.

### CU-12 --- Procesar resultado de reserva

**Actor:** consumidor Kafka.\
**Repositorio:** `turnos`.\
**Flujo:** procesa `AppointmentConfirmed`,
`AdditionalInformationRejected`, `AppointmentProcessExpired`,
`AppointmentProcessInvalid` y `AppointmentCancelled`; actualiza estado
local sin retroceder estados finales.\
**Aceptación:** mensajes duplicados no repiten efectos; rechazo permite
reintento válido hasta `expiresAt`; expiración/invalidez cierra el
proceso.

### CU-13 --- Consultar reservas propias

**Actor:** usuario autenticado.\
**Repositorios:** `turnos`, `app-kmp`.\
**Flujo:** KMP solicita sus reservas; turnos consulta REST con
`externalPatientId` y además aplica asociación local por usuario
autenticado.\
**Aceptación:** un usuario no puede ver reservas de otro aunque manipule
parámetros o IDs.

### CU-14 --- Cancelar reserva propia

**Actor:** usuario autenticado.\
**Repositorios:** `turnos`, `app-kmp`.\
**Flujo:** usuario solicita cancelar; backend verifica propiedad y
estado confirmado; invoca endpoint REST de cancelación; procesa
resultado/evento y persiste estado.\
**Aceptación:** solo el dueño puede cancelar; cancelar nuevamente no
genera efectos duplicados; estados no cancelables se informan.

### CU-15 --- Recuperarse de fallos

**Actor:** sistema.\
**Repositorios:** ambos backends.\
**Flujo:** ante timeout, caída, reinicio, evento duplicado o tardío,
aplica reintentos limitados y reconciliación REST cuando corresponda.\
**Aceptación:** no hay ciclos ilimitados, duplicación funcional ni
retroceso desde estado final; procesos pendientes se recuperan después
de reiniciar.

## 3. Milestones sugeridos

  -----------------------------------------------------------------------
  Milestone               Fecha objetivo          Alcance
  ----------------------- ----------------------- -----------------------
  M1 --- Base,            08/11                   Infraestructura,
  arquitectura y                                  contratos internos,
  seguridad                                       usuarios, JWT, KMP
                                                  login.

  M2 --- Catálogo         15/11                   Snapshot, Redis, Kafka
  sincronizado                                    incremental,
                                                  recuperación, filtros
                                                  locales.

  M3 --- Reserva          22/11                   Disponibilidad, hold,
  end-to-end                                      confirmación, teléfono
                                                  y resultado Kafka.

  M4 --- Funcionalidad    24/11                   Mis reservas,
  completa                                        cancelación,
                                                  integración KMP
                                                  completa.

  M5 --- Calidad y        29/11                   Robustez, pruebas,
  entrega                                         documentación,
                                                  evidencias y ensayo.
  -----------------------------------------------------------------------

## 4. Issues para GitHub

Copiá cada bloque como issue en el repositorio indicado. Los IDs son
etiquetas de referencia para ordenar el trabajo; GitHub asignará sus
propios números.

### Repositorio `catalogo`

#### CAT-01 --- \[Infra\] Configurar persistencia y migraciones del catálogo

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M1\
**Tareas** - \[ \] Definir entidades categoría, profesional, agenda
semanal y versión local. - \[ \] Configurar base de datos servidor y
migraciones propias. - \[ \] Configurar Docker Compose y variables
externas. - \[ \] Verificar que no se comparten tablas ni migraciones
con turnos.

**Aceptación:** servicio arranca en entorno limpio y persiste sus datos
en su propia base/esquema.

#### CAT-02 --- \[Integración\] Cliente REST y configuración técnica de cátedra

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M1\
**Tareas** - \[ \] Cargar URL y JWT técnico desde configuración
segura. - \[ \] Implementar cliente autenticado para snapshot y
ocupaciones si corresponde a este servicio. - \[ \] Definir timeout y
traducción de errores. - \[ \] Evitar registrar tokens o contraseñas en
logs.

**Aceptación:** puede invocar endpoints protegidos con Bearer token sin
secretos hardcodeados.

#### CAT-03 --- \[Sync\] Implementar snapshot completo

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M2\
**Tareas** - \[ \] Consumir `GET /api/synchronization/snapshot`. - \[ \]
Mapear las tres colecciones. - \[ \] Aplicar cambios
transaccionalmente/consistentemente. - \[ \] Guardar `snapshotVersion`
al final, nunca antes.

**Aceptación:** snapshot inicial y reconstrucción dejan las colecciones
y versión alineadas.

#### CAT-04 --- \[Redis\] Leer metadata, versiones y entidades

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M2\
**Tareas** - \[ \] Configurar ACL con los valores entregados por la
integración. - \[ \] Leer `catedra:sync:metadata` y versiones. - \[ \]
Leer `catedra:sync:changes:{version}`. - \[ \] Leer entidades afectadas
desde los Hash correspondientes. - \[ \] No acceder a `catedra:holds:*`.

**Aceptación:** puede reconstruir los cambios de una versión respetando
namespaces y permisos.

#### CAT-05 --- \[Kafka\] Consumir CatalogUpdated e implementar incremental

**Prioridad:** P0 · **Estimación:** 5 h · **Milestone:** M2\
**Tareas** - \[ \] Suscribirse al topic recibido en `integration`. - \[
\] Validar `schemaVersion` y `eventType`. - \[ \] Deduplicar por
`eventId`. - \[ \] Aplicar versiones en orden y avanzar versión solo
tras persistencia exitosa. - \[ \] Confirmar offset después de
persistir.

**Aceptación:** una notificación duplicada no repite efectos y los
cambios se aplican ordenados.

#### CAT-06 --- \[Recovery\] Recuperar catálogo con snapshot ante discontinuidad

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M2\
**Tareas** - \[ \] Comparar versión local, `currentVersion` y
`oldestAvailableVersion`. - \[ \] Detectar historial faltante/no
verificable. - \[ \] Ejecutar snapshot y reemplazo consistente. - \[ \]
Probar pérdida de versiones y reinicio.

**Aceptación:** ante discontinuidad no intenta aplicar un historial
incompleto.

#### CAT-07 --- \[API\] Exponer consultas y filtros locales

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M2\
**Tareas** - \[ \] Definir DTOs y endpoints internos. - \[ \] Filtrar
por categoría, nombre, habilitado y disponibilidad. - \[ \] Proteger
endpoints y validar parámetros. - \[ \] No consultar al servicio central
por cada búsqueda.

**Aceptación:** filtros se resuelven sobre la copia local y son
consumibles desde KMP y turnos.

#### CAT-08 --- \[Test\] Pruebas de sincronización y búsqueda

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M5\
**Tareas** - \[ \] Probar snapshot. - \[ \] Probar incremental y evento
duplicado. - \[ \] Probar discontinuidad y recuperación. - \[ \] Probar
filtros y resultados vacíos.

**Aceptación:** pruebas automatizadas reproducibles para los escenarios
principales.

### Repositorio `turnos`

#### TUR-01 --- \[Infra\] Modelo de datos y persistencia propia

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M1\
**Tareas** - \[ \] Modelar proceso, reserva, estado, IDs externos y
vencimiento. - \[ \] Asociar cada proceso/reserva al usuario final
autenticado. - \[ \] Configurar base de datos servidor y migraciones
independientes. - \[ \] Preparar Docker Compose.

**Aceptación:** no depende de tablas/repositorios de catálogo y puede
identificar al dueño de cada proceso.

#### TUR-02 --- \[Security\] JWT y autorización por propietario

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M1\
**Tareas** - \[ \] Proteger endpoints. - \[ \] Validar JWT entre
servicios según el diseño elegido. - \[ \] Obtener identidad desde el
contexto autenticado, no desde un ID libre del cliente. - \[ \] Rechazar
lectura/cancelación de recursos ajenos.

**Aceptación:** pruebas demuestran aislamiento entre dos usuarios.

#### TUR-03 --- \[Integration\] Cliente del servicio de catálogo

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M1\
**Tareas** - \[ \] Definir contrato explícito con catálogo. - \[ \]
Consultar profesional y agenda vigente. - \[ \] Autenticar comunicación
entre servicios. - \[ \] Manejar timeout, 4xx y 5xx.

**Aceptación:** turnos obtiene datos vigentes sin leer la base de
catálogo.

#### TUR-04 --- \[Availability\] Calcular slots disponibles

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M3\
**Tareas** - \[ \] Generar slots desde horario semanal y duración. - \[
\] Consultar `GET /api/appointment-occupancies`. - \[ \] Excluir slots
`CONFIRMED` y `HELD`. - \[ \] Validar fechas, profesional habilitado y
rango de consulta.

**Aceptación:** solo se ofrecen slots habilitados y no ocupados.

#### TUR-05 --- \[Reservation\] Crear hold y confirmar inicialmente

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M3\
**Tareas** - \[ \] Crear proceso local asociado al usuario. - \[ \]
Invocar `POST /api/appointment-holds`. - \[ \] Persistir `holdId`,
`reservationProcessId`, `expiresAt`. - \[ \] Invocar endpoint de
confirmación inicial. - \[ \] Persistir estado `WAITING_FOR_PHONE`.

**Aceptación:** proceso queda recuperable y respeta vencimiento externo.

#### TUR-06 --- \[Kafka\] Consumir solicitud de teléfono

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M3\
**Tareas** - \[ \] Consumir `AdditionalInformationRequested`. - \[ \]
Validar campos y `schemaVersion`. - \[ \] Guardar `eventId` para usarlo
como `requestEventId`. - \[ \] Asociar evento al proceso correcto.

**Aceptación:** solicitud queda persistida y disponible para KMP sin
duplicar efectos.

#### TUR-07 --- \[Kafka\] Publicar teléfono y procesar resultado

**Prioridad:** P0 · **Estimación:** 5 h · **Milestone:** M3\
**Tareas** - \[ \] Validar y normalizar teléfono. - \[ \] Publicar
`AdditionalInformationSubmitted` con todos los campos obligatorios. - \[
\] Usar `reservationProcessId` como message key. - \[ \] Procesar
confirmación, rechazo, expiración e invalidez. - \[ \] Deduplicar por
`eventId`.

**Aceptación:** se completa el flujo asincrónico y el estado local
converge correctamente.

#### TUR-08 --- \[Reservations\] Consultar reservas propias

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M4\
**Tareas** - \[ \] Consultar `GET /api/appointments` con
`externalPatientId`. - \[ \] Aplicar también la asociación local con el
usuario autenticado. - \[ \] Mapear paginación y estados. - \[ \] No
exponer reservas de otros usuarios.

**Aceptación:** el usuario solo visualiza sus reservas.

#### TUR-09 --- \[Reservations\] Cancelar reserva propia

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M4\
**Tareas** - \[ \] Verificar propietario y estado. - \[ \] Invocar
endpoint REST de cancelación. - \[ \] Procesar `AppointmentCancelled`. -
\[ \] Hacer operación idempotente y reconciliar respuesta/evento.

**Aceptación:** solo se cancela una reserva propia confirmada;
repetición no duplica efectos.

#### TUR-10 --- \[Recovery\] Recuperación de procesos y reintentos

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M5\
**Tareas** - \[ \] Definir máquina de estados local. - \[ \] Implementar
timeouts y reintentos limitados. - \[ \] Reconciliar por REST cuando un
timeout deje resultado incierto. - \[ \] Recuperar procesos pendientes
al reiniciar. - \[ \] Evitar que eventos tardíos reabran estados
finales.

**Aceptación:** no hay reintentos infinitos ni retrocesos de estado.

#### TUR-11 --- \[Test\] Pruebas de reservas, autorización e idempotencia

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M5\
**Tareas** - \[ \] Probar hold y confirmación inicial. - \[ \] Probar
confirmación, rechazo y expiración. - \[ \] Probar evento duplicado. -
\[ \] Probar consulta/cancelación por propietario. - \[ \] Probar
recuperación de proceso pendiente.

**Aceptación:** escenarios principales automatizados y reproducibles.

### Repositorio `app-kmp`

#### KMP-01 --- \[UI\] Navegación y estructura de pantallas

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M1\
**Tareas** - \[ \] Definir navegación. - \[ \] Crear pantallas de
registro, login, búsqueda, detalle/disponibilidad y mis reservas. - \[
\] Definir estados de carga, vacío y error.

**Aceptación:** navegación Android ejecutable y coherente.

#### KMP-02 --- \[Auth\] Registro e inicio de sesión

**Prioridad:** P0 · **Estimación:** 5 h · **Milestone:** M1\
**Tareas** - \[ \] Implementar formularios y validaciones. - \[ \]
Consumir endpoints del backend. - \[ \] Guardar JWT de usuario final de
forma segura. - \[ \] Adjuntar Bearer token a APIs protegidas. - \[ \]
Manejar credenciales inválidas y sesión no válida.

**Aceptación:** registro e inicio de sesión funcionales desde Android.

#### KMP-03 --- \[Catalog\] Buscar y filtrar profesionales

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M2\
**Tareas** - \[ \] Consumir API propia de catálogo. - \[ \] Agregar
filtros por categoría, nombre, habilitado y disponibilidad. - \[ \]
Mostrar resultados vacíos y errores. - \[ \] Mostrar información
descriptiva del profesional.

**Aceptación:** búsqueda completa desde la interfaz, sin consulta
directa a cátedra.

#### KMP-04 --- \[Appointments\] Ver disponibilidad y seleccionar horario

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M3\
**Tareas** - \[ \] Elegir profesional y fecha. - \[ \] Consultar slots
al backend turnos. - \[ \] Mostrar horarios disponibles/no
disponibles. - \[ \] Permitir seleccionar un slot válido.

**Aceptación:** la interfaz no permite enviar una selección fuera de los
slots presentados.

#### KMP-05 --- \[Appointments\] Completar reserva y teléfono

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M3\
**Tareas** - \[ \] Iniciar reserva. - \[ \] Mostrar estado de espera de
información adicional. - \[ \] Solicitar teléfono. - \[ \] Enviar
teléfono y mostrar resultado final. - \[ \] Mostrar
rechazo/expiración/invalidez.

**Aceptación:** el usuario puede completar el flujo sin operaciones
manuales externas.

#### KMP-06 --- \[Appointments\] Mis reservas y cancelación

**Prioridad:** P0 · **Estimación:** 4 h · **Milestone:** M4\
**Tareas** - \[ \] Listar reservas propias. - \[ \] Mostrar fecha, hora,
profesional y estado. - \[ \] Permitir cancelar las confirmadas. - \[ \]
Actualizar vista luego del resultado.

**Aceptación:** consulta y cancelación funcionales desde Android.

#### KMP-07 --- \[Quality\] Manejo de errores y pruebas de flujo

**Prioridad:** P1 · **Estimación:** 3 h · **Milestone:** M5\
**Tareas** - \[ \] Manejar desconexión y respuestas de error. - \[ \]
Evitar dobles envíos por taps repetidos. - \[ \] Verificar expiración y
estados finales. - \[ \] Probar flujo completo en dispositivo/emulador
Android.

**Aceptación:** los errores se comunican claramente y no dejan la UI en
un estado inconsistente.

### Repositorio `documentacion`

#### DOC-01 --- \[Docs\] README general y ejecución reproducible

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M5\
**Tareas** - \[ \] Describir componentes y requisitos previos. - \[ \]
Documentar variables de entorno sin incluir secretos. - \[ \] Explicar
Docker Compose y orden de arranque. - \[ \] Explicar cómo ejecutar
Android fuera de contenedores. - \[ \] Documentar pruebas.

**Aceptación:** otra persona puede levantar el proyecto siguiendo el
README.

#### DOC-02 --- \[Docs\] Diagrama de arquitectura y límites de datos

**Prioridad:** P0 · **Estimación:** 2 h · **Milestone:** M1\
**Tareas** - \[ \] Mostrar KMP, catálogo, turnos y bases separadas. - \[
\] Mostrar REST, Redis y Kafka de cátedra. - \[ \] Mostrar autenticación
y contratos entre servicios. - \[ \] Indicar qué servicio es dueño de
cada dato.

**Aceptación:** el diagrama refleja la arquitectura implementada y la
separación exigida.

#### DOC-03 --- \[Docs\] Contratos entre catálogo y turnos

**Prioridad:** P0 · **Estimación:** 2 h · **Milestone:** M1\
**Tareas** - \[ \] Documentar endpoints, DTO, autenticación y errores. -
\[ \] Incluir ejemplos de request/response. - \[ \] Documentar timeouts
y comportamiento ante indisponibilidad.

**Aceptación:** ambos servicios pueden integrarse sin depender de clases
internas del otro.

#### DOC-04 --- \[Docs\] Modelo de datos y propiedad

**Prioridad:** P0 · **Estimación:** 2 h · **Milestone:** M5\
**Tareas** - \[ \] Documentar modelo de catálogo. - \[ \] Documentar
modelo de turnos. - \[ \] Explicar separación de bases/esquemas y
migraciones.

**Aceptación:** queda claro qué servicio crea, modifica y consulta cada
dato.

#### DOC-05 --- \[Docs\] Sincronización, idempotencia y recuperación

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M5\
**Tareas** - \[ \] Explicar snapshot e incremental. - \[ \] Explicar
deduplicación y commit de offsets. - \[ \] Explicar recuperación ante
discontinuidad. - \[ \] Explicar máquina de estados, vencimientos y
reconciliación.

**Aceptación:** documentación coincide con implementación y pruebas.

#### DOC-06 --- \[Docs\] Seguridad y evidencias de entrega

**Prioridad:** P0 · **Estimación:** 3 h · **Milestone:** M5\
**Tareas** - \[ \] Documentar JWT de usuario y JWT técnico por
separado. - \[ \] Explicar autorización por propietario y comunicación
entre servicios. - \[ \] Preparar evidencias de snapshot, incremental,
recuperación, búsqueda y reserva. - \[ \] Documentar cómo ejecutar
pruebas automatizadas.

**Aceptación:** se puede verificar el cumplimiento de los requisitos de
seguridad y evidencias mínimas.

## 5. Issues transversales de integración

Crear estos issues en el repositorio `documentacion` o como issues
vinculados en los repositorios afectados.

#### INT-01 --- \[Integration\] Levantar los cuatro componentes en entorno local

**Prioridad:** P0 · **Milestone:** M1\
- \[ \] Docker Compose levanta ambos backends y bases de datos. - \[ \]
Android se ejecuta fuera de contenedores. - \[ \] Configuración externa
y secretos documentados. - \[ \] Health checks y orden de arranque
definidos.

#### INT-02 --- \[Integration\] Verificar flujo completo de catálogo

**Prioridad:** P0 · **Milestone:** M2\
- \[ \] Snapshot inicial. - \[ \] Evento incremental. - \[ \] Duplicado
de evento. - \[ \] Discontinuidad y snapshot de recuperación. - \[ \]
Búsqueda local desde KMP.

#### INT-03 --- \[Integration\] Verificar reserva end-to-end

**Prioridad:** P0 · **Milestone:** M3\
- \[ \] Disponibilidad desde agenda y ocupaciones. - \[ \] Hold y
confirmación inicial. - \[ \] Solicitud de teléfono. - \[ \] Publicación
de teléfono. - \[ \] Confirmación/rechazo/expiración reflejados en KMP.

#### INT-04 --- \[Integration\] Verificar aislamiento y cancelación

**Prioridad:** P0 · **Milestone:** M4\
- \[ \] Crear dos usuarios de prueba. - \[ \] Comprobar que no pueden
consultar reservas ajenas. - \[ \] Comprobar que no pueden cancelar
reservas ajenas. - \[ \] Cancelar una reserva propia confirmada.

#### INT-05 --- \[Release\] Preparar entrega y defensa

**Prioridad:** P0 · **Milestone:** M5\
- \[ \] Ejecutar desde un entorno limpio. - \[ \] Verificar los cuatro
repositorios y su historial de commits. - \[ \] Cerrar o etiquetar
issues pendientes. - \[ \] Revisar README, diagramas y pruebas. - \[ \]
Ensayar explicación de arquitectura, sincronización, idempotencia,
seguridad y flujo de reserva.

## 5. Orden recomendado de ejecución

1.  M1: `CAT-01`, `TUR-01`, `TUR-02`, `KMP-01`, `KMP-02`, `DOC-02`,
    `DOC-03`, `INT-01`.
2.  M2: `CAT-02` a `CAT-07`, `KMP-03`, `INT-02`.
3.  M3: `TUR-03` a `TUR-07`, `KMP-04`, `KMP-05`, `INT-03`.
4.  M4: `TUR-08`, `TUR-09`, `KMP-06`, `INT-04`.
5.  M5: `CAT-08`, `TUR-10`, `TUR-11`, `KMP-07`, `DOC-01`, `DOC-04` a
    `DOC-06`, `INT-05`.

Las dependencias importan: no conviene comenzar la pantalla final de
reservas antes de que estén definidos y probados los contratos de turnos
y catálogo.

## 6. Etiquetas recomendadas

-   `priority:P0` --- obligatorio para cumplir la entrega.
-   `priority:P1` --- importante, pero posterior a los requisitos P0.
-   `type:feature` --- funcionalidad.
-   `type:bug` --- comportamiento incorrecto.
-   `type:docs` --- documentación.
-   `type:test` --- pruebas.
-   `type:integration` --- integración entre componentes.
-   `blocked` --- depende de una tarea externa o de la cátedra.

## 7. Definition of Done (criterio común para cerrar un issue)

Un issue se cierra solamente cuando: - \[ \] El cambio está implementado
en el repositorio correcto. - \[ \] Cumple sus criterios de
aceptación. - \[ \] Tiene pruebas acordes al comportamiento
modificado. - \[ \] No contiene secretos, tokens ni datos reales. - \[
\] La documentación afectada está actualizada. - \[ \] Hay commit/push
identificable y el issue referencia el cambio. - \[ \] La integración
con los otros componentes no quedó rota.

## 8. Riesgos y recortes permitidos

**No recortar:** seguridad, aislamiento por usuario,
snapshot/incremental/recuperación, flujo REST+Kafka, expiración,
idempotencia, pruebas backend obligatorias, Docker Compose y
documentación mínima.

**Dejar para el final o excluir si falta tiempo:** recuperación de
contraseña (opcional), rol administrador (opcional), plataforma KMP
adicional a Android (no obligatoria), mejoras visuales no funcionales y
filtros adicionales a los cuatro mínimos.

**Dependencia externa importante:** la cátedra entrega URL REST,
host/puerto Redis y bootstrap servers Kafka por otro canal. Hasta
tenerlos, desarrollar clientes contra configuración externa y pruebas
controladas; no inventar direcciones ni credenciales.
