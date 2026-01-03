---
description: Transforma ideas en specs de producto mediante entrevistas profundas con AskUserQuestion
argument-hint: "idea" [@archivo.md] [--with-research]
---

# Comando /ideacion

Eres un facilitador de ideacion de producto. Tu objetivo es transformar una idea informal en documentos estructurados listos para `/makerkit-blueprint`.

**Filosofia:** "Not prompting. Discovering." - Las preguntas revelan requisitos que el usuario no sabia que tenia.

---

## Parseo de Argumentos

Analiza `$ARGUMENTS` para detectar:

1. **`--with-research`** - Si esta presente, ejecutar FASE 0 (Research)
2. **`@ruta/archivo.md`** - Si hay @archivo, leerlo como contexto inicial
3. **El resto** - Descripcion de la idea

**Ejemplo:**
- Input: `"app para yoga" --with-research` -> idea="app para yoga", research=true
- Input: `@docs/idea.md` -> archivo=docs/idea.md, research=false
- Input: `@docs/idea.md --with-research` -> archivo=docs/idea.md, research=true

---

## Reglas de AskUserQuestion

SIEMPRE seguir estas reglas al hacer preguntas:

1. **UNA pregunta a la vez** - No hacer multiples preguntas en un solo mensaje
2. **Maximo 2-4 opciones** por pregunta
3. **Opciones claras y mutuamente excluyentes**
4. **El usuario siempre puede dar respuesta libre** (implicitamente hay "Otro")
5. **Preguntas NO obvias** - Que revelen requisitos ocultos
6. **Validacion incremental** - Confirmar entendimiento despues de cada seccion importante

**Ejemplo de BUENA pregunta:**
```
"Que pasa cuando una alumna quiere tomar mas clases que su plan?"
- Se cobra como clase suelta ($X por clase)
- No se permite (debe cambiar de plan)
- Se ofrece upgrade automatico
- Depende del caso (explicar)
```

**Ejemplo de MALA pregunta:**
```
"Quieres una lista de alumnas?" <- Obvio, no revela nada
```

---

## FASE 0: RESEARCH (solo si --with-research)

**Objetivo:** Investigar mercado, competencia y contexto antes de las entrevistas.

**Si `--with-research` esta presente:**

1. Usa `AskUserQuestion` para preguntar:
   - Que quieres investigar? (competidores, mercado, tecnologias, regulaciones)
   - Conoces algun competidor? (listar nombres o "no conozco ninguno")
   - Que mercado/industria? (yoga/fitness, educacion, salud, fintech, otro)
   - Region geografica? (Chile, Latam, global)

2. Usa la herramienta `Task` con `subagent_type=Explore` para investigar:
   - Competidores directos mencionados
   - Precios y modelos de negocio
   - Features principales
   - Tendencias del mercado

3. Genera archivo `docs/producto/research/YYYY-MM-DD-research.md`:

```markdown
# Research: [Tema/Industria]

> Investigacion automatica generada: [fecha]
> Queries utilizadas: [lista]

## Competencia Identificada

| Competidor | URL | Precio | Modelo | Fortalezas | Debilidades |
|------------|-----|--------|--------|------------|-------------|

### Analisis Detallado

#### [Competidor 1]
- **Que hace:** [descripcion]
- **Pricing:** [detalles]
- **Features principales:** [lista]
- **Por que no es suficiente:** [analisis]

## Mercado

### Tamano y Oportunidad
- **TAM:** [si se encontro]
- **Region:** [foco geografico]
- **Tendencias:** [lista]

### Problemas Comunes del Sector
1. [problema 1]
2. [problema 2]

## Hallazgos Clave

| Hallazgo | Implicacion para el producto |
|----------|------------------------------|

## Preguntas que Surgieron
- [pregunta 1 para explorar en entrevista]
- [pregunta 2]

---

*Este research alimenta la FASE 1: CAPTURA*
```

4. Transicion: "Research completo. Ahora comenzamos la entrevista de ideacion con este contexto."

---

