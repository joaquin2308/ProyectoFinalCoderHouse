# Workflow Automatizado de Gestión de Comentarios en Redes Sociales
> Proyecto Final — IA Automation | CoderHouse 2026
> Alumno: Joaquín Capdevila

---

## Descripción

Workflow automatizado en **n8n** que procesa comentarios de redes sociales en tiempo real. Cuando se agrega un comentario en una hoja de Google Sheets, el sistema lo clasifica automáticamente usando **Groq (LLaMA 3.3)**, genera una respuesta personalizada según el tipo de comentario, notifica al equipo vía **Telegram** en caso de quejas urgentes, y registra todo en la misma hoja de cálculo.

**Flujo completo:**
```
Google Sheets Trigger → Agente Orquestador (LLM 1) → Code JS (Parser)
→ Tool Switch → LLM por rama → Telegram (si queja) → Update row in sheet
```

---

## Arquitectura del Workflow

| # | Nodo | Función |
|---|---|---|
| 1 | Google Sheets Trigger | Detecta nueva fila con comentario (Row Added) |
| 2 | Agente Orquestador | LLM 1 — Clasifica el comentario en 4 categorías y devuelve JSON |
| 3 | Code in JavaScript | Limpia la respuesta del LLM y extrae el campo `tool` |
| 4 | Tool Switch | Enruta el flujo según la categoría detectada |
| 5a | Generador de Disculpas | LLM 2 — Genera respuesta empática para quejas urgentes |
| 5b | Sugerencia Producto | LLM 3 — Genera respuesta entusiasta para sugerencias |
| 5c | Fallback LLM | LLM 4 — Genera respuesta genérica para comentarios ambiguos |
| 5d | (sin LLM) | Spam/Troll — Solo clasificación, sin respuesta |
| 6 | Send a text message | Telegram — Notifica al equipo solo en quejas urgentes |
| 7 | Update row in sheet | Registra clasificación, respuesta sugerida y fecha en el Sheet |

---

## Requisitos Previos

### Cuentas necesarias
- ✅ Cuenta de **Google** (Google Sheets)
- ✅ Cuenta en **Groq** → [console.groq.com](https://console.groq.com)
- ✅ Bot de **Telegram** creado con @BotFather
- ✅ Instancia de **n8n** (cloud o self-hosted)

### APIs y credenciales necesarias
| Servicio | Dónde obtenerla |
|---|---|
| **Groq API Key** | [console.groq.com](https://console.groq.com) → API Keys → Create API Key |
| **Google Sheets OAuth** | Configurar credencial Google en n8n |
| **Telegram Bot Token** | @BotFather en Telegram → /newbot |
| **Telegram Chat ID** | Enviar mensaje al bot y consultar `https://api.telegram.org/bot<TOKEN>/getUpdates` |

---

## Configuración Paso a Paso

### Paso 1 — Preparar el Google Sheet
1. Creá una hoja en Google Sheets con estas columnas:
   - `ID` | `Usuario` | `Comentario` | `Red Social` | `Fecha` | `Clasificacion` | `Respuesta Sugerida`
2. Populate manualmente `ID`, `Usuario`, `Comentario` y `Red Social` por ahora (en producción vendría de APIs de redes sociales)

### Paso 2 — Configurar el Trigger en n8n
1. Agregá el nodo **Google Sheets Trigger**
2. Conectá tu cuenta de Google
3. Configurá:
   - **Document:** `ComentariosRedes`
   - **Sheet:** `Hoja 1`
   - **Trigger On:** `Row Added`
   - **Poll Times:** `Every Minute`

### Paso 3 — Configurar el Agente Orquestador
1. Conectá el nodo **Groq Chat Model** con tu API Key
2. Modelo recomendado: `llama-3.3-70b-versatile`
3. Pegá el prompt del clasificador (ver `prompts.txt`)
4. Asegurate que el campo **Prompt (User Message)** termine con:
   ```
   Comentario a clasificar: {{ $json["Comentario"] }}
   ```

### Paso 4 — Configurar el Code in JavaScript
El nodo JS cumple dos funciones:
1. Limpia la respuesta del LLM (elimina bloques markdown ` ```json `)
2. Extrae el campo `tool` para enrutar al Tool Switch
3. Si el JSON falla, clasifica por palabras clave como fallback

### Paso 5 — Configurar el Tool Switch
1. Modo: **Rules**
2. Agregar 4 reglas:
   - `{{ $json.tool }}` is equal to `queja_urgente`
   - `{{ $json.tool }}` is equal to `sugerencia_producto`
   - `{{ $json.tool }}` is equal to `spam_o_troll`
   - `{{ $json.tool }}` is equal to `fallback`

### Paso 6 — Configurar los LLMs por rama
Conectá un nodo **Groq** en cada rama con su prompt correspondiente (ver `prompts.txt`):
- Rama `queja_urgente` → Generador de Disculpas
- Rama `sugerencia_producto` → Sugerencia Producto
- Rama `fallback` → Fallback LLM
- Rama `spam_o_troll` → directo al Update (sin LLM)

### Paso 7 — Configurar Telegram
1. Creá un bot con @BotFather y copiá el token
2. Enviá un mensaje al bot y obtené tu Chat ID via API
3. En n8n, agregá el nodo **Telegram → Send a text message**
4. Conectalo después del nodo Generador de Disculpas
5. Configurá el mensaje con los datos del comentario y la respuesta sugerida

### Paso 8 — Configurar Update row in sheet
1. Conectá todas las ramas al nodo **Google Sheets → Update Row**
2. Columna de match: `ID`
3. Campos a actualizar:
   - `Clasificacion` → `{{ $node["Code in JavaScript"].json["tool"] }}`
   - `Respuesta Sugerida` → expresión con `$if` para cada rama
   - `Fecha` → `{{ $now.toFormat('dd/MM/yyyy HH:mm') }}`

---


## Casos de Prueba

Para verificar el funcionamiento correcto, probá con estos comentarios:

| Categoría esperada | Comentario de prueba |
|---|---|
| `queja_urgente` | "No puedo entrar a mi cuenta desde ayer, ya reinstalé la app y sigue sin funcionar" |
| `sugerencia_producto` | "Estaría bueno poder exportar los reportes en PDF" |
| `spam_o_troll` | "💩💩💩" |
| `fallback` | "Hola buenos días, ¿tienen atención al cliente por acá?" |

**Resultado esperado por caso:**
- `queja_urgente` → Respuesta empática + notificación Telegram + registro en Sheet
- `sugerencia_producto` → Respuesta entusiasta + registro en Sheet
- `spam_o_troll` → Sin respuesta + clasificación registrada en Sheet
- `fallback` → Respuesta genérica + registro en Sheet

---

## Mejoras Futuras

- Conectar con APIs reales de Twitter/Instagram para que los comentarios lleguen automáticamente
- Agregar análisis de urgencia numérica (campo `urgency` del JSON ya está disponible)
- Dashboard en Google Sheets con métricas diarias de clasificación
- Notificaciones diferenciadas por nivel de urgencia en Telegram
- Integración con CRM para seguimiento de quejas urgentes no resueltas

---

Joaquín Capdevila
Proyecto Final — IA Automation
CoderHouse 2026
