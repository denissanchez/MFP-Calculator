# Verificador Tributario Peru

Aplicacion web para verificar calculos tributarios peruanos (IGV) y generar el JSON payload listo para enviar a un facturador electronico.

**[Ver demo en vivo](https://denissanchez.github.io/calculator/)**

## Funcionalidades

- **Tasa IGV personalizable** — por defecto 18%, se recalcula todo en tiempo real
- **Decimales de redondeo configurables** — 2 o 3 decimales para verificar cual coincide con el facturador
- **IGV por diferencia** — el IGV se calcula como `totalConIgv - base`, no como `base * tasa`, evitando discrepancias de centimos
- **Tipos de operacion** — Gravada, Inafecta y Bonificada con sus respectivos `tipoAfeIgv` (10, 30, 15)
- **Precision en enteros** — todos los totales se suman como unidades enteras (`10^N`) para evitar errores de punto flotante
- **Generacion de JSON** — payload compatible con facturador electronico SUNAT (empresa, venta, cliente, detVentas)
- **Datos configurables** — empresa, cliente, serie, correlativo, moneda, forma de pago
- **Copiar al portapapeles** — un click para copiar el JSON generado

## Uso

1. Abrir `index.html` en el navegador (no requiere servidor)
2. Configurar la tasa de IGV si es diferente a 18%
3. Configurar los decimales de redondeo (2 o 3) segun lo que acepte el facturador
4. Agregar productos con codigo, descripcion, unidad de medida, cantidad y precio unitario
5. Seleccionar si el precio ingresado incluye IGV o no
6. Verificar los calculos en el resumen y copiar el JSON generado

## Logica de calculo

### El problema: IGV por multiplicacion directa

El enfoque intuitivo para calcular el IGV de un item es:

```
igv = montoBaseIgv * tasaIgv / 100
```

Sin embargo, al redondear cada IGV individual a 2 decimales y luego sumarlos, el total puede diferir en 1 o 2 centimos respecto al valor que espera el facturador electronico. Esto ocurre porque el facturador trabaja con los totales redondeados y obtiene el IGV global por diferencia, no por suma de IGVs individuales.

**Ejemplo del problema:**

| Item     | Precio con IGV | Base (÷ 1.18)  | Base redondeada | IGV = base * 18% | IGV redondeado |
|----------|---------------|-----------------|-----------------|------------------|----------------|
| NUCITA   | 14.05         | 11.906779...    | 11.91           | 2.1438           | **2.14**       |
| AZUCAR   | 8.85          | 7.500000        | 7.50            | 1.3500           | **1.35**       |
| **Suma** |               |                 | **19.41**       |                  | **3.49**       |

Hasta aqui coincide, pero en otros casos la suma de IGVs individuales redondeados **no coincide** con `totalConIgv - totalBases`. Esa diferencia de centimos causa rechazos en el facturador.

### La solucion: IGV por diferencia

En lugar de calcular el IGV como `base * tasa`, se calcula como:

```
igv = totalConIgv - base
```

Tanto a nivel de cada item como en los totales del comprobante. Esto garantiza que la suma siempre sea consistente.

### Calculo por item (operacion gravada)

Dado un precio unitario **con IGV**, cantidad y tasa (ej: 18%):

```
1. valorUnitario       = precio / (1 + tasa/100)              → alta precision (10 decimales)
2. montoPrecioUnitario = precio                                → tal cual ingresado
3. montoValorVenta     = round(valorUnitario * cantidad, N)    → base redondeada a N decimales
4. montoTotalConIgv    = round(montoPrecioUnitario * cant, N)  → total redondeada a N decimales
5. igv                 = round(montoTotalConIgv - montoValorVenta, N)  → POR DIFERENCIA
6. montoBaseIgv        = montoValorVenta
7. totalImpuestos      = igv
```

Si el precio se ingresa **sin IGV**, se invierte: `valorUnitario = precio` y `montoPrecioUnitario = precio * (1 + tasa/100)`.

### Calculo por item (operacion inafecta)

No hay IGV. El precio unitario es el valor de venta directamente:

```
1. valorUnitario    = precio
2. montoValorVenta  = round(precio * cantidad, N)
3. montoTotalConIgv = montoValorVenta    → igual a la base (no hay impuesto)
4. igv              = 0
```

### Calculo por item (operacion bonificada / gratuita)

El valor de venta es 0 (es gratuita), pero se calcula el IGV sobre la base para fines tributarios:

```
1. valorUnitario    = 0
2. montoValorVenta  = 0
3. montoTotalConIgv = 0
4. montoBaseIgv     = round(precio * cantidad, N)
5. igv              = round(montoBaseIgv * tasa / 100, N)  → multiplicacion directa (no hay "total con IGV")
```

### Totales del comprobante

Los totales se acumulan en **unidades enteras** (`soles * 10^N`) para evitar errores de punto flotante en las sumas:

```
Para cada item gravado:
  gravUnits      += toUnits(montoValorVenta)     → suma de bases
  gravTotalUnits += toUnits(montoTotalConIgv)    → suma de totales con IGV

Para cada item inafecto:
  inafUnits += toUnits(montoValorVenta)

IGV total:
  igvUnits = gravTotalUnits - gravUnits          → POR DIFERENCIA (no suma de IGVs)

Total a pagar:
  montoImporteVenta = gravTotalUnits + inafUnits
```

Donde `toUnits(soles) = Math.round(soles * 10^N)` y `N` es el numero de decimales configurado.

### Decimales de redondeo (N)

El parametro **Decimales** controla a cuantos decimales se redondean los montos por item (`montoValorVenta`, `montoTotalConIgv`, `igv`) y los totales.

- **N = 2** (default): Trabaja en centimos. Es lo mas comun.
- **N = 3**: Trabaja en milesimos. Algunos facturadores aceptan o requieren 3 decimales.

Cambiar este valor recalcula todos los items y totales en tiempo real, permitiendo verificar rapidamente cual configuracion produce valores que coinciden con el facturador.

### Ejemplo completo paso a paso

**Configuracion:** IGV 18%, 2 decimales

**Item 1: NUCITA** — precio con IGV = S/ 14.05, cantidad = 1

```
valorUnitario    = 14.05 / 1.18            = 11.9067796610 (10 dec)
montoValorVenta  = round(11.9067796610 * 1, 2) = 11.91
montoTotalConIgv = round(14.05 * 1, 2)         = 14.05
igv              = round(14.05 - 11.91, 2)      = 2.14
```

**Item 2: AZUCAR** — precio con IGV = S/ 8.85, cantidad = 1

```
valorUnitario    = 8.85 / 1.18             = 7.5000000000 (10 dec)
montoValorVenta  = round(7.5 * 1, 2)           = 7.50
montoTotalConIgv = round(8.85 * 1, 2)          = 8.85
igv              = round(8.85 - 7.50, 2)        = 1.35
```

**Totales (en centimos, N=2):**

```
gravUnits      = 1191 + 750  = 1941   → S/ 19.41  (Op. Gravadas)
gravTotalUnits = 1405 + 885  = 2290   → S/ 22.90  (Total items con IGV)
igvUnits       = 2290 - 1941 = 349    → S/ 3.49   (IGV por diferencia)
montoImporteVenta = 2290              → S/ 22.90  (Total a pagar)
```

**Resultado en el JSON:**

```json
{
  "totalOpGravadas": 19.41,
  "mtoIgv": 3.49,
  "totalImpuestos": 3.49,
  "valorVenta": 19.41,
  "montoImporteVenta": 22.90,
  "subTotal": 22.90
}
```

### Ejemplo con 3 decimales

Mismos items, pero con **N = 3**:

**Item 1: NUCITA**

```
montoValorVenta  = round(11.9067796610 * 1, 3) = 11.907
montoTotalConIgv = round(14.05 * 1, 3)         = 14.050
igv              = round(14.050 - 11.907, 3)    = 2.143
```

**Item 2: AZUCAR**

```
montoValorVenta  = round(7.5 * 1, 3)           = 7.500
montoTotalConIgv = round(8.85 * 1, 3)          = 8.850
igv              = round(8.850 - 7.500, 3)      = 1.350
```

**Totales (en milesimos, N=3):**

```
gravUnits      = 11907 + 7500  = 19407  → S/ 19.407
gravTotalUnits = 14050 + 8850  = 22900  → S/ 22.900
igvUnits       = 22900 - 19407 = 3493   → S/ 3.493
montoImporteVenta = 22900               → S/ 22.900
```

## Estructura del JSON generado

```json
{
  "empresa": { "ruc", "ubigeo", "direccion", "distrito", "codAnexo" },
  "venta": {
    "tipoOperacion", "tipoDoc", "serie", "correlativo", "fechaEmision",
    "formaPago", "moneda", "totalOpGravadas", "mtoIgv", "totalImpuestos",
    "valorVenta", "montoImporteVenta", "subTotal"
  },
  "cliente": { "tipoDoc", "razonSocial", "numDoc", "ubigeo" },
  "detVentas": [{
    "codigoProd", "unidadMedida", "descripcion", "cantidad",
    "valorUnitario", "montoValorVenta", "montoBaseIgv", "porcentajeIgv",
    "igv", "tipoAfeIgv", "totalImpuestos", "montoPrecioUnitario"
  }]
}
```

Campos opcionales que aparecen segun los tipos de operacion presentes:
- `totalInafectas` — solo si hay items inafectos
- `totalGratuitas`, `mtoIgvGratuitas` — solo si hay items bonificados

## Tecnologia

HTML + CSS + JavaScript vanilla. Sin dependencias, sin build, un solo archivo.
