# AI Recruiter Agent — Sourcing IT

Agente de IA conversacional construido en n8n que actúa como AI Recruiter Senior especializado en Talent Acquisition IT. Recibe la descripción de una vacante por chat, busca candidatos en LinkedIn, los evalúa con scoring automático y guarda los mejores en un shortlist en Airtable.

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

---

## ¿Qué hace?

1. El recruiter describe una vacante en lenguaje natural por el chat de n8n
2. El agente analiza los requisitos y construye una query optimizada para buscar perfiles en LinkedIn vía Google (SerpAPI)
3. Evalúa cada candidato con un score de match del 0 al 100
4. Los candidatos con score ≥ 60 se guardan automáticamente en Airtable
5. Al finalizar presenta un resumen rankeado con próximos pasos sugeridos

---

## Arquitectura

```
[Chat Trigger]
      │
      ▼
[¿Es una vacante?] ── NO (< 15 chars) ──▶ [Respuesta Saludo]
      │
      SÍ
      │
      ▼
[AI Recruiter Agent]  ◀── [Claude Sonnet 4.6]
      │                ◀── [Memoria de Sesión (20 msgs)]
      │
      ├──▶ [buscar_candidatos]         → SerpAPI / Google Search
      ├──▶ [guardar_candidato_shortlist] → Airtable (POST)
      └──▶ [consultar_shortlist]        → Airtable (GET)
```

---

## Nodos del workflow

| # | Nodo | Tipo | Función |
|---|------|------|---------|
| 1 | Chat con Recruiter | `chatTrigger` | Punto de entrada. Recibe el mensaje vía chat UI de n8n |
| 2 | ¿Es una vacante? | `IF` | Si el mensaje tiene más de 15 caracteres pasa al agente; si no, devuelve saludo |
| 3 | Respuesta Saludo | `Set` | Mensaje de bienvenida para inputs cortos |
| 4 | AI Recruiter Agent | `agent` (toolsAgent) | Orquesta el flujo completo de sourcing |
| 5 | Anthropic Chat Model | `lmChatAnthropic` | Claude Sonnet 4.6 como LLM del agente |
| 6 | Memoria de Sesión | `memoryBufferWindow` | Contexto de las últimas 20 interacciones |
| 7 | buscar_candidatos | `httpRequestTool` | Búsqueda de perfiles LinkedIn en Google vía SerpAPI |
| 8 | guardar_candidato_shortlist | `httpRequestTool` | Guarda candidatos con score ≥ 60 en Airtable |
| 9 | consultar_shortlist | `httpRequestTool` | Lee el shortlist actual para evitar duplicados |

---

## Flujo de ejecución

```
1. Recruiter envía la descripción de la vacante por chat

2. Filtro: si el mensaje tiene menos de 15 caracteres → saludo, fin

3. El agente analiza la vacante:
   - Hard skills requeridas
   - Nivel de seniority
   - Stack tecnológico
   - Idioma y ubicación

4. Consulta Airtable para obtener URLs ya guardadas (evitar duplicados)

5. Construye la search query, ejemplo:
   site:linkedin.com/in "Senior React Developer" "TypeScript" "Argentina"

6. Llama a SerpAPI → extrae de cada resultado:
   - Nombre completo, cargo actual, URL de LinkedIn, snippet del perfil

7. Calcula match score (0-100) por candidato

8. Para cada candidato con score ≥ 60 y URL no duplicada:
   → Llama a guardar_candidato_shortlist (Airtable POST)

9. Presenta resumen final rankeado por score descendente
```

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
| **Anthropic** | Claude Sonnet 4.6 como LLM | Credencial `Anthropic account` en n8n |
| **SerpAPI** | Búsqueda de perfiles LinkedIn vía Google | API key en el parámetro `api_key` del nodo `buscar_candidatos` |
| **Airtable** | Almacenamiento del shortlist | Credencial `Airtable Personal Access Token account` en n8n |

### Campos requeridos en Airtable

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `Nombre` | Texto | Nombre completo del candidato |
| `Rol Actual` | Texto | Cargo actual |
| `Ubicación` | Texto | Ciudad y país |
| `LinkedIn URL` | URL | Perfil de LinkedIn |
| `Experiencia` | Texto largo | Resumen de experiencia |
| `Educación` | Texto | Formación académica |
| `Skills Match` | Texto | Skills que matchean, separadas por coma |
| `Skills Faltantes` | Texto | Skills que faltan, separadas por coma |
| `Seniority` | Select | Junior / Semi Senior / Senior / Staff/Lead |
| `Match Score` | Número | 0 a 100 |
| `Vacante Aplicada` | Texto | Nombre o descripción de la vacante |
| `Estado` | Select | Nuevo / En proceso / Contactado / Descartado |

---

## Configuración

### Prerrequisitos

- n8n instalado (self-hosted o cloud)
- Cuenta de [Anthropic](https://console.anthropic.com) con API key
- Cuenta de [SerpAPI](https://serpapi.com) con API key
- Cuenta de [Airtable](https://airtable.com) con una base y tabla configurada

### Pasos

1. Importar el workflow JSON en n8n
2. Configurar las credenciales:
   - `Anthropic account` → API key de Anthropic
   - `Airtable Personal Access Token account` → token de Airtable
3. En el nodo `buscar_candidatos`, reemplazar el valor de `api_key` con tu API key de SerpAPI
4. Verificar el ID de la base y tabla de Airtable en los nodos `guardar_candidato_shortlist` y `consultar_shortlist`
5. Activar el workflow

---

## Cómo usar

1. Abrir el workflow en n8n y hacer clic en el botón **Chat**
2. Escribir la vacante en lenguaje natural:

```
Busco un Senior Backend Developer con Python y FastAPI,
experiencia en arquitectura de microservicios,
preferentemente en Argentina, inglés avanzado requerido.
```

3. El agente realiza la búsqueda, evaluación y guardado automáticamente
4. Al finalizar devuelve un resumen con tabla rankeada y próximos pasos sugeridos

---

## Output esperado

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

> Saludos y mensajes cortos (menos de 15 caracteres) no consumen herramientas ni créditos de API.

---

## Stack

- [n8n](https://n8n.io) — orquestación de workflows
- [Claude Sonnet 4.6](https://anthropic.com) — modelo de lenguaje
- [SerpAPI](https://serpapi.com) — búsqueda Google / LinkedIn
- [Airtable](https://airtable.com) — base de datos del shortlist
