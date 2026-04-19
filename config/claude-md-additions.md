# Adiciones a `~/.claude/CLAUDE.md`

Copia y pega el bloque de abajo al final de tu `CLAUDE.md` global (el que está en `~/.claude/CLAUDE.md`, no el de proyecto). Esto le enseña a Claude **3 quirks críticos** del MCP de NotebookLM que, si no se conocen, cuestan horas de debugging.

---

````markdown
## NotebookLM MCP: verificar auth ANTES de cualquier fase autónoma

La sesión autenticada del MCP `notebooklm-mcp` no es perdurable — expira sin aviso. Antes de cualquier flujo autónomo que use NotebookLM MCP para investigación (p.ej. `mcp__notebooklm-mcp__research_start`, `mcp__notebooklm-mcp__notebook_query`), **confirmar primero que el auth sigue válido**.

**IMPORTANTE — health check correcto:** `mcp__notebooklm-mcp__server_info` es una llamada LOCAL que retorna success incluso con cookies expiradas. **No sirve como health check de auth.** El check real es:

```
mcp__notebooklm-mcp__notebook_list
```

Esta golpea Google y devuelve "Authentication expired" cuando las cookies ya no son válidas.

**Si devuelve error de auth (flujo end-to-end que Claude ejecuta, NO el usuario):**

1. **Claude lanza `nlm login` via Bash** con `run_in_background=true` (abre Chrome y espera OAuth). El único paso humano es completar el flow en el navegador que se abrió — no le pidas al usuario que corra el comando.

2. **Monitor** el output hasta ver `Successfully authenticated` o `Credentials saved to`. Ubicación de credenciales en Windows: `C:\Users\<user>\.notebooklm-mcp-cli\profiles\default`.

3. **CRÍTICO — recargar tokens en el MCP corriendo:** el MCP server cachea credenciales en memoria al arrancar y **NO** las recarga de disco automáticamente. Después de `nlm login` exitoso, llamar:

   ```
   mcp__notebooklm-mcp__refresh_auth
   ```

   Sin este paso, `notebook_list` sigue devolviendo "Authentication expired" aunque las cookies ya estén en disco — confunde el debug y se pierde tiempo.

4. **Verificar** con un nuevo `notebook_list`. Debe retornar `status: success`.

5. **Fallback** (solo si `nlm login` CLI falla de verdad): `mcp__notebooklm-mcp__save_auth_tokens` con cookies extraídas manualmente.

6. **NO proceder** con la fase autónoma hasta que un `notebook_list` vuelva con status success.

**Aplicable a notebooklm-mcp-cli v0.5.25.** El comportamiento de `server_info` como no-auth-check y la necesidad de `refresh_auth` manual después de `nlm login` son quirks del MCP que pueden o no seguir vigentes en releases futuros. Si el upstream cambia el comportamiento, confirmar contra los release notes del repo antes de aplicar ciegamente.
````

---

## Opcional: preferencia de MCPs sobre WebSearch nativo

Si quieres que Claude priorice tus MCPs de research sobre el `WebSearch` builtin en tus proyectos, añade esto a tu CLAUDE.md **de proyecto** (no al global — es por-proyecto según el tech stack):

```markdown
## Investigación

Preferir MCPs sobre WebSearch puro: `mcp__exa__*`, `mcp__firecrawl__*`, `mcp__notebooklm-mcp__*`, `mcp__context7__*`. Para queries grandes existe la skill `deep-research`.
```
