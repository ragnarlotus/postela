# Sync and notifications

**Grupo:** `RFSN`

**Numeración:** `RFSN001`, `RFSN002`, `RFSN003`…

## Alcance

Este grupo define la supervisión de cambios en el servidor de correo, su comunicación en tiempo real con Postela Web y la recuperación de las conexiones interrumpidas.

Postela consta de un backend en Go y un frontend SPA en Vue 3. El correo permanece en el servidor IMAP; la supervisión no implica descargar ni almacenar localmente todos los mensajes.

Las operaciones de lectura, modificación y envío del correo se especificarán en sus respectivos grupos. La autenticación inicial, la custodia de credenciales y la gestión general de sesiones se detallarán en los requisitos correspondientes.

## Requisitos funcionales

### Supervisión del servidor IMAP

#### RFSN001 — Conexión persistente de supervisión

El backend debe mantener **una única conexión IMAP persistente por contexto de cuenta supervisada** mientras exista una sesión activa que requiera dicha supervisión.

Esta conexión se destina a recibir notificaciones. No debe abrirse una conexión persistente por cada carpeta ni utilizarse para ejecutar las operaciones ordinarias de lectura y modificación de mensajes, que emplearán conexiones bajo demanda.

#### RFSN002 — Suscripción a cambios de todas las carpetas personales

Tras autenticar la conexión de supervisión, el backend debe activar `CONDSTORE` y establecer una suscripción `NOTIFY PERSONAL` que cubra las carpetas personales de la cuenta.

La suscripción debe incluir los eventos `MessageNew`, `MessageExpunge`, `FlagChange`, `MailboxName` y `SubscriptionChange`. La recepción de eventos no debe requerir seleccionar individualmente cada carpeta mediante `SELECT` ni mantener un comando `IDLE`.

#### RFSN003 — Detección de cambios en los mensajes

El backend debe detectar, mediante las notificaciones IMAP recibidas, la llegada y desaparición de mensajes en cualquier carpeta supervisada, incluidos los cambios que produzca el movimiento de un mensaje entre carpetas.

Cada carpeta afectada debe tratarse de forma independiente, sin exigir que el backend identifique el mensaje concreto que originó la notificación.

#### RFSN004 — Detección de cambios de indicadores

El backend debe detectar cambios de indicadores de los mensajes —por ejemplo, leído o no leído— en las carpetas supervisadas, incluidas las que no están abiertas en el frontend.

Debe utilizar la información facilitada por las notificaciones y `CONDSTORE`, sin requerir una sincronización incremental basada en el seguimiento de `MODSEQ` por mensaje.

#### RFSN005 — Detección de cambios en la estructura de carpetas

El backend debe detectar la creación, eliminación, renombrado y cambio de suscripción de carpetas.

Cuando el servidor proporcione el nombre anterior de una carpeta renombrada mediante `OLDNAME`, deberá conservarse esa información para actualizar correctamente la estructura mostrada. El atributo `\\NonExistent` debe interpretarse como indicación de eliminación; una respuesta `LIST` adicional sin ese atributo no debe interpretarse por sí sola como una nueva creación.

### Actualización de la interfaz

#### RFSN006 — Actualización de contadores de carpetas

Ante una notificación de cambio, Postela debe actualizar los contadores y estados afectados de la carpeta correspondiente con los datos disponibles del servidor, incluidos el número de mensajes y el de mensajes no leídos cuando se faciliten.

Las notificaciones IMAP pueden contener solo algunos campos. Los campos ausentes no deben interpretarse como valores cero ni como cambios.

#### RFSN007 — Actualización del listado visible

Si cambia la carpeta cuyo listado está viendo el usuario, Postela debe solicitar nuevamente **solo la página de mensajes que esté visible**, respetando su paginación y ordenación, y sustituir con el resultado el listado mostrado.

No debe recalcular diferencias entre listados ni descargar el contenido de todas las páginas para identificar altas, bajas o movimientos. Si la carpeta afectada no está abierta, bastará con actualizar sus contadores y estados; no se refrescará su listado de mensajes.

#### RFSN008 — Actualización del árbol de carpetas

Cuando se reciba una notificación de creación, eliminación, renombrado o suscripción, Postela debe reflejar el cambio correspondiente en el árbol de carpetas del frontend.

El resultado debe ser coherente aunque el servidor emita varias notificaciones relacionadas con una misma operación.

### Comunicación en tiempo real con el frontend

#### RFSN009 — Canal WebSocket único por sesión

