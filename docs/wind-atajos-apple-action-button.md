# Wind + Apple Shortcuts: registrar una transacción desde el Action Button del iPhone

> Investigación y propuesta técnica. Fecha: 2026-10-04.
> Contexto: los videos virales de "registra tus gastos con el botón de acción" lo hacen hacia Excel, Numbers, Google Sheets o Notion. Aquí se explica cómo funciona Apple Shortcuts por dentro, qué hacen exactamente esos atajos, qué tiene Wind hoy para conectarse, y cuál es el camino recomendado para que el botón de acción registre una transacción directamente en Wind.

---

## 0. Resumen ejecutivo

**Sí se puede, y la mayor parte de la infraestructura ya existe en Wind.** El atajo de los videos es siempre la misma receta: un botón físico dispara un atajo, el atajo pide monto y categoría, y al final llama a una acción "agrega una fila" (Numbers/Excel) o a una API (Notion, Google Sheets vía webhook). Para Wind solo cambia el último paso: en lugar de "agregar fila", el atajo hace un `POST` a la API pública de Wind con una API Key.

Hallazgos clave del lado de Wind (ver sección 3):

- Wind ya tiene **sección Desarrolladores con API Keys (`wind_sk_…`), permisos por endpoint, rate limit, documentación OpenAPI y webhooks**. La documentación visible agrupa endpoints de Transacciones, pero las acciones documentadas aparentan ser solo **Listar** y **Ver**: falta (o hay que confirmar) un `POST` para crear transacciones desde la API pública.
- **Navi ya tiene la acción "Crear transacción"** con flujo de propuesta y aprobación. Eso abre la opción de "dicta el gasto y Navi lo interpreta".
- Wind es una **web app React (windapp.ai)**, sin app nativa iOS. Esto descarta por ahora App Intents nativos y deep links a PWA, pero no afecta la vía API, que es la que usan los videos de Notion y Google Sheets.

**Recomendación:** Fase 1 = atajo oficial "Registrar en Wind" + endpoint de creación de transacciones en la API pública + página "Conecta tu iPhone" en Wind que entrega la API Key y el link del atajo. Es la ruta con menor esfuerzo, funciona en cualquier iPhone con iOS 17+, y es exactamente lo que el usuario ya aprendió a hacer con Excel. Las fases 2 y 3 (dictado con IA, automatización con Apple Pay y notificaciones bancarias, app nativa con App Intents) se construyen encima.

---

## 1. Cómo funciona Apple Shortcuts (modelo mental completo)

### 1.1 Qué es un atajo

Un atajo es una secuencia de **acciones** que se ejecutan en orden. Cada acción recibe la salida de la anterior ("Shortcut Input" / variables mágicas) y produce una salida. Hay cuatro familias de acciones:

| Familia | Ejemplos | Para qué sirve en nuestro caso |
|---|---|---|
| Scripting | `Ask for Input`, `Choose from Menu`, `Choose from List`, `If / Otherwise If`, `Repeat with Each`, `Dictionary`, `Get Dictionary Value`, `Text`, `Format Date`, `Show Result`, `Show Notification`, `Show Alert` | Pedir monto, categoría y concepto; armar el JSON; mostrar confirmación |
| Web | `Get Contents of URL`, `Open URL`, `Open X-Callback URL`, `Get Dictionary from Input` | Llamar a la API de Wind, abrir la transacción creada |
| Medios y dispositivo | `Dictate Text`, `Take Photo`, `Select Photos`, `Extract Text from Image`, `Get Current Location` | Dictar el gasto, fotografiar el comprobante |
| Acciones de apps (App Intents) | Numbers "Add Row to Table", WhatsApp "Send Message", Notion, etc. | Solo existen si la app es nativa y las publica. Wind hoy no tiene app nativa |

### 1.2 `Get Contents of URL` (la acción que hace todo el trabajo)

Es un cliente HTTP completo dentro del atajo:

- Métodos `GET`, `POST`, `PUT`, `PATCH`, `DELETE`.
- Headers arbitrarios (por ejemplo `Authorization: Bearer wind_sk_…` o `X-API-Key`).
- Cuerpo `JSON` (diccionario clave-valor con variables), `Form` (multipart, permite adjuntar una foto como archivo) o `File`.
- Si la respuesta es JSON, Shortcuts la convierte automáticamente en un **Dictionary** que se lee con `Get Dictionary Value` (soporta notación punto para anidados).

Dos detalles de diseño importantes:

