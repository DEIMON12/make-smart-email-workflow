# Arquitectura v1: Router con filtros excluyentes

| Campo | Detalle |
|---|---|
| **Estado** | Implementada y probada |
| **Escenario** | `make-email-automation` · [`blueprints/v1-reglas.blueprint.json`](../blueprints/v1-reglas.blueprint.json) |
| **Componente central** | Módulo [2] Router "Clasificar correo" |
| **Fecha** | Noviembre de 2026 |

Este documento explica cómo la v1 garantiza que **cada correo siga exactamente una ruta**, por qué se descartó dejar que un correo pase por varias rutas, cómo se compara con la alternativa **If-else + Merge** de Make y cuánto cuesta en créditos.

---

## Contenido

1. [Resumen de la decisión](#1-resumen-de-la-decisión)
2. [Cómo evalúa las rutas un Router de Make](#2-cómo-evalúa-las-rutas-un-router-de-make)
3. [Primera coincidencia mediante una cascada de prioridad](#3-primera-coincidencia-mediante-una-cascada-de-prioridad)
4. [Por qué no permitir varias coincidencias](#4-por-qué-no-permitir-varias-coincidencias)
5. [Alternativa: If-else + Merge](#5-alternativa-if-else--merge)
6. [La ruta de respaldo](#6-la-ruta-de-respaldo)
7. [Costo en créditos](#7-costo-en-créditos)
8. [Orden de las acciones dentro de cada ruta](#8-orden-de-las-acciones-dentro-de-cada-ruta)
9. [Mantenimiento: agregar una palabra clave](#9-mantenimiento-agregar-una-palabra-clave)
10. [Referencias](#10-referencias)

---

## 1. Resumen de la decisión

**Contexto.** Cada correo debe recibir una sola respuesta y generar una sola fila en el registro. Las categorías se solapan: un mismo correo puede contener "urgente" y "factura".

**Decisión.** Un **Router con cuatro rutas en orden de prioridad**, donde el filtro de cada ruta **excluye las palabras clave de las rutas anteriores**. Esto convierte el Router, que por naturaleza admite varias coincidencias, en un clasificador de **primera coincidencia**.

**Consecuencias.**

| ✅ A favor | ⚠️ En contra |
|---|---|
| Exactamente una respuesta y una fila por correo | Las palabras clave se repiten como exclusiones: 116 condiciones en total |
| Costo mínimo: 3 créditos por correo, porque el Router y los filtros no cobran | Agregar una palabra clave obliga a editar varias rutas (sección 9) |
| Cada ruta muestra su propio contador en el historial de ejecuciones | Las acciones (responder, registrar, marcar) se repiten en las cuatro ramas: 12 módulos |
| Cada rama se puede personalizar de forma independiente | |

---

## 2. Cómo evalúa las rutas un Router de Make

Hay tres propiedades del Router que explican el diseño ([Router](https://help.make.com/router), [If-else y Merge](https://help.make.com/if-else-and-merge)):

1. **Evalúa todas las rutas, en orden.** El Router *"runs all conditions in the set order"*: no se detiene en la primera ruta que coincide. Un correo sigue **cada** ruta cuyo filtro cumpla.
2. **Las rutas se ejecutan en secuencia, no en paralelo.** Make termina la primera ruta antes de empezar la segunda. Por eso, la alternativa a la primera coincidencia no es una "ejecución paralela", sino una **ejecución múltiple**: el mismo correo recorre varias ramas, una tras otra.
3. **Las rutas no se pueden volver a unir.** Lo que ocurre después de una rama se queda en esa rama. Por eso cada ruta repite sus tres acciones.

Además, el Router y los filtros **no consumen créditos** ([consumo de créditos](https://help.make.com/how-features-use-credits)), así que evaluar 116 condiciones no tiene costo. Solo cobran los módulos que se ejecutan.

---

## 3. Primera coincidencia mediante una cascada de prioridad

### 3.1 Conjuntos de palabras clave

| Símbolo | Categoría | El correo contiene, en asunto o cuerpo… |
|:---:|---|---|
| **U** | Urgente | `urgente` o `emergencia` |
| **F** | Facturación | `factura`, `pago` o `cobro` |
| **C** | Consulta | `duda`, `consulta` o `pregunta` |

Todas las comparaciones usan el operador de Make **Contains (case insensitive)** (`text:contain:ci`) o su negación (`text:notcontain:ci`).

### 3.2 Filtro de cada ruta

| Ruta | Filtro lógico | Grupos OR en Make | Condiciones |
|---|---|:---:|:---:|
| 1 · Urgente | `U` | 4 | 4 |
| 2 · Facturación | `F ∧ ¬U` | 6 | 30 |
| 3 · Consulta | `C ∧ ¬U ∧ ¬F` | 6 | 66 |
| 4 · Respaldo | `¬U ∧ ¬F ∧ ¬C` | 1 | 16 |
| | | **Total** | **116** |

En el editor de filtros de Make, los **grupos se combinan con OR** y las **condiciones dentro de un grupo, con AND**. Por eso cada palabra positiva crea un grupo, y dentro de él se repiten todas las exclusiones. Por ejemplo, el primer grupo de la ruta 2:

```
Asunto contiene "factura"
  AND Asunto NO contiene "urgente"      AND Cuerpo NO contiene "urgente"
  AND Asunto NO contiene "emergencia"   AND Cuerpo NO contiene "emergencia"
```

### 3.3 Garantía: exactamente una ruta por correo

Las cuatro expresiones forman una **partición** de todos los correos posibles:

```
¿Contiene U? ── sí ──►  Urgente
      │ no
      ▼
¿Contiene F? ── sí ──►  Facturación
      │ no
      ▼
¿Contiene C? ── sí ──►  Consulta
      │ no
      ▼
   General
```

- **Mutuamente excluyentes:** cada ruta exige que *no* se cumpla ninguna condición de las rutas anteriores, así que dos rutas nunca pueden cumplirse a la vez.
- **Exhaustivas:** si un correo no cumple U, F ni C, cumple exactamente `¬U ∧ ¬F ∧ ¬C`, que es la ruta de respaldo.

**Verificación.** Se simularon los filtros tal como están en el blueprint, con todas las combinaciones posibles de las 8 palabras clave en asunto y cuerpo. En ninguna combinación pasó un número de rutas distinto de 1.

---

## 4. Por qué no permitir varias coincidencias

Supongamos el diseño "natural" de un Router: cada ruta con **solo sus palabras positivas**. Tomemos este correo:

> **Asunto:** URGENTE: error en la factura 4512
>
> **Cuerpo:** No coincide el monto

| Diseño | Rutas que se ejecutan | Respuestas al cliente | Filas en la hoja | Créditos del correo |
|---|---|:---:|:---:|:---:|
| **v1: cascada de prioridad** | Urgente | 1 | 1 | **3** |
| Filtros solo positivos, respaldo nativo | Urgente, Facturación | 2 | 2 | 6 |
| Filtros solo positivos, respaldo sin filtro | Urgente, Facturación, General | 3 | 3 | 9 |

El costo se resume en una fórmula:

```
créditos por correo = 3 × (número de rutas que cumple el correo)
```

Permitir varias coincidencias tiene tres consecuencias:

1. **Respuestas contradictorias.** El cliente recibiría a la vez *"prioridad Alta"* y *"prioridad Media"* para el mismo correo.
2. **Registro distorsionado.** Un correo contaría dos o tres veces en la hoja. Eso invalida las métricas por categoría y, sobre todo, la comparación entre la v1 y la v2, que depende de que cada fila sea un correo.
3. **Créditos desperdiciados.** Cada ruta extra cuesta 3 créditos sin aportar nada: responde a un correo que ya fue respondido y lo marca como leído otra vez.

La cascada elimina las tres sin costo adicional, porque las condiciones extra viven en los filtros y estos no consumen créditos.

---

## 5. Alternativa: If-else + Merge

### 5.1 El problema de unir ramas

Con un Router, las ramas no se pueden volver a unir. Para que las tres acciones se ejecuten una sola vez después de clasificar habría que guardar la categoría en variables o en un data store y agregar más módulos. Eso hace el escenario más complejo a cambio de poco.

Make ofrece dos módulos que resuelven exactamente esto ([If-else y Merge](https://help.make.com/if-else-and-merge)):

- **If-else:** evalúa las condiciones en orden y *"only the first condition that evaluates as true runs"*. Es primera coincidencia nativa, con una ruta **Else** como respaldo.
- **Merge:** reúne las ramas del If-else en un solo flujo y pasa los datos de la rama activa a los módulos siguientes.

Ninguno de los dos consume créditos.

### 5.2 Cómo se vería el escenario

```
[1] Gmail · Watch emails
       │
       ▼
    If-else ─┬─ 1. U?  ──►  Do nothing ─┐
             ├─ 2. F?  ──►  Do nothing ─┤
             ├─ 3. C?  ──►  Do nothing ─┤
             └─ Else   ──►  Do nothing ─┤
                                        ▼
                                      Merge  (salidas: categoría · prioridad · plantilla HTML)
                                        │
                                        ▼
         [Gmail] Responder  ──►  [Sheets] Add a Row  ──►  [Gmail] Marcar leído
```

Las ramas sin módulos propios llevan un *Do nothing*, que Make inserta automáticamente para que los datos pasen al Merge.

### 5.3 Comparación

| Criterio | Router + cascada (v1) | If-else + Merge |
|---|---|---|
| Primera coincidencia | Emulada con exclusiones | Nativa |
| Condiciones | 116 | 16 |
| Módulos de acción (responder, registrar, marcar) | 12 | 3 |
| Créditos por correo | 3 | 3 |
| Agregar una palabra urgente | 28 condiciones en 4 rutas | 2 condiciones en 1 rama |
| Riesgo de duplicados por olvido | Sí (sección 9) | No |
| Personalizar una sola categoría | Directo, cada rama es independiente | Con módulos dentro de la rama, antes de Merge |

**El costo en créditos es idéntico**, así que la ventaja de If-else + Merge está en el mantenimiento, no en el ahorro.

**Decisión para la v1.** Se conserva el Router porque el escenario ya está construido y probado, y la cascada garantiza el mismo resultado. La migración a If-else + Merge queda planeada para la **v1.1**, antes de la v2: el clasificador con Gemini encaja de forma natural como una condición más en un If-else.

---

## 6. La ruta de respaldo

La ruta 4 no usa la opción de **ruta de respaldo nativa** del Router (*fallback route*, que solo se ejecuta si el correo no cumplió ninguna otra ruta). En su lugar, tiene un filtro explícito con las 16 negaciones.

Ambas opciones dan el mismo resultado mientras el filtro esté completo. El filtro explícito tiene un riesgo: si se agrega una palabra a otra ruta y se olvida en el respaldo, un correo con esa palabra pasaría por las dos rutas. Por ejemplo, terminaría registrado como *Facturación* y también como *General*.

**Recomendación:** marcar la ruta 4 como fallback nativo y quitarle el filtro. Para hacerlo, abre el filtro de la ruta 4, elige **Yes** en *Fallback route* y guarda.

---

## 7. Costo en créditos

### 7.1 Por componente

| Componente | Créditos | Fuente |
|---|:---:|---|
| Trigger *Watch emails* | 1 por revisión, aunque no haya correos nuevos | [Make](https://help.make.com/how-features-use-credits) |
| Router y filtros | 0 | [Make](https://help.make.com/how-features-use-credits) |
| *Send an email* | 1 por correo | |
| *Add a Row* | 1 por correo | |
| *Mark an email as read* | 1 por correo | |

```
créditos al mes ≈ revisiones al mes + 3 × correos procesados
```

### 7.2 El costo fijo del sondeo

| Intervalo | Revisiones al mes | Créditos solo por revisar | ¿Cabe en el plan gratuito (1,000)? |
|---|:---:|:---:|:---:|
| 15 min | ≈2,880 | ≈2,880 | ❌ se agota en ≈10 días |
| 30 min | ≈1,440 | ≈1,440 | ❌ |
| 60 min | ≈720 | ≈720 | ✅ quedan ≈280 créditos (≈90 correos) |

El plan gratuito incluye 1,000 créditos al mes ([precios](https://www.make.com/en/pricing)) y su intervalo mínimo es de 15 minutos ([comparativa de precios 2026](https://www.activepieces.com/blog/makecom-pricing-2026-plans-credits-and-real-costs)). Ese intervalo mínimo tiene un costo fijo mayor que todo el saldo mensual del plan. Para operar todo el mes en el plan gratuito, se recomienda un intervalo de 60 minutos. Si se usa un plan de pago, el intervalo se puede ajustar según la urgencia requerida.

---

## 8. Orden de las acciones dentro de cada ruta

```
Responder  →  Registrar  →  Marcar como leído
```

- **Marcar como leído va al final.** El correo solo queda como atendido en Gmail cuando la respuesta y el registro terminaron. Si algo falla antes, sigue sin leer, que es una señal visible de que requiere revisión manual.
- **Responder va antes de registrar.** El registro refleja una respuesta que ya se envió. La desventaja: si *Add a Row* falla, el cliente ya recibió respuesta pero no hay fila. Para la v1 se acepta, porque la hoja es un registro de apoyo, no la fuente de verdad.

---

## 9. Mantenimiento: agregar una palabra clave

Con la cascada, una palabra clave aparece una vez de forma **positiva** en su ruta y varias veces de forma **negativa** en las rutas posteriores. Ejemplo: agregar `urgencia` a la categoría Urgente.

| Ruta | Cambio | Condiciones nuevas |
|---|---|:---:|
| 1 · Urgente | 2 grupos nuevos: asunto contiene / cuerpo contiene `urgencia` | 2 |
| 2 · Facturación | En **cada uno** de sus 6 grupos: asunto y cuerpo NO contienen `urgencia` | 12 |
| 3 · Consulta | En **cada uno** de sus 6 grupos: asunto y cuerpo NO contienen `urgencia` | 12 |
| 4 · Respaldo | Asunto y cuerpo NO contienen `urgencia` | 2 |
| | | **28** |

### Lista de verificación

- [ ] Agregar la palabra como positiva en su ruta (asunto **y** cuerpo).
- [ ] Agregarla como negativa en **todos los grupos** de **todas las rutas posteriores**.
- [ ] Agregarla como negativa en la ruta de respaldo, o usar el fallback nativo (sección 6).
- [ ] Enviar un correo de prueba que contenga la palabra nueva junto con una palabra de otra categoría, y confirmar que solo se ejecuta una ruta.
- [ ] Exportar el blueprint, quitar los datos personales y actualizar `blueprints/`.

Este costo de mantenimiento es la principal razón para migrar a If-else + Merge (sección 5), donde el mismo cambio son 2 condiciones en una sola rama.

---

## 10. Referencias

- Make Help Center: [Router](https://help.make.com/router)
- Make Help Center: [If-else and Merge](https://help.make.com/if-else-and-merge)
- Make Help Center: [How features use credits](https://help.make.com/how-features-use-credits)
- Make Help Center: [Scenario settings](https://help.make.com/scenario-settings)
- Make: [Pricing](https://www.make.com/en/pricing)
