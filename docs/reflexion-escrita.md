# Reflexión Escrita sobre Metodología y Desarrollo con Agentes de IA

**Proyecto:** Sistema de Gestión Odontológica "La Muelita Cuyana"  
**Cátedra:** Metodología de Sistemas I — Tecnicatura Universitaria en Programación (UTN FRM)  
**Equipo de Desarrollo:**  
* Sebastián Gómez (Lead Developer / Tech Lead)
* Tomás Ruti (Backend Developer)
* Diego González (Backend Developer)
* Gaspar González (Backend Developer)
* Facundo López Romera (Frontend Specialist)

---

## 1. ¿Qué información del Discovery resultó incorrecta, no verificable o inventada por el agente? ¿Cómo lo detectaron?

Durante la fase inicial de Discovery, el agente generó propuestas sobredimensionadas y descontextualizadas respecto de la realidad operativa de un consultorio odontológico mediano en Argentina. En particular, detectamos tres puntos críticos de información no verificable o alucinada:

1. **Integración con API oficial de WhatsApp Cloud Business y pasarelas de pago internacionales:** El agente asumió como requisito mandatorio la integración directa con Meta for Business (con costos fijos en dólares por mensaje saliente) y cobro online previo mediante Stripe. Al contrastarlo con el benchmark de consultorios reales y nuestro modelo de negocio local, detectamos que las clínicas barriales operan con cobro en mostrador (efectivo/transferencia) y no incurren en costos de mensajería empresarial para recordatorios básicos.
2. **Telemedicina / Consultas virtuales:** El agente propuso un módulo de videollamadas para atención odontológica remota. Como equipo detectamos inmediatamente la inviabilidad funcional del requerimiento: el acto odontológico es eminentemente físico, manual e instrumental (inspección de piezas, operatoria, radiología). 
3. **Cálculo estático de turnos sin contemplar duración de tratamientos:** Inicialmente el modelo propuso una grilla fija de turnos de 15 minutos idéntica para cualquier especialidad, omitiendo que procedimientos quirúrgicos o de conducto demandan bloques variables (30 a 60 minutos).

**¿Cómo lo detectamos?**  
Lo detectamos mediante la confrontación crítica en el checklist de Discovery: cruzamos las suposiciones de la IA contra entrevistas reales de operatoria odontológica y definimos supuestos explícitos en `knowledge-base/09_decisiones_y_supuestos.md` (SU-01 y SU-02), recortando el alcance hacia un MVP sólido con adaptadores simulados (mock/log para WhatsApp) y notificaciones por correo electrónico SMTP estándar.

---

## 2. ¿En qué etapa usaron *Ajustar*, qué estaba mal y qué habría pasado si hubieran apretado *Continuar*?

Utilizamos la acción **Ajustar** de forma activa en dos momentos bisagra del ciclo de especificación (OpenSpec):

### Caso A: En el Propose de C-06 (`turnos-agenda-backend`)
* **Qué estaba mal:** En la propuesta original de la IA, la validación de colisiones de turnos se resolvía en la capa de servicio mediante una simple consulta booleana en memoria (`findByOdontologoAndFechaHora`).
* **Qué habría pasado si hubiéramos apretado Continuar:** En un entorno concurrente donde dos recepcionistas o pacientes intentan reservar al mismo profesional en el mismo horario simultáneamente, ambas transacciones hubieran leído la base de datos como "libre" antes de escribir, produciendo una condición de carrera (*race condition*) y registrando turnos duplicados para el mismo odontólogo.
* **El ajuste aplicado:** Rechazamos el diseño preliminar y exigimos incorporar un **bloqueo pesimista a nivel de base de datos (`PESSIMISTIC_WRITE`)** con índice compuesto y respuesta controlada `409 Conflict`. Esto garantizó consistencia ACID absoluta ante accesos concurrentes.

### Caso B: En el Propose de C-08 (`historia-clinica-evoluciones`)
* **Qué estaba mal:** El agente generó un diseño estándar con endpoints de CRUD completo para las notas de evolución clínica, incluyendo operaciones `PUT` (modificar) y `DELETE` (borrar).
* **Qué habría pasado si hubiéramos apretado Continuar:** Habríamos violado la legislación sanitaria vigente (Ley 26.529 de Derechos del Paciente e Historia Clínica en Argentina), la cual estipula taxativamente que los asientos médicos son documentos legales inalterables que no pueden ser borrados ni modificados una vez asentados.
* **El ajuste aplicado:** Modificamos la especificación para imponer **inmutabilidad clínica estricta**: la entidad se anotó con `@Immutable`, se bloquearon mutaciones en JPA y se restringió la creación exclusivamente a usuarios autenticados con rol `ODONTOLOGO`.

---

## 3. ¿Qué diferencia notaron entre implementar con una especificación aprobada y pedir código con un prompt suelto?

La diferencia entre ambos enfoques fue la línea divisoria entre el caos y el desarrollo profesional predecible:

