# Repo judiff — Dashboard financiero

## Qué es esto

Dashboard estático publicado en GitHub Pages en `/panel/`.
Se ve en: https://juanfonsecax.github.io/judiff/panel/

Cubre dos negocios distintos:
- **Mercado Libre** — dos cuentas (Carlos y Juan), venta de interruptores y otros.
- **Drop** — dropshipping Colombia + Guatemala, plata repartida en 3 cajas
  (Mercury USD, Dropi Colombia COP, Dropi Guatemala GTQ).

## Arquitectura — leer antes de tocar nada

```
panel/
  index.html    <- código: CSS, HTML, lógica de gráficos (Chart.js)
  datos.json    <- TODOS los números
```

**Regla principal: `index.html` NO se toca en las actualizaciones mensuales.**
Actualizar el dashboard = reemplazar `datos.json`. Nada más.

`index.html` carga `datos.json` con fetch al abrir la página y de ahí saca
`DATOS` (Mercado Libre) y `DROP` (dropshipping). Si algo falla, muestra un
mensaje en pantalla en vez de quedar en blanco.

Solo hay que editar `index.html` si se quiere cambiar el diseño o agregar una
gráfica nueva. En ese caso, probar antes de subir (ver abajo).

## Estructura de datos.json

```
{
  "DATOS": {
    "monedaLocale": "es-CO",
    "actualizado": "YYYY-MM-DD",
    "cuentas": {
      "carlos": { "nombre", "descripcion", "meses": { "YYYY-MM": {...} } },
      "juan":   { ... }
    }
  },
  "DROP": {
    "saldos": { "YYYY-MM": { etiqueta, tasas:{usdCop,gtqCop}, plataformas:{...} } },
    "gastos": { "YYYY-MM": { etiqueta, periodo, usdCop, totalUsd, categorias:{...} } }
  }
}
```

Cada mes de una cuenta:
```
"YYYY-MM": {
  "etiqueta": "Julio 2026",
  "ventasConcretadas": 0,   <- tal cual del reporte de ML
  "cargos": 0,              <- tal cual del reporte de ML
  "impuestos": 0,           <- tal cual del reporte de ML
  "recibiste": 0,           <- tal cual del reporte de ML
  "productos": [
    { "nombre": "...", "u": 0, "recibiste": 0, "cost": 0 }
  ]
}
```

## Reglas de cálculo — no improvisar

- `ventasConcretadas`, `cargos`, `impuestos`, `recibiste` se copian **tal cual**
  del reporte oficial "Cargos e inversiones" de Mercado Libre. No recalcular.
- `cost` = unidades × costo unitario, sacado del archivo de stock
  (columna "Precio en Casa").
- `utilidadNeta` = suma de (`recibiste` − `cost`) **solo** de productos con costo cargado.
- Producto sin costo conocido → `"cost": null`. **Nunca inventar un costo ni poner 0.**
  Los productos con `cost: null` quedan fuera de la utilidad y se listan aparte.

## Cómo actualizar (proceso mensual)

Juan pasa los documentos del mes:
- Reporte de costos de ML — cuenta Carlos
- Reporte de costos de ML — cuenta Juan
- Archivo de stock con costo unitario por referencia
- Extracto de la tarjeta Mercury (USD)
- Saldos de cierre de las 3 cajas (Mercury, Dropi CO, Dropi GT)
- Tasas de cambio a usar (usdCop, gtqCop)

### De dónde sale el reporte de ML (ruta exacta)

En cada cuenta: **Métricas → Costos → selector de período → Fecha
personalizada → elegir el mes.**

Ese reporte trae, en un solo lugar, las cuatro cifras de cabecera
(ventas concretadas, cargos e inversiones, impuestos, recibiste) **y** el
detalle por publicación (unidades y monto recibido). No son dos documentos
distintos: es uno solo.

**Ojo con los meses de 31 días.** ML no deja seleccionar más de 30 días, así
que un mes de 31 llega en dos partes: del 1 al 30, y luego el día 31 aparte.
Hay que **sumar las dos partes** antes de cargar el mes. No cargar solo la
primera parte: quedaría un día de ventas por fuera sin que nada lo advierta.

En julio 2026 el reporte de costos del día 31 suelto no se pudo sacar. Si
vuelve a pasar, el día que falte se estima así:

1. Juan pasa ese día desde la sección **Ventas** (trae precio unitario y
   unidades por orden, pero no cargos ni impuestos).
2. `ventas del día` = suma de (unidades × precio unitario) de cada orden.
3. Se calculan las **tasas efectivas** que la cuenta tuvo en la parte que sí
   vino del reporte de costos: `cargos ÷ ventas` e `impuestos ÷ ventas`.
4. Se aplican esas tasas a las ventas del día para estimar cargos e
   impuestos; `recibiste = ventas − cargos − impuestos`.
5. El reparto por publicación usa la tasa `recibiste ÷ ventas` del mismo mes.

El método es defendible porque un día pesa ~1-2% del mes, y porque usa el
comportamiento real de esa cuenta ese mes, no un promedio inventado.
**El mes queda marcado con `nota` diciendo qué día está estimado.**

Cuidado al estimar: las órdenes que ML muestra como "despacharemos el
paquete el N de agosto" pueden contarse como concretadas en agosto. Si el
corte siguiente no cuadra por poco, revisar primero eso.

### Lo que el detalle por publicación NO cubre

La suma de `recibiste` de todas las publicaciones no cuadra exactamente con
el `recibiste` de cabecera. La diferencia son cargos que ML no atribuye a una
publicación (devoluciones, envíos Full), que viven en pestañas aparte del
mismo reporte.

Esto no es un error: las cuatro cifras de cabecera se copian tal cual, y el
detalle por publicación se usa solo para el ranking de productos y la
utilidad. Son dos cálculos independientes a propósito.

Pasos:
1. Agregar el mes nuevo dentro de `meses` de cada cuenta. **No borrar meses viejos.**
2. Agregar el corte a `DROP.saldos` y el mes a `DROP.gastos`.
3. Actualizar `DATOS.actualizado` con la fecha de hoy.
4. Avisar explícitamente si algún producto quedó con `cost: null`.
5. Probar (ver abajo) y luego commit + push.

## Probar antes de subir — obligatorio

```
cd panel && python3 -m http.server 8000
```

Abrir http://localhost:8000 y verificar que:
- Las tarjetas de arriba muestran números, no ceros ni "NaN"
- El selector de mes trae el mes nuevo
- Las gráficas pintan
- La pestaña de Drop carga

Abrir `index.html` con doble clic **no funciona** (el navegador bloquea fetch de
archivos locales). Siempre con el servidor.

## Al terminar

Commit descriptivo del estilo `datos: agrega julio 2026` y push a `main`.
GitHub Pages publica solo en ~1 minuto.

El historial de Git es el respaldo. Si un mes queda un número mal, se puede ver
qué cambió y devolverse.
