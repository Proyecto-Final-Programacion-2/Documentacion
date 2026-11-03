# Plan de trabajo diario --- Proyecto Integrador de Programación 2

**Período:** 2 al 29 de noviembre de 2026\
**Entrega prevista:** últimos días de noviembre (la cátedra todavía debe
confirmar día y horario).\
**Repositorios:** `app-kmp`, `catalogo`, `turnos` y `documentacion`.

## 1. Criterio de planificación

El enunciado exige una aplicación Android KMP funcional y dos backends
independientes en Java/Spring Boot. El servicio central de la cátedra es
externo: no hay que implementarlo, sino integrarlo mediante REST, Redis
y Kafka.

La planificación prioriza primero que los tres componentes funcionen
integrados y después robustez, pruebas y documentación. No conviene
terminar cada repositorio de forma aislada y dejar la integración para
el final.

**Dedicación sugerida:** 3 horas de lunes a viernes y 4--5 horas sábados
y domingos (aprox. 24--26 horas semanales). Si un día no podés
cumplirlo, trasladá la tarea al próximo bloque; no saltees las pruebas
de integración.

**Bloque diario sugerido** - 15 min: revisar issues y elegir el objetivo
del día. - 2 h 15 min: implementar una tarea concreta. - 30 min: probar,
corregir y registrar decisiones. - 15 min: commit/push y actualizar el
issue.

Usá ramas cortas por issue, commits frecuentes y un `main` siempre
ejecutable. En cada issue indicá repositorio, criterios de aceptación y
dependencias.

## 2. Calendario día por día

  --------------------------------------------------------------------------------------------
  Fecha                       Dedicación Objetivo y tareas                   Resultado
                                                                             esperado
  ---------------- --------------------- ----------------------------------- -----------------
  Lun 02/11                          3 h Leer enunciado y contrato           Inventario real
                                         completos. Revisar los cuatro       del proyecto e
                                         repositorios, estructura,           issues
                                         tecnologías, estado actual y        priorizados.
                                         pendientes. Crear tablero y         
                                         milestones.                         

  Mar 03/11                          3 h Definir arquitectura, límites de    Decisiones
                                         datos, contratos entre `catalogo` y documentadas y
                                         `turnos`, autenticación entre       diagrama inicial.
                                         servicios y modelo de estados de    
                                         reserva.                            

  Mié 04/11                          3 h Preparar Docker Compose para        Infraestructura
                                         backends y bases de datos servidor. local
                                         Separar bases/esquemas, usuarios y  reproducible.
                                         migraciones.                        

  Jue 05/11                          3 h Configuración externa, perfiles,    Ambos backends
                                         variables de entorno, manejo de     arrancan sin
                                         secretos y conexión base de datos   secretos en el
                                         en ambos servicios.                 código.

  Vie 06/11                          3 h Implementar registro/login de       API de
                                         usuario final, hashing de           autenticación
                                         contraseñas y JWT en el backend     probada.
                                         correspondiente.                    

  Sáb 07/11                       4--5 h Completar autorización JWT,         Seguridad base
                                         endpoints protegidos y aislamiento  del backend
                                         de datos por usuario. Agregar       lista.
                                         pruebas básicas.                    

  Dom 08/11                       4--5 h KMP: navegación, pantallas de       Usuario puede
                                         registro/login, cliente HTTP,       registrarse e
                                         almacenamiento seguro de sesión y   iniciar sesión
                                         manejo de errores.                  desde Android.

  Lun 09/11                          3 h Cuenta técnica de cátedra:          Conectividad REST
                                         registrar una vez con Postman,      de cátedra
                                         autenticar y guardar configuración  configurada.
                                         fuera de Git. Configurar REST en    
                                         ambos backends.                     

  Mar 10/11                          3 h Catálogo: entidades, repositorios,  Modelo local del
                                         migraciones y endpoint interno para catálogo
                                         consultar categorías, profesionales implementado.
                                         y agendas.                          

  Mié 11/11                          3 h Catálogo: obtener snapshot REST y   Primera
                                         aplicarlo de forma consistente;     sincronización
                                         persistir `snapshotVersion` solo al completa
                                         completar las tres colecciones.     demostrable.

  Jue 12/11                          3 h Catálogo: cliente Redis, lectura de Lectura de Redis
                                         metadata, versiones, cambios y      implementada.
                                         entidades; validar namespaces y     
                                         permisos.                           

  Vie 13/11                          3 h Catálogo: consumidor Kafka          Actualización
                                         `CatalogUpdated`, deduplicación por incremental
                                         `eventId` y sincronización          demostrable.
                                         incremental ordenada.               

  Sáb 14/11                       4--5 h Catálogo: detectar                  Recuperación
                                         discontinuidad/versión fuera de     completa y
                                         ventana y reconstruir mediante      consistencia
                                         snapshot. Probar reinicio y         verificadas.
                                         notificación duplicada.             

  Dom 15/11                       4--5 h Catálogo: búsquedas y filtros       Casos de búsqueda
                                         locales por categoría, nombre,      resueltos sin
                                         habilitado y disponibilidad.        consultar cátedra
                                         Exponer API protegida para KMP y    por cada
                                         turnos.                             búsqueda.

  Lun 16/11                          3 h KMP: pantalla de búsqueda, filtros  Búsqueda
                                         y listado de profesionales usando   funcional desde
                                         API de catálogo. Estados de carga,  Android.
                                         vacío y error.                      

  Mar 17/11                          3 h Turnos: modelo de datos propio,     Persistencia de
                                         migraciones y asociación            turnos lista.
                                         obligatoria de procesos/reservas    
                                         con usuario final.                  

  Mié 18/11                          3 h Turnos: consultar catálogo y        Disponibilidad
                                         ocupaciones REST; construir slots   calculada
                                         combinando agenda semanal y         correctamente.
                                         ocupaciones `CONFIRMED`/`HELD`.     

  Jue 19/11                          3 h Turnos: crear hold y confirmar      Inicio de reserva
                                         inicialmente por REST; persistir    implementado.
                                         IDs, vencimiento y estado local.    

  Vie 20/11                          3 h Turnos: consumidor Kafka de         Flujo llega al
                                         acciones; procesar                  pedido de
                                         `AdditionalInformationRequested` y  teléfono.
                                         mostrar solicitud de teléfono a     
                                         KMP.                                

  Sáb 21/11                       4--5 h Turnos: publicar                    Flujo asincrónico
                                         `AdditionalInformationSubmitted`;   completo probado.
                                         procesar confirmación, rechazo,     
                                         expiración e invalidez. Deduplicar  
                                         eventos.                            

  Dom 22/11                       4--5 h KMP: selección de fecha/horario,    Reserva completa
                                         ingreso de teléfono, estados de     desde la app
                                         proceso y resultado final.          Android.

  Lun 23/11                          3 h Turnos: listar reservas filtradas   Consulta y
                                         por usuario, consultar estado por   cancelación
                                         REST y cancelar reserva confirmada  funcionales.
                                         propia.                             

  Mar 24/11                          3 h KMP: pantalla de mis reservas,      Ciclo de vida de
                                         detalle/estado y acción de          reserva visible
                                         cancelar; probar aislamiento entre  en Android.
                                         dos usuarios.                       

  Mié 25/11                          3 h Robustez: timeouts, reintentos      Recuperación ante
                                         limitados, reconciliación tras      fallos
                                         timeout, reinicio con procesos      principales.
                                         pendientes y errores REST.          

  Jue 26/11                          3 h Pruebas automatizadas obligatorias: Suite mínima
                                         autenticación, sincronización,      ejecutable.
                                         reserva, autorización e             
                                         idempotencia. Corregir fallos.      

  Vie 27/11                          3 h Prueba integral desde Docker        Evidencias de
                                         Compose y Android: snapshot,        aceptación
                                         incremental, búsqueda, reserva      completas o lista
                                         confirmada, reserva expirada,       final de
                                         consulta y cancelación.             bloqueos.

  Sáb 28/11                       4--5 h Completar README, arquitectura,     Documentación
                                         contratos internos, modelos de      alineada con el
                                         datos, seguridad, sincronización,   código.
                                         idempotencia y recuperación.        

  Dom 29/11                       4--5 h Ensayo de defensa, revisar issues,  Versión candidata
                                         limpiar logs/configuración,         de entrega.
                                         comprobar repositorios,             
                                         tags/releases y pasos de ejecución. 
                                         No agregar funcionalidades nuevas.  
  --------------------------------------------------------------------------------------------