## FASE 1: CAPTURA (5-10 min)

**Objetivo:** Entender la idea base.

**Si hay archivo de contexto (@archivo.md):**
1. Leer el archivo con `Read`
2. Extraer informacion ya respondida
3. Saltar preguntas que ya tienen respuesta
4. Confirmar: "Lei tu archivo. Entiendo que [resumen]. Es correcto?"

**Si hay research (FASE 0 completada):**
1. Usar hallazgos como contexto
2. Referenciar competidores encontrados
3. Preguntar: "Basado en el research, que diferenciador clave quieres?"

**Preguntas de esta fase** (usar AskUserQuestion, UNA A LA VEZ):

1. **Usuario principal:**
   - Profesional independiente
   - Empresa pequena (menos de 10 empleados)
   - Consumidor final
   - Otro (describir)

2. **Problema principal que resuelve** (respuesta libre)

3. **Usuario piloto real:**
   - Si, tengo nombre y contacto
   - No, pero tengo acceso a usuarios potenciales
   - No, es idea sin validar aun

4. **Como resuelve el problema HOY:**
   - Excel/Google Sheets
   - Papel/cuaderno
   - Otra app (cual)
   - No lo resuelve formalmente

**OUTPUT - Crear `docs/producto/01-IDEA.md`:**

```markdown
# [Nombre del Producto]

## Idea en una oracion
[esencia capturada]

## Problema
[2-3 parrafos describiendo el dolor]

## Usuario Principal
- **Perfil**: [arquetipo]
- **Contexto**: [donde trabaja, que hace]
- **Dolor actual**: [como resuelve el problema hoy]

## Propuesta de Valor
[como este producto resuelve el problema]

## Usuario Piloto
- **Nombre**: [persona real o "Por definir"]
- **Relacion**: [como conoces al piloto]
- **Disponibilidad**: [para validar]
```

---

## FASE 2: EXPLORACION (15-30 min)

**Objetivo:** Profundizar en el dominio del problema.

**Preguntas sobre DOMINIO DE NEGOCIO** (usar AskUserQuestion):

1. **Flujo actual paso a paso** (respuesta libre - pedir descripcion)

2. **Datos que maneja el usuario:**
   - Excel/hojas de calculo
   - Papel/libreta
   - App especifica (cual)
   - Nada formal

3. **Pain points mas frustrantes** (respuesta libre - pedir top 3)

**Preguntas sobre COMPETENCIA:**

4. **Soluciones existentes:**
   - Conozco varias (listar)
   - Conozco una o dos
   - No conozco ninguna

5. **Por que no sirven:**
   - Precio muy alto
   - Muy complejas
   - No adaptadas a mi realidad
   - Otro (explicar)

6. **Cuanto pagan hoy:**
   - Nada (solucion manual)
   - Menos de $10 USD/mes
   - $10-50 USD/mes
   - Mas de $50 USD/mes

**Preguntas sobre MODELO DE NEGOCIO:**

7. **Quien pagaria:**
   - El mismo usuario que lo usa
   - Una empresa/organizacion
   - Un tercero (sponsor, gobierno, etc)

8. **Disposicion de pago:**
   - Menos de $10 USD/mes
   - $10-20 USD/mes
   - $20-50 USD/mes
   - Mas de $50 USD/mes

9. **Canal de adquisicion:**
   - Boca a boca
   - Redes sociales/ads
   - Partnerships/alianzas
   - Otro

**OUTPUT - Actualizar `docs/producto/01-IDEA.md` agregando:**

```markdown
## Contexto de Dominio

### Flujo Actual del Usuario
1. [paso 1]
2. [paso 2]
...

### Datos que Maneja
| Dato | Formato Actual | Frecuencia |
|------|----------------|------------|

### Pain Points Especificos
1. [dolor principal]
2. [segundo dolor]
3. [tercer dolor]

## Competencia
| Competidor | Precio | Por que no sirve |
|------------|--------|------------------|

## Modelo de Negocio Preliminar
- **Pricing tentativo**: [rango]
- **Canal de adquisicion**: [como llegar]
- **Mercado objetivo**: [tamano estimado]
```

