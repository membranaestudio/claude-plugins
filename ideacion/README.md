# Plugin: Ideacion

> Transforma ideas informales en specs de producto mediante entrevistas profundas.

**Filosofia:** "Not prompting. Discovering." - Las preguntas revelan requisitos que el usuario no sabia que tenia.

---

## Comando

### `/ideacion`

Inicia una sesion de ideacion estructurada que transforma una idea informal en documentos de producto listos para `/makerkit-blueprint`.

**Uso:**

```bash
# Solo entrevista
/ideacion "app para gestionar cobros de yoga"

# Con archivo de contexto (salta preguntas ya respondidas)
/ideacion @docs/idea-inicial.md

# Con research automatico
/ideacion "app para gestionar cobros" --with-research

# Combo: archivo + research
/ideacion @docs/idea-inicial.md --with-research
```

---

## Fases

| Fase | Tiempo | Output | Proposito |
|------|--------|--------|-----------|
| **0. RESEARCH** | 15-30 min | `research/YYYY-MM-DD-research.md` | Solo si `--with-research` |
| **1. CAPTURA** | 5-10 min | `01-IDEA.md` | Usuario, problema, piloto |
| **2. EXPLORACION** | 15-30 min | Actualiza `01-IDEA.md` | Dominio, competencia, modelo negocio |
| **3. DISENO** | 30-60 min | `02-DESIGN.md` | MVP, campos, estados, permisos, UI |
| **4. CONSOLIDACION** | 10-15 min | `03-FEATURE-SPECS.md` | Pre-flight check, specs para blueprint |

---

## Output

El plugin genera 3 archivos en `docs/producto/`:

```
docs/producto/
+-- research/                  # Solo si uso --with-research
|   +-- YYYY-MM-DD-research.md
+-- 01-IDEA.md                 # Contexto, problema, usuario, competencia
+-- 02-DESIGN.md               # Diseno detallado de features
+-- 03-FEATURE-SPECS.md        # Specs listos para /makerkit-blueprint
```

---

## Integracion con el Flujo

```
/ideacion "idea informal"
    |
docs/producto/03-FEATURE-SPECS.md
    |
/makerkit-blueprint "Implementar [feature]"
    |
docs/arquitectura/XX-feature.md
    |
/ralph-loop (implementacion autonoma)
    |
Codigo funcionando
```

---

## Principios de Diseno

1. **Una pregunta a la vez** - No bombardear al usuario
2. **Preferir opcion multiple** - Reduce carga cognitiva con `AskUserQuestion`
3. **Validacion incremental** - Presentar en secciones, validar despues
4. **Proponer defaults** - Cuando usuario dice "no se", proponer patron MakerKit
5. **YAGNI ruthlessly** - Eliminar features innecesarias

---

## Manejo de "No se"

Cuando el usuario no tiene respuesta, el plugin propone defaults marcados:

| Pregunta sin respuesta | Default sugerido | Marcado |
|------------------------|------------------|---------|
| Que estados tiene? | `active`, `archived` | `[DEFAULT]` |
| Que validaciones? | required para campos criticos | `[DEFAULT]` |
| Side effects? | ninguno | `[PENDIENTE]` |
| Permisos por rol? | CRUD basico para owner | `[PENDIENTE]` |
| Que componente UI? | Sheet para create/edit | `[DEFAULT]` |

Los campos marcados `[PENDIENTE]` indican a blueprint que DEBE preguntar.

---

## Pre-flight Check

Antes de generar `03-FEATURE-SPECS.md`, el plugin verifica:

### Obligatorios
- Entity name definido (singular, lowercase)
- Account type especificado (personal/team)
- Al menos 1 operacion definida
- Al menos 3 campos definidos
- account_id relation SI es team account

### Recomendados
- Estados definidos O marcados [DEFAULT]
- Permisos por rol definidos O marcados [PENDIENTE]
- UI preferences definidas
- Relations con FK explicitas

---

## Autor

**YogaBuddhi**

Creado como parte del sistema de desarrollo MakerKit + Claude Code.

---

*Plugin version 1.0.0*
