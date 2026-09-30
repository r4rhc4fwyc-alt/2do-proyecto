# Generador de sigilos

Aplicación de una sola página (`index.html`, HTML+CSS+JS puro, sin
frameworks ni dependencias) que reduce una frase a un sigilo simbólico
y lo deja trazar a mano sobre dos geometrías distintas.

Este archivo existe para que una sesión nueva de Claude Code entienda
el proyecto sin tener que releer el historial completo de la
conversación donde se construyó. El código en sí también tiene
comentarios explicando el "por qué" de cada decisión no obvia — este
documento es el mapa de más alto nivel.

## Contexto del repositorio

El README todavía dice "Somno app" porque ese era el plan original del
repo; el dueño decidió usarlo para este proyecto en su lugar y prefirió
dejar el README como está por ahora (no tocarlo a menos que lo pida).

## Objetivo pedagógico

El usuario está aprendiendo a usar Claude Code mientras se construye
esto. Cuando se agregan features nuevas, conviene explicar en el chat
qué archivo se toca y qué hace cada fragmento antes de que apruebe,
no solo aplicar el cambio en silencio.

## Cómo probar cambios

- No hay build ni servidor: `index.html` se abre directo en un
  navegador, o se sirve como archivo estático.
- Para probar interacción (dibujo con Pointer Events, resize, etc.) se
  usó Playwright vía `NODE_PATH=$(npm root -g) node -e "..."` con
  `chromium.launch({executablePath: '/opt/pw-browsers/chromium'})`,
  screenshots incluidas, en vez de asumir que el código "debería"
  funcionar. Vale la pena seguir haciendo esto para cambios en el
  canvas o en la interacción táctil.
- Para que el usuario probara en Safari/iPad real durante el
  desarrollo (antes de mergear), se publicó una copia como Artifact de
  Claude (ver más abajo). El repo también tiene GitHub Pages activado
  en `main` → https://r4rhc4fwyc-alt.github.io/2do-proyecto/ — esa es
  la forma "real" de compartir la app (pública, sin login, funciona
  para cualquiera).

### Publicar/actualizar la copia de prueba en Artifacts

Cuando el usuario pide "probar en el Artifact": el Artifact tool no
acepta un documento HTML completo (con `<!DOCTYPE>`/`<html>`/`<head>`/
`<body>`) — solo el contenido interno (un `<title>`, un `<style>`, el
resto del body, y el `<script>`), porque el resto lo agrega el propio
sistema. Hay que copiar `index.html`, sacar esas etiquetas envolventes
y el `<meta viewport>` (el Artifact pone el suyo), guardar el
resultado como archivo aparte en el scratchpad, y publicarlo con
`url` apuntando al Artifact ya existente para que actualice el mismo
link en vez de crear uno nuevo. El Artifact es solo para pruebas
rápidas del propio usuario — el `index.html` del repo es la fuente de
verdad.

## Decisiones de arquitectura (el porqué, no solo el qué)

### Dos canvases superpuestos, no uno

`bgCanvas` (la rueda/kamea de fondo, se redibuja seguido) y
`drawCanvas` (transparente, encima, solo para el trazo del usuario)
son elementos `<canvas>` separados. Esto evita tener que "recordar y
repintar" el trazo del usuario cada vez que cambia el fondo (modo,
geometría, "Visualizar sigilo", rotación) — el trazo simplemente vive
en su propia capa y nunca se toca al redibujar el fondo.

### Los trazos se guardan como puntos, no como píxeles ya pintados

Cada trazo es un array de puntos `{x, y}` (no solo lo que quedó
pintado en el canvas). Esto es lo que permite: deshacer trazo por
trazo (se hace `pop()` del array y se repinta el resto), rotar/invertir
sin perder calidad, y sobrevivir a un resize.

### Coordenadas normalizadas (0–1), no píxeles absolutos

Un punto se guarda como fracción del ancho del canvas
(`toNormalizedPoint`/`toPixelPoint`), no en píxeles fijos. Antes se
guardaban en píxeles y cualquier cambio de tamaño de ventana borraba
todo (las coordenadas viejas ya no significaban nada); con
coordenadas normalizadas, un resize simplemente reescala el dibujo.
El canvas siempre es cuadrado (`aspect-ratio: 1/1`), así que una sola
división/multiplicación por `size` no deforma nada.

### Transform (rotación/inversión) no destructivo

