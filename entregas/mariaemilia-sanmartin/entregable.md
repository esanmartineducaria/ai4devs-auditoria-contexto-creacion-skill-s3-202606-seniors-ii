# Entregable · Sesión 3 — Copilotos IA

- **Nombre / usuario:** María Emilia San Martín - esanmartineducaria
- **Fecha de entrega:** 21/06/2026
- **Repo auditado en la Parte A**: Monorepo Java de mi trabajo, SAAS 13 años productivo y en desarrollo continuo

---

## 1. Hallazgos de la auditoría (Parte A)

> 3-5 cosas que el agente **no pudo inferir** del código y que tendrías que decirle explícitamente.
> Redáctalas para que otra persona las entienda sin contexto adicional. **Sin código propietario ni secretos.**

1. Que trabajamos con SSR, las tecnologías (HTMX, Freemarker), y convenciones que usamos como no exponer objetos de modelo.
2. Si bien infirió que hacemos TDD no detectó que trabajamos con DDD, que hacemos foco en el lenguaje ubicuo
3. No detectó un framework propio de trabajo con tests, trabajamos de una manera particular para evitar setups de tests que nos lleven mucho tiempo.

---

## 2. SKILL.md de la skill creada (Parte B)

> Pega aquí el contenido completo de tu `.claude/skills/<nombre-skill>/SKILL.md`
> (o enlaza al archivo en tu repositorio sandbox).