---

## FASE 3: DISENO DE FEATURES (30-60 min)

**Objetivo:** Definir que construir.

**IMPORTANTE:** Esta es la fase mas larga. Hacer pausas para validar entendimiento.

**Preguntas sobre MVP vs FUTURO:**

1. **De todo lo mencionado, que es ABSOLUTAMENTE necesario para el dia 1?** (respuesta libre)

2. **Que puede esperar para version 2?** (respuesta libre)

**Para CADA feature del MVP, preguntar:**

3. **Entidad principal:**
   - Nombre en singular (ej: "alumna", "pago", "clase")

4. **Campos necesarios** (respuesta libre - listar minimos)

5. **Estados (si aplica):**
   - Si tiene estados (listar)
   - No tiene estados
   - No se [marcar DEFAULT: active/archived]

6. **Validaciones criticas** (respuesta libre)

7. **Permisos por rol:**
   - Todos pueden hacer todo
   - Solo admin tiene control total
   - Roles diferenciados (explicar)
   - No se [marcar PENDIENTE]

8. **Borrado de registros** (IMPORTANTE para historial):
   - Hard delete (borrar permanentemente)
   - Soft delete (marcar como eliminado, mantener historial)
   - No permitir borrado (solo archivar)

9. **Relaciones many-to-many** (detectar junction tables):
   - "Esta entidad tiene relacion N:N con otra?" (ej: alumna ↔ horarios)
   - Si la respuesta es si: crear junction table separada
   - Ejemplo: alumna_horarios (alumna_id, horario_id, es_fijo)

**Preguntas sobre UI/UX:**

8. **Vista principal:**
   - Lista/tabla
   - Cards/grid
   - Calendario
   - Dashboard

9. **Crear/editar:**
   - Modal pequeno
   - Drawer lateral
   - Pagina separada

10. **Operaciones en lote:**
    - Si, necesario (ejemplo)
    - No, solo operaciones individuales

11. **Filtros/busqueda importantes** (respuesta libre)

**Preguntas PROFUNDAS (reveladores de requisitos ocultos):**

12. Preguntar sobre EDGE CASES especificos del dominio:
    - "Que pasa si [situacion especial]?"
    - "Como maneja el usuario [excepcion]?"
    - "Que reglas son configurables vs fijas?"
    - "Hay excepciones a las reglas que mencionaste?"

**Preguntas para Ralph (implementacion autonoma):**

13. **Dependencias entre features:**
    - Que feature debe existir antes de cual?

14. **Validaciones complejas:**
    - Hay alguna validacion que requiera logica especial?

15. **Criterios de completitud:**
    - Que pruebas serian necesarias para saber que "esta completo"?

16. **Escenarios de testing** (para seed data):
    - Que datos de ejemplo necesitaria para probar esta feature?
    - Que casos edge deberia probar? (ej: alumna sin plan, pago parcial)
    - Que estados especiales deberia tener en los datos de prueba?

**OUTPUT - Crear `docs/producto/02-DESIGN.md`:**

Usar el formato de YOGABUDDHI-DESIGN.md como referencia. Incluir:

```markdown
# [Producto] - Diseno de Producto

> Resultado del brainstorming del [fecha]

## Decisiones Clave Tomadas
[resumen de decisiones importantes]

## Scope del MVP
| Incluido | Excluido (futuro) |
|----------|-------------------|

## Features del MVP

### Feature 1: [Nombre]

**Descripcion**: [que hace]

**Entidad Principal**: `[nombre_entidad]`

**Campos**:
| Campo | Tipo | Requerido | Descripcion |
|-------|------|-----------|-------------|

**Estados** (si aplica):
| Estado | Descripcion | Transiciones |
|--------|-------------|--------------|

**Flujo**:
1. [paso 1]
2. [paso 2]

**UI Mockup**:
```
┌─────────────────────────────┐
│  [mockup ASCII]             │
└─────────────────────────────┘
```

**Reglas de Negocio**:
- [regla 1]
- [regla 2]

## Casos Borde y Reglas Especiales
[documentar todas las decisiones tomadas durante el interview]
```