1. **No expone el status code HTTP.** La acción devuelve el cuerpo aunque el servidor responda 4xx/5xx. Por eso el endpoint para atajos debe responder siempre JSON con un campo explícito tipo `"ok": true/false` y `"message"`, y el atajo decide con un `If`.
2. **Tiempo límite.** Cuando el atajo corre en segundo plano (automatizaciones) iOS lo mata alrededor de los 30 segundos. Desde el Action Button corre en primer plano y hay más margen, pero el endpoint debe ser rápido (ideal < 2 s).

### 1.3 Cómo se dispara un atajo

| Disparador | Requisito | Notas |
|---|---|---|
| **Action Button** | iPhone 15 Pro / 16 / 17 (todas las series con botón de acción) | Ajustes → Botón de acción → Atajo → elegir atajo. Un solo atajo; si quieres varias opciones se usa `Choose from Menu` dentro del atajo |
| Back Tap (doble/triple toque en la espalda) | Cualquier iPhone con iOS 14+ | Ajustes → Accesibilidad → Tocar → Toque posterior. Menos confiable |
| Siri | Cualquiera | "Oye Siri, registrar en Wind". Pide monto y categoría por voz |
| Widget de pantalla bloqueada / pantalla de inicio | Cualquiera | Un toque |
| Control Center | iOS 18+ | Los atajos se pueden poner como "control". Las apps nativas pueden publicar `ControlWidget` propios |
| Apple Watch | watchOS | Se activa "Mostrar en Apple Watch" |
| Automatizaciones | iOS 17+ | Ver 1.5. Pueden correr sin confirmación ("Run Immediately") |

### 1.4 Apple Intelligence dentro de los atajos: `Use Model` (iOS 26+, mejorado en iOS 27)

- Acción que manda un prompt a un modelo: **On-Device**, **Private Cloud Compute** (Cloud / Cloud Pro), o **ChatGPT** como extensión.
- Puede recibir texto, variables, fotos, eventos, y **puede devolver un `Dictionary` nativo** (salida estructurada) además de texto, lista, número o booleano. Esto permite: "Gasté 450 en Uber al aeropuerto con la tarjeta azul" → `{ "tipo": "egreso", "monto": 450, "concepto": "Uber al aeropuerto", "categoria": "Transporte" }`.
- El modelo cloud analiza imágenes (recibo fotografiado); el modelo on-device rechaza imágenes.
- Opción **Follow Up** para corregir la respuesta antes de pasarla a la siguiente acción.
- Requiere iPhone 15 Pro o posterior con Apple Intelligence activado. **Español (México) está soportado.** Hay límites de uso diario.
- iOS 27 agregó: modelos más capaces, consulta web, y un **Transcript inspector** para depurar qué recibió el modelo.

Implicación: el dictado con IA es un gran "wow", pero no todos los usuarios de Wind tendrán un iPhone compatible. Por eso conviene que el parseo de lenguaje natural también exista del lado del servidor (Navi), y `Use Model` sea un acelerador opcional.

### 1.5 Automatizaciones relevantes (iOS 17 → iOS 27)

| Trigger | Desde | Datos que entrega | Limitaciones |
|---|---|---|---|
| **Wallet / Transaction** (pago con Apple Pay) | iOS 17 (en iOS 26 se renombró "Wallet") | Monto, comercio, tarjeta/pase, fecha | Solo pagos **NFC en tienda física**; no compras in-app ni web. Depende de que el banco notifique rápido (1 s a 1 min). Reportes de timeouts en iOS 18 sobre todo con Mastercard, bug abierto en Apple |
| **App is opened / closed** | iOS 17 | Ninguno (solo el evento) | Útil para "al cerrar la app del banco, pregúntame si registro algo" |
| **Notificación recibida de [app]** con filtro por palabras | **iOS 27** | El texto de la notificación | Permite capturar la notificación del banco ("Compra por $450.00 en UBER") y parsearla con `Use Model`. Muy relevante en México donde Apple Pay no cubre todo |
| Screenshot guardado | iOS 27 | La captura | Capturar un comprobante de transferencia SPEI desde la app del banco |
| Hora, ubicación, NFC tag | iOS 13+ | — | Recordatorios de captura |

Todas pueden configurarse con "Run Immediately" (sin confirmación), aunque iOS siempre notifica que corrió.

### 1.6 Guardar configuración (API Key) dentro del atajo

Opciones, de más a menos recomendable:

1. **Storage actions de iOS 27** (`Get Value` / `Set Value`): valores persistentes entre ejecuciones, pueden ser "globales" (compartidos entre atajos) y se sincronizan por iCloud. Apple los menciona explícitamente para API keys. Es la solución limpia, pero requiere iOS 27.
2. **Archivo de configuración en la carpeta de Shortcuts en iCloud Drive** (`Save File` / `Get File`): patrón clásico, funciona desde iOS 13. El atajo pide la key la primera vez y la guarda en `Shortcuts/Wind/config.json`.
3. **Import Questions**: al compartir el atajo se marcan campos como "pregunta al importar" y el usuario los llena al instalarlo. Sencillo, pero la key queda embebida en el atajo y cambiar de key implica editarlo.

