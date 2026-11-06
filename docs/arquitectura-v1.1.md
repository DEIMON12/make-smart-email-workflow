# Arquitectura v1.1: If-else + Merge con expresiones regulares

| Campo | Detalle |
|---|---|
| **Estado** | Versión actual |
| **Escenario** | `make-email-automation` · [`blueprints/v1.1-reglas.blueprint.json`](../blueprints/v1.1-reglas.blueprint.json) |
| **Reemplaza a** | [v1: Router con filtros excluyentes](arquitectura-v1.md) |
| **Fecha** | Noviembre de 2026 |

La v1.1 conserva el mismo comportamiento de la v1: cada correo se clasifica en una sola categoría, recibe una respuesta y genera una fila. Cambia la forma de lograrlo, para que sea **más fácil de mantener, más precisa y más barata**.

---

## Contenido

1. [Qué cambió y por qué](#1-qué-cambió-y-por-qué)
2. [Flujo](#2-flujo)
3. [If-else + Merge](#3-if-else--merge)
4. [Clasificación con expresiones regulares](#4-clasificación-con-expresiones-regulares)
5. [Filtrado en dos capas](#5-filtrado-en-dos-capas)
6. [Marcar como leído desde el trigger](#6-marcar-como-leído-desde-el-trigger)
7. [Costo en créditos](#7-costo-en-créditos)
8. [Verificación](#8-verificación)
9. [Migrar desde la v1](#9-migrar-desde-la-v1)
10. [Limitaciones que se mantienen](#10-limitaciones-que-se-mantienen)
11. [Referencias](#11-referencias)

---

## 1. Qué cambió y por qué

| Aspecto | v1 | v1.1 | Beneficio |
|---|---|---|---|
| Clasificador | Router con 4 rutas | If-else con 3 condiciones + Else | Primera coincidencia nativa |
| Exclusividad | 16 a 66 condiciones de exclusión por ruta | Ninguna: la da el If-else | Sin riesgo de duplicados por olvido |
| Condiciones | 116 | 3 de clasificación + 2 del prefiltro | Un solo lugar por categoría |
| Coincidencia | "contiene" (subcadena) | Expresión regular con límite de palabra | Menos falsos positivos y más variantes |
| Acciones | 12 módulos (3 por ruta) | 2 módulos después del Merge | Un solo punto para cambiar respuestas o registro |
| Marcar como leído | 1 módulo por ruta | Opción del trigger | 1 crédito menos por correo |
| Correos no deseados | Se respondían y registraban | Excluidos por la búsqueda de Gmail y el prefiltro | Sin créditos desperdiciados ni bucles |
| Créditos por correo | 3 | 2 | −33 % |

---

## 2. Flujo

```
          ┌──────────────────────────────────────────────┐
          │ [1] Gmail · Watch emails                     │
          │ Gmail search: Recibidos · no leídos          │
          │ sin boletines, no-reply ni autorrespuestas   │
          │ marca como leído al obtener                  │
          └──────────────────────┬───────────────────────┘
                                 │  prefiltro (regex · 0 créditos)
                                 ▼
          ┌──────────────────────────────────────────────┐
          │ [2] If-else "Clasificar correo"              │
          │ gana la primera condición verdadera          │
          └───┬───────────┬───────────┬───────────┬──────┘
              │ 1. U      │ 2. F      │ 3. C      │ Else
              ▼           ▼           ▼           ▼
           Urgente   Facturación  Consulta     General
              │           │           │           │
              └───────────┴──────┬────┴───────────┘
                                 ▼
          ┌──────────────────────────────────────────────┐
          │ [5] Merge                                    │
          │ salidas: categoria · prioridad · plantilla   │
          └──────────────────────┬───────────────────────┘
                                 ▼
          ┌──────────────────────────────────────────────┐
          │ [6] Gmail · Responder con {{5.plantilla}}    │
          └──────────────────────┬───────────────────────┘
                                 ▼
          ┌──────────────────────────────────────────────┐
          │ [7] Google Sheets · Registrar el caso        │
          └──────────────────────────────────────────────┘
```

---

## 3. If-else + Merge

El módulo **If-else** evalúa sus condiciones en orden y *"only the first condition that evaluates as true runs"*. Si ninguna se cumple, se ejecuta la rama **Else** ([Make: If-else y Merge](https://help.make.com/if-else-and-merge)). Esto da la primera coincidencia de forma nativa, sin las exclusiones que necesitaba el Router de la v1.

Las ramas no ejecutan acciones: cada una tiene un módulo *Do nothing* y todas se unen en el **Merge**. El Merge define tres salidas, con un valor por rama y en el mismo orden que las ramas:

| Salida | Urgente | Facturación | Consulta | Else |
|---|---|---|---|---|
| `categoria` | Urgente | Facturación | Consulta | General |
| `prioridad` | Alta | Media | Baja | Baja |
| `plantilla` | HTML de urgente | HTML de facturación | HTML de consulta | HTML general |

Después del Merge, los módulos usan esas salidas como cualquier otro dato: `{{5.categoria}}`, `{{5.prioridad}}` y `{{5.plantilla}}`. Así, la respuesta y el registro existen **una sola vez** en el escenario.

Ni el If-else ni el Merge consumen créditos.

---

## 4. Clasificación con expresiones regulares

### 4.1 Expresiones

Cada condición evalúa el **asunto y el cuerpo juntos** (`{{1.subject}} {{1.fullTextBody}}`) con el operador **Matches pattern (case insensitive)**, que no distingue mayúsculas de minúsculas.

| Orden | Categoría | Expresión regular |
|:---:|---|---|
| 1 | Urgente | `\b(urge\b\|urgen\|emergencia\|asap\b)` |
| 2 | Facturación | `\b(factur\|pag[aeéoóu]\|cobr[ao]\|reembols\|cfdi)` |
| 3 | Consulta | `\b(duda\|consult\|pregunt)` |
| — | General | *(Else)* |

### 4.2 Por qué el límite de palabra va solo al inicio

Make se ejecuta sobre JavaScript, y las expresiones se diseñaron y probaron con ese motor. En JavaScript, `\b` considera letras únicamente `a–z`, `0–9` y `_`: una vocal con acento cuenta como "no letra". Por eso, una expresión como `\bpagó\b` **nunca coincide**, ya que no existe límite de palabra entre la "ó" y el espacio siguiente.

La solución es usar **raíces con límite solo al inicio**:

- `\bfactur` coincide con *factura, facturas, facturación, facturar*.
- `\bpag[aeéoóu]` coincide con *pago, pagos, pagar, pagué, pagó, paga*.
- El `\b` inicial evita coincidencias dentro de otra palabra: *in**duda**blemente*, *a**pag**ar*, *re**cobr**ó*, *s**urge***.

Las palabras cortas y sin acento llevan `\b` también al final para coincidir completas: `urge\b` cubre "me urge" sin chocar con *urgente*, que ya cubre `urgen`, y `asap\b` evita coincidir dentro de otras palabras.

### 4.3 Casos verificados

| Categoría | Debe coincidir | No debe coincidir |
|---|---|---|
| Urgente | URGENTE, urgencia, me urge, urgen, urgentemente, emergencia, ASAP | surge, pasaporte, ventana emergente, purgen |
| Facturación | factura, facturación, pago, pagué, pagó, cobro, cobrar, reembolso, CFDI | apagar, página, recobró, refactor |
| Consulta | duda, consulta, consultar, pregunta, preguntar | indudablemente, reunión del viernes |

### 4.4 Mantenimiento

Para agregar una palabra basta con editar **una sola expresión**. Por ejemplo, para que "inmediato" cuente como urgente:

```
\b(urge\b|urgen|emergencia|asap\b|inmediat)
```

En la v1 el mismo cambio requería 28 condiciones repartidas en 4 rutas ([detalle](arquitectura-v1.md#9-mantenimiento-agregar-una-palabra-clave)).

Antes de cambiar una expresión, conviene probarla con casos positivos y negativos en [regex101.com](https://regex101.com) con el sabor **ECMAScript (JavaScript)** y la opción `i`.

---

## 5. Filtrado en dos capas

### Capa 1: búsqueda de Gmail en el trigger

```
in:inbox is:unread -from:me
-category:(promotions OR social OR updates OR forums)
-from:(noreply OR no-reply OR mailer-daemon OR postmaster)
-subject:("respuesta automática" OR "automatic reply" OR "out of office" OR "fuera de la oficina")
-unsubscribe -"cancelar suscripción" -"cancelar suscripcion" -"anular suscripción" -"darte de baja"
```

En el blueprint, la búsqueda va en una sola línea; aquí se muestra en varias para que sea legible.

| Parte | Excluye |
|---|---|
| `in:inbox` | Todo lo que no esté en Recibidos: archivados, enviados, spam y papelera |
| `-category:(…)` | Promociones, Social, Notificaciones y Foros. Se excluyen estas categorías en lugar de pedir `category:primary`, porque en cuentas sin pestañas (común en Google Workspace) no existe "Principal" y la búsqueda no encontraría nada |
| `-from:me` | Correos enviados desde la misma cuenta, lo que evita que el escenario se responda a sí mismo |
| `-from:(noreply …)` | Remitentes que no aceptan respuesta |
| `-subject:(…)` | Respuestas automáticas |
| `-unsubscribe -"cancelar suscripción" …` | Boletines y suscripciones. En cuentas sin pestañas llegan a Recibidos sin categoría, así que `-category:(…)` no los detecta; casi todos incluyen un enlace para darse de baja |

Lo que excluye esta capa **ni siquiera llega a Make**: sigue sin leer en Gmail y no consume créditos.

### Capa 2: prefiltro antes del If-else

Un filtro en la conexión hacia el If-else, que tampoco consume créditos, descarta lo que la búsqueda de Gmail no detecte. Se continúa solo si se cumplen las dos condiciones:

| Campo | Operador | Expresión |
|---|---|---|
| Asunto | Does not match pattern (ci) | `respuesta autom[aá]tica\|automatic reply\|auto-?reply\|fuera de (la )?oficina\|out of office\|delivery status notification\|no se pudo entregar\|undeliverable` |
| Remitente | Does not match pattern (ci) | `no-?reply\|mailer-daemon\|postmaster\|notifications?@` |

Las expresiones son más amplias que la búsqueda de Gmail: cubren variantes como "Respuesta automatica" sin acento, `auto-reply`, avisos de entrega fallida y `notifications@`.

**Nota:** como el trigger marca los correos como leídos al obtenerlos (sección 6), un correo descartado por esta capa queda **leído pero sin respuesta**. Esto solo afecta a lo que pase la capa 1, que normalmente es ruido.

---

## 6. Marcar como leído desde el trigger

El trigger usa la opción **"Mark email message(s) as read when fetched"** en lugar de un módulo *Mark an email as read* al final de cada ruta.

| | Módulo al final (v1) | Opción del trigger (v1.1) |
|---|---|---|
| Créditos por correo | 1 | 0 |
| Si falla el envío | El correo queda sin leer, como señal de pendiente | El correo ya está leído |

Se acepta la contrapartida porque la hoja de Google Sheets es el registro de lo atendido: un correo leído que no aparece en la hoja es un fallo identificable. Además, Make desactiva el escenario después de varios errores seguidos (*Scenario settings → Errors before deactivation*), así que un problema persistente no pasa desapercibido.

---

## 7. Costo en créditos

| Componente | v1 | v1.1 |
|---|:---:|:---:|
| Trigger, por revisión | 1 | 1 |
| Filtros, If-else y Merge | 0 | 0 |
| Responder | 1 | 1 |
| Registrar | 1 | 1 |
| Marcar como leído | 1 | 0 |
| **Por correo procesado** | **3** | **2** |
| Por boletín o respuesta automática | 3 | 0 |

**Ejemplo:** 100 correos al mes, revisando de lunes a viernes de 8:00 a 18:00 cada 30 minutos, que son ≈440 revisiones al mes.

| Versión | Créditos al mes |
|---|:---:|
| v1, cada 15 min todos los días | 2,880 + 300 = **≈3,180** |
| v1.1, horario laboral cada 30 min | 440 + 200 = **≈640** |

El horario se configura en Make con **Advanced scheduling** en el reloj del trigger; no forma parte del blueprint.

---

## 8. Verificación

El blueprint se probó con dos simulaciones, porque desde el repositorio no se puede ejecutar dentro de Make:

1. **Expresiones regulares.** 44 casos positivos y negativos con el motor de JavaScript y la opción `i`, incluidas palabras con acento. Todos pasan.
2. **Blueprint de punta a punta.** Se leyeron los filtros y las condiciones tal como quedaron en el JSON, se aplicó la primera coincidencia del If-else y se calcularon las salidas del Merge:

| Asunto | Cuerpo | Remitente | Resultado |
|---|---|---|---|
| URGENTE: Se me cayó mi lente | Prueba de filtro | cliente | Urgente · Alta |
| URGENTE: error en la factura 4512 | No coincide el monto | cliente | Urgente · Alta |
| Ya pagué | ¿Me confirman? | cliente | Facturación · Media |
| Envío de CFDI | Adjunto XML | cliente | Facturación · Media |
| Tengo una duda | ¿Cuál es el horario? | cliente | Consulta · Baja |
| Hola | Indudablemente fue un buen día | cliente | General · Baja |
| Surge un problema | con la impresora | cliente | General · Baja |
| Automatic reply: Re: URGENTE | I am out of office | cliente | Ignorado por el prefiltro |
| Alerta de seguridad | urgente: revisa tu cuenta | `no-reply@…` | Ignorado por el prefiltro |

También se comprobó que la estructura del If-else, el Merge y el trigger coincide con una exportación real de Make.

---

## 9. Migrar desde la v1

1. Importa la v1.1 como un **escenario nuevo** (ver el [README](../README.md#instalación-importar-el-blueprint)). No sobrescribas el de la v1 hasta confirmar que todo funciona.
2. Conecta Gmail en los módulos 1 y 6, y Google en el módulo 7. Apunta el módulo 7 a la misma hoja `Registro de Correos`: las columnas no cambian.
3. **Desactiva el escenario v1 antes de activar la v1.1.** Si ambos están activos, cada correo recibe dos respuestas.
4. Envía los correos de prueba desde otra cuenta y revisa una fila nueva por correo, con `Versión Make = v1.1 - Reglas`.

---

## 10. Limitaciones que se mantienen

- **Texto citado.** El cuerpo de una respuesta incluye el mensaje anterior. Si el correo original decía "urgente", la respuesta del cliente también se clasifica como Urgente.
- **Sin comprensión del contexto.** *"Esto no es urgente"* se clasifica como Urgente. Resolver la negación y la intención es el objetivo de la v2 con Gemini.
- **Asunto con "Re:" acumulado.** La respuesta se envía como `Re: <asunto>`, así que un asunto que ya empieza con "Re:" queda como "Re: Re:".
- **Respuesta Enviada** guarda el nombre de la plantilla, no el texto enviado.

---

## 11. Referencias

- Make Help Center: [If-else and Merge](https://help.make.com/if-else-and-merge)
- Make Help Center: [How features use credits](https://help.make.com/how-features-use-credits)
- Make Help Center: [Scenario settings](https://help.make.com/scenario-settings)
- Google: [Refine searches in Gmail](https://support.google.com/mail/answer/7190)
- MDN: [Word boundary assertion `\b`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Word_boundary_assertion)
