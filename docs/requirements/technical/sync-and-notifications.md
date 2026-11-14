# Sync and notifications

**Grupo:** `RTSN`

**Numeración:** `RTSN001`, `RTSN002`, `RTSN003`…

**Requisitos funcionales relacionados:** [Sync and notifications](../functional/sync-and-notifications.md).

## Alcance

Este documento especifica los mecanismos de implementación de la supervisión de correo, la comunicación en tiempo real y las reconexiones de Postela. Los comportamientos que deben percibir los usuarios se definen exclusivamente en los requisitos funcionales `RFSN`.

Postela utiliza un backend en Go y un frontend SPA en Vue 3. **Postela no almacena mensajes de correo**: permanecen en el servidor IMAP y se consultan cuando es necesario. La configuración, las preferencias y otros datos propios de la aplicación se documentarán en sus grupos correspondientes.

## Requisitos técnicos

### Supervisión IMAP

#### RTSN001 — Conexión persistente de supervisión

El backend mantendrá **una única conexión IMAP persistente por contexto de cuenta supervisada** para la recepción de eventos, sin abrir una conexión persistente por carpeta.

Esta conexión será independiente de las conexiones IMAP de corta duración utilizadas, bajo demanda, para consultar y modificar mensajes. La supervisión no requerirá seleccionar carpetas mediante `SELECT`.

**Funcionales:** RFSN001–RFSN004.

#### RTSN002 — Transporte y autenticación IMAP

La conexión de supervisión utilizará IMAP autenticado y cifrado mediante STARTTLS, con validación del certificado del servidor. No se establecerá una sesión de supervisión operativa antes de completar correctamente su autenticación.

La custodia de las credenciales y la autenticación inicial del usuario se definirán en los requisitos de seguridad y sesiones.

**Funcionales:** RFSN001, RFSN011.

#### RTSN003 — Activación de CONDSTORE y NOTIFY PERSONAL

Después de autenticarse, el backend ejecutará `ENABLE CONDSTORE` y establecerá una suscripción `NOTIFY SET` con el selector `PERSONAL`.

Se solicitarán los eventos `MessageNew`, `MessageExpunge`, `FlagChange`, `MailboxName` y `SubscriptionChange`. La suscripción deberá cubrir las carpetas personales sin ejecutar `SELECT` individualmente.

Se utilizará `CONDSTORE` para obtener las notificaciones necesarias de cambios de indicadores en carpetas no seleccionadas. **No se utilizarán `IDLE`, seguimiento de `MODSEQ` por mensaje ni `CHANGEDSINCE` para una sincronización incremental.**

**Funcionales:** RFSN001–RFSN004, RFSN011.

#### RTSN004 — Recepción asíncrona de respuestas IMAP

La biblioteca IMAP del backend deberá procesar continuamente las respuestas no solicitadas del servidor, incluso cuando no haya comandos IMAP pendientes. Las notificaciones de mensajes y cambios de indicadores se recibirán mediante `STATUS`; las de estructura y suscripción de carpetas, mediante `LIST`.

El procesamiento de eventos no deberá bloquear la lectura de respuestas de la conexión.

**Funcionales:** RFSN001–RFSN005.

#### RTSN005 — Interpretación de STATUS parcial

Las respuestas `STATUS` pueden incluir solo los atributos afectados por un cambio. El backend deberá identificar los atributos efectivamente presentes y no interpretar los campos ausentes como valores cero.

Se tendrán en cuenta, cuando aparezcan, `MESSAGES`, `UNSEEN`, `UIDNEXT` y `HIGHESTMODSEQ`. Este último sirve como indicador de cambios de flags; no se utilizará para mantener una copia incremental de los mensajes.

**Funcionales:** RFSN002–RFSN004, RFSN006.

#### RTSN006 — Interpretación de LIST y cambios de carpeta

Para procesar eventos `MailboxName` y `SubscriptionChange`, el backend interpretará las respuestas `LIST` y sus atributos.

En particular, deberá reconocer `OLDNAME` para identificar renombrados, `\\NonExistent` para identificar eliminaciones y `\\Subscribed` para conocer el estado de suscripción.

Varias respuestas `LIST` pueden corresponder a una misma operación. Una respuesta sin atributos, por sí sola, no deberá interpretarse necesariamente como la creación de una carpeta.

**Funcionales:** RFSN005, RFSN008.

### Actualización de datos y comunicación con Vue

#### RTSN007 — Conexión WebSocket por sesión

El frontend mantendrá **una conexión WebSocket bidireccional por sesión activa** con el backend.

Por ese único canal circularán las solicitudes de los distintos módulos, sus respuestas y los eventos asíncronos de correo. No se abrirán WebSockets adicionales por carpeta o funcionalidad.

**Funcionales:** RFSN009.

#### RTSN008 — Enrutamiento de notificaciones

El backend identificará la cuenta y las sesiones destinatarias de cada cambio detectado en IMAP y transmitirá únicamente las notificaciones que les correspondan.

El canal de notificaciones permitirá distinguir los cambios en mensajes, indicadores y estructura de carpetas, así como el nombre de la carpeta afectada.

El formato concreto de los mensajes WebSocket se definirá durante la implementación.

**Funcionales:** RFSN006–RFSN010.

#### RTSN009 — Recarga de la página de mensajes visible

