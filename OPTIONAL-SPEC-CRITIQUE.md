# Lección Opcional: Crítica y Validación de Documentos de Especificación

> Esta lección es independiente del flujo principal de 3 horas. Cubre cómo usar los comandos `/speckit.analyze` y `/speckit.checklist` para detectar errores, incongruencias y huecos en los documentos generados durante el taller.

## Por qué importa esta lección

Un riesgo real del Spec-Driven Development con IA es asumir que los documentos generados son correctos solo porque "se ven bien". Los archivos `spec.md`, `plan.md` y `tasks.md` pueden tener problemas que no son evidentes a simple vista: una regla de negocio definida en la especificación que el plan técnico no contempla, tareas que no cubren todos los componentes del plan, o decisiones arquitectónicas que contradicen la constitución.

Spec Kit incluye dos comandos built-in diseñados para esto. No son extensiones ni plugins externos, vienen con la instalación base.

---

## Comando 1: `/speckit.analyze`

### Qué hace

Realiza un análisis de consistencia cruzada entre todos los artefactos de una feature: `constitution.md`, `spec.md`, `plan.md` y `tasks.md`. Produce un reporte de solo lectura (no modifica ningún archivo) que identifica:

* **Inconsistencias:** Decisiones contradictorias entre artefactos. Por ejemplo, la spec dice "validar montos negativos" pero las tareas no incluyen un paso para implementar esa validación.
* **Huecos de cobertura:** Componentes definidos en el plan que no tienen una tarea correspondiente, o reglas de negocio en la spec que no aparecen reflejadas en el plan.
* **Violaciones constitucionales:** Cualquier decisión en plan o tasks que contradiga las reglas definidas en la constitución. Estas se marcan como hallazgos CRITICAL.
* **Ambiguedades:** Términos o requisitos que se usan de forma distinta entre documentos.
* **Duplicaciones:** Tareas o requisitos repetidos que podrían generar trabajo doble o conflictos.

### Cuándo ejecutarlo

Después de `/speckit.tasks` y antes de `/speckit.implement`. Es el punto del flujo donde todos los artefactos de especificación ya existen pero todavía no hay código. Encontrar problemas aquí es barato; encontrarlos durante la implementación significa rehacer trabajo.

### Cómo usarlo en el taller

En el chat de GitHub Copilot, escribe:

```
/speckit.analyze
```

Sin argumentos adicionales. El comando lee automáticamente los artefactos del feature activo.

Si quieres dirigir el análisis a un área específica, puedes agregar contexto:

```
/speckit.analyze Enfócate en verificar que todas las reglas de negocio de transferencias 
definidas en spec.md tienen cobertura completa en tasks.md
```

### Ejemplo de lo que produce

El reporte tendrá una estructura similar a esta (el formato exacto depende de la versión de Spec Kit y el modelo que uses):

```
## Reporte de Análisis de Consistencia

### Hallazgos CRITICAL
- [CONSTITUCIÓN] La constitución exige pruebas unitarias obligatorias, pero 
  tasks.md no incluye una tarea específica para tests del endpoint de consulta 
  de saldo (solo cubre transferencias).

### Hallazgos WARN
- [COBERTURA] spec.md define la regla "No permitir transferir a la misma cuenta", 
  pero plan.md no menciona dónde se implementará esta validación (¿Service? ¿Controller?).
- [AMBIGUEDAD] spec.md usa "cuenta" y plan.md usa "account" indistintamente 
  en los nombres de endpoints, lo cual es correcto dado la política de idioma, 
  pero podría confundir durante la implementación.

### Hallazgos INFO
- [DUPLICACIÓN] La restricción "sin base de datos" aparece en spec.md, plan.md 
  y tasks.md. Es redundante pero no dañina.
```

### Qué hacer con el reporte

Los hallazgos CRITICAL se deben resolver antes de implementar. Puedes pedirle a Copilot que actualice el artefacto afectado:

```
@workspace El análisis de /speckit.analyze encontró que tasks.md no incluye 
una tarea para pruebas unitarias del endpoint de consulta de saldo. 
Agrega una subtarea dentro de la Tarea 5 que cubra este caso.
```

Los hallazgos WARN son juicio del equipo. Algunos vale la pena resolver, otros son aceptables para un MVP.

---

## Comando 2: `/speckit.checklist`

### Qué hace

Genera checklists de calidad personalizados que validan completitud, claridad y consistencia de los requisitos dentro de un dominio específico. A diferencia de `/speckit.analyze` que compara artefactos entre sí, `/speckit.checklist` evalúa la calidad de un solo artefacto contra criterios de un dominio concreto.

