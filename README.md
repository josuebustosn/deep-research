<div align="center">

# `deep-research`

**El botón de Research de claude.ai, replicado dentro de Claude Code.**

Dispara Exa Deep Researcher + NotebookLM DeepResearch en paralelo, cross-valida hallazgos entre ambos motores, y persiste todo en tu proyecto — con un solo comando.

[![Built for Claude Code](https://img.shields.io/badge/Built%20for-Claude%20Code-D97757?style=flat-square)](https://docs.claude.com/en/docs/claude-code/overview)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](./LICENSE)
[![Español](https://img.shields.io/badge/Idioma-Espa%C3%B1ol-red?style=flat-square)](#)

</div>

---

## ¿Qué es esto?

Un set de **skills + rules + config MCP** para Claude Code que replica la experiencia del botón "Research" de claude.ai (el que visita 50+ fuentes y sintetiza) — pero **dentro de tu terminal**, integrado a tu proyecto, y con cross-validation entre dos motores de research independientes.

```
Tú:  /deep-research investiga Bifrost vs LiteLLM para producción
     │
     ▼
  Claude dispara en paralelo ─────┬──▶ Exa research-pro ──┐
                                  │                        ├──▶ SYNTHESIS.md
                                  └──▶ NotebookLM deep ────┘    (cross-validated)
     │
     ▼
  Persiste 4 archivos en .planning/deep-research/<fecha>-<slug>/
  y te da un resumen de 5 líneas con el veredicto.
```

**Por qué dos motores en paralelo:** cada uno tiene carácter distinto — Exa es bueno con GitHub issues, threads de comunidad, y citas directas; NotebookLM es bueno con síntesis narrativa y papers. Correr ambos y sintetizar las convergencias atrapa alucinaciones y expande cobertura. El extra de ~$1.30 USD vale 100x lo que cuesta.

---

## ¿Qué incluye este repo?

```
deep-research/
├── skills/
│   ├── deep-research/SKILL.md     ← orquestador principal (dispara motores + sintetiza)
│   └── find-docs/SKILL.md          ← wrapper de Context7 para docs de libraries
├── rules/
│   └── context7.md                 ← rule global: siempre leer docs con ctx7
├── config/
│   ├── mcp-servers.json            ← template de los 5 MCPs (placeholders para keys)
│   └── claude-md-additions.md      ← snippet a pegar en tu CLAUDE.md global
└── README.md                       ← este archivo
```

---

## Tabla de contenidos

- [Quickstart (para tu Claude)](#quickstart-para-tu-claude)
- [Instalación manual paso a paso](#instalación-manual-paso-a-paso)
- [Arquitectura](#arquitectura)
- [Cómo usarlo](#cómo-usarlo)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Créditos](#créditos)

---

## Quickstart (para tu Claude)

> [!TIP]
> **¿Tu Claude Code está instalando esto por ti?** Si le dijiste "instala este repo", léele este README completo y ejecuta los pasos en orden. Pide al usuario sus API keys **solo cuando llegues al paso 4**. No improvises — los quirks de NotebookLM (paso 5) están documentados porque son trampas reales.

**Pre-requisitos que tu sistema necesita:**

| Requisito | Verificar con | Si falta |
|---|---|---|
| Claude Code instalado | `claude --version` | [docs.claude.com](https://docs.claude.com/en/docs/claude-code/overview) |
| Python 3.10+ | `python --version` | [python.org](https://python.org) |
| Node.js 18+ | `node --version` | [nodejs.org](https://nodejs.org) |
| Git | `git --version` | [git-scm.com](https://git-scm.com) |

**Cuentas que necesitarás (tener a mano antes de empezar):**

| Servicio | Para qué | Obligatorio | Gratis |
|---|---|---|---|
| [Exa](https://exa.ai) | Deep research motor #1 | Sí | Trial, después pay-per-use (~$1.30/query deep) |
| Cuenta Google | NotebookLM (motor #2) | Sí | 100% gratis |
| [Firecrawl](https://firecrawl.dev) | Scraping/crawling de URLs | Sí | Free tier generoso |
| [Context7](https://context7.com) | Docs de libraries | No (funciona sin cuenta) | Free tier sin auth |
| [Papersflow](https://papersflow.ai) | Papers académicos | No (opcional) | Guest mode funciona |

---

## Instalación manual paso a paso

### 1. Clonar el repo

```bash
git clone https://github.com/josuebustosn/deep-research.git
cd deep-research
```

### 2. Instalar el CLI de NotebookLM

El paquete `notebooklm-mcp-cli` provee **dos binarios**: `nlm` (para OAuth login) y `notebooklm-mcp` (el servidor MCP que Claude Code habla). Elige uno:

**Opción A — con `uv` (recomendado, más rápido):**
```bash
# Instalar uv si no lo tienes: https://docs.astral.sh/uv/
uv tool install notebooklm-mcp-cli
```

**Opción B — con `pip`:**
```bash
pip install notebooklm-mcp-cli
```

Verificar:
```bash
nlm --version              # debe imprimir: nlm version 0.5.x
notebooklm-mcp --help      # debe imprimir las flags del MCP
```

### 3. Autenticar NotebookLM (una sola vez)

```bash
nlm login
```

Esto abre Chrome → te pide OAuth con tu cuenta Google → guarda cookies en `~/.notebooklm-mcp-cli/profiles/default`.

> [!WARNING]
> La sesión **expira con el tiempo sin aviso**. Cuando eso pase, tu Claude detectará el error de auth y te dirá que re-corras `nlm login`. El flujo está documentado en `config/claude-md-additions.md` — es por eso que incluimos ese archivo.

### 4. Configurar los MCPs en `~/.claude.json`

Abre tu `~/.claude.json` (en Windows: `C:\Users\<usuario>\.claude.json`). Busca la clave `mcpServers` (si no existe, créala en la raíz del JSON). Agrega los 5 MCPs desde `config/mcp-servers.json` de este repo.

**Campos a reemplazar:**

- `<TU_FIRECRAWL_API_KEY>` → tu API key de Firecrawl (la consigues en [firecrawl.dev/app/api-keys](https://firecrawl.dev/app/api-keys))

Exa autentica por OAuth al primer request — no necesitas poner key en la URL. NotebookLM usa las cookies que guardó `nlm login`. Context7 y Papersflow funcionan sin auth.

**Ejemplo del resultado final en `~/.claude.json`:**

```json
{
  "mcpServers": {
    "exa": {
      "type": "http",
      "url": "https://mcp.exa.ai/mcp?tools=web_search_exa,web_search_advanced_exa,get_code_context_exa,crawling_exa,company_research_exa,linkedin_search_exa,deep_researcher_start,deep_researcher_check"
    },
    "notebooklm-mcp": {
      "type": "stdio",
      "command": "notebooklm-mcp",
      "args": [],
      "env": {}
    },
    "context7": { "type": "http", "url": "https://mcp.context7.com/mcp" },
    "papersflow": { "type": "http", "url": "https://doxa.papersflow.ai/mcp" },
    "firecrawl": {
      "type": "http",
      "url": "https://mcp.firecrawl.dev/<TU_FIRECRAWL_API_KEY>/v2/mcp"
    }
  }
}
```

### 5. Copiar skills y rules a tu `~/.claude/`

**Linux/macOS/Git Bash:**
```bash
mkdir -p ~/.claude/skills ~/.claude/rules
cp -r skills/deep-research ~/.claude/skills/
cp -r skills/find-docs ~/.claude/skills/
cp rules/context7.md ~/.claude/rules/
```

**PowerShell (Windows):**
```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.claude\skills", "$HOME\.claude\rules" | Out-Null
Copy-Item -Recurse skills\deep-research "$HOME\.claude\skills\"
Copy-Item -Recurse skills\find-docs "$HOME\.claude\skills\"
Copy-Item rules\context7.md "$HOME\.claude\rules\"
```

### 6. Pegar el snippet en tu CLAUDE.md global

Abre `~/.claude/CLAUDE.md` (créalo si no existe) y **pega al final** el contenido del bloque de `config/claude-md-additions.md` de este repo.

Este snippet le enseña a Claude **3 quirks del MCP de NotebookLM** que, sin ellos, te costarían horas debuggueando auth:
1. `server_info` **no** sirve como health check (retorna success con cookies expiradas).
2. Después de `nlm login`, hay que llamar `refresh_auth` manualmente para que el MCP recargue cookies de disco.
3. El flujo completo de re-auth end-to-end que Claude debe ejecutar sin intervención humana más que el OAuth en browser.

### 7. Verificar que todo corre

Reinicia Claude Code (cierra y abre de nuevo, para que cargue los MCPs). Después, en cualquier proyecto:

```
> /deep-research investiga cuál es el mejor MCP de NotebookLM para Claude Code en 2026
```

Si todo está bien, Claude debería:
- Detectar la skill `deep-research`
- Hacer el preflight de NotebookLM auth
- Disparar Exa y NotebookLM en paralelo
- Persistir 4 archivos (`exa-report.md`, `nlm-report.md`, `SYNTHESIS.md`, `sources.md`) en `.planning/deep-research/<fecha>-<slug>/`
- Darte un resumen con el veredicto

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLAUDE.md (global)                      │
│  Instrucciones sobre NotebookLM auth (3 quirks críticos)        │
└────────────────────────────────┬────────────────────────────────┘
                                 │ informa
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Skills + Rules                            │
│  ┌──────────────┐  ┌──────────┐  ┌─────────────────┐            │
│  │ deep-research│  │ find-docs│  │ rules/context7  │            │
│  │ (orquestador)│  │ (docs)   │  │ (rule global)   │            │
│  └──────┬───────┘  └─────┬────┘  └─────────────────┘            │
└─────────┼────────────────┼──────────────────────────────────────┘
          │ dispara        │ consulta
          ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        MCP Servers                              │
│  ┌─────┐  ┌──────────────┐  ┌──────────┐  ┌──────────┐  ┌────┐  │
│  │ exa │  │notebooklm-mcp│  │ context7 │  │papersflow│  │ fc │  │
│  │HTTP │  │    stdio     │  │  HTTP    │  │   HTTP   │  │HTTP│  │
│  └──┬──┘  └──────┬───────┘  └─────┬────┘  └─────┬────┘  └─┬──┘  │
└─────┼────────────┼────────────────┼─────────────┼─────────┼─────┘
      │            │                │             │         │
      ▼            ▼                ▼             ▼         ▼
   exa.ai   Google NotebookLM   context7.com  papersflow   firecrawl
            (via OAuth cookies)               (guest OK)   (API key)
```

**Flujo de una query de research:**

1. Usuario escribe `/deep-research <pregunta>` (o la skill se dispara proactivamente por keywords como "investiga a fondo").
2. Claude hace preflight: `mcp__notebooklm-mcp__notebook_list` para verificar auth. Si falla → relanza `nlm login` + `refresh_auth` + re-verifica.
3. Claude dispara **Exa + NotebookLM en paralelo** (ambos async).
4. Polling: NotebookLM bloquea server-side hasta 180s (eficiente), Exa retorna status instantáneo.
5. Al completarse NotebookLM, Claude **re-lee con `compact=false`** (importante — el default trunca 70% del contenido).
6. Claude sintetiza convergencias y divergencias entre ambos motores.
7. Persiste 4 archivos + scan del proyecto por referencias al topic (por si hay decisions que revisar).
8. Resumen tight al usuario con veredicto de 1 línea + paths + cost.

---

## Cómo usarlo

### Activación explícita

```
/deep-research <cualquier pregunta de investigación>
```

### Activación por keywords

La skill se dispara automáticamente con cualquiera de estos:

- `"deep research ..."`
- `"investigación profunda sobre ..."`
- `"investiga a fondo ..."`
- `"comprehensive research on ..."`
- `"lanza investigación de ..."`

### Flags útiles

| Flag | Qué hace |
|---|---|
| `--source=exa` | Solo Exa (si NotebookLM auth está jodido) |
| `--source=nlm` | Solo NotebookLM (si quieres costo $0) |
| `--source=papers` | Agrega papersflow (académico) |
| `--source=all` | Exa + NLM + papers + context7 |
| `--mode=fast` | NLM fast mode (30s, 10 fuentes) |
| `--model=exa-research` | Exa balanced en vez de pro (más barato) |
| `--no-persist` | No escribe archivos, solo resumen inline |
| `--slug=<nombre>` | Override del slug auto-generado |

### Cuándo NO usarlo

- Preguntas factuales rápidas → usa `WebSearch` builtin
- Dudas de API de libraries → usa `/find-docs` (context7)
- Preguntas sobre tu propio proyecto → `Read` directo
- Debug de errores específicos → wrong tool

---

## Troubleshooting

<details>
<summary><b>NotebookLM devuelve "Authentication expired" aunque acabo de hacer login</b></summary>

El MCP cachea credenciales en memoria al arrancar y **no** las recarga de disco. Después de `nlm login`, tienes que llamar:

```
mcp__notebooklm-mcp__refresh_auth
```

Sin eso, sigue usando las cookies viejas. Este es el quirk más común.

</details>

<details>
<summary><b>El reporte de NotebookLM viene truncado / muy corto</b></summary>

`research_status` default es `compact=true`. Una vez que el status es `completed`, tienes que relamar con `compact=false` para pull del reporte completo. Skipping this pierde 70%+ del contenido.

El skill `deep-research/SKILL.md` de este repo ya maneja esto — si lo ves truncado es porque tu versión del skill está desactualizada o alguien lo modificó.

</details>

<details>
<summary><b>Exa retorna "quota exceeded" o "trial ended"</b></summary>

Tu trial de Exa se acabó. Opciones:
1. Agregar payment en [exa.ai/settings/billing](https://exa.ai/settings/billing) — pay-per-use, research-pro son ~$1.30/query
2. Usar `--source=nlm` para correr solo NotebookLM (gratis)

</details>

<details>
<summary><b>notebook_list retorna success pero research_start falla</b></summary>

Muy raro. Verifica:
- Que tu cuenta Google tiene acceso a NotebookLM (normalmente sí, pero en algunos dominios corporativos está bloqueado)
- Que no tienes 2FA forzado por una extensión que bloquea el OAuth headless

Fallback: abre [notebooklm.google.com](https://notebooklm.google.com) en tu browser manualmente y verifica que funciona. Si ahí funciona pero el MCP no, regenera cookies con `nlm login`.

</details>

<details>
<summary><b>Claude no detecta la skill `deep-research`</b></summary>

- Reinicia Claude Code (cerrar y abrir — así re-escanea `~/.claude/skills/`).
- Verifica que el archivo está en `~/.claude/skills/deep-research/SKILL.md` (no en un subdir raro).
- Verifica que el frontmatter YAML del SKILL.md no está roto (líneas `---` al inicio y fin del bloque).

</details>

<details>
<summary><b>Los MCPs no aparecen en la lista de tools disponibles</b></summary>

- Verifica tu `~/.claude.json` es JSON válido (usa `jq . ~/.claude.json` o similar).
- Reinicia Claude Code.
- Corre `claude mcp list` (si el comando existe en tu versión) para ver el estado de cada MCP.
- Para los HTTP: prueba `curl <url>` — si retorna HTML de landing, el endpoint está vivo.
- Para `notebooklm-mcp` stdio: corre `notebooklm-mcp --help` directo en terminal. Si falla, reinstala con `uv tool install --reinstall notebooklm-mcp-cli`.

</details>

---

## FAQ

**¿Cuánto cuesta usar esto?**

- NotebookLM: **$0** (cuota generosa de Google, gratis).
- Context7: **$0** en tier sin auth.
- Papersflow: **$0** en guest mode.
- Firecrawl: **$0** en free tier (suficiente para uso personal).
- Exa: **trial gratis**, después ~$1.30 por query deep-research-pro. Si usas `--source=nlm` queda todo gratis, pero pierdes el cross-validation.

**¿Por qué no usar solo el `WebSearch` nativo de Claude Code?**

Porque el `WebSearch` es un hit y listo — no explora múltiples fuentes, no sintetiza, no cross-valida. Para preguntas chicas (2-3 sources) sí sirve. Para decisiones de arquitectura donde necesitas cruzar evidencia de 20-40 fuentes distintas, no alcanza.

**¿Por qué dos motores en paralelo en vez de solo uno?**

Cross-validation atrapa alucinaciones. Si Exa y NotebookLM dicen lo mismo con fuentes independientes, tu confianza sube mucho. Si divergen, eso es una señal importante de que el topic está contestado. Un solo motor te da la falsa sensación de certeza.

**¿Funciona en Windows, macOS, Linux?**

Sí, los tres. `notebooklm-mcp-cli` es Python, funciona cross-platform. Los MCPs HTTP obviamente son agnósticos al OS. El paso manual de copiar skills a `~/.claude/` tiene comandos para bash y PowerShell en [Instalación](#instalación-manual-paso-a-paso).

**¿Puedo customizar las skills?**

Sí. Después de copiarlas a `~/.claude/skills/`, son tuyas. Editarlas cambia cómo Claude se comporta. Si mejoras algo, mándame un PR.

**¿Esto reemplaza las skills de GSD (`get-shit-done`)?**

No. Es complementario. GSD es un framework de planeación de proyectos. `deep-research` es una skill de investigación que *puede* ser usada dentro de un flujo GSD (p.ej. antes de decidir stack tech en `/gsd-discuss-phase`), pero también funciona standalone en cualquier proyecto.

---

## Créditos

- **Inspirado en** el botón "Research" de [claude.ai](https://claude.ai) — este setup lo replica dentro de Claude Code.
- **NotebookLM MCP:** [jacob-bd/notebooklm-mcp-cli](https://github.com/jacob-bd/notebooklm-mcp-cli) — los binarios `nlm` y `notebooklm-mcp`.
- **Exa Deep Researcher:** [exa.ai](https://exa.ai)
- **Context7:** [context7.com](https://context7.com)
- **Firecrawl:** [firecrawl.dev](https://firecrawl.dev)
- **Papersflow:** [papersflow.ai](https://papersflow.ai)

## License

[MIT](./LICENSE) — úsalo, modifícalo, compártelo.

---

<div align="center">

**¿Problemas? ¿Sugerencias?** Abre un [issue](https://github.com/josuebustosn/deep-research/issues) o manda un PR.

</div>