Sea cual sea la opción, **el atajo guarda texto plano** en el dispositivo: cualquiera que abra el atajo puede ver la key. Por eso la key que use el atajo debe ser **una key dedicada, con permisos mínimos (crear transacción + leer catálogos), revocable y con fecha de expiración opcional**. Nunca el JWT de sesión del usuario.

### 1.7 Distribución de atajos a los usuarios de Wind

- Se comparte como **link de iCloud** (`https://www.icloud.com/shortcuts/<id>`). El usuario toca "Obtener atajo" y listo; desde iOS 15 ya no hace falta activar "atajos no confiables".
- Un link de iCloud es una **foto fija**: si Wind publica una nueva versión, es un link nuevo. Patrón estándar: el atajo lleva un número de versión y al correr consulta `GET /shortcut/version`; si hay una nueva, ofrece abrir el nuevo link.
- Alternativa: hospedar el archivo `.shortcut` firmado en windapp.ai (desde iOS 15 los atajos exportados van firmados). También existe `shortcuts://run-shortcut?name=…&input=text&text=…` para que la web de Wind o Navi en WhatsApp **invoquen** el atajo con parámetros.

### 1.8 Deep links y web apps (lo que NO funciona)

- `Open URL` hacia `https://windapp.ai/...` abre **Safari**, aunque el usuario tenga Wind instalado como PWA en la pantalla de inicio. iOS no permite deep link a PWAs desde Shortcuts (solo desde notificaciones push).
- Universal Links y URL schemes propios (`wind://…`) requieren **app nativa**.
- `Open X-Callback URL` sirve para ir a otra app/atajo y regresar con resultado; útil solo entre apps nativas.

### 1.9 App Intents (la vía "nativa", para cuando Wind tenga app)

Si Wind publica una app iOS (aunque sea un contenedor delgado sobre la web app), con el framework **App Intents** (Swift) una acción se declara una vez y aparece automáticamente en: la app Atajos, Siri ("Registra un gasto de 450 en Uber en Wind"), Spotlight, el **Action Button** (como Control), Control Center y pantalla bloqueada (`ControlWidget`, iOS 18+), widgets interactivos, y `Use Model` (iOS 26+, el modelo recibe tus entidades como JSON). Esquema mínimo:

```swift
struct RegistrarTransaccionIntent: AppIntent {
    static var title: LocalizedStringResource = "Registrar transacción en Wind"
    static var openAppWhenRun = false

    @Parameter(title: "Tipo") var tipo: TipoTransaccion          // AppEnum: ingreso / egreso
    @Parameter(title: "Monto") var monto: Double
    @Parameter(title: "Concepto") var concepto: String
    @Parameter(title: "Categoría") var categoria: CategoriaEntity? // AppEntity con EntityQuery → lista dinámica desde la API

    func perform() async throws -> some IntentResult & ProvidesDialog & ReturnsValue<String> {
        let folio = try await WindAPI.crearTransaccion(tipo, monto, concepto, categoria)
        return .result(value: folio, dialog: "Listo, registré \(monto.formatted(.currency(code: "MXN"))) en Wind (\(folio)).")
    }
}

struct WindShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(intent: RegistrarTransaccionIntent(),
                    phrases: ["Registrar gasto en \(.applicationName)", "Registra una transacción en \(.applicationName)"],
                    shortTitle: "Registrar transacción", systemImageName: "plus.circle")
    }
}
```

Con React Native/Expo existe `expo-app-intents` (SDK 58, alpha) y config plugins; con Capacitor se agrega el código Swift en el target iOS. No depende del stack web de Wind.

---

## 2. Qué hacen exactamente los atajos de los videos (Excel / Numbers / Sheets / Notion)

Receta común, acción por acción:

1. **Trigger**: Action Button (o Back Tap, o automatización de Apple Pay).
2. `Ask for Input` (Number): "¿Cuánto?".
3. `Choose from Menu`: Comida / Transporte / Súper / Servicios / Otro.
4. `Ask for Input` (Text) opcional: nota o comercio.
5. `Current Date` + `Format Date`.
6. **Destino**, que es lo único que cambia entre videos:
   - **Numbers**: acción nativa "Add Row to Top/Bottom of Table" (solo funciona con Numbers; Excel para iOS no publica acciones de Shortcuts).
   - **Excel**: Power Automate con el trigger "When an HTTP request is received" → "Add a row into a table" (Excel Online). El atajo hace `POST` JSON a la URL del flow. O sea: **es una API**.
   - **Google Sheets**: Apps Script publicado como web app que recibe `POST` y hace `appendRow`. **También es una API.**
   - **Notion**: `Get Contents of URL` → `POST https://api.notion.com/v1/pages` con `Authorization: Bearer secret_…`. **API con token.**
