# AGENTS.md — Viborita LCD

Juego de la viborita para un parcial de programación. **No hay `package.json`, ni build,
ni tests, ni linter, ni CI.** Son cuatro archivos planos y versionados; nada se transpila
y nada se instala.

## Verificación

`game.js` se carga con `defer`, no usa módulos ES ni `fetch`: **basta con abrir
`index.html` en el navegador** (`file://`, doble clic). No hay servidor, no hay paso previo.

Como no hay red de seguridad automática, la revisión manual del comportamiento es el único
control de calidad. Si tocas puntuación, record o el estado de la partida, verifícalo jugando.

## Arquitectura y acoplamiento

`index.html` contiene todo el DOM (el aparato + la ficha lateral), `game.js` es un único
IIFE con `"use strict"` (365 líneas, sin módulos ni exports), `styles.css` lleva los tokens
`:root` y clases BEM (`bloque__elemento`, `bloque--modificador`).

Tres acoplamientos que fallan en silencio:

- **`game.js` resuelve el DOM una sola vez** en el nivel superior del IIFE
  (`game.js:18-33`, antes de `arrancar()`). Todo `id` que use tiene que existir en
  `index.html`; renombrar uno no da error de compilación, revienta en el primer pintado.
- **El tamaño del lienzo vive en el HTML**, no en el JS. `width=488 height=344`
  (`index.html:28`) es exactamente `COLS*CELDA + MARCO*2` × `ROWS*CELDA + MARCO*2`
  (`20*24+8` × `14*24+8`). Si cambías `COLS`, `ROWS`, `CELDA` o `MARCO` (`game.js:8-11`),
  actualiza también el `<canvas>` o todo el dibujo se desplaza.
- **El canvas lee sólo tres variables CSS** (`game.js:35-40`): `--lcd`, `--lcd-apagado`,
  `--tinta-lcd`. Si las renembras, el lienzo cae callado a los colores de reserva
  hardcodeados. Ninguna otra variable de `:root` llega al canvas.

## Dónde vive cada regla

- **Puntuación y velocidad**: en un solo sitio, dentro de `paso()` (`game.js:154-159`) —
  `puntos += 10 * nivel`, `nivel = floor(manzanas/5)+1`,
  `tickMs = max(70, 150 - manzanas*3)`.
- **Récord**: `localStorage["viborita-lcd.mejor"]`, siempre en `try/catch`
  (`leerRecord`/`guardarRecord`). Se lee sólo al arrancar y se escribe sólo al perder;
  no hay guardado continuo.
- **Estados**: `estado` ∈ `"listo" | "jugando" | "pausa" | "fin"`. Toda transición pasa por
  `comenzar()`, `alternarPausa()` o `perder()`.
- **El bucle nunca se detiene**: `cuadro()` se reprograma siempre (`game.js:279-295`) y
  `dibujar()` corre en cada frame, jugando o no. Los parpadeos se derivan del timestamp
  `ahora`, no del estado.
- **`prefers-reduced-motion`**: el flag `quieto` (`game.js:42`) apaga el parpadeo de la
  manzana y el de la muerte. Cualquier animación nueva debe respetarlo.

## Al agregar UI

- Botón en el aparato → `.tecla` (más `.tecla--blanda` si es de texto).
- Botón en la **ficha lateral** → **no** reutilices `.tecla`: esa clase es el hardware
  oscuro del aparato. La ficha se construye con `--regla`, `--suelo-veta`, `--tinta-suave`
  y mayúsculas de 11px. `.ficha` ya es `grid` con `gap`, así que un elemento nuevo entre
  `dl.datos` y `section.controles` no necesita márgenes propios.
- Los botones de dirección se cablean con un bucle sobre `[data-dir]`; uno nuevo con ese
  atributo queda conectado solo.
- **Atajo global vs. botón**: el `keydown` está en `document` y los botones de dirección
  hacen `stopPropagation()` en Enter/Espacio para no dispararse dos veces
  (`game.js:317-329`). Un botón nuevo con atajo debe hacer lo mismo.
- HUD (`#marcador`, `#record`) y tabla de datos (`d-*`) se pintan juntos y sólo desde
  `pintarDatos()`: añade la fila en `index.html` **y** su entrada en el objeto `salida`
  (`game.js:25-33`).

## Estilo del código

- Comentarios, `id`, clases y textos de UI **en español**.
- 2 espacios, punto y coma siempre, `const`/`let` sin `var`.
- Constantes de ajuste en MAYÚSCULAS agrupadas arriba (`game.js:8-16`); helpers de una
  línea como flecha (`enMalla`, `cifras`).
- Todo dibujo pasa por `punto(x, y, w, h)` (`game.js:198`), que aplica el `MARCO`.
- Secciones comentadas: `/* ---------- nombre ---------- */` en CSS,
  `/* ---------------- nombre ---------------- */` en JS.

## Tarea vigente: `CAMBIOS.md`

`CAMBIOS.md` es el enunciado del parcial: tres features, **cada una en su propia rama y su
propio worktree, en paralelo**, integradas al final en `main`.

- `feature/borrar-record` — botón "Borrar mejor marca" bajo la tabla de datos, con
  confirmación; pone el récord en 0 en pantalla y en `localStorage`.
- `feature/manzana-dorada` — la manzana 7, 14, 21... vale el triple, se dibuja hueca
  (sólo contorno) y no parpadea.
- `feature/racha` — vale el doble si se come 15 pasos o menos después de la anterior; la
  primera manzana de la partida nunca cuenta como racha.

Crear los worktrees desde la raíz del repo:

```bash
git worktree add ../vib-borrar-record  -b feature/borrar-record
git worktree add ../vib-dorada        -b feature/manzana-dorada
git worktree add ../vib-racha         -b feature/racha
```

Regla de integración: si una manzana es dorada y además llega en racha, los multiplicadores
se multiplican (x6). Ojo: "pasos" son invocaciones de `paso()`, no milisegundos — así que
el contador de la racha se avanza en el mismo `paso()` que la colisión.

## Integración: choques ya analizados

Las tres ramas nacen del mismo commit y no hay `.gitattributes`: git fusiona por defecto
y todo conflicto se resuelve a mano.

- `feature/borrar-record` es ortogonal: inserta un botón entre `</dl>` y
  `<section class="controles">` (`index.html:64-66`) y un listener en la sección
  "mandos" de `game.js`. Mézclala primero o última, da igual.
- `feature/manzana-dorada` y `feature/racha` **colisionan en `paso()`**
  (`game.js:140-163`): ambas necesitan la condición de colisión (`:154`),
  `puntos += 10 * nivel` (`:157`) y `comida = celdaLibre()` (`:159`). Constantes
  (`:8-16`) y estado (`:77-80`) también se tocan, pero son choques mecánicos.
- **El peligro no lo marca git**: cada rama dejará la línea de puntos en `* 3` o en
  `* 2`; integradas como dos merges secuenciales, la segunda pisa la primera y el árbol
  queda limpio pero puntúa mal. Integra dorada + racha **en un solo commit manual** con
  multiplicadores explícitos (`puntos += 10 * nivel * multiplicador`), sin alterar el
  orden existente nivel→puntos.
- Verificación post-merge, jugando: la 7ª manzana sola (x3), dos seguidas (x2) y la 7ª
  en racha (x6).