`transformByGeoMode[geometria][modo] = {angle, flipH, flipV}` se
aplica solo al MOMENTO DE DIBUJAR (`applyTransform` en
`redrawStrokes`), nunca se "hornea" en los puntos guardados. Al
capturar un trazo nuevo mientras ya hay una rotación activa, se aplica
la transformación INVERSA antes de guardar (`inverseTransform`), para
que lo que ves dibujarse bajo el dedo sea siempre WYSIWYG y la
rotación nunca se duplique.

### Geometría (rueda/kamea) × Modo (guiado/libre) = 4 cajones independientes

`strokesByGeoMode` y `transformByGeoMode` tienen la forma
`{ rueda: {guiado, libre}, kamea: {guiado, libre} }`. Cada una de las
4 combinaciones tiene su propio historial de trazos y su propio
transform, totalmente independiente. Esto es lo que permite cambiar
libremente entre cualquiera de las 4 para comparar, sin bloqueos ni
pérdida de datos. (Hubo una versión anterior con bloqueo de geometría
mientras hubiera trazos — se sacó por pedido explícito del usuario en
favor de esta independencia total.)

### El código de la rueda nunca se tocó al agregar el kamea

Cuando se agregó el kamea como segunda geometría, fue requisito
explícito no modificar el código que ya dibujaba la rueda.
`drawBackground()` tiene 4 líneas agregadas al principio que desvían a
`drawKameaBackground()` si la geometría es kamea y hacen `return`; el
resto de la función (el dibujo de la rueda) sigue exactamente igual
que antes, sin tocar. `computeLetterPositions` (rueda) tampoco se
tocó. El kamea es una sección aparte y autocontenida.

### Mapeo de letras: mismo índice alfabético para ambas geometrías

`ALPHA_INDEX[letra] = i + 1` (A=1, B=2, ... Ñ en su lugar entre N y O,
... Z=27), usando el mismo orden en que la rueda ya distribuye las 27
letras del alfabeto español en círculo. En el kamea, ese número es
directamente el número de celda a buscar en `KAMEA_CELL_OF_NUMBER`
(el cuadrado mágico de Mercurio, 8×8, tradición hermética — números 1
a 64, constante mágica 260 por fila/columna/diagonal).

### La Ñ nunca se confunde con la N

`reduceToSigil()` protege la Ñ con un carácter guardián antes de
`normalize('NFD')` para quitar tildes, porque esa normalización
Unicode separaría la Ñ en "N" + tilde combinante igual que separa
"É" en "E" + tilde — perdiendo la diferencia entre N y Ñ si no se
protege primero.

## Historial de features (orden en que se construyeron)

1. Algoritmo de reducción (mayúsculas → sin tildes con Ñ protegida →
   solo letras válidas → sin vocales → consonantes únicas en orden).
2. Rueda: 27 letras en círculo, modo guiado (numerado + línea tenue) y
   libre (solo puntos), trazo con Pointer Events (`setPointerCapture`,
   `touch-action: none`, `preventDefault()` en `pointermove` —
   necesario para que el trazo funcione bien en iPad/Safari sin
   disparar gestos de navegación).
3. Deshacer por trazo (guardar puntos, no píxeles).
4. "Visualizar sigilo": oculta puntos/números/línea en modo guiado
   para ver el trazo limpio contra la rueda neutra.
5. Modo guiado/libre independientes entre sí (cajón de trazos propio
   por modo) para poder comparar sin bloqueos.
6. Rotación (slider 0–359°, de a un dedo, se descartó un dial circular
   a mano por ser innecesariamente complejo) + inversión horizontal/
   vertical, no destructivo.
7. Kamea de Mercurio como segunda geometría seleccionable, sin tocar
   el código de la rueda.
8. Geometría también independiente por combinación (4 cajones en vez
   de 2), sacando el bloqueo de geometría.
9. Coordenadas normalizadas para que el resize de ventana no borre el
   trazo.

## Flujo de trabajo de git en este proyecto

- Rama de trabajo: `claude/admiring-bardeen-bb1fl6`.
- El PR #1 (con todo el historial de features de arriba) ya se
  mergeó a `main`. Cualquier trabajo nuevo arranca la rama de nuevo
  desde el `main` actualizado (no se apila sobre el PR ya mergeado) y
  termina en un PR nuevo.
- GitHub Pages está activo en `main` /root:
  https://r4rhc4fwyc-alt.github.io/2do-proyecto/
