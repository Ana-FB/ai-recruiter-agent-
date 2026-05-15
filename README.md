# 🤖 AI Recruiter Agent — Sourcing IT

![n8n](https://img.shields.io/badge/n8n-workflow-orange?logo=n8n&logoColor=white)
![Claude](https://img.shields.io/badge/Claude-Sonnet_4.6-8B5CF6?logo=anthropic&logoColor=white)
![Slack](https://img.shields.io/badge/Slack-integration-4A154B?logo=slack&logoColor=white)
![Airtable](https://img.shields.io/badge/Airtable-shortlist-18BFFF?logo=airtable&logoColor=white)
![SerpAPI](https://img.shields.io/badge/SerpAPI-LinkedIn_search-green)
![License](https://img.shields.io/badge/license-All%20Rights%20Reserved-red)

Agente de IA conversacional construido en **n8n** que actúa como AI Recruiter Senior especializado en Talent Acquisition IT. Recibe la descripción de una vacante por Slack, busca candidatos en LinkedIn, los evalúa con scoring automático y guarda los mejores en un shortlist en Airtable.

> 👤 **Creado por [Ana Ferreira](https://www.linkedin.com/in/anaferreirabezerra)** — HR Operations Analyst · People Analytics · HR Automation Specialist · 📩 [ani.fb95@gmail.com](mailto:ani.fb95@gmail.com) · [WhatsApp](https://wa.me/5491135077374)

---

## Tabla de contenidos

- [¿Qué hace?](#qué-hace)
- [Arquitectura](#arquitectura)
- [Nodos del workflow](#nodos-del-workflow)
- [Flujo de ejecución](#flujo-de-ejecución)
- [Criterios de scoring](#criterios-de-scoring)
- [Integraciones y credenciales](#integraciones-y-credenciales)
- [Configuración](#configuración)
- [Cómo usar](#cómo-usar)
- [Output esperado](#output-esperado)
- [Límites de operación](#límites-de-operación)
- [Stack](#stack)

---

## ¿Qué hace?

1. El recruiter describe una vacante en lenguaje natural por un canal de Slack
2. El agente responde de inmediato confirmando que está buscando
3. Analiza los requisitos y construye una query optimizada para buscar perfiles en LinkedIn vía Google (SerpAPI)
4. Evalúa cada candidato con un score de match del 0 al 100
5. Los candidatos con score ≥ 60 se guardan automáticamente en Airtable
6. Al finalizar presenta un resumen rankeado con próximos pasos sugeridos

---

## Arquitectura

```
[Slack Trigger]
      │
      ▼
[Filtro Bot]  ── es bot ──▶ (descartado, sin conexión)
      │
      es humano
      │
      ▼
[¿Es una vacante?] ── NO (< 15 chars) ──▶ [Respuesta Saludo] ──▶ [Slack Send Saludo]
      │
      SÍ
      │
      ▼
[Slack Send Procesando]   → "Entendido, estoy buscando..."
      │
      ▼
[AI Recruiter Agent]  ◀── [Claude Sonnet 4.6]
      │               ◀── [Memoria de Sesión (20 msgs)]
      │
      ├──▶ [buscar_candidatos]           → SerpAPI / Google Search
      ├──▶ [guardar_candidato_shortlist] → Airtable (POST)
      └──▶ [consultar_shortlist]         → Airtable (GET)
      │
      ▼
[Slack Send Respuesta]    → resumen final rankeado
```

---

## Nodos del workflow

| # | Nodo | Tipo | Función |
|---|------|------|---------|
| 1 | Slack Trigger | `slackTrigger` | Escucha mensajes en todo el workspace vía webhook |
| 2 | Filtro Bot | `IF` | Descarta mensajes de bots para evitar loops infinitos |
| 3 | ¿Es una vacante? | `IF` | Si el mensaje tiene más de 15 caracteres pasa al agente; si no, devuelve saludo |
| 4 | Slack Send Procesando | `slack` | Respuesta inmediata: "Estoy buscando candidatos..." |
| 5 | AI Recruiter Agent | `agent (toolsAgent)` | Orquesta el flujo completo de sourcing |
| 6 | Anthropic Chat Model | `lmChatAnthropic` | Claude Sonnet 4.6 como LLM del agente |
| 7 | Memoria de Sesión | `memoryBufferWindow` | Contexto de las últimas 20 interacciones |
| 8 | buscar_candidatos | `httpRequestTool` | Búsqueda de perfiles LinkedIn en Google vía SerpAPI |
| 9 | guardar_candidato_shortlist | `httpRequestTool` | Guarda candidatos con score ≥ 60 en Airtable |
| 10 | consultar_shortlist | `httpRequestTool` | Lee el shortlist actual para evitar duplicados |
| 11 | Slack Send Respuesta | `slack` | Envía el resumen final rankeado al canal |
| 12 | Respuesta Saludo | `Set` | Mensaje de bienvenida para inputs cortos |
| 13 | Slack Send Saludo | `slack` | Envía el saludo al canal |

---

## Flujo de ejecución

1. Recruiter envía la descripción de la vacante en el canal de Slack

2. **Filtro Bot**: si el mensaje viene de un bot → descartado (evita loops)

3. **Filtro**: si el mensaje tiene menos de 15 caracteres → saludo, fin

4. El agente responde de inmediato: `"Entendido, estoy buscando candidatos..."`

5. El agente analiza la vacante:
   - Hard skills requeridas
   - Nivel de seniority
   - Stack tecnológico
   - Idioma y ubicación

6. Consulta Airtable para obtener URLs ya guardadas (evitar duplicados)

7. Construye la search query, ejemplo:
   ```
   site:linkedin.com/in "Senior Data Engineer" "Spark" "Databricks" "Latinoamérica"
   ```

8. Llama a SerpAPI → extrae de cada resultado:
   - Nombre completo, cargo actual, URL de LinkedIn, snippet del perfil

9. Calcula match score (0–100) por candidato

10. Para cada candidato con **score ≥ 60** y URL no duplicada:
    → Llama a `guardar_candidato_shortlist` (Airtable POST)

11. Presenta resumen final rankeado por score descendente en Slack

---

## Criterios de scoring

| Criterio | Peso |
|----------|------|
| Hard skills match | 40% |
| Seniority adecuado | 20% |
| Industria relevante | 10% |
| Nivel de inglés estimado | 10% |
| Liderazgo / mentoring | 10% |
| Ubicación / timezone | 10% |

**Umbral de corte:** score ≥ 60 para guardar en shortlist.

**Seniority detectado:**
- **Junior** — 0 a 2 años, tareas asistidas
- **Semi Senior** — 2 a 4 años, autonomía parcial
- **Senior** — más de 5 años, ownership técnico
- **Staff/Lead** — liderazgo técnico, decisiones de arquitectura

---

## Integraciones y credenciales

| Servicio | Uso | Configuración |
|----------|-----|---------------|
| Anthropic | Claude Sonnet 4.6 como LLM | Credencial Anthropic account en n8n |
| Slack | Canal de entrada y salida del agente | Credencial Slack account en n8n + Slack App con webhook |
| SerpAPI | Búsqueda de perfiles LinkedIn vía Google | API key en el parámetro `api_key` del nodo `buscar_candidatos` |
| Airtable | Almacenamiento del shortlist | Credencial Airtable Personal Access Token account en n8n |

### Campos requeridos en Airtable

| Campo | Tipo | Descripción |
|-------|------|-------------|
| Nombre | Texto | Nombre completo del candidato |
| Rol Actual | Texto | Cargo actual |
| Ubicación | Texto | Ciudad y país |
| LinkedIn URL | URL | Perfil de LinkedIn |
| Experiencia | Texto largo | Resumen de experiencia |
| Educación | Texto | Formación académica |
| Skills Match | Texto | Skills que matchean, separadas por coma |
| Skills Faltantes | Texto | Skills que faltan, separadas por coma |
| Seniority | Select | Junior / Semi Senior / Senior / Staff/Lead |
| Match Score | Número | 0 a 100 |
| Vacante Aplicada | Texto | Nombre o descripción de la vacante |
| Estado | Select | Nuevo / En proceso / Contactado / Descartado |

---

## Configuración

### Prerrequisitos

- [n8n](https://n8n.io) instalado (self-hosted o cloud)
- Cuenta de [Anthropic](https://console.anthropic.com) con API key
- Cuenta de [Slack](https://api.slack.com/apps) con una Slack App creada
- Cuenta de [SerpAPI](https://serpapi.com) con API key
- Cuenta de [Airtable](https://airtable.com) con una base y tabla configurada

### Pasos

1. Importar el workflow JSON en n8n
2. Configurar las credenciales:
   - **Anthropic account** → API key de Anthropic
   - **Slack account** → token de la Slack App
   - **Airtable Personal Access Token account** → token de Airtable
3. En el nodo `buscar_candidatos`, reemplazar el valor de `api_key` con tu API key de SerpAPI
4. Verificar el ID de la base y tabla de Airtable en los nodos `guardar_candidato_shortlist` y `consultar_shortlist`

### Configuración de la Slack App

1. Crear una nueva app en [https://api.slack.com/apps](https://api.slack.com/apps) usando un manifest
2. Habilitar **Event Subscriptions** y agregar la URL del webhook de n8n:
   ```
   https://tu-dominio.com/webhook/TU_WEBHOOK_ID
   ```
3. Suscribirse al evento `message.channels`
4. Instalar la app en el workspace y agregarla al canal deseado
5. Activar el workflow en n8n **antes** de verificar el webhook en Slack

> **Nota:** Si n8n está detrás de Cloudflare Access, crear una política de tipo **Bypass** para la ruta `/webhook` que permita acceso de todos (Everyone), de lo contrario Slack no podrá verificar el endpoint.

---

## Cómo usar

1. Agregar la Slack App al canal donde quieras usar el agente
2. Escribir la vacante en lenguaje natural:

```
Busco un Senior Backend Developer con Python y FastAPI,
experiencia en arquitectura de microservicios,
preferentemente en Argentina, inglés avanzado requerido.
```

3. El agente responde de inmediato confirmando que está buscando
4. Realiza la búsqueda, evaluación y guardado automáticamente
5. Al finalizar devuelve un resumen con tabla rankeada y próximos pasos sugeridos

---

## Output esperado

### Confirmación inmediata

```
Entendido, estoy buscando candidatos en LinkedIn... dame un momento.
```

### Por candidato (score ≥ 60)

```json
{
  "candidate_name": "Nombre Apellido",
  "linkedin_url": "https://linkedin.com/in/username",
  "current_role": "Cargo actual del candidato",
  "location": "Ciudad, País",
  "seniority_detected": "Senior",
  "match_score": 0,
  "skills_match": ["skill_1", "skill_2"],
  "missing_skills": ["skill_faltante"],
  "strengths": ["fortaleza_1", "fortaleza_2"],
  "red_flags": ["red_flag_1"],
  "summary": "Análisis crítico del perfil en 2-3 líneas."
}
```

### Resumen final

```
Total perfiles analizados: N
Total perfiles saltados por duplicado: N
Total guardados en Airtable: N

| Nombre | Rol | Ubicación | Score |
|--------|-----|-----------|-------|
| ...    | ... | ...       | ...   |

Próximos pasos sugeridos: ...
```

---

## Límites de operación

| Límite | Valor |
|--------|-------|
| Perfiles analizados por búsqueda | 10 |
| Llamadas a `buscar_candidatos` por conversación | 2 |
| Guardados en Airtable por conversación | 10 |
| Mensajes en memoria de sesión | 20 |

> **Nota:** Saludos y mensajes cortos (menos de 15 caracteres) no consumen herramientas ni créditos de API.

---

## Stack

| Herramienta | Rol |
|-------------|-----|
| [n8n](https://n8n.io) | Orquestación de workflows |
| [Claude Sonnet 4.6](https://anthropic.com) | Modelo de lenguaje |
| [Slack](https://slack.com) | Interfaz de chat |
| [SerpAPI](https://serpapi.com) | Búsqueda Google / LinkedIn |
| [Airtable](https://airtable.com) | Base de datos del shortlist |

---

## 👤 Autor

**Ana Ferreira**  
HR Operations Analyst · People Analytics · HR Automation Specialist · BI & Data Analyst  
SQL · Power BI · Looker Studio · n8n · Make · SAP SuccessFactors

📩 **Contacto:** [ani.fb95@gmail.com](mailto:ani.fb95@gmail.com) · [LinkedIn](https://www.linkedin.com/in/anaferreirabezerra) · [WhatsApp](https://wa.me/5491135077374)

---

## ⚠️ Licencia

© 2025 Ana Ferreira. Todos los derechos reservados.

Este proyecto es de **uso personal y educativo**. Queda prohibida su reproducción, distribución o uso comercial sin autorización expresa de la autora.
