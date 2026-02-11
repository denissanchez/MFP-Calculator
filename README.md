# Verificador Tributario Peru

Aplicacion web para verificar calculos tributarios peruanos (IGV) y generar el JSON payload listo para enviar a un facturador electronico.

**[Ver demo en vivo](https://denissanchez.github.io/calculator/)**

## Funcionalidades

- **Tasa IGV personalizable** — por defecto 18%, se recalcula todo en tiempo real
- **Tipos de operacion** — Gravada, Inafecta y Bonificada con sus respectivos `tipoAfeIgv` (10, 30, 15)
- **Precision en centimos** — todos los totales se suman como enteros para evitar errores de punto flotante
- **Generacion de JSON** — payload compatible con facturador electronico SUNAT (empresa, venta, cliente, detVentas)
- **Datos configurables** — empresa, cliente, serie, correlativo, moneda, forma de pago
- **Copiar al portapapeles** — un click para copiar el JSON generado

## Uso

1. Abrir `index.html` en el navegador (no requiere servidor)
2. Configurar la tasa de IGV si es diferente a 18%
3. Agregar productos con codigo, descripcion, unidad de medida, cantidad y precio unitario
4. Seleccionar si el precio ingresado incluye IGV o no
5. Verificar los calculos en el resumen y copiar el JSON generado

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

## Tecnologia

HTML + CSS + JavaScript vanilla. Sin dependencias, sin build, un solo archivo.