7. `Show Notification`: "Gasto registrado".

Conclusión: **la variante Notion es idéntica a lo que necesita Wind.** El usuario ya sabe pegar un token y un atajo; nosotros le damos el atajo hecho, con la API Key generada desde Wind y categorías/cuentas reales en vez de un menú fijo.

Lo que Wind puede hacer mejor que una hoja de cálculo:

- Categorías, cuentas bancarias, clientes y proveedores reales (menús dinámicos desde la API).
- Folio, link a la transacción creada, comprobante adjunto.
- Conciliación posterior con movimientos bancarios y facturas (CFDI) que ya hace Wind.
- Permisos por usuario (la key hereda los permisos de quien la creó) y auditoría.
- Webhook `transaction_created` ya existente para avisar a terceros.

---

## 3. Qué tiene Wind hoy (hallazgos de la investigación)

Fuente: landing `windapp.mx` y bundle público de la web app `windapp.ai` (cadenas i18n y rutas; no se accedió a datos ni a endpoints autenticados).

| Área | Hallazgo | Implicación |
|---|---|---|
| Plataforma | Web app React (Vite) en `windapp.ai`; backend en el mismo host bajo `/api/*` con JWT de sesión. Sin app en App Store (el "Wind App" de la App Store es una app de clima sin relación). Sin `apple-app-site-association` real | App Intents, Controls y deep links nativos quedan para una fase posterior. La vía API es la viable hoy |
| Transacciones | Rutas `/finance/transactions`, `/finance/transactions/new`, `/finance/transactions/:id`. Formulario con: tipo (Ingreso/Egreso), monto, moneda, fecha y hora, concepto/descripción, forma de pago, cuenta bancaria, categoría, cliente o proveedor (opcional), facturas/cotizaciones relacionadas, comprobante, link de pago | El payload del atajo debe cubrir los obligatorios: tipo, monto, fecha, concepto, categoría y cuenta. El resto es opcional |
| API pública | Sección **Desarrolladores**: API Keys con alias, expiración opcional, activar/desactivar, "Key ID: `wind_sk_…`", aviso "esta clave solo se mostrará una vez", "Todas las peticiones requieren el header … con una API Key válida", **rate limit**, "Permiso requerido" por endpoint, documentación en app (`/api/public-api-docs/openapi` y PDF) con secciones Clientes, Proveedores, Proyectos, Cotizaciones, Facturas, **Transacciones**, Documentos Fiscales, Campos Configurables, Usuarios, Roles. Las acciones documentadas visibles son **Listar** y **Ver** | Ya existe autenticación, permisos y docs. **Falta confirmar/crear `POST` de transacciones** y endpoints ligeros de catálogos (categorías, cuentas) |
| Webhooks | Webhooks salientes con eventos de creación/actualización/eliminación, reintentos y detalle de entregas; permisos `transaction_created`, `transaction_updated`, `transaction_categorized` | Nada nuevo que construir para avisar a terceros cuando el atajo cree algo |
| Navi | Acciones propuestas por Navi: **Crear transacción**, Editar transacción, Crear cliente/proveedor/proyecto/cotización… con estados `PENDING_APPROVAL` → `EXECUTED`, autorización por aprobadores configurados, historial de aprobaciones. Endpoints `/api/navi/conversations/{id}/chat` y `/runs/{id}/stream`. Herramienta para enviar archivos por **WhatsApp** | Navi ya "entiende" una transacción en lenguaje natural. Se puede reutilizar su parser para el dictado, y el canal WhatsApp como segunda puerta de entrada |
| PWA | `manifest.json` dice `"name": "TIDE App"`, `display: standalone`, `theme_color #29B17B` | Bug de branding a corregir. iOS igual no permite abrir la PWA desde un atajo |
| Permisos | Permisos finos como `transaction_read_amount`, `transaction_read_client`, etc. | La key del atajo puede limitarse a crear transacción y leer catálogos |

---

## 4. Opciones de arquitectura

