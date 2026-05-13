# ADR-002 – Validación y parametrización de consultas MCP a movimientos

- Estado: proposed
- Fecha: 2026-05-13
- Contexto del curso: Desarrollo Seguro de Software – agente de reportes conversacionales del banco digital.

## Contexto

El producto es un banco digital con app y web en producción que expone un agente conversacional capaz de responder preguntas en lenguaje natural sobre los movimientos del cliente. El agente usa un LLM que interpreta el prompt, llama herramientas vía MCP contra un backend con acceso a la base de datos de movimientos y luego compone una respuesta textual.

En la especificación ejecutable de la clase 2 se fijó como invariante de seguridad que ningún movimiento de otro cliente puede aparecer en la respuesta, jamás. En el threat model de la sesión 3 se identificaron amenazas de elevación de privilegios y divulgación de información asociadas a la construcción dinámica de consultas a movimientos a partir del texto del usuario. Entre ellas destacan el cruce de cuentas (pedir movimientos de otro cliente), la inyección de instrucciones hostiles en el prompt y el abuso de herramientas MCP con permisos excesivos. Estas amenazas se mapearon a STRIDE (Elevation of Privilege, Information Disclosure), OWASP Top 10 para LLM (por ejemplo, LLM01: Prompt Injection, LLM05: Sensitive Information Disclosure) y OWASP Agentic AI (tool abuse, over‑permissioned tools).

Hoy el agente podría delegar al modelo la construcción de filtros y condiciones directamente sobre el SQL o sobre la interfaz MCP, lo que abre la puerta a inyecciones, consultas no auditadas y bypass de controles de autorización si el backend expone endpoints demasiado genéricos. Además, la presión por entregar respuestas “útiles” con lenguaje natural puede empujar al equipo a aceptar caminos rápidos (vibe coding) donde la capa de datos queda subordinada al output del LLM en vez de a contratos explícitos.

Las fuerzas en juego son: costo de ingeniería para definir vistas y endpoints específicos, fricción para el equipo de datos y backend al tener que versionar esquemas y contratos, impacto en latencia por capas adicionales de validación, y dependencia de tooling de pruebas automatizadas para verificar que las consultas implementadas cumplen las invariantes de seguridad acordadas con el equipo.

## Decisión

Todas las consultas de movimientos que el agente realiza vía MCP se implementarán exclusivamente como operaciones parametrizadas sobre vistas de solo lectura, definidas y revisadas por el equipo de backend; el LLM nunca construye ni envía SQL ni un DSL de consultas a partir del prompt del usuario.

Cada nueva capacidad de reporte se incorporará como un endpoint o función backend explícito (por ejemplo, `get_movimientos_por_rango`, `get_gasto_por_categoria`), con parámetros tipados y validados, revisado en pull request y cubierto por un test que verifica que no es posible obtener movimientos de otro cliente ni ejecutar filtros arbitrarios.

La capa MCP solo expondrá estas funciones de alto nivel; los identificadores de cliente se obtendrán exclusivamente de la identidad autenticada (token/JWT) y nunca desde el texto del usuario, headers manipulables u otras fuentes no confiables, alineando esta decisión con la invariante definida en la spec ejecutable.

## Consecuencias positivas

- Reduce de forma significativa el riesgo de elevación de privilegios y exfiltración de datos al eliminar el SQL dinámico influenciado por el prompt y limitar la superficie de consulta a vistas predefinidas y revisadas.
- Facilita la verificación automática: cada endpoint MCP puede tener tests unitarios y de integración que ejerciten los parámetros límites, reproduzcan misuse stories y verifiquen la invariante “ningún movimiento de otro cliente aparece en la respuesta, jamás”.
- Mejora la trazabilidad y la auditoría, porque cada pregunta del cliente se traduce en una o más llamadas a funciones conocidas, registradas con parámetros concretos en logs de aplicación y de base de datos, lo que simplifica la investigación de incidentes.
- Alinea el diseño con buenas prácticas de DevOps: los contratos de datos se versionan en el repositorio, pasan por revisión entre pares y se validan en la pipeline de CI antes de llegar a producción, en vez de depender del comportamiento emergente del modelo.

## Consecuencias negativas

- Aumenta el esfuerzo inicial de diseño, porque el equipo debe anticipar los reportes más probables, diseñar vistas o endpoints específicos y mantenerlos en el tiempo a medida que cambian las necesidades de negocio.
- Reduce la flexibilidad del agente frente a preguntas completamente nuevas; mientras no exista un endpoint adecuado, el modelo deberá responder con una explicación honesta de limitación en lugar de improvisar una consulta ad hoc, lo que puede percibirse como menos “mágico” para el usuario.
- Introduce una dependencia fuerte en la disciplina del equipo: si alguien expone un endpoint genérico sin pasar por el mismo estándar de parametrización y revisión, se reabre el vector de ataque, por lo que esta decisión debe acompañarse de reglas claras en `AGENTS.md` y políticas de revisión de código.
- Puede incrementar la latencia o el consumo de recursos cuando un reporte complejo requiere múltiples llamadas a funciones especializadas en lugar de una sola consulta libre, lo que obliga a ajustar caching y diseño de vistas para compensar.

## Conexión con el threat model

Esta decisión responde directamente a la amenaza de cruce de cuentas y exfiltración de movimientos mediante prompt injection que induce al agente a construir consultas más amplias o a incluir identificadores de otros clientes en los filtros (STRIDE: Elevation of Privilege / Information Disclosure; OWASP LLM01, LLM05; OWASP Agentic AI – tool abuse).

El control asociado en la tabla “amenaza–control” es: “consultas a movimientos solo vía vistas parametrizadas y endpoints de solo lectura, sin SQL dinámico influenciado por el prompt del usuario”. Este control se prueba mediante un conjunto de tests automatizados que reproducen las misuse stories definidas en la especificación ejecutable (pedir movimientos de otro cliente, inyectar instrucciones en el prompt, forzar filtros arbitrarios) y verifican que la invariante de seguridad se mantiene.