**Si la entrega es el 30/11:** reservar ese día para una última
ejecución limpia, verificar enlaces y entregar. No planificar
desarrollos grandes para esa jornada.

## 3. Hitos y prioridades

  -----------------------------------------------------------------------
  Hito                    Fecha objetivo          Condición para darlo
                                                  por terminado
  ----------------------- ----------------------- -----------------------
  H1 --- Base y seguridad 08/11                   Infraestructura,
                                                  usuarios finales, JWT y
                                                  KMP login funcionando.

  H2 --- Catálogo         15/11                   Snapshot, incremental,
  sincronizado                                    recuperación, filtros
                                                  locales y API propia.

  H3 --- Reserva          22/11                   Hold, confirmación
  end-to-end                                      inicial, teléfono por
                                                  Kafka y resultado
                                                  final.

  H4 --- Aplicación       24/11                   Búsqueda,
  completa                                        disponibilidad,
                                                  reservas propias y
                                                  cancelación desde KMP.

  H5 --- Calidad y        29/11                   Pruebas, robustez,
  entrega                                         documentación y
                                                  demostración
                                                  reproducible.
  -----------------------------------------------------------------------

## 4. Reglas para no desviarse del alcance

-   No implementar funcionalidades médicas reales, pagos, historias
    clínicas ni videollamadas: están fuera de alcance.
-   La recuperación de contraseña y el rol administrador son opcionales;
    dejarlos para después de los requisitos obligatorios.
-   No duplicar el catálogo en el servicio de turnos. Consultar al
    servicio de catálogo mediante un contrato propio.
-   No compartir tablas, repositorios ni migraciones entre servicios.
-   La app Android debe completar el flujo; Postman sirve para el alta
    técnica inicial, no para completar reservas manualmente.
-   No enviar el JWT técnico de cátedra a KMP ni guardar credenciales en
    Git.
-   Los datos de prueba deben ser ficticios.
-   La fuente de verdad de los contratos externos es
    `INTEGRATION_REFERENCE-v2.md`.

## 5. Rutina de cierre diaria

Al terminar cada jornada: 1. Ejecutar las pruebas relacionadas con lo
modificado. 2. Hacer commit con mensaje claro y subir la rama. 3.
Actualizar el issue con lo terminado, lo pendiente y cualquier bloqueo.
4. Si una tarea se demora, reducir primero extras visuales o
funcionalidades opcionales; no eliminar seguridad, integración, pruebas
obligatorias ni documentación.
