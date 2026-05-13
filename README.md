# 🤖 AI Recruiter Agent — Sourcing IT

![n8n](https://img.shields.io/badge/n8n-workflow-EA4B71?style=flat-square&logo=n8n&logoColor=white)
![Claude](https://img.shields.io/badge/Claude-Sonnet_4.6-6C3BDB?style=flat-square)
![Airtable](https://img.shields.io/badge/Airtable-shortlist-18BFFF?style=flat-square&logo=airtable&logoColor=white)
![SerpAPI](https://img.shields.io/badge/SerpAPI-LinkedIn_Search-4285F4?style=flat-square)
![Status](https://img.shields.io/badge/status-production-brightgreen?style=flat-square)

> Agente de IA conversacional construido en n8n que actúa como AI Recruiter Senior especializado en Talent Acquisition IT. Recibe la descripción de una vacante por chat, busca candidatos en LinkedIn, los evalúa con scoring automático y guarda los mejores en un shortlist en Airtable.

---

## 📸 Workflow

![AI Recruiter Agent Workflow](./workflow-preview.png)

---

## ⚡ ¿Qué hace?

1. El recruiter describe una vacante en **lenguaje natural** por el chat de n8n
2. El agente analiza los requisitos y construye una **query optimizada** para buscar perfiles en LinkedIn vía SerpAPI
3. Evalúa cada candidato con un **score de match del 0 al 100**
4. Los candidatos con score ≥ 60 se guardan automáticamente en **Airtable**
5. Al finalizar presenta un **resumen rankeado** con próximos pasos sugeridos

---

## 🏗️ Arquitectura

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
      ├──▶ [buscar_candidatos]            → SerpAPI / Google Search
      ├──▶ [guardar_candidato_shortlist]  → Airtable (POST)
      └──▶ [consultar_shortlist]          → Airtable (GET)
```

---

## 🧩 Nodos del workflow

| # | Nodo | Tipo | Función |
|---|------|------|---------|
| 1 | Chat con Recruiter | `chatTrigger` | Punto de entrada — recibe el mensaje vía chat UI de n8n |
| 2 | ¿Es una vacante? | `IF` | Si el mensaje tiene más de 15 caracteres pasa al agente; si no, devuelve saludo |
| 3 | Respuesta Saludo | `Set` | Mensaje de bienvenida para inputs cortos |
| 4 | AI Recruiter Agent | `agent` (toolsAgent) | Orquesta el flujo completo de sourcing |
| 5 | Anthropic Chat Model | `lmChatAnthropic` | Claude Sonnet 4.6 como LLM del agente |
| 6 | Memoria de Sesión | `memoryBufferWindow` | Contexto de las últimas 20 interacciones |
| 7 | buscar_candidatos | `httpRequestTool` | Búsqueda de perfiles LinkedIn en Google vía SerpAPI |
| 8 | guardar_candidato_shortlist | `httpRequestTool` | Guarda candidatos con score ≥ 60 en Airtable |
| 9 | consultar_shortlist | `httpRequestTool` | Lee el shortlist actual para evitar duplicados |

---

## 🔄 Flujo de ejecución

```
1. Recruiter envía la descripción de la vacante por chat

2. Filtro: si el mensaje tiene menos de 15 caracteres → saludo, fin

3. El agente analiza la vacante:
   - Hard skills requeridas
   - Nivel de seniority
   - Stack tecnológico
   - Idioma y ubicación

4. Consulta Airtable para obtener URLs ya guardadas (evitar duplicados)

5. Construye la search query optimizada, ejemplo:
   site:linkedin.com/in "Senior React Developer" "TypeScript" "Argentina"

6. Llama a SerpAPI → extrae de cada resultado:
   - Nombre completo, cargo actual, URL de LinkedIn, snippet del perfil

7. Calcula match score (0–100) por candidato

8. Para cada candidato con score ≥ 60 y URL no duplicada:
   → Llama a guardar_candidato_shortlist (Airtable POST)

9. Presenta resumen final rankeado por score descendente
```

---

## 📊 Criterios de scoring

| Criterio | Peso |
|----------|------|
| Hard skills match | 40% |
| Seniority adecuado | 20% |
| Industria relevante | 10% |
| Nivel de inglés estimado | 10% |
| Liderazgo / mentoring | 10% |
| Ubicación / timezone | 10% |

**Umbral de corte:** score ≥ 60 para guardar en shortlist.

**Seniority detectado automáticamente:**

| Nivel | Experiencia | Descripción |
|-------|-------------|-------------|
| Junior | 0–2 años | Tareas asistidas |
| Semi Senior | 2–4 años | Autonomía parcial |
| Senior | +5 años | Ownership técnico |
| Staff / Lead | Variable | Liderazgo técnico, decisiones de arquitectura |

---

## 🔌 Integraciones y credenciales

| Servicio | Uso | Configuración |
|----------|-----|---------------|
| **Anthropic** | Claude Sonnet 4.6 como LLM | Credencial `Anthropic account` en n8n |
| **SerpAPI** | Búsqueda de perfiles LinkedIn vía Google | API key en el nodo `buscar_candidatos` |
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

## ⚙️ Configuración

### Prerrequisitos

- n8n instalado (self-hosted o cloud)
- Cuenta de Anthropic con API key activa
- Cuenta de SerpAPI con API key
- Cuenta de Airtable con base y tabla configuradas

### Pasos

1. Importar el workflow JSON en n8n *(disponible bajo solicitud — ver sección de contacto)*
2. Configurar las credenciales:
   - `Anthropic account` → API key de Anthropic
   - `Airtable Personal Access Token account` → token de Airtable
3. En el nodo `buscar_candidatos`, reemplazar el valor de `api_key` con tu clave de SerpAPI
4. Verificar el ID de la base y tabla de Airtable en los nodos `guardar_candidato_shortlist` y `consultar_shortlist`
5. Activar el workflow y abrir el chat

---

## 💬 Cómo usar

1. Abrir el workflow en n8n y hacer clic en el botón **Chat**
2. Escribir la vacante en lenguaje natural:

```
Busco un Senior Backend Developer con Python y FastAPI,
experiencia en arquitectura de microservicios,
preferentemente en Argentina, inglés avanzado requerido.
```

3. El agente realiza la búsqueda, evaluación y guardado automáticamente
4. Al finalizar devuelve un resumen rankeado con tabla y próximos pasos sugeridos

---

## 📤 Output esperado

### Por candidato (score ≥ 60)

```json
{
  "candidate_name": "Nombre Apellido",
  "linkedin_url": "https://linkedin.com/in/username",
  "current_role": "Cargo actual del candidato",
  "location": "Ciudad, País",
  "seniority_detected": "Senior",
  "match_score": 82,
  "skills_match": ["Python", "FastAPI", "microservicios"],
  "missing_skills": ["Kubernetes"],
  "strengths": ["Stack alineado", "Experiencia en scale-ups"],
  "red_flags": [],
  "summary": "Perfil sólido con ownership técnico comprobado. Stack 100% alineado a la vacante."
}
```

### Resumen final

```
Total perfiles analizados: 10
Total perfiles saltados por duplicado: 2
Total guardados en Airtable: 5

| Nombre          | Rol                     | Ubicación        | Score |
|-----------------|-------------------------|------------------|-------|
| Nombre Apellido | Senior Backend Dev      | Buenos Aires, AR |  82   |
| Nombre Apellido | Backend Engineer        | Córdoba, AR      |  74   |
| ...             | ...                     | ...              | ...   |

Próximos pasos sugeridos: Iniciar outreach por LinkedIn a los 3 perfiles con score > 75.
```

---

## 🚧 Límites de operación

| Límite | Valor |
|--------|-------|
| Perfiles analizados por búsqueda | 10 |
| Llamadas a `buscar_candidatos` por conversación | 2 |
| Guardados en Airtable por conversación | 10 |
| Mensajes en memoria de sesión | 20 |

> ℹ️ Saludos y mensajes cortos (menos de 15 caracteres) no consumen herramientas ni créditos de API.

---

## 💼 ¿Quieres este agente?

El workflow **no está disponible públicamente** para proteger la propiedad intelectual.

Ofrezco **implementación personalizada** adaptada a tu stack, proceso de recruiting y herramientas existentes.

### ¿Qué incluye la implementación?

- ✅ Configuración completa del workflow en tu instancia de n8n
- ✅ Integración con tus credenciales (Anthropic, SerpAPI, Airtable)
- ✅ Adaptación del sistema prompt a tu proceso de recruiting
- ✅ Ajuste de criterios de scoring a tus vacantes habituales
- ✅ Soporte post-implementación

📩 **Contacto:** [tu-email@dominio.com] · [LinkedIn] · [WhatsApp]

---

## 👤 Autor

**Ferreira** — Automation & People Analytics Builder  
Especializado en workflows de IA aplicados a HR Tech y Talent Acquisition.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-conectar-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/tu-perfil)

---

*Este repositorio es de carácter demostrativo. El workflow completo está disponible únicamente bajo solicitud de implementación.*
