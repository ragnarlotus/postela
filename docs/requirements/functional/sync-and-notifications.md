# Sync and notifications

**Grupo:** `RFSN`

**Numeración:** `RFSN001`, `RFSN002`, `RFSN003`…

**Detalles de implementación:** [Requisitos técnicos de sincronización y notificaciones](../technical/sync-and-notifications.md).

## Alcance

Este grupo define **qué debe hacer Postela** para detectar cambios en el correo, mantener actualizada la interfaz y recuperarse de las interrupciones de conexión.

**Postela no almacena ningún mensaje de correo.** Los mensajes permanecen en el servidor de correo y se consultan cuando es necesario.

Las operaciones de lectura, modificación y envío de mensajes, así como la autenticación inicial y la gestión general de sesiones, se definirán en sus grupos correspondientes. Los protocolos, las conexiones y las bibliotecas utilizados para cumplir estos comportamientos se especifican en los requisitos técnicos.

## Requisitos funcionales

### Supervisión de cambios

#### RFSN001 — Supervisión continua del correo

Mientras la cuenta esté activa, Postela debe detectar los cambios que se produzcan en el correo sin que el usuario tenga que actualizar manualmente la interfaz.

#### RFSN002 — Supervisión de todas las carpetas

Postela debe supervisar los cambios de todas las carpetas personales de la cuenta, incluidas las que no estén abiertas en la interfaz.

#### RFSN003 — Detección de entradas y salidas de mensajes

Postela debe detectar cuándo se incorpora o desaparece un mensaje de una carpeta, tanto por recepción o eliminación como por movimientos entre carpetas.

#### RFSN004 — Detección de cambios de indicadores

Postela debe detectar los cambios en el estado de los mensajes, como marcar un correo como leído o no leído, independientemente de la carpeta en la que se encuentren.

#### RFSN005 — Detección de cambios en carpetas

Postela debe detectar la creación, eliminación y renombrado de carpetas, así como sus cambios de suscripción.

### Actualización de la interfaz

#### RFSN006 — Actualización de contadores

Cuando cambie el contenido o el estado de una carpeta, Postela debe actualizar sus contadores afectados, incluidos el número total de mensajes y el de mensajes no leídos.

#### RFSN007 — Actualización de la página visible

Si cambia una carpeta cuyo listado de mensajes está abierto, Postela debe volver a cargar **solo la página que está viendo el usuario**, conservando la paginación y ordenación seleccionadas, y sustituir el listado anterior por el actualizado.

Si la carpeta afectada no está abierta, se actualizarán sus contadores sin recargar su listado de mensajes.

#### RFSN008 — Actualización del árbol de carpetas

Cuando se cree, elimine, renombre o cambie la suscripción de una carpeta, el árbol de carpetas mostrado debe reflejar el resultado sin requerir una actualización manual.

### Comunicación en tiempo real

#### RFSN009 — Comunicación bidireccional

Postela debe permitir que la interfaz envíe solicitudes al backend, reciba sus respuestas y obtenga notificaciones de cambios en tiempo real durante la sesión.

#### RFSN010 — Aislamiento de notificaciones

Los cambios de una cuenta deben comunicarse únicamente a las sesiones autorizadas para esa cuenta. Ninguna sesión debe recibir información de otra cuenta que no le corresponda.

### Desconexiones y recuperación

#### RFSN011 — Detección de pérdida de conexión con el correo

Postela debe detectar cuando deja de estar disponible la conexión utilizada para supervisar el servidor de correo.

#### RFSN012 — Reconexión con el servidor de correo

Ante una pérdida inesperada de esa conexión, Postela debe realizar **hasta tres intentos de reconexión** antes de considerar que el servidor de correo no está disponible.

#### RFSN013 — Reanudación de la supervisión

Si Postela recupera la conexión con el servidor de correo, debe reanudar la supervisión de todas las carpetas y sus cambios.

#### RFSN014 — Fallo definitivo del servidor de correo

Si fallan los tres intentos de reconexión con el servidor de correo, Postela debe finalizar la sesión afectada e informar al usuario de que no ha sido posible conectar con el servidor de correo.

#### RFSN015 — Reconexión de la interfaz con el backend

Si la interfaz pierde inesperadamente la conexión con el backend, debe realizar **hasta tres intentos de reconexión** antes de considerar que el backend no está disponible.

#### RFSN016 — Tratamiento de cierres controlados

Cuando Postela finalice una conexión deliberadamente e informe de su motivo, la interfaz debe actuar según ese motivo y no intentar reconectar si se ha indicado una finalización definitiva de la sesión.

Si la conexión se interrumpe sin recibir esa indicación, debe tratarse como una desconexión inesperada.

#### RFSN017 — Fallo definitivo del backend

Si fallan los tres intentos de reconexión con el backend, la interfaz debe finalizar la sesión, realizar el logout e informar al usuario de que no ha sido posible conectar con el backend.

#### RFSN018 — Recuperación del estado de la interfaz

Después de recuperar correctamente una conexión con el servidor de correo o con el backend, Postela debe reconstruir el estado de la interfaz **como después de iniciar sesión**, sin volver a pedir credenciales si la sesión sigue siendo válida.

Esta recuperación incluirá el árbol de carpetas, sus contadores y la página de mensajes que se esté mostrando, sin cargar todos los mensajes de la cuenta.