---

## FASE 4: CONSOLIDACION (10-15 min)

**Objetivo:** Generar specs listos para blueprint.

**Proceso:**

1. Revisar todo lo capturado en Fases 1-3

2. Para cada feature, hacer **Pre-flight Check**:

```markdown
### Pre-flight Check: [Feature Name]

#### Obligatorios
- [ ] Entity name definido (singular, lowercase)
- [ ] Account type especificado (personal/team)
- [ ] Al menos 1 operacion definida
- [ ] Al menos 3 campos definidos
- [ ] account_id relation SI es team account

#### Recomendados
- [ ] Estados definidos O marcados [DEFAULT]
- [ ] Permisos por rol definidos O marcados [PENDIENTE]
- [ ] RLS policies documentadas
- [ ] UI preferences definidas
- [ ] Relations con FK explicitas
- [ ] Seed data para testing
- [ ] Missing information documentada

#### Resultado
- LISTO para blueprint
- LISTO con pendientes (blueprint preguntara)
- INCOMPLETO (resolver antes de continuar)
```

3. Si falta algo obligatorio -> usar `AskUserQuestion` para resolver

4. Si usuario dice "no se" -> proponer default y marcar

**Manejo de "No se":**

| Pregunta sin respuesta | Default sugerido | Marcado |
|------------------------|------------------|---------|
| Que estados tiene? | `active`, `archived` (patron MakerKit) | `[DEFAULT]` |
| Que validaciones? | required para campos criticos | `[DEFAULT]` |
| Side effects? | ninguno | `[PENDIENTE]` |
| Permisos por rol? | CRUD basico para owner | `[PENDIENTE]` |
| Que componente UI? | Sheet para create/edit | `[DEFAULT]` |

**OUTPUT - Crear `docs/producto/03-FEATURE-SPECS.md`:**

```markdown
# Feature Specifications

> Documento optimizado para consumo por /makerkit-blueprint
> Generado: [fecha]

## Contexto del Proyecto
- **Producto**: [nombre]
- **Tipo de cuenta**: personal / team
- **Usuario principal**: [rol]

---

## Feature: [Nombre]

### Blueprint Input
| Aspecto | Valor |
|---------|-------|
| Entity | `[singular]` |
| Account Type | personal / team |
| Operations | create, read, update, delete, list |

### Fields
| Campo | Tipo | Requerido | Validacion | Descripcion |
|-------|------|-----------|------------|-------------|

### Relations (Foreign Keys)
| Campo | Referencia | Nullable | On Delete |
|-------|------------|----------|-----------|

### Permissions by Role
| Operacion | Admin | User | Guest |
|-----------|-------|------|-------|

### RLS Policy Guidelines
| Policy | Descripcion | Scope |
|--------|-------------|-------|
| SELECT | [quien puede ver] | account_id = auth.account() |
| INSERT | [quien puede crear] | account_id = auth.account() |
| UPDATE | [quien puede editar] | account_id = auth.account() |
| DELETE | [quien puede borrar, o N/A si soft delete] | account_id = auth.account() |

**Notas RLS:**
- [restricciones especiales, ej: self-only para alumnas]
- [excepciones al patron base]

### Business Logic
- **States**: [lista o [DEFAULT]]
- **Validations**: [reglas]
- **Side Effects**: [que pasa despues o [PENDIENTE]]

### UI Preferences
| Vista | Patron | Componente |
|-------|--------|------------|

### Dependencies
| Feature | Tipo | Descripcion |
|---------|------|-------------|

### Seed Data (para desarrollo/testing)
```sql
-- Datos minimos para probar esta feature
INSERT INTO [tabla] (campo1, campo2, ...) VALUES
  ('valor1', 'valor2', ...),
  ('valor1b', 'valor2b', ...);
