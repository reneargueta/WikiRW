# Estructura de Archivos en Wiki.js (Git Storage)

**Propósito:** Entender cómo Wiki.js traduce el path de una página a una ruta de archivo real dentro del repositorio Git, para poder planificar la taxonomía del wiki (carpetas, subpáginas, jerarquía) antes de migrar contenido.

---

## Regla fundamental

Cuando Wiki.js usa Git como storage, cada página se guarda como un archivo cuya ruta es exactamente el **path de la página + la extensión del tipo de contenido**:

```
{page.path}.{extensión}
```

Ejemplo: una página con path `research-pocs/token-economics` y contenido en markdown se guarda como:
```
research-pocs/token-economics.md
```

No hay una capa de mapeo intermedia ni una base de datos separada llevando el registro de "dónde vive cada página" — **la estructura de carpetas del repositorio ES la jerarquía de navegación del wiki**.

---

## Wiki.js no tiene "carpetas" en el sentido tradicional

Un detalle importante de cómo funciona Wiki.js: **nunca necesitas crear una carpeta explícitamente**. Si creas una página con path `/universo/planetas/tierra`, Wiki.js infiere automáticamente que existen los niveles `/universo` y `/universo/planetas`, sin que tú tengas que crear nada de forma manual — el sistema los deriva directamente del path.

Esto le da flexibilidad, pero significa que la jerarquía visual (breadcrumbs, árbol de navegación) depende completamente de cómo nombres los paths, no de una estructura de carpetas que administres por separado.

---

## Cómo crear una "página padre" con subpáginas debajo

Este es el patrón más importante para tu taxonomía. Si quieres una sección con:
- Una página de aterrizaje/resumen (ej. "Research & POCs")
- Varias subpáginas debajo (ej. "Token Economics", "Traceability")

La estructura de archivos queda así:

```
research-pocs.md                    ← página padre (path: /research-pocs)
research-pocs/
  ├── token-economics.md            ← subpágina (path: /research-pocs/token-economics)
  ├── traceability.md               ← subpágina (path: /research-pocs/traceability)
  └── ai-assistant-overview.md      ← subpágina (path: /research-pocs/ai-assistant-overview)
```

Esto **no genera conflicto** en el filesystem porque `research-pocs.md` (archivo) y `research-pocs/` (carpeta) son dos cosas distintas — una tiene extensión, la otra no.

**Por qué conviene crear siempre la página padre:** si no existe `research-pocs.md`, al hacer clic en el breadcrumb "research-pocs" desde cualquier subpágina, el usuario cae en una página inexistente/vacía. La documentación oficial de Wiki.js recomienda explícitamente crear esta página "de aterrizaje" por esa razón.

---

## Prefijo de idioma (locale)

Si una página está en el idioma **default** configurado en la instancia, el path se guarda tal cual, sin prefijo.

Si una página está en un idioma **distinto** al default, Wiki.js antepone el código de idioma como carpeta:

```
es/research-pocs/token-economics.md
```

Para un wiki con contenido en un solo idioma (el caso más probable para el BAP), esto no afecta en la práctica — pero es relevante si en el futuro se agrega contenido bilingüe.

---

## Limitación conocida: navegación de subpáginas en el sidebar

Un problema reportado consistentemente por usuarios de Wiki.js: el panel de navegación lateral **no muestra automáticamente las subpáginas (hijos)** cuando estás parado en la página padre — solo muestra los "hermanos" (siblings) al mismo nivel. Hay que hacer clic explícito para expandir el árbol y ver los hijos.

**Por qué esto importa para la evaluación:** es exactamente el tipo de fricción de navegación que un sistema puramente jerárquico puede generar — algo a tener en cuenta si Wiki.js termina siendo la recomendación final, ya que la experiencia de "explorar" el wiki no es tan fluida como en herramientas como Confluence o BookStack.

---

## Limitación conocida: imágenes y assets con rutas relativas

Otro gotcha real y bastante reportado: **las imágenes no soportan rutas relativas a la carpeta del archivo `.md`** de la forma en que uno esperaría (al estilo GitHub/GitLab, donde una imagen junto al archivo markdown simplemente funciona).

- Una imagen colocada junto al `.md` se ve bien mientras trabajas en local, pero **se rompe al hacer push al repositorio remoto**.
- La única forma soportada actualmente es subir las imágenes a una carpeta de assets a **nivel raíz** del repo, no junto a cada página.

**Por qué esto importa:** si el contenido migrado incluye diagramas, capturas de pantalla o imágenes de apoyo (como las que ya usamos en la guía de troubleshooting de GitHub sync), hay que planificar una carpeta de assets centralizada desde el inicio, en lugar de asumir que cada sección puede tener sus propias imágenes junto a sus páginas.

---

## Resumen práctico para la taxonomía del BAP

| Necesitas | Cómo hacerlo |
|---|---|
| Una sección top-level (ej. "Onboarding") | Crear página con path `onboarding` |
| Contenido dentro de esa sección | Crear páginas con path `onboarding/algo` |
| Que el breadcrumb de las subpáginas funcione | Asegurarte de que exista una página en el path padre (`onboarding.md`) |
| Jerarquía de varios niveles | Simplemente usar más segmentos en el path (`onboarding/pega/setup-inicial`) — no hace falta "crear carpetas" |
| Imágenes/diagramas en las páginas | Subirlas a una carpeta de assets a nivel raíz del repo, no junto a cada `.md` |