Los autores de Spec Kit lo describen como "pruebas unitarias para texto en inglés" (o español, en nuestro caso): criterios objetivos y medibles para evaluar la calidad de una especificación antes de que se genere código.

### Cuándo ejecutarlo

Después de `/speckit.specify` o `/speckit.plan`, antes de avanzar al siguiente paso. Es especialmente útil cuando la especificación cubre un dominio regulado (banca, salud, gobierno) donde la completitud no es negociable.

### Cómo usarlo en el taller

En el chat de GitHub Copilot, escribe el comando con el dominio que quieres validar:

```
/speckit.checklist Valida que la especificación incluye todos los requisitos 
de seguridad necesarios para una API bancaria, incluyendo validación de inputs, 
manejo de errores y trazabilidad.
```

Otro ejemplo orientado a la experiencia del desarrollador:

```
/speckit.checklist Verifica que el plan técnico es suficientemente claro para 
que un desarrollador junior pueda implementar cada componente sin ambiguedad. 
Evalúa: nombres de clases, rutas de archivos, contratos de API, y manejo de errores.
```

### Ejemplo de lo que produce

```
## Checklist de Validación: Seguridad API Bancaria

- [x] Validación de inputs numéricos (montos negativos, cero)
- [x] Validación de cuenta origen != cuenta destino
- [x] Validación de saldo suficiente
- [ ] Manejo de errores: No se define el formato estándar de respuesta 
      para errores (¿ProblemDetails? ¿Custom?)
- [ ] Trazabilidad: La constitución exige Correlation ID pero spec.md 
      no lo menciona en los contratos de respuesta
- [x] Seed data con cuentas de prueba definidas
- [ ] Rate limiting: No mencionado. Para un lab no es necesario, 
      pero debería documentarse como decisión explícita de exclusión.
```

### Refinamiento iterativo

El checklist está diseñado para un ciclo de "identificar, corregir, revalidar". Después de corregir los items pendientes, puedes volver a correr el mismo checklist para verificar:

```
/speckit.checklist Re-evalúa el checklist de seguridad anterior 
después de los cambios aplicados a spec.md
```

---

## Combinando ambos comandos

El flujo recomendado para una validación completa es:

1. Generar spec.md con `/speckit.specify`
2. Correr `/speckit.checklist` sobre spec.md para validar completitud de dominio
3. Corregir los huecos encontrados
4. Generar plan.md con `/speckit.plan`
5. Correr `/speckit.checklist` sobre plan.md para validar claridad técnica
6. Generar tasks.md con `/speckit.tasks`
7. Correr `/speckit.analyze` para validar consistencia cruzada entre los tres artefactos
8. Resolver hallazgos CRITICAL
9. Proceder con `/speckit.implement`

Para el taller de 3 horas esto sería excesivo. La sugerencia práctica es correr `/speckit.analyze` una sola vez después de generar `tasks.md` (entre la Parte 5 y la Parte 6), dedicándole unos 10 minutos. Es suficiente para demostrar el valor sin romper el ritmo.

---

## Prerrequisitos

Ninguno adicional. Ambos comandos vienen incluidos en Spec Kit desde la versión 0.8.x. Si tu instalación es reciente (la que se usa en el taller principal), ya los tienes.

Validar disponibilidad:

```bash
# Después de specify init, los comandos ya están registrados.
# Verificar que existan los templates:
ls .specify/templates/commands/analyze.md
ls .specify/templates/commands/checklist.md
```

Si los archivos no existen, actualizar Spec Kit:

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git --force
specify init . --ai copilot --force
```

---

## Limitaciones a tener en cuenta

Estos comandos dependen completamente del modelo de lenguaje que estés usando en Copilot Chat. El "análisis" no es determinista: es el modelo leyendo tus documentos y emitiendo juicios. Eso implica que puede haber falsos positivos (hallazgos que no son realmente problemas) y falsos negativos (problemas reales que no detecta). No sustituyen una revisión humana; la complementan.

La propuesta abierta de Issue #1323 en el repositorio de Spec Kit pide un comando `/speckit.review` que actuaría como quality gate final integrando análisis de constitución, código muerto, drift de spec y métricas de cobertura. A la fecha no está implementado, pero vale la pena seguirle la pista.

---

## Tiempo estimado

| Actividad | Tiempo |
| --- | --- |
| Correr `/speckit.analyze` y leer el reporte | 5 min |
| Discutir hallazgos y decidir qué corregir | 5 min |
| Correr `/speckit.checklist` (opcional, si hay tiempo) | 5 min |

Total: 10-15 minutos.