```markdown
	---
	name: htmx
	description: Guía para trabajar con HTMX en templates FreeMarker. Usala cuando te pidan crear o modificar elementos interactivos (`.html`) en `/src/main/resources/templates/`.
	---

	## Principio fundamental

	**Nunca recargar la página.** Toda interacción actualiza únicamente la parte del DOM afectada. Cada acción del usuario debe tener feedback visible: un resultado actualizado, un mensaje de éxito, o un mensaje de error.

	---

	## Manejo de errores

	### Errores no manejados
	No los interceptes. Son capturados automáticamente por la infraestructura global de la aplicación. No agregues lógica de manejo para errores inesperados en los templates ni en los controllers.

	### Errores manejados (400, 500)
	Los errores de negocio se comunican devolviendo un **partial FreeMarker** con el mensaje correspondiente, usando un código HTTP 400 o 500 según el caso.

	En el template, usá siempre `hx-ext="response-targets"` con `hx-target-error` para apuntar al elemento de feedback:

	```html
	<button hx-post="/recursos/guardar"
			hx-target="#resultado"
			hx-swap="innerHTML"
			hx-ext="response-targets"
			hx-target-error="#mensaje-error">
	Guardar
	</button>
	```

	No especifiques el código HTTP en `hx-target-error` (evitá `hx-target-400`, `hx-target-500`). Usá siempre `hx-target-error` genérico.

	---

	## Targets y feedback

	### Cuándo usar un elemento fijo en el DOM
	Cuando el mensaje de feedback es contextual a una sección que siempre está visible (un formulario, un panel). El elemento receptor ya existe en el layout y simplemente se actualiza.

	```html
	<div id="mensaje"></div>

	<form hx-post="/recursos/guardar"
		hx-target="#resultado"
		hx-swap="innerHTML"
		hx-ext="response-targets"
		hx-target-error="#mensaje">
	...
	</form>
	```

	### Cuándo insertar dinámicamente
	Cuando el feedback es efímero o no hay un lugar fijo lógico para él. En ese caso el partial devuelto por el servidor incluye el elemento completo y se inserta en un contenedor.

	### Éxito y error en el mismo elemento
	Cuando el feedback es un mensaje simple (no un formulario completo ni una tabla), podés apuntar éxito y error al mismo elemento. El partial FreeMarker devuelto por el servidor incluye el estilo o clase correspondiente.

	```html
	<div id="mensaje"></div>

	<button hx-post="/recursos/eliminar"
			hx-target="#mensaje"
			hx-swap="innerHTML"
			hx-ext="response-targets"
			hx-target-error="#mensaje">
	Eliminar
	</button>
	```

	El servidor devuelve en éxito:
	```html
	<#-- _mensaje-exito.html -->
	<p class="mensaje-exito">Recurso eliminado correctamente.</p>
	```

	El servidor devuelve en error (con status 400 o 500):
	```html
	<#-- _mensaje-error.html -->
	<p class="mensaje-error">${mensaje}</p>
	```

	---

	## Estructura de partials en el servidor

	### Controller — respuesta exitosa
	```java
	@PostMapping("/guardar")
	public ModelAndView guardar(@RequestBody RecursoDto dto) {
		recursoService.guardar(dto);
		var mv = new ModelAndView("recurso/_fila");
		mv.addObject("item", dto);
		return mv;
	}
	```

	### Controller — error manejado
	```java
	@PostMapping("/guardar")
	public ModelAndView guardar(@RequestBody RecursoDto dto) {
		try {
			recursoService.guardar(dto);
			return new ModelAndView("recurso/_fila").addObject("item", dto);
		} catch (NegocioException e) {
			var mv = new ModelAndView("shared/_mensaje-error");
			mv.addObject("mensaje", e.getMessage());
			mv.setStatus(HttpStatus.BAD_REQUEST);
			return mv;
		}
	}
	```

	---

	## Reglas de diseño

	- **Elegí el target según el contexto.** No hay IDs convención globales. El selector debe ser el más específico y semánticamente correcto para cada caso.
	- **Un partial por responsabilidad.** No devuelvas HTML mezclado con lógica de múltiples secciones en un mismo partial.
	- **`hx-swap` por defecto es `innerHTML`.** Usá `outerHTML` solo cuando necesitás reemplazar el elemento contenedor completo (por ejemplo, una fila de tabla).
	- **No uses JS para manejar respuestas HTMX** salvo casos excepcionales. Si sentís que necesitás un `htmx:afterRequest` para algo, revisá si el servidor puede devolver el HTML directamente.
	- **`hx-ext="response-targets"` va siempre en el elemento que dispara la request**, no en un ancestro.

	---

	## Checklist antes de entregar un template

	- [ ] Ninguna acción recarga la página completa
	- [ ] Toda acción tiene un `hx-target` definido para el caso exitoso
	- [ ] Toda acción que puede fallar tiene `hx-ext="response-targets"` y `hx-target-error`
	- [ ] Los errores no manejados no tienen manejo explícito en el template
	- [ ] Los partials de error devuelven status 400 o 500 desde el controller
	- [ ] Los targets son selectores específicos al contexto, no genéricos globales

```

---

## 3. Diario de decisiones

*Skill creada:* skill para crear archivos de templates de mi proyecto con HTMX y Freemarker, sobre todo para siempre considerar manejo de errores consistente

*Decisiones de diseño tomadas:*
- Decisión 1: darle ejemplos
- Decisión 2: decirle que sí y que no debe hacer
- Decisión 3: traté de mantenerlo "chico" para que no ocupe tanto contexto

*Qué me resultó fácil:*
- El armado principal que lo hice con un prompt a Claude

*Qué me resultó ambiguo o difícil de decidir:*
- Cuantos ejemplos darle, si referenciar a un archivo de código real o no

*Tiempo real invertido:*
- 40 minutos

*Qué probarías si tuvieras más tiempo:*
- iteraría más veces con ejemplos reales para ir puliendo lo que no quedaba bien

*¿Usaste IA para crear la skill?* (qué partes generaste con IA y qué partes decidiste tú)
- sí, pero edité lo que no me parecía correcto

### Resultado de la prueba (Paso 8)

- ¿Se activó cuando lo esperabas? Sí
- ¿El resultado fue el que querías? No del todo
- Si no, ¿qué crees que falló? (no la "arregles" — documenta el primer intento)
