# Changelog - Plugin /ideacion

Registro de mejoras basadas en testing real.

---

## [1.1.0] - 2026-01-02

### Agregado

**Nuevas preguntas en FASE 3:**
- Pregunta 8: Borrado de registros (hard delete vs soft delete vs no delete)
- Pregunta 9: Detección de relaciones many-to-many (junction tables)
- Pregunta 16: Escenarios de testing (para seed data)

**Nuevas secciones en output 03-FEATURE-SPECS.md:**
- `### RLS Policy Guidelines` - Template de políticas RLS por operación
- `### Seed Data` - SQL de ejemplo + escenarios de prueba
- `### Missing Information` - Tabla de ambigüedades para blueprint

**Pre-flight Check actualizado:**
- Verificación de RLS policies documentadas
- Verificación de seed data para testing
- Verificación de missing information documentada

### Razón del cambio

Análisis comparativo entre output generado vs. documentos originales de YogaBuddhi reveló:
- Score inicial: 7.3/10
- Faltaba: seed data, RLS policies, detección de junction tables
- Ambigüedad: soft delete no preguntado explícitamente

---

## [1.0.0] - 2026-01-02

### Versión inicial

- 4 fases: CAPTURA, EXPLORACIÓN, DISEÑO, CONSOLIDACIÓN
- Fase 0 opcional: RESEARCH con `--with-research`
- Soporte para archivo de contexto con `@archivo.md`
- Output en 3 archivos: 01-IDEA.md, 02-DESIGN.md, 03-FEATURE-SPECS.md
- Pre-flight check para validar completitud
- Integración con workflow `/makerkit-blueprint` → `/ralph-loop`

---

## Roadmap de mejoras futuras

- [ ] Preguntas sobre índices de base de datos
- [ ] Detección automática de computed fields
- [ ] Integración con MCP para análisis de codebase existente
- [ ] Soporte para múltiples idiomas en templates
