# ADR-0005: Montos en unidades menores de la moneda

- **Estado:** Propuesto
- **Fecha:** 2026-11-26

## Contexto

Los importes atraviesan tarifas por noche, impuestos porcentuales, descuentos, penalidades de cancelación y reembolsos parciales. `float` introduce errores de redondeo inaceptables. Además, cada moneda tiene su propia cantidad de decimales (USD y EUR tienen 2, JPY y CLP tienen 0).

## Decisión

- Los importes se guardan y se transportan como **enteros en la unidad menor** de la moneda (centavos para USD): columnas `BIGINT` y `int` en Python.
- Todo importe viaja con su moneda (código ISO 4217, `CHAR(3)`). No existe un importe sin moneda.
- Los cálculos intermedios con porcentajes usan `Decimal` y se redondean a la unidad menor **por línea del desglose**, con `ROUND_HALF_UP`. El total es la suma de las líneas ya redondeadas, así el desglose siempre suma exacto.
- La API expone los importes en unidades menores junto con `currency`. El ejemplo de desglose del plan inicial (`405`, `40.5`) pasa a `40500`, `4050`.
- En la versión 1, cada propiedad opera en una sola moneda.

## Alternativas consideradas

- **`NUMERIC(12,2)` y `Decimal` en todos lados.** Válida y legible, pero fija 2 decimales para todas las monedas y obliga a convertir al hablar con proveedores que trabajan en unidades menores, como Stripe.
- **`float`.** Descartado.

## Consecuencias

- Hace falta una tabla de exponentes por moneda para convertir y formatear.
- La regla de redondeo es parte del dominio y se testea (impuestos, descuentos, penalidades parciales).
- El frontend formatea los importes a partir de la moneda y su exponente.
