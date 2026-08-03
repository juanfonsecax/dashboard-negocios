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
- Reporte "Cargos e inversiones" de ML — cuenta Carlos
- Reporte "Cargos e inversiones" de ML — cuenta Juan
- Detalle de unidades y monto recibido por publicación
- Archivo de stock con costo unitario por referencia
- Extracto de la tarjeta Mercury (USD)
- Saldos de cierre de las 3 cajas (Mercury, Dropi CO, Dropi GT)
- Tasas de cambio a usar (usdCop, gtqCop)

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