| # | Opción | Qué hay que construir en Wind | Experiencia | Cuándo |
|---|---|---|---|---|
| **A** | **Atajo oficial + API pública** (recomendada) | `POST` transacciones en API pública, `GET` categorías y cuentas, `POST` comprobante, página "Conecta tu iPhone" | Botón → monto → categoría → listo en 3-5 s. Funciona en iOS 17+ en cualquier iPhone | Fase 1 |
| **B** | Atajo que abre el formulario web prellenado | `/finance/transactions/new` lee query params (`?type=expense&amount=450&concept=Uber`) | Abre Safari con el formulario listo para revisar y guardar. Requiere sesión iniciada | Fase 1 como fallback y como "ver/editar" tras crear |
| **C** | Atajo → Navi (lenguaje natural) | Endpoint `quick-capture` que use el parser de Navi, o permitir que la API Key hable con Navi; opcionalmente auto-aprobación para el canal "atajo" | Dictas "gasté 450 en Uber" y Wind lo interpreta; hoy requeriría aprobar la acción propuesta | Fase 2 |
| **D** | Automatizaciones: Apple Pay (iOS 17+) y notificaciones del banco (iOS 27) | Mismo endpoint de A, más estado "por categorizar" | Pagas con el iPhone y la transacción aparece sola; o la notificación de BBVA/Banorte se convierte en transacción | Fase 2 |
| **E** | App nativa (contenedor) con App Intents + Control | App iOS mínima, intents que llamen a la API | "Oye Siri, registra un gasto en Wind"; Action Button como Control sin pasar por la app Atajos; widget interactivo | Fase 3 |

Por qué A primero: es lo que ya hacen los videos (versión Notion), no requiere app nativa, y cada pieza se reutiliza en C, D y E.

---

## 5. Diseño detallado del atajo "Registrar en Wind" (Fase 1)

### 5.1 Flujo

```
Action Button
 └─ [1] Cargar configuración (API Key, URL base, cuenta default)
 └─ [2] Choose from Menu: "Egreso" | "Ingreso" | "Dictar" | "Foto de comprobante"
      ├─ Egreso / Ingreso
      │    ├─ Ask for Input (Number) "¿Cuánto?"
      │    ├─ Ask for Input (Text) "¿Concepto?" (placeholder: "Describe brevemente la transacción")
      │    ├─ GET categorías?type=expense  → Choose from List
      │    ├─ GET cuentas bancarias        → Choose from List (se omite si hay cuenta default)
      │    └─ POST transacción
      ├─ Dictar
      │    ├─ Dictate Text (es-MX)
      │    ├─ Use Model (Output: Dictionary) con esquema  ← solo iPhone compatible
      │    │   (si no hay Apple Intelligence: POST quick-capture con el texto crudo, Wind lo interpreta)
      │    ├─ Show Result / Choose from Menu "Guardar" | "Corregir"
      │    └─ POST transacción
      └─ Foto de comprobante
           ├─ Take Photo
           ├─ Use Model (Cloud) "extrae total, comercio y fecha" (opcional)
           ├─ POST transacción
           └─ POST comprobante (Form multipart con la foto)
 └─ [3] Leer respuesta: ok, folio, url
      ├─ ok = true  → Show Notification "Registrado en Wind: $450.00 · Uber (T-0421)"  [+ Open URL opcional]
      └─ ok = false → Show Alert con message → ofrecer "Abrir en Wind" (opción B con los datos prellenados)
```

### 5.2 Acciones concretas (lo que se arma en la app Atajos)

| # | Acción | Configuración |
|---|---|---|
| 1 | `Get File` (carpeta Shortcuts/Wind/config.json) o `Get Value` (iOS 27) | Si falla → `Ask for Input` "Pega tu API Key de Wind" → `Save File` / `Set Value` |
| 2 | `Get Dictionary from Input` | Variables `api_key`, `base_url`, `cuenta_default` |
| 3 | `Choose from Menu` | "¿Qué quieres registrar?" con 4 opciones |
| 4 | `Ask for Input` | Tipo Number, "¿Cuánto?", teclado numérico |
| 5 | `Ask for Input` | Tipo Text, "Concepto" |
| 6 | `Get Contents of URL` | `GET {base_url}/transaction-categories?type=expense` · Header `Authorization: Bearer {api_key}` |
| 7 | `Get Dictionary Value` → `Repeat with Each` → `Choose from List` | Mostrar `name`, guardar `id` (truco estándar: diccionario nombre→id y elegir por clave) |
| 8 | `Current Date` → `Format Date` ISO 8601 | Zona horaria del dispositivo |
| 9 | `Dictionary` | Ver JSON en 5.3 |
| 10 | `Get Contents of URL` | `POST {base_url}/transactions` · Headers `Authorization`, `Content-Type: application/json`, `Idempotency-Key: {UUID}` · Body JSON = diccionario anterior |
| 11 | `Get Dictionary Value` `ok` → `If` | Rama éxito / error |
| 12 | `Show Notification` o `Show Alert` | Texto con monto, concepto y folio |

Ajustes del atajo: nombre "Registrar en Wind", ícono con color de Wind, "Mostrar en Apple Watch" activado, "Pin en Control Center" opcional. Luego Ajustes → Botón de acción → Atajo → "Registrar en Wind".

