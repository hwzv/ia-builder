# PRD-001: A263-PDP — Gestión de inscripción de postulantes

## Contexto y Problema
Actualmente el flujo de inscripciones de postulantes es sincrono, centraliza muchas responsabilidades e integraciones con servicios institucionales.  

Usuarios:
- CRM: Envia eventos de inscripciones.
- PDP: Recepciona, registra la inscripcion y ejecuta integraciones con sistemas institucionales.
- Servicios institucionales:
Active Directory, keycloak, Servicio de correo 

## Objetivos
Desacoplar el procesamiento de las inscripciones de las integraciones con servicios institucionales.

## Requerimientos Funcionales
- RF-01: El sistema debe recibir las solicitudes de inscripción enviadas por CRM.
- RF-02: El sistema debe procesar las solicitudes de inscripción.
- RF-03: El sistema debe publicar eventos correspondientes a las solicitudes de inscripción.
- RF-04: El sistema debe ejecutar asincronamente las integraciones con los servicios institucionales.
- RF-05: El sistema debe evitar el procesamiento de inscripciones duplicadas.

## Requerimientos No Funcionales
- RNF-01: Las operaciones de procesamiento de las solicitudes de inscripción deben tener un tiempo <2s en p95.

## Criterios de Aceptación
- AC-01 (RF-01,RF-02,RF-03): Dado una solicitud de inscripción de entrada, cuando se reciba una solicitud valida, entonces el sistema debe registrarla y publicar un evento de dominio.
- AC-02 (RF-04): Dado un evento de dominio, cuando sea aceptada, entonces el sistema debe permitir su procesamiento asincrono.
- AC-03 (RF-05): Dado una solicitud de inscripcion que ya fue procesado, cuando el sistema reciba la misma inscripcion, entonces el sistema debe evitar la duplicación.
- AC-04 (RNF-01): Dado un testing, cuando se ejecute la prueba, entonces el tiempo de respuesta debe ser <2s.

## Fuera de Alcance
- Migrar completamente el sistema Portal del Postulante.

## Riesgos y Dependencias
- Riesgo: La entrega de mensajes puede producir duplicados → mitigación: Implementar claves de idempotencia.
- Dependencia: Disponibilidad de los servicios institucionales .
