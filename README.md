# make-smart-email-workflow

![Make](https://img.shields.io/badge/Make-escenario-6D00CC?logo=make&logoColor=white)
![Gmail](https://img.shields.io/badge/Gmail-trigger%20%2B%20respuesta-EA4335?logo=gmail&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Google%20Sheets-registro-34A853?logo=googlesheets&logoColor=white)
![Versión](https://img.shields.io/badge/versión-v1.1%20·%20reglas-blue)

Automatización en [Make](https://www.make.com) que **clasifica los correos entrantes de Gmail, responde automáticamente y registra cada caso en Google Sheets**.

La **versión 1.1** es un clasificador determinista basado en expresiones regulares. La **versión 2** (planeada) usará un modelo de IA (Gemini) y conservará el mismo registro, de modo que ambas versiones puedan compararse.

---

## Contenido

- [Qué hace](#qué-hace)
- [Arquitectura](#arquitectura)
- [Reglas de clasificación](#reglas-de-clasificación)
- [Qué correos se ignoran](#qué-correos-se-ignoran)
- [Configurar Gmail según tu cuenta](#configurar-gmail-según-tu-cuenta)
- [Registro en Google Sheets](#registro-en-google-sheets)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Instalación: importar el blueprint](#instalación-importar-el-blueprint)
- [Pruebas](#pruebas)
- [Consumo de créditos](#consumo-de-créditos)
- [Limitaciones conocidas](#limitaciones-conocidas)
- [Versiones y roadmap](#versiones-y-roadmap)

---

## Qué hace

En cada revisión, el escenario busca en **Recibidos** los correos no leídos enviados por personas (sin boletines, promociones ni notificaciones) y, por cada uno:

1. **Lo clasifica** en una de cuatro categorías: Urgente, Facturación, Consulta o General.
2. **Responde al remitente** con una plantilla según la categoría.
3. **Registra el caso** en Google Sheets: fecha, remitente, asunto, categoría y prioridad.

Cada correo recibe **exactamente una** respuesta y genera **exactamente una** fila. Los boletines, las notificaciones, las respuestas automáticas y los correos de la propia cuenta **se ignoran** y no consumen créditos. Revisa [Configurar Gmail según tu cuenta](#configurar-gmail-según-tu-cuenta) antes de activarlo.

---

## Arquitectura

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

Los números entre corchetes son los IDs de los módulos en el blueprint.

- **If-else** evalúa las condiciones en orden y ejecuta **solo la primera** que se cumple. Si ninguna se cumple, ejecuta la rama **Else**.
- **Merge** reúne las cuatro ramas y entrega la `categoria`, la `prioridad` y la `plantilla` de la rama que se ejecutó.
- La respuesta y el registro existen **una sola vez** en el escenario, después del Merge.

El diseño completo, incluido el porqué de cada decisión, está en [`docs/arquitectura-v1.1.md`](docs/arquitectura-v1.1.md).

---

## Reglas de clasificación

Cada condición busca en el **asunto y el cuerpo juntos**, con expresiones regulares que no distinguen mayúsculas.

| Orden | Categoría | Expresión regular | Ejemplos que coinciden | Prioridad |
|:---:|---|---|---|:---:|
| 1 | 🔴 **Urgente** | `\b(urge\b\|urgen\|emergencia\|asap\b)` | urgente, urgencia, me urge, urgentemente, emergencia, ASAP | Alta |
| 2 | 🟠 **Facturación** | `\b(factur\|pag[aeéoóu]\|cobr[ao]\|reembols\|cfdi)` | factura, facturación, pago, pagué, pagó, cobro, cobrar, reembolso, CFDI | Media |
| 3 | 🔵 **Consulta** | `\b(duda\|consult\|pregunt)` | duda, consulta, consultar, pregunta, preguntar | Baja |
| — | ⚪ **General** | *(ninguna de las anteriores)* | | Baja |

El `\b` inicial exige que la palabra **empiece** ahí. Por eso *indudablemente*, *surge*, *pasaporte*, *apagar* o *página* ya no se clasifican por error. Si un correo coincide con varias categorías, gana la de menor número de orden: *"URGENTE: error en la factura"* se clasifica como **Urgente**.

### Respuestas automáticas

Cada respuesta se envía como `Re: <asunto original>`:

| Categoría | Mensaje |
|---|---|
| Urgente | Hemos recibido tu mensaje y lo hemos marcado como **Urgente** con prioridad **Alta**. Nuestro equipo lo atenderá lo antes posible. |
| Facturación | Hemos recibido tu mensaje relacionado con **Facturación** y lo hemos clasificado con prioridad **Media**. Revisaremos tu solicitud y te responderemos pronto. |
| Consulta | Hemos recibido tu consulta y la hemos clasificado como **Consulta**, con prioridad **Baja**. Nuestro equipo revisará tu mensaje y te responderá pronto. |
| General | Hemos recibido tu mensaje y lo hemos clasificado como **General**, con prioridad **Baja**. Revisaremos tu solicitud y te responderemos pronto. |

---

## Qué correos se ignoran

El escenario responde **solo a correos entrantes de personas**. El filtrado ocurre en dos capas, y ninguna consume créditos:

| Capa | Dónde | Ignora |
|---|---|---|
| 1 | Búsqueda de Gmail en el trigger | Todo lo que no esté en **Recibidos**, correos de la propia cuenta, boletines y suscripciones, Promociones, Social, Notificaciones y Foros, `noreply`, `mailer-daemon`, `postmaster` y asuntos de respuesta automática |
| 2 | Filtro antes del If-else | Variantes que la búsqueda no detecte: *auto-reply*, avisos de entrega fallida, `notifications@`, "Respuesta automatica" sin acento |

Lo que se descarta en la capa 1 sigue sin leer en Gmail. Lo que se descarta en la capa 2 queda marcado como leído, porque el trigger ya lo obtuvo.

### Búsqueda de Gmail del trigger

```
in:inbox is:unread -from:me
-category:(promotions OR social OR updates OR forums)
-from:(noreply OR no-reply OR mailer-daemon OR postmaster)
-subject:("respuesta automática" OR "automatic reply" OR "out of office" OR "fuera de la oficina")
-unsubscribe -"cancelar suscripción" -"cancelar suscripcion" -"anular suscripción" -"darte de baja"
```

En el módulo Gmail (1) la búsqueda va en una sola línea.

| Parte | Para qué |
|---|---|
| `in:inbox` | Solo correos en **Recibidos**: no toma archivados, enviados, spam ni papelera |
| `is:unread` | Solo correos sin leer |
| `-from:me` | Ignora correos enviados desde la misma cuenta, para que el escenario no se responda a sí mismo |
| `-category:(…)` | Ignora Promociones, Social, Notificaciones y Foros |
| `-from:(noreply …)` | Ignora remitentes que no aceptan respuesta |
| `-subject:(…)` | Ignora respuestas automáticas |
| `-unsubscribe -"cancelar suscripción" …` | Ignora boletines: casi todos incluyen un enlace para darse de baja |

---

## Configurar Gmail según tu cuenta

La misma búsqueda se comporta distinto según el tipo de cuenta. Antes de activar el escenario, **pega la búsqueda en el buscador de Gmail**: lo que aparezca es exactamente lo que el trigger va a procesar, a partir del punto de inicio que elijas.

### Cuenta con pestañas (Gmail personal)

Gmail separa los correos en Principal, Promociones, Social, etc. La exclusión `-category:(…)` funciona sin ajustes.

### Cuenta sin pestañas (Google Workspace: escuela o empresa)

Muchas cuentas institucionales tienen las pestañas desactivadas:

- **No uses `category:primary`.** En estas cuentas no existe "Principal" y la búsqueda devuelve **0 resultados**, aunque haya correos sin leer. Por eso el blueprint excluye categorías en lugar de pedir "Principal".
- **Los boletines llegan a Recibidos sin categoría.** Los cubre la exclusión de `unsubscribe` y "cancelar suscripción". Si alguno se cuela, agrega su remitente a la búsqueda: `-from:boletin@ejemplo.com` o `-from:ejemplo.com`.

### Bandeja con muchos correos sin leer

El trigger tiene la opción **Choose where to start** (clic derecho en el módulo Gmail):

| Opción | Qué procesa | Recomendación |
|---|---|---|
| **From now on** | Solo correos que lleguen después de activar | Segura para uso normal |
| **Since specific date** | Correos desde la fecha y hora indicadas | Útil para probar con correos recientes |
| **All** | **Todos** los correos que cumplan la búsqueda | ⚠️ Evítala si la bandeja tiene correos viejos sin leer: el escenario les respondería, hasta 10 por revisión |

### Bandeja compartida o de soporte

Si la cuenta recibe correos de muchos tipos y solo quieres atender algunos, deja que Gmail los separe primero:

1. En Gmail, crea un filtro (**Configuración → Filtros y direcciones bloqueadas → Crear un filtro**) que aplique la etiqueta `soporte` a los correos que te interesan; por ejemplo, los enviados a cierta dirección o desde cierto dominio.
2. En el trigger, cambia el inicio de la búsqueda a `label:soporte is:unread` y conserva el resto de las exclusiones.

### Otros ajustes útiles

| Necesidad | Agrega a la búsqueda |
|---|---|
| Solo correos de un dominio | `from:@empresa.com` |
| Ignorar un remitente | `-from:remitente@ejemplo.com` |
| Ignorar correos con adjuntos | `-has:attachment` |
| Solo correos recientes | `newer_than:2d` |

---

## Registro en Google Sheets

Hoja: **`Registro de Correos`**. Los encabezados deben escribirse exactamente así, porque el blueprint los usa como identificadores de columna. La plantilla está en [`templates/registro-correos.csv`](templates/registro-correos.csv).

| Columna | Contenido | Ejemplo |
|---|---|---|
| **Fecha** | Fecha del correo, hora del centro de México (`YYYY-MM-DD HH:mm`) | `2026-11-06 10:30` |
| **Remitente** | Dirección del remitente | `usuario@ejemplo.com` |
| **Asunto** | Asunto original | `URGENTE: Se me cayó mi lente de contacto` |
| **Categoría** | Urgente · Facturación · Consulta · General | `Urgente` |
| **Prioridad** | Alta · Media · Baja | `Alta` |
| **Versión Make** | Versión del clasificador que procesó el correo | `v1.1 - Reglas` |
| **Respuesta Enviada** | Plantilla de respuesta utilizada | `Urgente` |

La columna **Versión Make** permite comparar las versiones sobre la misma hoja (`v1 - Reglas`, `v1.1 - Reglas` y, más adelante, `v2 - Gemini IA`).

---

## Estructura del repositorio

```
make-smart-email-workflow/
├── README.md                           ← este documento
├── CHANGELOG.md                        ← cambios por versión
├── .gitignore
├── blueprints/
│   ├── v1.1-reglas.blueprint.json      ← versión actual: If-else + Merge
│   └── v1-reglas.blueprint.json        ← versión anterior: Router
├── docs/
│   ├── arquitectura-v1.1.md            ← diseño actual, expresiones regulares y créditos
│   └── arquitectura-v1.md              ← diseño anterior, conservado como referencia
├── templates/
│   └── registro-correos.csv            ← encabezados de la hoja de registro
└── assets/
    └── escenario-v1.png                ← captura del escenario v1 en Make
```

---

## Instalación: importar el blueprint

### Requisitos

- Una cuenta de [Make](https://www.make.com). El plan gratuito sirve; revisa antes la sección [Consumo de créditos](#consumo-de-créditos).
- Una cuenta de **Gmail**, que será la bandeja vigilada y la que envíe las respuestas.
- Una cuenta de **Google** con acceso a Google Sheets. Puede ser la misma.

### 1. Crear la hoja de registro

1. Descarga [`templates/registro-correos.csv`](templates/registro-correos.csv).
2. En Google Sheets: **Archivo → Importar → Subir** y elige el archivo.
3. Cambia el nombre de la pestaña a **`Registro de Correos`**, exactamente así.

Si ya usabas la v1, puedes reutilizar la misma hoja: las columnas no cambian.

### 2. Importar el escenario

1. En Make, entra a **Scenarios → Create a new scenario**.
2. En la barra inferior del editor, abre el menú **⋯** y elige **Import Blueprint**.
3. Selecciona [`blueprints/v1.1-reglas.blueprint.json`](blueprints/v1.1-reglas.blueprint.json) y pulsa **Save**.

### 3. Conectar tus cuentas

El blueprint no incluye conexiones. Abre cada módulo y selecciona o crea la tuya:

| Módulo | Conexión |
|---|---|
| [1] Gmail · Watch emails | **Gmail** |
| [6] Gmail · Responder | **Gmail** |
| [7] Google Sheets · Registrar | **Google** |

### 4. Apuntar a tu hoja

En el módulo **[7] Registrar**:

1. En **Spreadsheet ID**, reemplaza `REEMPLAZAR_CON_EL_ID_DE_TU_HOJA` con el ID de tu hoja, que es la parte de la URL entre `/d/` y `/edit`. También puedes buscarla con el botón de búsqueda.
2. En **Sheet Name**, elige `Registro de Correos`.
3. Verifica que **Categoría** y **Prioridad** muestren los datos del Merge (`5. categoria` y `5. prioridad`). En el módulo [6], **Content** debe mostrar `5. plantilla`. Si alguno aparece como texto en lugar de un dato, vuelve a seleccionarlo desde las salidas del Merge.

### 5. Probar y activar

1. Si tenías la v1 activa, **desactívala primero**. Con las dos activas, cada correo recibiría dos respuestas.
2. Desde **otra cuenta de correo**, envía uno de los [correos de prueba](#pruebas).
3. Clic derecho en **Gmail (1) → Choose where to start → Since specific date** con una hora un poco anterior a tu correo de prueba. No uses **All** si la bandeja tiene correos viejos sin leer.
4. Pulsa **Run once** y revisa el historial de ejecución: solo una rama del If-else debe mostrar actividad.
5. En el reloj del módulo 1, configura el horario. Te recomendamos **Advanced scheduling** con días y horas laborales (ver [Consumo de créditos](#consumo-de-créditos)).
6. Antes de activar, cambia el punto de inicio a **From now on** y activa el escenario con el interruptor **Scheduling**.

---

## Pruebas

Resultados esperados de la lógica del blueprint, obtenidos con una simulación de sus filtros y condiciones (detalle en [`docs/arquitectura-v1.1.md`](docs/arquitectura-v1.1.md#8-verificación)):

| Asunto | Cuerpo | Resultado esperado |
|---|---|---|
| URGENTE: Se me cayó mi lente | Prueba de filtro | Urgente · Alta |
| URGENTE: error en la factura 4512 | No coincide el monto | Urgente · Alta |
| Pregunta sobre el pago | Es urgente | Urgente · Alta |
| Ya pagué | ¿Me confirman? | Facturación · Media |
| Envío de CFDI | Adjunto XML | Facturación · Media |
| Tengo una duda | ¿Cuál es el horario? | Consulta · Baja |
| Hola | Indudablemente fue un buen día | General · Baja |
| Reunión del viernes | Nos vemos a las 10 | General · Baja |
| Automatic reply: Re: URGENTE | I am out of office | Ignorado |

Después de cada prueba, comprueba que haya **una** respuesta en el correo de prueba y **una** fila nueva en la hoja, con `Versión Make = v1.1 - Reglas`.

---

## Consumo de créditos

Según la [documentación de Make](https://help.make.com/how-features-use-credits), los filtros, el If-else y el Merge **no consumen créditos**. El trigger consume **1 crédito por revisión, aunque no encuentre correos**.

| Concepto | Créditos |
|---|:---:|
| Cada revisión del trigger | 1 |
| Cada correo procesado (responder + registrar) | 2 |
| Cada correo ignorado | 0 |

Con poco volumen, lo que más cuesta es **revisar**. Limita las revisiones con **Advanced scheduling** en el reloj del módulo 1:

| Programación | Revisiones al mes | + 100 correos | Total |
|---|:---:|:---:|:---:|
| Cada 15 min, todos los días | ≈2,880 | 200 | ≈3,080 |
| Cada 60 min, todos los días | ≈720 | 200 | ≈920 |
| L–V, 8:00–18:00, cada 30 min | ≈440 | 200 | ≈640 |
| L–V, 8:00–18:00, cada 60 min | ≈220 | 200 | ≈420 |

> ⚠️ El plan gratuito de Make incluye **1,000 créditos al mes** ([precios](https://www.make.com/en/pricing)). Revisar cada 15 minutos todos los días lo agota en unos 10 días, solo con las revisiones.

---

## Limitaciones conocidas

- **Texto citado.** En una respuesta encadenada, el cuerpo incluye el mensaje anterior: si este decía "urgente", la respuesta también se clasifica como Urgente.
- **Sin comprensión del contexto.** *"Esto no es urgente"* se clasifica como Urgente. Resolverlo es el objetivo de la v2.
- **Marcado como leído al obtener.** Si el envío de la respuesta falla, el correo ya aparece como leído. La hoja de registro es la referencia de lo que sí se atendió.
- **Asunto "Re: Re:".** Si el asunto original ya empieza con "Re:", la respuesta lo repite.

---

## Versiones y roadmap

- [x] **v1 · Reglas.** Router con filtros excluyentes. [Diseño](docs/arquitectura-v1.md) · [blueprint](blueprints/v1-reglas.blueprint.json)
- [x] **v1.1 · Reglas mejoradas.** If-else + Merge, expresiones regulares, filtrado de ruido y 2 créditos por correo. [Diseño](docs/arquitectura-v1.1.md) · [blueprint](blueprints/v1.1-reglas.blueprint.json)
- [ ] **v2 · Gemini IA.** Clasificar y redactar la respuesta con Gemini, conservando el mismo registro (`Versión Make = v2 - Gemini IA`) y usando las reglas de la v1.1 como respaldo si la IA falla.

El detalle de cada cambio está en el [`CHANGELOG.md`](CHANGELOG.md).
