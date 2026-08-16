# Repo dashboard-negocios — Dashboard financiero

## Qué es esto

Dashboard estático publicado en GitHub Pages.
Se ve en: https://juanfonsecax.github.io/dashboard-negocios/

Cubre **dos negocios completamente distintos**, que nunca se mezclan:
- **Mercado Libre** — dos cuentas (Carlos y Juan), venta de interruptores y
  otros, en Colombia.
- **Drop** — dropshipping con LLC en Estados Unidos, operando en Colombia y
  Guatemala. Otro producto, otra operación, otras cuentas.

**No cruzar los dos negocios en ningún análisis.** La pauta de Facebook que
sale de las cuentas de Drop es de Drop: no explica ni afecta las ventas de
Mercado Libre. Son dos negocios separados, cada uno con su propio bloque de
datos (`DATOS` y `DROP`) y su propia pestaña.

En el mismo repo, aparte, vive una segunda app sin relación con el dashboard:
**Bitácora del Efecto Compuesto** (hábitos y objetivos) en `bitacora/`. Guarda
sus datos en `localStorage` del navegador, no en el repo. No tocarla al hacer
las actualizaciones mensuales.

## Arquitectura — leer antes de tocar nada

```
index.html        <- dashboard: CSS, HTML, lógica de gráficos (Chart.js)
datos.json        <- TODOS los números
bitacora/         <- la otra app, independiente
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
  "nota": "...",            <- opcional, ver abajo
  "productos": [
    { "nombre": "...", "u": 0, "recibiste": 0, "cost": 0 }
  ]
}
```

`nota` es opcional y sirve para **avisar en pantalla** cuando algo del mes no
salió tal cual del reporte (por ejemplo un día estimado). Si está, el panel la
pinta debajo de las tarjetas. Si no está, no se pinta nada. Úsala siempre que
un número no venga directo de la fuente: sin eso, en seis meses nadie recuerda
qué estaba estimado.

## Reglas de cálculo — no improvisar

- `ventasConcretadas`, `cargos`, `impuestos`, `recibiste` se copian **tal cual**
  del reporte de costos de Mercado Libre (ruta abajo). No recalcular.
- Comprobación rápida de que se leyó bien el reporte: `ventasConcretadas −
  cargos − impuestos` tiene que dar `recibiste` exacto, y la suma de
  `ventasConcretadas` de las publicaciones con unidades > 0 tiene que dar la
  cifra de cabecera. Si eso cuadra, no se perdió ninguna línea.
- `cost` = unidades × costo unitario, sacado del export de stock de Notion
  (columna "Precio en Casa").
- **El stock de Notion puede venir desactualizado.** En julio 2026 traía cinco
  precios inflados (Módulo Mini, Enchufe Normal, Válvula, Gu10, Enchufe
  Monitor) que habrían bajado el margen de Juan de 52% a 43%. Antes de cargar,
  comparar el costo unitario contra el del mes anterior y **preguntar por
  cualquier referencia que se mueva más de ~15%**.
- `utilidadNeta` = suma de (`recibiste` − `cost`) **solo** de productos con costo cargado.
- Producto sin costo conocido → `"cost": null`. **Nunca inventar un costo ni poner 0.**
  Los productos con `cost: null` quedan fuera de la utilidad y se listan aparte.

## Cómo actualizar (proceso mensual)

Juan pasa los documentos del mes:
- Reporte de costos de ML — cuenta Carlos
- Reporte de costos de ML — cuenta Juan
- Archivo de stock con costo unitario por referencia
- Extractos de los bancos de Drop en USD
- Saldos de cierre de las cajas de Drop
- Tasas de cambio a usar (usdCop, gtqCop)

### Los bancos de Drop cambian — ojo con esto

Hasta julio 2026 el banco era **Mercury**. En julio hubo un problema y se
vació: quedó en $3,18 y se pasó a **Relay**. Desde agosto los gastos salen de
Relay.

Consecuencias al hacer el corte:
- Los gastos de un mes pueden venir de **más de un banco**. Julio 2026 es el
  caso: casi todo de Mercury y un Namecheap de $29,95 de Relay. Pedir los dos
  extractos y sumar, no asumir que hay uno solo.
- **Un giro entre cuentas propias NO es un gasto.** En julio salieron $3.300 a
  una cuenta familiar y $4.400 a Relay. Esa plata sigue siendo del negocio: va
  en `saldos` como otra caja, nunca en `gastos`. Contarla como gasto habría
  mostrado $9.377 de gastos en un mes de $1.707.
- Los **"Credit account payment"** del extracto de Mercury tampoco son gastos:
  son el pago de la tarjeta desde la cuenta corriente. Los gastos reales están
  en el extracto de la **tarjeta**, no en el de la cuenta.

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
python3 -m http.server 8000
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

El sitio lo publica `.github/workflows/pages.yml`, que corre solo en cada push
a `main` y tarda ~30 segundos. También se puede lanzar a mano
(`workflow_dispatch`) si hiciera falta republicar sin cambiar nada.

Ese workflow existe por una razón concreta: el mecanismo automático heredado
de Pages **no se dispara con pushes hechos por una app de GitHub**, así que los
cambios llegaban al repo pero el sitio publicado se quedaba atrás. No volver a
"Deploy from a branch" en Settings → Pages sin tener eso en cuenta.

El historial de Git es el respaldo. Si un mes queda un número mal, se puede ver
qué cambió y devolverse.

## Estado actual y pendientes

Última actualización: **julio 2026** cargado en las dos cuentas.

Mercado Libre de julio está **completo**: las dos cuentas cargadas y ningún
producto sin costo.

Pendiente:
- **Drop de julio**: falta el extracto de Mercury, los saldos de las 3 cajas y
  las tasas de cambio. `DROP.saldos` solo llega a julio y `DROP.gastos` a junio.

### Costos confirmados que no salían obvios del stock

Referencias del stock de Notion que no se pueden deducir del título de la
publicación en ML. Confirmadas por Juan, sirven para los meses siguientes:

| Publicación en ML | Costo unitario |
|---|---|
| Bombilla Inteligente Wifi 12w | $10.819 (mismo que *Bombilla 15w x1*) |
| Interruptor Smart Switch 10A 110v | $9.829 (*Switch Amarillo*) |
| Vii Interruptor Smart Switch 10A 110v | $9.829 (el mismo producto) |
| Capacitor Condensador sin Neutro | $851 |