El frontend debe establecer **una conexión WebSocket bidireccional por sesión activa** con el backend.

El mismo canal debe servir para las solicitudes del frontend, las respuestas del backend y las notificaciones de los distintos módulos de Postela, sin establecer conexiones WebSocket adicionales por carpeta o funcionalidad.

#### RFSN010 — Propagación de notificaciones

El backend debe comunicar por WebSocket los cambios relevantes detectados en IMAP para que el frontend actualice los contadores, el árbol de carpetas o la página de mensajes visible según corresponda.

Las notificaciones deben quedar asociadas a la cuenta y sesión que corresponda, evitando que cambios de una cuenta se comuniquen a sesiones ajenas.

### Desconexiones y recuperación

#### RFSN011 — Detección de pérdida de conexión IMAP

El backend debe detectar la pérdida inesperada de la conexión persistente de supervisión IMAP y dejar de considerarla operativa desde ese momento.

#### RFSN012 — Reintentos de reconexión IMAP

Cuando se pierda la conexión de supervisión IMAP, el backend debe realizar **hasta tres intentos de reconexión** antes de declarar el fallo definitivo.

Los intervalos entre intentos y los tiempos de espera concretos se definirán durante la implementación.

#### RFSN013 — Restauración de la supervisión IMAP

Cada reconexión IMAP exitosa debe restablecer una sesión autenticada y volver a activar `CONDSTORE` y `NOTIFY PERSONAL` con los eventos establecidos en RFSN002.

La supervisión solo debe considerarse recuperada cuando la suscripción vuelva a estar operativa.

#### RFSN014 — Fallo definitivo de conexión con el servidor de correo

Si se agotan los tres intentos de reconexión IMAP, el backend debe comunicar al frontend que no ha sido posible conectar con el servidor de correo y cerrar de forma controlada el WebSocket correspondiente, dando por finalizada la sesión afectada.

Este cierre debe identificarse como intencionado para que el frontend no inicie por él otros tres intentos de reconexión WebSocket.

#### RFSN015 — Reintentos de reconexión WebSocket

Si el frontend pierde inesperadamente la conexión WebSocket con el backend, debe realizar **hasta tres intentos de reconexión**.

El frontend es responsable de iniciar estos intentos. La recuperación debe preservar la sesión cuando siga siendo válida, sin solicitar de nuevo las credenciales al usuario.

#### RFSN016 — Identificación de cierres controlados

Los cierres intencionados de WebSocket deben comunicar un motivo identificable por el frontend, mediante el mecanismo de cierre del protocolo o un mensaje de aplicación previo al cierre.

Si el motivo indica una finalización definitiva de la sesión, el frontend no debe intentar reconectar. Si la conexión termina sin recibir un motivo de cierre controlado, deberá tratarse como una pérdida inesperada y aplicar RFSN015.

#### RFSN017 — Fallo definitivo de conexión con el backend

Si el frontend agota sus tres intentos de reconexión WebSocket sin éxito, debe finalizar la sesión local, realizar el logout y mostrar un mensaje que indique que no ha sido posible conectar con el backend.

#### RFSN018 — Recuperación del estado después de reconectar

Tras recuperar satisfactoriamente una conexión IMAP o WebSocket, Postela debe **reconstruir el estado de la interfaz como tras un inicio de sesión**, sin volver a solicitar las credenciales mientras la sesión continúe siendo válida.

La reconstrucción debe recuperar el árbol de carpetas, sus contadores y la página de mensajes visible. No debe descargar todos los mensajes ni implementar una sincronización incremental de los cambios ocurridos durante la desconexión.

## Decisiones de alcance

- La supervisión de todas las carpetas personales se basa en `NOTIFY PERSONAL + CONDSTORE`; no utiliza `IDLE`, `SELECT` por carpeta ni consultas periódicas para detectar cambios.
- La conexión persistente de supervisión IMAP es independiente de las conexiones bajo demanda usadas para operar sobre los mensajes.
- La actualización de listados se realiza sustituyendo la página visible, sin algoritmos de comparación de mensajes ni seguimiento incremental de `MODSEQ`.
- Los reintentos son **tres por conexión interrumpida**, distinguiendo los fallos de comunicación con el servidor de correo de los fallos de comunicación con el backend.
- No se fijan aquí los intervalos de reintento, los tiempos de espera, el formato de los mensajes WebSocket ni los códigos de cierre concretos; son decisiones de implementación pendientes.
