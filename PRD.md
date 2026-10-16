# PRD-001: A263-PDP — Sincronización de inscripciones de postulantes desde CRM

## Contexto y Problema
Un postulante paga el derecho de inscripción al examen de admisión de una carrera en un periodo. CRM Dynamics (Azure), plataforma de captación y fuente de verdad de la inscripción, la registra y puede luego modificarla o cancelarla. Por cada operación (creación, modificación, cancelación) CRM envía un evento de integración a A263-PDP (AWS), que sincroniza la inscripción y ejecuta las integraciones institucionales: Active Directory, Keycloak y servicio de correo.

Hoy, A263-PDP sincroniza y ejecuta las integraciones en un único flujo síncrono: si una integración falla o tarda, falla o se demora toda la sincronización. Un postulante que ya pagó puede quedar sin accesos para rendir su examen, y una inscripción cancelada puede conservar accesos activos.

Personas:
- Postulante: pagó su derecho de inscripción; necesita recibir sus accesos a tiempo para el examen y que sus datos (incluida la carrera) estén correctos.
- Equipo de admisión / operaciones: necesita detectar inscripciones con integraciones incompletas o desincronizadas y resolverlas sin intervención técnica.
- CRM Dynamics (sistema, fuente de verdad): necesita una confirmación rápida y confiable de cada evento enviado.
- Servicios institucionales (Active Directory, Keycloak, correo): reciben las integraciones; su disponibilidad no debe bloquear la sincronización. 

## Objetivos
- Ningún postulante que pagó queda sin accesos por la falla de un servicio institucional.
- Cada postulante tiene una sola identidad institucional, sin importar cuántas inscripciones tenga.
- Ningún postulante sin inscripciones activas conserva accesos habilitados.
- CRM recibe la confirmación de cada evento sin esperar a las integraciones.
- El estado de cada inscripción en A263-PDP coincide con el de CRM.
- Equipo de admisión / Operaciones identifica y resuelve inscripciones incompletas sin intervención técnica.

## Requerimientos Funcionales
- RF-01: El sistema debe recibir los eventos de inscripción (creación, modificación, cancelación) enviados por CRM.
- RF-02: El sistema debe validar que cada evento tenga eventId, aggregateId, eventType, sequence, eventVersion soportada, postulante (tipo_documento, num_doc, nombre, apellido, email), info_academica (campus, carrera, modalidad_estudio) y periodo_lectivo.
- RF-03: El sistema debe sincronizar una inscripción nueva al recibir un evento de creación.
- RF-04: El sistema debe ignorar un evento InscripcionCreada cuyo aggregateId ya está sincronizado.
- RF-05: El sistema debe reemplazar los datos de una inscripción con la inscripción completa recibida en un evento InscripcionModificada.
- RF-06: El sistema debe marcar como cancelada una inscripción al recibir un evento de cancelación.
- RF-07: El sistema debe publicar un evento de dominio por cada cambio de inscripción sincronizado.
- RF-08: El sistema debe crear la cuenta en Active Directory solo si no existe una para el num_doc del postulante.
- RF-09: El sistema debe validar el login para el num_doc del postulante en Keycloak.
- RF-10: El sistema debe asociar una nueva inscripción a la identidad existente del postulante cuando su num_doc ya tiene cuenta.
- RF-11: El sistema debe enviar el correo de inscripción después de que la cuenta en AD esté disponible y se valido el login en Keycloak.
- RF-12: El sistema debe ignorar un evento cuyo eventId ya fue procesado.
- RF-13: El sistema debe ignorar un evento cuyo sequence es menor o igual al último aplicado en ese aggregateId.
- RF-14: El sistema debe reintentar una integración fallida.
- RF-15: El sistema debe marcar como "incompleta" una inscripción cuya integración agotó los reintentos.
- RF-16: El sistema debe permitir a admisión/operaciones reprocesar una inscripción incompleta.
  
## Requerimientos No Funcionales
- RNF-01: La confirmación de recepción de un evento a CRM es <2s p95.
- RNF-02: El 99 % de los postulantes recibe sus accesos en ≤2min desde el evento de creación.
- RNF-03: Cada integración fallida se reintenta hasta 3 veces antes de marcarse incompleta.
- RNF-04: El 100 % de las inscripciones es trazable de punta a punta por su correlationId, aggregateId y eventId.

## Criterios de Aceptación
Recepción y validación

- AC-01 (RF-01): Dado un evento válido de cada tipo (InscripcionCreada, InscripcionModificada, InscripcionCancelada), cuando CRM lo envía, entonces A263-PDP confirma su recepción y lo registra con su eventId.
- AC-02 (RF-02): Dado un evento InscripcionCreada sin info_academica.carrera, cuando A263-PDP lo recibe, entonces lo rechaza indicando a CRM el campo faltante y no crea ni modifica ninguna inscripción.
- AC-03 (RF-02): Dado un evento con eventVersion 2 y solo la versión 1 soportada, cuando A263-PDP lo recibe, entonces lo rechaza indicando a CRM "versión no soportada" y no crea ni modifica ninguna inscripción.

Sincronización

