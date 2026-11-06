# Changelog

Todos los cambios relevantes del escenario. El formato se basa en [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/).

## [v1.1] · Noviembre de 2026

Blueprint: [`blueprints/v1.1-reglas.blueprint.json`](blueprints/v1.1-reglas.blueprint.json) · Diseño: [`docs/arquitectura-v1.1.md`](docs/arquitectura-v1.1.md)

### Cambiado
- **Router → If-else + Merge.** La primera coincidencia ahora es nativa: ya no hacen falta condiciones de exclusión. Las acciones se definen una sola vez, después del Merge.
- **Palabras clave → expresiones regulares** con límite de palabra al inicio. Las condiciones bajan de 116 a 3, una por categoría, y el asunto y el cuerpo se evalúan juntos.
- **Gmail search** en el trigger: solo correos no leídos de **Recibidos**, sin boletines ni suscripciones, sin promociones, redes sociales ni notificaciones, sin `no-reply`, sin correos propios y sin respuestas automáticas. No usa `category:primary`, porque en cuentas sin pestañas (Google Workspace) devuelve 0 resultados.
- **Marcar como leído al obtener** el correo desde el trigger, en lugar de un módulo por ruta.
- **Versión Make** en el registro: `v1.1 - Reglas`.

### Agregado
- **Prefiltro gratuito** antes de clasificar: ignora respuestas automáticas y remitentes de sistema que no detecte la búsqueda de Gmail.
- **Palabras clave nuevas.** Urgente: `urge`, `urgen`, `urgencia`, `urgentemente`, `ASAP`. Facturación: `facturación`, `pagar`, `pagué`, `pagó`, `cobrar`, `reembolso`, `CFDI`. Consulta: `consultar`, `preguntar`.

### Corregido
- `duda` ya no coincide con "in**duda**blemente".
- Variantes como "urgencia" o "me urge" ahora sí se clasifican como Urgente.
- Riesgo de bucle con respuestas automáticas y con correos enviados desde la misma cuenta.

### Créditos
- Cada correo procesado cuesta **2 créditos** en lugar de 3. Los boletines, las notificaciones y las respuestas automáticas ya no consumen créditos.

### Comportamiento distinto a la v1
- Un correo con "urgencia" y "factura", como *"Urgencia Facturación"*, ahora es **Urgente**. Antes era Facturación.

## [v1] · Noviembre de 2026

Blueprint: [`blueprints/v1-reglas.blueprint.json`](blueprints/v1-reglas.blueprint.json) · Diseño: [`docs/arquitectura-v1.md`](docs/arquitectura-v1.md)

- Primera versión: Router con 4 rutas y filtros mutuamente excluyentes, respuestas por plantilla, registro en Google Sheets y marcado como leído.