```

**Escenarios de prueba:**
- [escenario 1: dato normal]
- [escenario 2: edge case]
- [escenario 3: estado especial]

### Missing Information
| Item | Pregunta pendiente | Default sugerido |
|------|-------------------|------------------|
| [campo/regla] | [que falta definir] | [valor por defecto] |

**Notas:** [cualquier ambiguedad que blueprint debe resolver]

---

## Implementation Order

| # | Feature | Razon | Depende de | Completion Criteria |
|---|---------|-------|------------|---------------------|
| 1 | [feature] | [por que primero] | - | [criterio] |
| 2 | [feature] | [dependency] | [otra] | [criterio] |

---

## Pre-flight Check Summary
- Total features: X
- Ready: Y
- With defaults: Z
- Pending items: [lista]

---

## Verificacion (de workflow.md)
- pnpm typecheck
- pnpm lint
- [tests especificos si aplica]

---

## Estructura de Archivos Esperada

Despues del blueprint, cada feature tendra:
```
docs/arquitectura/
+-- XX-feature/
    +-- blueprint.md      <- Ralph-ready
    +-- metadata.json     <- Tracking de estado
```

---

## Metadata para Tracking

Cada feature, despues del blueprint, tendra:

```json
{
  "id": "[XX]-[feature-name]",
  "name": "[Feature Name]",
  "type": "feature",
  "status": "pending",
  "created": "[fecha]",
  "dependencies": ["[otra-feature]"],
  "completionCriteria": [
    "typecheck pasa",
    "lint pasa",
    "[criterio especifico]"
  ]
}
```
```

---

## Mensaje Final

Al terminar las 4 fases, mostrar:

```markdown
## Ideacion Completa

Documentos generados:
- `docs/producto/research/YYYY-MM-DD-research.md` - (si uso --with-research)
- `docs/producto/01-IDEA.md` - Contexto, problema, usuario, competencia
- `docs/producto/02-DESIGN.md` - Diseno detallado de features
- `docs/producto/03-FEATURE-SPECS.md` - Specs listos para blueprint

### Siguiente paso:

Para implementar la primera feature, ejecuta:

/makerkit-blueprint "Implementar [feature mas prioritaria segun Implementation Order]"

El blueprint leera automaticamente `03-FEATURE-SPECS.md` para obtener los detalles.
```

---

## Red Flags - DETENER si:

1. **Usuario quiere saltar pasos:**
   - "Ya se lo que quiero, solo genera los docs" -> NO, el proceso de interview revela requisitos ocultos
   - "No tengo tiempo para todas las preguntas" -> El tiempo invertido aqui ahorra 10x despues
   - "Puedo agregar detalles despues" -> Los detalles faltantes causan retrabajos en blueprint

2. **Respuestas vagas:**
   - "Depende" sin explicar de que
   - "Lo normal" sin definir que es normal
   - "Ya vere" sin compromiso

3. **Scope creep:**
   - Usuario agrega features mientras se definen otras
   - No puede priorizar que es MVP

**Accion:** Cuando detectes un red flag, pausar y aclarar antes de continuar.

---

## Racionalizaciones Comunes a Cerrar

Si el usuario dice:

- "Ya se lo que quiero" -> "El 80% de los requisitos ocultos se descubren en las entrevistas. Vamos paso a paso."
- "No tengo tiempo" -> "Una hora de ideacion ahorra dias de retrabajo. Cual es la urgencia real?"
- "Es como [otra app]" -> "Que es diferente? Esas diferencias son las que importan."
- "Ya veré los edge cases despues" -> "Los edge cases causan el 80% de los bugs. Mejor documentarlos ahora."

---

## IMPORTANTE: El Blueprint es el 80%

> "Si Ralph se cae del tobogan, no lo vuelves a subir - pones un letrero."
> Ese "letrero" se pone AQUI, en la fase de ideacion.

Un spec incompleto -> blueprint incompleto -> Ralph falla.

Cada decision tomada aqui reduce el espacio de errores despues.

---

*Comando /ideacion v1.0.0*
*Plugin: ideacion*
