# Rasterizado del texto y `mix-blend-mode: difference`

Demo estática, sin build ni dependencias. `index.html` en la raíz: se publica en
GitHub Pages tal cual está (Settings → Pages → Deploy from a branch → root).

Tres párrafos con el mismo texto, la misma tipografía, el mismo tamaño y el mismo
peso. Lo único que cambia es cómo se pintan:

| | CSS |
|---|---|
| Referencia | `color: #000;` |
| Esquema habitual | `color: #fff; mix-blend-mode: difference;` |
| Propuesta | `color: #000; filter: invert(1); mix-blend-mode: difference;` |

Sobre fondo blanco los tres se leen negros. Al scrollear, los párrafos quedan fijos
y la imagen pasa por debajo: los dos blendeados se invierten a claro y el de
referencia queda negro sobre la foto.

## El síntoma

El texto blendeado se ve más pesado que el mismo texto sin blend. Mismo `font-family`,
mismo `font-size`, mismo `font-weight` — nada de eso cambia entre los tres párrafos.
La diferencia está solamente en el color con el que se pinta el glifo.

## La causa

El rasterizador no produce la misma máscara de cobertura para cualquier color: le
aplica una corrección de gamma en función del color de pintado y del fondo estimado
(`SkMaskGamma` en Skia; el equivalente en DirectWrite). Hace falta porque el
alpha-blending ocurre en sRGB no lineal, así que sin corregir, el texto claro sobre
oscuro y el oscuro sobre claro no salen con el mismo peso percibido.

Pintar en blanco y dejar que `difference` lo lleve a negro pide la corrección de
"claro sobre oscuro" y recién después invierte el resultado. El texto no queda sin
corregir: queda corregido al revés. Por eso engorda.

## El arreglo

Pintar en el color final —el que el rasterizador va a ver— e invertir el resultado
ya rasterizado con `filter: invert(1)`.

El orden de la spec es `filter` → `blend` → composición, así que el píxel final es
idéntico al del esquema habitual. Lo único que cambia es en qué color se rasterizó
el glifo.

No hay forma de conseguirlo sin `filter`: `difference` contra negro es la identidad,
así que pintar directamente en negro y blendear no invierte nada.

## Limitaciones

- **El antialiasing subpíxel se pierde igual.** `mix-blend-mode` fuerza la composición
  en un grupo transparente, y ahí no hay LCD text. Es inherente al blend, no se recupera
  con este arreglo ni con ningún otro.
- **`filter: invert(1)` invierte todo lo que pinte el elemento.** Ninguna imagen ni
  `background` propio puede caer adentro.
- **Un stacking context en un ancestro sigue rompiendo el blend.** Cualquier `z-index`,
  `transform`, `opacity < 1`, `filter`, `will-change`, `isolation` o `position: fixed`
  arriba en el árbol y el texto deja de componer contra lo que hay detrás.
- **Adentro del elemento con el filter, los colores se escriben normales.** No hay que
  invertirlos a mano. Ojo con `:focus-visible`: el `outline` también se invierte.

## Estructura de la página

Los párrafos y la imagen son hermanos directos de `<body>`, sin ningún wrapper: el
blend solo ve el backdrop de su propio stacking context, y cualquier ancestro que cree
uno lo aísla de la imagen.

Para que los párrafos se queden fijos mientras la imagen pasa, el `position: sticky`
está en el párrafo mismo, no en un contenedor. Sticky crea stacking context, pero en
el mismo elemento que lleva el blend: aísla a sus descendientes, no a sí mismo — igual
que el `filter` de la propuesta. Verificado en Chrome (Windows): el blend compone contra
la imagen sin problemas, así que no hizo falta la alternativa de poner la imagen
`position: fixed` a pantalla completa detrás con `z-index: -1`.

Los rótulos rojos van fuera de los párrafos, como hermanos: adentro del de la propuesta
el `filter: invert(1)` los pondría cian, y adentro del habitual el `difference` los
alteraría.

### Layout

- Los tres casos van contra el margen izquierdo, en una columna de 65ch.
- El texto explicativo va en flujo, arriba de los tres casos, con los mismos estilos
  que el `<pre>` de un rótulo.
- La imagen ocupa el 70% del ancho de los párrafos y va centrada dentro de esa
  columna, así que los párrafos fijos la cruzan por el medio y sobresalen a los
  costados. Conserva su relación de aspecto, sin recorte. En pantallas angostas pasa
  al 100% del ancho.
- Arriba de los tres casos van tres miniaturas (`shot-1.png`, `shot-2.png`,
  `shot-3.png`) que abren en un lightbox hecho con `:target`, sin JS. Los overlays
  están al final del `<body>` y con `z-index` propio: se pintan por encima de los
  párrafos, así que nunca entran en su backdrop. Son hermanos, no ancestros, de modo
  que su `position: fixed` no aísla nada. Para reemplazar una captura alcanza con
  pisar el archivo con el mismo nombre.
- Las posiciones `top` del apilado fijo salen de `--pad-top`, `--label-h` y `--pitch`
  en `:root`, medidas contra el render real. `--pitch` coincide con el ritmo del flujo
  normal para que los bloques no se solapen al fijarse. **Si cambia el tamaño, el
  interlineado o el largo del texto, hay que volver a medir esos tres valores.**
- Debajo de 700px la medida se achica, los párrafos crecen en alto y el apilado ya no
  entra en el viewport: ahí se desactiva el sticky y todo queda en flujo normal. El
  blend sigue funcionando, pero la comparación de los tres sobre la imagen es de
  pantalla ancha.

## Tipografía

Univers LT Pro 55, servida con `@font-face` desde el repo (`UniversLTStd.woff2`), a
`font-size: 18px` y `line-height: 20px`. El descriptor `font-weight: 400` del `@font-face`
sólo declara qué contiene el archivo: no se declara `font-weight` en ningún elemento, ni
`font-synthesis`, ni `text-rendering`, ni ajustes de suavizado.

Nota de licencia: Univers es una tipografía licenciada. Publicar el `.woff2` en GitHub
Pages lo deja descargable desde un repo público, así que conviene verificar que la
licencia cubra uso web en ese dominio antes de publicar.

## Lo que no se hace acá, y por qué

- **`-webkit-font-smoothing: antialiased`.** Es no-op en Windows, que es justamente donde
  el efecto se ve. Y si todo el texto de la página está blendeado, no arregla ninguna
  inconsistencia: solo adelgaza todo en macOS.
- **Bajar el `font-weight` del texto blendeado.** Tapa el síntoma en un tamaño y se
  desincroniza en el resto; además queda atado a la tipografía y se rompe si cambia.

## Dónde se ve

El efecto se nota claramente en Windows con Chrome y con Firefox, en tamaños de lectura
(18–20px). En macOS puede no notarse: el pipeline de rasterizado es otro.