* **Con un prompt suelto:** En pruebas informales previas, observamos que pedir código directo genera soluciones aisladas: el modelo inventa nombres de atributos (ej. `fecha_turno` en un archivo y `appointmentDate` en otro), usa tipos de datos dispares (timestamp numérico vs. ISO-8601), ignora patrones arquitectónicos (mete lógica de negocio en controladores) y genera dependencias circulares. Además, ante la falta de contratos, cada integrante del equipo recibía código incompatible con el de sus compañeros.
* **Con una especificación aprobada (OpenSpec + TDD):**
  1. **Contratos innegociables:** El DTO de entrada, el DTO de salida y los códigos HTTP estaban fijados en el `proposal.md` y `specs/` antes de codificar.
  2. **Eliminación del código innecesario (*bloatware*):** Al regirnos por Strict TDD, la IA solo tenía permitido escribir el código estrictamente necesario para hacer pasar los tests unitarios e integrales que reflejaban los escenarios del spec.
  3. **Paralelismo real:** Diego pudo implementar pacientes (C-05) mientras Tomás implementaba profesionales (C-04) sin romper nada, porque ambos se acoplaban a los contratos del Core (C-02) sin ambigüedades.

---

## 4. ¿Qué parte de la fundación (knowledge-base, roadmap, reglas) le aportó más valor al ciclo OPSX y cuál menos?

### Lo que más valor aportó:
1. **El Roadmap de dependencias (`CHANGES.md`):** Fue el mapa de navegación operativo. Establecer claramente el camino crítico (`C-01 → C-02 → C-03 → C-04 → C-06 → C-07`) y los puntos de sincronización (*gates*) evitó que los integrantes intentaran construir interfaces sobre servicios que todavía no tenían contratos estables.
2. **Las Reglas Duras (`AGENTS.md`):** Instrucciones como *"TDD estricto obligatorio"*, *"Prohibido exponer entidades JPA en controladores"* y *"Respuestas uniformes vía GlobalExceptionHandler"* actuaron como un compilador de buenas prácticas para la IA, evitando regresiones en los 116 tests del sistema.

### Lo que menos valor aportó:
* **Los documentos descriptivos narrativos de la Knowledge Base (ej. `10_preguntas_abiertas.md` o secciones genéricas de visión):** Si bien fueron útiles para el arranque conceptual, una vez que el proyecto entró en la fase de implementación de código, la IA rara vez consultaba reflexiones teóricas abiertas. Toda la fuerza operativa residió en los contratos de datos (`05_modelo_de_datos.md`), las reglas de negocio (`06_reglas_de_negocio.md`) y las especificaciones atómicas de cada change en `openspec/`.

---

## 5. En grupo: qué hizo cada integrante y en qué etapa

El trabajo se coordinó mediante el modelo de **Git Forking Workflow**, donde cada integrante trabajó en su repositorio bifurcado y submitted Pull Requests hacia el repositorio central de la organización:

```
                                  ┌── C-04 [Tomas] ──┐
                                  │                  ├── C-06 [Gaspar] ──┬── C-07 [Facundo] (Frontend)
C-01 [Seba] ──► C-02 [Seba] ──► C-03 [Tomas]         │                   ├── C-08 [Diego]   (Clínica)
                                  └── C-05 [Diego] ──┘                   └── C-09 [Gaspar]  (Notificaciones)
```

| Integrante | Rol | Etapa / Changes Asignados | Tareas Clave Desarrolladas |
| :--- | :--- | :--- | :--- |
| **Sebastián Gómez** | Lead Developer / Tech Lead | **Fase 0: Cimientos y Core**<br>• `C-01` foundation-setup<br>• `C-02` core-models | Setup del monorepo, configuración de Docker Compose (PostgreSQL 15), Gradle y TypeScript. Creación de `AuditableEntity` (auditoría y soft-delete), entidad `Usuario`, `GlobalExceptionHandler` y seed inicial. Auditoría y merge de los PRs #5 al #11, resolución de conflictos de ramas y persistencia de memoria Engram. |
| **Tomás Ruti** | Backend Developer | **Fase 1: Seguridad y Catálogo**<br>• `C-03` auth-jwt<br>• `C-04` profesionales-service | Arquitectura de seguridad con Spring Security 6 stateless y JWT HMAC-SHA256. Filtro de autorización con denegación 403 a cuentas inactivas. Entidad `Odontologo`, catálogo público sin auth y CRUD administrativo protegido. *(PR #5 y PR #6)*. |
| **Diego González** | Backend Developer | **Fase 1 y 2: Pacientes y Clínica**<br>• `C-05` pacientes-service<br>• `C-08` historia-clinica-evoluciones | Entidad `Paciente` con vinculación de historia clínica automática y soft-delete. Módulo legal de `EvolucionClinica` con inmutabilidad estricta (`@Immutable`), restricción RBAC para odontólogos y transición automática de turnos a `ATENDIDO`. *(PR #7 y PR #10)*. |
| **Gaspar González** | Backend Developer | **Fase 2: Agenda y Eventos**<br>• `C-06` turnos-agenda-backend<br>• `C-09` notificaciones-turnos | Motor de agenda con cálculo de slots disponibles y bloqueo pesimista (`PESSIMISTIC_WRITE`) anti-colisión. Arquitectura desacoplada de notificaciones con `@EventListener` asíncrono y adapters tolerantes a fallos para Email y WhatsApp mock. *(PR #8 y PR #9)*. |
| **Facundo López Romera** | Frontend Specialist | **Fase 2: Interfaz Web y UX**<br>• `C-07` agenda-web-frontend | Arquitectura cliente en TypeScript vanilla modular y HTML5/CSS3 consumiendo tokens de diseño. Vistas de `login.html`, `registro.html` y `dashboard.html`. Cliente HTTP tipado con Bearer JWT, calendario interactivo de turnos, modales accesibles y cancelación en vivo. *(PR #11)*. |
