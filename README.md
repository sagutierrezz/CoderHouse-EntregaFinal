# 🚗 Sunchales Motors AI — Asistente Comercial con n8n

> Workflow de automatización que convierte Telegram en un asesor comercial con IA para una concesionaria.

---

## 📋 Tabla de Contenidos

1. [Descripción](#descripción)
2. [Requisitos Previos](#requisitos-previos)
3. [Credenciales necesarias](#credenciales-necesarias)
4. [Configuración paso a paso](#configuración-paso-a-paso)
5. [Estructura del Google Sheets](#estructura-del-google-sheets)
6. [Pruebas Reproducibles](#pruebas-reproducibles)

---

## Descripción

**Sunchales Motors AI** es un bot de Telegram que actúa como asesor comercial ("Enzo") para una concesionaria de vehículos. El workflow:

- Recibe mensajes de texto **y notas de voz** por Telegram
- Transcribe el audio con **OpenAI Whisper**
- Clasifica la consulta (categoría, modelo, nivel de interés) con **GPT-4.1-mini**
- Responde con información del stock en tiempo real desde **Google Sheets**
- Registra automáticamente los prospectos en una hoja de seguimiento
- Mantiene **memoria conversacional** por usuario dentro de cada sesión

---

## Requisitos Previos

| Requisito | Versión mínima | Notas |
|-----------|---------------|-------|
| n8n | 1.40+ | Self-hosted o cloud |
| Cuenta OpenAI | — | Con acceso a GPT-4.1-mini y Whisper |
| Bot de Telegram | — | Creado con @BotFather |
| Google Sheets | — | Cuenta con OAuth2 habilitado |


---

## Credenciales Necesarias

Antes de importar, preparar las siguientes credenciales en n8n:

### 1. Telegram Bot Token

1. Abrir Telegram y buscar `@BotFather`
2. Ejecutar `/newbot` y seguir las instrucciones
3. Copiar el **Bot Token** (formato: `123456789:ABCdef...`)
4. En n8n → Settings → Credentials → New → **Telegram API**
5. Pegar el token

### 2. OpenAI API Key

1. Ir a [platform.openai.com](https://platform.openai.com)
2. Settings → API Keys → Create new secret key
3. En n8n → Settings → Credentials → New → **OpenAI API**
4. Pegar la API Key

> ⚠️ Asegurate de tener créditos disponibles y acceso al modelo `gpt-4.1-mini`.

### 3. Google Sheets OAuth2

1. Ir a [console.cloud.google.com](https://console.cloud.google.com)
2. Crear un proyecto → APIs & Services → Enable **Google Sheets API**
3. Create credentials → OAuth 2.0 Client ID → Web application
4. Agregar como Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`
5. En n8n → Settings → Credentials → New → **Google Sheets OAuth2 API**
6. Pegar Client ID y Client Secret → conectar cuenta


---

## Estructura del Google Sheets

Necesitás una planilla con **al menos 2 hojas**:

### Hoja 1: "Stock" (gid=0)

Inventario de vehículos — esta es la fuente de datos del agente:

| Modelo | Precio USD | KM | Año | Categoria |
|--------|-----------|-----|-----|-----------|
| Toyota Hilux SR 4x4 | 28500 | 45000 | 2022 | Camioneta |
| Volkswagen Polo | 18000 | 12000 | 2023 | Auto |
| Honda CB500F | 7200 | 8000 | 2021 | Moto |
| Ford Ranger XLT | 31000 | 0 | 2024 | Camioneta |

> ⚠️ Las columnas deben tener encabezados en la primera fila. El Agente lee la hoja completa.

### Hoja 2: "Hoja 4" (para prospectos)

Registro automático de clientes interesados:

| Cliente | Categoria | Modelo | Interes |
|---------|-----------|--------|---------|
| 12345678 | Camioneta | Toyota Hilux | Alto |

> Esta hoja se llena automáticamente. Crear los encabezados manualmente la primera vez.

---

## Pruebas Reproducibles

Una vez configurado el workflow, ejecutar estas pruebas para verificar el funcionamiento:

### Test 1 — Consulta de texto simple

**Input (mensaje en Telegram):**
```
Hola, ¿tienen camionetas disponibles?
```

**Comportamiento esperado:**
- Switch → rama Texto ✓
- Message a model clasifica: `{"categoria": "Camioneta", "interes": "Medio", "modelo": ""}`
- AI Agent consulta Google Sheets
- Responde listando camionetas disponibles con precios
- NO registra en Hoja 4 (modelo vacío → If = false)

**Verificación:** La respuesta de Enzo debe mencionar al menos una camioneta con precio en USD.

---

### Test 2 — Consulta con modelo específico

**Input:**
```
Quiero info de la Hilux, ¿cuánto sale?
```

**Comportamiento esperado:**
- Message a model clasifica: `{"categoria": "Camioneta", "interes": "Alto", "modelo": "Toyota Hilux"}`
- Code in JavaScript parsea el JSON correctamente
- If → true (modelo no vacío) → **Agregar cliente** crea fila en Hoja 4
- AI Agent consulta stock y responde con precio y detalles
- Enzo termina con una pregunta de cierre

**Verificación:**
1. Respuesta en Telegram con precio de Hilux
2. Nueva fila en Google Sheets Hoja 4 con: `[chat_id, "Camioneta", "Toyota Hilux", "Alto"]`

---

### Test 3 — Consulta general (sin modelo)

**Input:**
```
¿Qué horarios tienen?
```

**Comportamiento esperado:**
- Clasifica como: `{"categoria": "Consulta_general", "interes": "Bajo", "modelo": ""}`
- If → false → NO registra en Hoja 4
- AI Agent responde con información general de la agencia

---

### Test 4 — Nota de voz

**Input:** Enviar una nota de voz de Telegram diciendo:
```
"Buenas, quiero ver opciones de autos usados, tengo para permutar un Corsa 2014"
```

**Comportamiento esperado:**
- Switch → rama Audio ✓
- Get a file descarga el OGG
- Transcribe a recording → transcripción del audio
- Clasificación: `{"categoria": "Auto", "interes": "Medio", "modelo": ""}`
- Enzo responde mencionando la política de permutas (autos desde 2015)

**Verificación:** Enzo debe mencionar "2015" o la política de permutas en su respuesta.

---

### Test 5 — Memoria conversacional

**Input (mismo chat, mensajes consecutivos):**

Mensaje 1: `Hola, ¿tienen Hilux?`
Mensaje 2: `¿Y cuántos kilómetros tiene?`

**Comportamiento esperado:**
- El segundo mensaje, aunque no menciona "Hilux", recibe una respuesta sobre el kilometraje de la Hilux (gracias a Simple Memory).

**Verificación:** La respuesta al segundo mensaje referencia el vehículo mencionado en el primero.

---

### Verificación del Registro de Leads

Después de ejecutar los Tests 2 y 3:

1. Abrir Google Sheets → Hoja 4
2. Verificar que:
   - Test 2 agregó 1 fila (con modelo "Toyota Hilux")
   - Test 3 NO agregó fila (modelo vacío)

---

## Archivos de la Entrega

| Archivo                             | Descripción                                           |
| ----------------------------------- | ----------------------------------------------------- |
| `EntregaFinal.json`                 | Workflow n8n                                          |
| `Documentación Entrega final.pdf`   | Documentación técnica completa                        |
| `prompts_LLM.txt`                   | Todos los prompts de los nodos LLM extraídos del JSON |
| `README.md`                         | Este archivo — instrucciones de importación y pruebas |