- AC-04 (RF-03): Dado un evento InscripcionCreada válido con un aggregateId nuevo, cuando se procesa, entonces existe exactamente una inscripción con ese aggregateId, su num_doc, carrera, periodo_lectivo, sequence 1 y estado "activa".
- AC-05 (RF-04): Dada la inscripción "6304b037-…" ya sincronizada, cuando llega otro InscripcionCreada con el mismo aggregateId y distinto eventId, entonces no se crea un segundo registro, sus datos no cambian y se confirma la recepción a CRM.
- AC-06 (RF-05): Dada una inscripción con sequence 1 en carrera PSICO, cuando llega un InscripcionModificada con sequence 2, carrera ARQUI y otro email, entonces todos los datos de la inscripción son iguales a los del evento con sequence 2.
- AC-07 (RF-06): Dada una inscripción activa con sequence 2, cuando llega un InscripcionCancelada con sequence 3, entonces la inscripción queda en estado "cancelada" con sequence 3.
- AC-08 (RF-07): Dado un cambio de inscripción sincronizado (creación, modificación o cancelación), cuando se confirma, entonces se publica exactamente un evento de dominio con su aggregateId, tipo de operación y correlationId.

Integraciones

- AC-09 (RF-08): Dado el num_doc "61887110" sin cuenta en AD, cuando se sincroniza su primera inscripción, entonces existe exactamente una cuenta en AD para ese num_doc.
- AC-10 (RF-08, RF-10): Dado el num_doc "61887110" con cuenta en AD por una inscripción anterior, cuando se sincroniza una nueva inscripción con otro aggregateId, entonces no se crea una segunda cuenta y la nueva inscripción queda asociada a la identidad existente.
- AC-11 (RF-09): Dada una cuenta en AD disponible para el num_doc, cuando se valida el login en Keycloak, entonces la integración con Keycloak queda "completa"; si la validación falla, queda "fallida" y se aplica el reintento (RF-14).
- AC-12 (RF-11): Dada la cuenta en AD disponible y el login validado en Keycloak, cuando se completan ambas integraciones, entonces se envía exactamente un correo de inscripción al email del postulante.
- AC-13 (RF-11): Dada la cuenta en AD creada y la validación de login en Keycloak fallida, cuando se procesa la inscripción, entonces no se envía el correo de inscripción.

Idempotencia y orden

- AC-14 (RF-12): Dado el evento "evt-test-001" ya procesado, cuando CRM lo reenvía con el mismo eventId, entonces no se aplica de nuevo, no se ejecuta ninguna integración ni se reenvía el correo, y se confirma la recepción a CRM.
- AC-15 (RF-13): Dada una inscripción con sequence 3 aplicado, cuando llega un evento del mismo aggregateId con sequence 2, entonces se ignora y la inscripción conserva los datos del sequence 3.
- AC-16 (RF-13): Dada una inscripción con sequence 3 aplicado, cuando llega otro evento del mismo aggregateId con sequence 3 y distinto eventId, entonces se ignora.

Resiliencia y operación

- AC-17 (RF-14, RNF-03): Dado Active Directory no disponible, cuando se procesa una inscripción, entonces la creación de cuenta se reintenta exactamente 3 veces.
- AC-18 (RF-15): Dado que los 3 reintentos de una integración fallan, cuando se agota el último, entonces la inscripción queda "incompleta", conserva sus datos sincronizados y no se repiten las integraciones que ya estaban completas.
- AC-19 (RF-16): Dada una inscripción "incompleta" y el servicio ya disponible, cuando admisión/operaciones la reprocesa, entonces se ejecutan solo las integraciones pendientes, la inscripción pasa a "completa" y el correo se envía una sola vez.

No funcionales

- AC-20 (RNF-01): Dado un lote de 1 000 eventos válidos enviados, cuando se mide el tiempo de confirmación a CRM, entonces el p95 es < 2 s.
- AC-21 (RNF-02): Dadas 1 000 inscripciones nuevas con los servicios institucionales disponibles, cuando se procesan, entonces al menos 990 postulantes reciben su correo de inscripción en ≤ 2 min desde el evento de creación.
- AC-22 (RNF-04): Dada una inscripción procesada, cuando se busca por su correlationId, entonces se obtienen la recepción, la sincronización, el evento de dominio y cada integración, todos con su eventId y aggregateId.

## Fuera de Alcance 
- Gestión del examen de admisión, sus resultados y el ingreso a la carrera.
- Administración del catálogo de carreras y periodos.
- Migrar completamente el sistema Portal del Postulante.
- Deshabilitar accesos al cierre de un periodo.

## Riesgos y Dependencias
- Riesgo: la entrega de mensajes produce eventos duplicados → mitigación: idempotencia por ID de evento.
- Riesgo: los eventos de una misma inscripción llegan desordenados → mitigación: versión por inscripción enviada por CRM; se descartan las versiones anteriores y la cancelación es definitiva.
- Dependencia: contrato de eventos acordado con CRM: envelope (eventId, aggregateId, eventType, sequence por aggregateId, eventVersion como versión del formato, occurredAt, correlationId) y data (inscripcion completa, postulante); eventType: InscripcionCreada, InscripcionModificada, InscripcionCancelada.
- Dependencia: CRM solo envía InscripcionCreada cuando el derecho de inscripción está pagado.
- Dependencia: Disponibilidad de los servicios institucionales .