### 5.3 Contrato propuesto para la API (lo que debe construir backend)

Base: la URL base que ya muestra la sección Desarrolladores. Autenticación: el header que ya exige la API pública con la API Key.

```http
POST /transactions
Authorization: Bearer wind_sk_xxxxxxxx
Content-Type: application/json
Idempotency-Key: 6D2A4C1E-...

{
  "type": "expense",                       // "expense" | "income"
  "amount": 450.00,
  "currency": "MXN",
  "transaction_date": "2026-10-04T13:22:00-06:00",
  "concept": "Uber al aeropuerto",
  "category_id": 12,
  "bank_account_id": 3,
  "payment_method": "card",                // opcional
  "client_id": null,                       // opcional
  "provider_id": null,                     // opcional
  "source": "ios_shortcut",                // para analítica y para el badge "Capturado desde iPhone"
  "shortcut_version": "1.0"
}
```

Respuesta **siempre 200 con JSON** (Shortcuts no ve el status code):

```json
{ "ok": true, "id": 9876, "folio": "T-0421", "url": "https://windapp.ai/finance/transactions/9876" }
{ "ok": false, "code": "validation_error", "message": "La categoría 12 no existe para egresos." }
```

Endpoints de apoyo:

| Endpoint | Para qué |
|---|---|
| `GET /transaction-categories?type=expense\|income` | Menú dinámico de categorías (`id`, `name`). Respuesta pequeña, cacheable |
| `GET /bank-accounts` | Menú de cuentas (`id`, `name`, `is_default`) |
| `POST /transactions/{id}/receipt` (multipart `file`) | Adjuntar la foto del comprobante que tomó el atajo |
| `POST /transactions/quick-capture` `{ "text": "gasté 450 en uber…" }` | Interpretación en servidor con el parser de Navi. Devuelve la transacción propuesta (`ok`, campos, `confidence`) o la crea con estado "por revisar". Hace que el dictado funcione en iPhones sin Apple Intelligence |
| `GET /shortcut/version` | `{ "version": "1.2", "install_url": "https://www.icloud.com/shortcuts/…" }` para auto-actualizar el atajo |

Reglas:

- **Idempotencia** por header (el usuario puede tocar dos veces el botón; las automatizaciones reintentan).
- La key del atajo debe poder crearse con **scope mínimo**: crear transacción + leer catálogos. Sugerencia de alias automático: "Atajo iPhone de {nombre}".
- Registrar `source=ios_shortcut` para medir adopción y mostrarlo en el detalle de la transacción.
- Latencia objetivo < 2 s (las automatizaciones tienen ~30 s totales, y el usuario está mirando el teléfono).

### 5.4 Frontend / producto en Wind

1. **Página "Conecta tu iPhone"** (Perfil de usuario o Desarrolladores): botón "Crear key para mi iPhone" (scope mínimo, se muestra una vez), botón "Obtener el atajo" (link de iCloud) con QR para abrirlo desde la computadora, y pasos ilustrados para asignar el Action Button, Back Tap, Siri y Control Center.
2. `/finance/transactions/new` debe **leer query params** (`type`, `amount`, `concept`, `category_id`, `bank_account_id`) para la opción B y para el fallback de errores.
3. Corregir `manifest.json` (nombre "Wind", íconos, `start_url`).
4. Badge "Capturado desde iPhone" cuando `source=ios_shortcut`.
5. Backlog: ítem "Conectar Wind con Atajos de Apple (Action Button)" en Ubicación "Transacciones", y un ítem de Infraestructura/Desarrolladores para el `POST` de transacciones en la API pública.

---

## 6. Fase 2: captura automática y lenguaje natural

### 6.1 Automatización con Apple Pay (trigger Wallet)

- Shortcuts → Automatización → Wallet (Transaction) → elegir tarjetas → "Run Immediately".
- El atajo recibe monto, comercio y tarjeta. Hace `POST /transactions` con `category_id` vacío y `status: "por_categorizar"` (o la categoría que Wind infiera por comercio). Wind ya tiene el evento `transaction_categorized`, así que la categorización posterior ya es un concepto del producto.
- Limitaciones a comunicar al usuario: solo pagos NFC en tienda, depende del banco, y hay un bug abierto en iOS 18 con algunas Mastercard. Es un complemento, no la vía principal.

### 6.2 Automatización con notificaciones del banco (iOS 27)