Cuando cambie la carpeta abierta, el backend atenderá la solicitud de refresco recuperando **únicamente la página de mensajes visible**, con su paginación y ordenación actuales, mediante una conexión IMAP bajo demanda.

El frontend sustituirá el listado mostrado por el resultado. No se compararán listados para reconstruir altas, bajas o movimientos, ni se consultarán páginas no visibles.

Si cambia una carpeta que no está abierta, se actualizarán sus contadores y estado sin recargar su listado de mensajes.

**Funcionales:** RFSN006, RFSN007.

#### RTSN010 — Ausencia de almacenamiento de mensajes

El backend consultará los mensajes directamente al servidor IMAP y los entregará al frontend cuando sean solicitados. **No almacenará mensajes de correo en SQLite ni mantendrá un repositorio local de mensajes.**

Los datos de las respuestas se tratarán para atender la operación solicitada, sin crear una copia persistente del correo.

**Funcionales:** RFSN007, RFSN018.

### Gestión de desconexiones

#### RTSN011 — Detección de desconexión IMAP

El backend deberá detectar la finalización o el fallo de la conexión de supervisión IMAP y marcarla como no operativa.

La detección no deberá depender de que el usuario realice una operación en la interfaz. El mecanismo concreto de comprobación de conexiones inactivas y sus tiempos de espera quedan pendientes de definir.

**Funcionales:** RFSN011.

#### RTSN012 — Reconexión IMAP y restauración de la suscripción

Ante una desconexión involuntaria, el backend efectuará **hasta tres intentos de reconexión**.

Cada intento que consiga conectar deberá restablecer la autenticación y ejecutar nuevamente `ENABLE CONDSTORE` y la suscripción `NOTIFY PERSONAL` con los eventos de RTSN003.

No se considerará recuperada la supervisión hasta completar estas operaciones. Los intervalos entre intentos y tiempos de espera se decidirán durante la implementación.

**Funcionales:** RFSN012, RFSN013.

#### RTSN013 — Finalización por fallo definitivo de IMAP

Una vez agotados los tres intentos de reconexión IMAP, el backend deberá comunicar al frontend el motivo del fallo y cerrar de forma controlada su conexión WebSocket, finalizando la sesión afectada.

El frontend deberá reconocer este cierre como definitivo y no iniciar un nuevo ciclo de tres reconexiones WebSocket por la misma causa.

**Funcionales:** RFSN014, RFSN016.

#### RTSN014 — Reconexión WebSocket

El frontend será responsable de reconectar su WebSocket cuando se interrumpa inesperadamente, con **un máximo de tres intentos**.

Si vuelve a conectar y la sesión sigue siendo válida, podrá continuar sin solicitar de nuevo las credenciales. La conservación temporal de la sesión en el backend durante la interrupción se detallará en los requisitos de sesiones.

Los intervalos y tiempos de espera concretos quedan pendientes de definir.

**Funcionales:** RFSN015.

#### RTSN015 — Identificación de cierres controlados

Las finalizaciones deliberadas de WebSocket deberán transmitir un motivo reconocible para el frontend, usando un código de cierre WebSocket apropiado o un mensaje de aplicación antes del cierre.

Un cierre comunicado como definitivo impedirá iniciar reintentos automáticos. Si la conexión se interrumpe sin que el cliente reciba una indicación fiable de finalización controlada, se aplicará la política de reconexión inesperada.

Los códigos y mensajes concretos se definirán durante la implementación.

**Funcionales:** RFSN014, RFSN016.

#### RTSN016 — Fallo definitivo de WebSocket

Si fallan los tres intentos de reconexión WebSocket, el frontend deberá finalizar la sesión local, ejecutar el logout y mostrar un error que identifique al backend como destino inaccesible.

**Funcionales:** RFSN017.

#### RTSN017 — Reconstrucción del estado tras recuperar la conexión

Tras recuperar una conexión IMAP o WebSocket, Postela ejecutará el flujo de obtención del estado inicial de la interfaz, **sin solicitar de nuevo las credenciales si la sesión sigue siendo válida**.

Se recuperarán el árbol de carpetas, sus contadores y la página de mensajes que esté visible. No se descargarán todos los mensajes ni se procesará un historial incremental de eventos perdidos durante la interrupción.

**Funcionales:** RFSN018.

## Validación realizada

El mecanismo central de supervisión se ha probado con **Dovecot 2.3.21.1** y **Go 1.24.4**, utilizando `github.com/kiliant/go-imap` **v1.3.1** como biblioteca candidata.

Se verificaron la autenticación mediante STARTTLS, `ENABLE CONDSTORE`, `NOTIFY PERSONAL` y la recepción asíncrona de `STATUS` y `LIST` para mensajes nuevos, cambios de leído/no leído, movimientos, creación de carpetas, suscripciones, renombrados con `OLDNAME` y eliminaciones con `\\NonExistent`.

Estas pruebas demuestran la compatibilidad del mecanismo en el entorno comprobado; **no suponen todavía la adopción definitiva de la biblioteca** ni validan su comportamiento durante desconexiones prolongadas o reconexiones.

## Decisiones técnicas pendientes

No se fijan aún la política de tiempos de espera y separación entre reintentos, la detección de conexiones inactivas, los códigos y formatos exactos del protocolo WebSocket ni el plazo durante el que el backend conserva una sesión tras perder su WebSocket. Esos detalles se acordarán cuando corresponda, sin alterar los comportamientos funcionales definidos.