- Trigger "Cuando reciba una notificación de [BBVA México / Banorte / Santander…]" con filtro por "Compra" o "Cargo".
- `Use Model` (Dictionary) extrae monto y comercio del texto de la notificación, o se manda el texto crudo a `quick-capture` y Wind lo interpreta.
- Cubre compras en línea y pagos con tarjeta física, que Apple Pay no cubre. Es la pieza que más se parece al "piloto automático" que promete el landing de Wind.

### 6.3 Dictado con Navi

- Hoy Navi propone la acción "Crear transacción" y el usuario la aprueba en la UI. Para el atajo, dos caminos: (a) `quick-capture` reutiliza el parser y responde la propuesta para que el atajo la confirme con un `Choose from Menu` "Guardar / Corregir", o (b) se permite auto-aprobar cuando el canal es el atajo y la confianza es alta.
- Puerta alterna sin backend nuevo: el atajo usa la acción "Send Message via WhatsApp" (WhatsApp sí publica acciones de Shortcuts) hacia el número de Navi con el texto dictado; la aprobación ocurre en WhatsApp. Más lento, pero cero desarrollo.

---

## 7. Fase 3: app nativa con App Intents

Cuando Wind tenga app en App Store (puede ser un contenedor delgado), los App Intents de la sección 1.9 dan: acción "Registrar transacción" lista en la app Atajos sin configurar nada, Siri por voz con parámetros, **Control para el Action Button y Control Center** (sin pasar por Atajos), widget interactivo "+ Gasto" en la pantalla de inicio, y entidades (categorías, cuentas, clientes) que `Use Model` puede razonar. La API Key desaparece: el intent usa la sesión de la app.

---

## 8. Seguridad y límites

- La key vive en texto plano dentro del atajo o en iCloud Drive del usuario. Mitigación: scope mínimo, revocación desde Desarrolladores, expiración opcional, "último uso" visible, y alertar si una key del atajo se usa desde una IP/UA que no parece iOS Shortcuts.
- Rate limit por key ya existe; el atajo hace 2-3 llamadas por captura.
- No pedir datos sensibles en el atajo (RFC, CLABE). Solo monto, concepto, categoría, cuenta.
- Apple Intelligence (Use Model) manda el texto a Private Cloud Compute o ChatGPT según configuración del usuario; para datos sensibles preferir `quick-capture` en servidor de Wind.
- Los atajos compartidos por link de iCloud pasan por validación de Apple; no incluir keys reales al publicar la plantilla (usar Import Questions o configuración en archivo).

---

## 9. Plan de trabajo sugerido

| Fase | Entregables | Esfuerzo aproximado |
|---|---|---|
| 1 | `POST /transactions`, `GET` categorías y cuentas, `POST` comprobante, `GET /shortcut/version`; página "Conecta tu iPhone"; query params en el formulario; atajo "Registrar en Wind" v1 (menú: Egreso / Ingreso / Foto); QA en iPhone 15 Pro+ y en un iPhone sin Action Button (Back Tap/Widget) | Backend 3-5 días, frontend 2-3 días, atajo y pruebas 1-2 días |
| 2 | `quick-capture` con Navi; rama "Dictar" (Use Model + fallback servidor); plantillas de automatización Apple Pay y notificación bancaria (iOS 27); fix manifest PWA | 1-2 semanas |
| 3 | App iOS contenedor + App Intents (`RegistrarTransaccionIntent`, `ControlWidget`, App Shortcuts) | A evaluar junto con la decisión de app nativa |

Primer paso concreto: confirmar en la documentación de la API pública (sección Desarrolladores, con sesión) el nombre exacto del header de autenticación y si ya existe `POST` de transacciones. Si existe, la Fase 1 se reduce a los endpoints de catálogo, la página de conexión y el atajo.

---

## 10. Fuentes

Apple (documentación y WWDC):
- Shortcuts User Guide: "Request your first API" (`Get Contents of URL`): https://support.apple.com/guide/shortcuts/request-your-first-api-apd58d46713f/ios
- "Use Apple Intelligence in Shortcuts" (Use Model, modelos, Follow Up, idiomas): https://support.apple.com/guide/shortcuts/use-apple-intelligence-in-shortcuts-tpg3vrvwmclv/ios
- "What's new in Shortcuts for iOS 26": https://support.apple.com/en-us/125148
- Share shortcuts / Import Questions: https://support.apple.com/guide/shortcuts/share-shortcuts-apdf01f8c054/ios
- Run a shortcut from a URL (`shortcuts://run-shortcut`): https://support.apple.com/guide/shortcuts/run-a-shortcut-from-a-url-apd624386f42/ios
- Use another app's URL scheme / x-callback-url: https://support.apple.com/guide/shortcuts/use-another-apps-url-scheme-apd68802640c/ios
- Setting triggers (automatizaciones): https://support.apple.com/guide/shortcuts/setting-triggers-apde31e9638b/ios
- Get Dictionary Value / Dictionaries / Lists: https://support.apple.com/guide/shortcuts/get-dictionary-value-action-apdf01294032/ios · https://support.apple.com/guide/shortcuts/dictionaries-apd43b69f337/ios
- WWDC25 "Develop for Shortcuts and Spotlight with App Intents" (Use Model + App Entities, salida Dictionary): https://developer.apple.com/videos/play/wwdc2025/260/
- WWDC25 "Get to know App Intents": https://developer.apple.com/videos/play/wwdc2025/244/
- WWDC26 "What's new in Shortcuts" (automatización por notificación, Storage/Get-Set Value, Transcript inspector): https://developer.apple.com/videos/play/wwdc2026/310/
- WWDC24 "Bring your app's core features to users with App Intents": https://developer.apple.com/videos/play/wwdc2024/10210/
- Apple Intelligence: idiomas y disponibilidad (incluye español de México): https://www.apple.com/ios/feature-availability/ · https://support.apple.com/en-us/121115

Action Button, Controls, iOS 27:
- MacRumors, asignar Controls al Action Button (iOS 18): https://www.macrumors.com/how-to/assign-control-center-iphone-action-button/
- 9to5Mac, iOS 27 Shortcuts 35+ acciones nuevas: https://9to5mac.com/2026/09/29/ios-27s-shortcuts-app-adds-35-new-improved-actions-heres-whats-new/
- MacRumors, iOS 27 crea atajos con lenguaje natural: https://www.macrumors.com/2026/06/09/apple-shortcuts-ios-27-upgrade/
- 9to5Mac, acciones nuevas en iOS 26: https://9to5mac.com/2025/12/09/ios-26s-shortcuts-app-adds-25-new-actions-heres-everything-new/
- MacStories sobre Use Model (límites, Dictionary, imágenes): https://www.macstories.net/notes/i-have-many-questions-about-apples-updated-foundation-models-and-the-great-use-model-action-in-shortcuts/

Atajos de gastos (los "videos"):
- Finny, Action Button expense tracker: https://getfinny.app/blog/action-button-expense-tracker-iphone-15-16-pro
- Finny, tutorial general: https://getfinny.app/blog/iphone-expense-tracker-shortcut-tutorial-2026
- Finny, Numbers: https://getfinny.app/blog/track-expenses-apple-numbers-shortcut-2026
- Easlo, gastos a Notion con Apple Shortcuts y con Apple Wallet: https://www.easlo.co/blog/add-expenses-to-notion-using-apple-shortcuts · https://www.easlo.co/blog/add-expenses-to-notion-when-you-tap-your-apple-wallet
- Alex de Pablos, atajo iOS + Notion API: https://alexdepablos.com/en/software-engineering/connect-ios-shortcut-with-notion/
- Medium, atajo + Google Sheets (Apps Script): https://medium.com/@lopesjuliano/como-configurar-um-atalho-do-iphone-para-atualizar-sua-planilha-de-despesas-no-google-sheets-a94709554e11
- Power Automate "When an HTTP request is received" → Excel "Add a row into a table": https://www.spguides.com/add-rows-to-excel-in-power-automate/
- CashJot, Apple Pay tracking (campos, límites, iOS 27): https://www.cashjot.com/blog/apple-pay-expense-tracking
- Graham Haley, Apple Pay automation (solo NFC): https://grahamhaley.co.uk/2024/11/19/apple-pay-automation/
- Apple Developer Forums, timeouts del trigger Transaction (bug abierto): https://developer.apple.com/forums/thread/765516
- TikTok ES: @milecrow30 "atajo control de gastos" y automatización con Apple Pay; @thequickflow atajo de gastos a Notion

Técnico:
- Superwall, App Intents field guide (código AppIntent, AppShortcutsProvider, ControlWidget): https://superwall.com/blog/an-app-intents-field-guide-for-ios-developers
- Expo App Intents (SDK 58): https://docs.expo.dev/versions/v58.0.0/sdk/app-intents/
- RoutineHub, POST con Shortcuts: https://blog.routinehub.co/how-to-send-a-post-request-with-apple-shortcuts/
- Matthew Cassinelli, referencia de `Get Contents of URL`: https://matthewcassinelli.com/actions/get-contents-of-url/
- Automators Talk, multipart/form con fotos: https://talk.automators.fm/t/send-post-request-with-get-contents-of-url/15943
- PWA deep links en iOS (no es posible desde Shortcuts): https://intercom.help/progressier/en/articles/6902113-complete-guide-to-pwa-deep-links
- Michael Tsai, actualizar atajos compartidos (links de iCloud son snapshot): https://mjtsai.com/blog/2026/05/20/updating-shared-shortcuts/
