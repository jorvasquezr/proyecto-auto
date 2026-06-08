# Plan de Entrega — Verificador LTL con Autómatas de Büchi
**Proyecto Final · Teoría de Autómatas · Jorge Vasquez**

---

## Qué es el proyecto

Un **model checker** — una herramienta que verifica matemáticamente si un sistema reactivo cumple una propiedad de comportamiento infinito.

El usuario define un sistema (estados + transiciones) y una propiedad en LTL (Lógica Temporal Lineal). El verificador responde:
- ✅ **Verificado** — la propiedad se cumple en todos los comportamientos posibles
- ❌ **Violada** — aquí está el contraejemplo exacto donde falla

El motor interno usa **Autómatas de Büchi** — el modelo formal más avanzado del curso.

---

## Por qué es el proyecto correcto

| Criterio | Por qué cumple |
|---|---|
| Uso correcto del modelo formal (30%) | Büchi es el núcleo real del verificador, no decorativo |
| Originalidad (25%) | Model checking va muy por encima del reto de clase de 60 min |
| Calidad técnica (20%) | 4 algoritmos encadenados: LTL parser → Büchi → producto → detección de ciclos |
| Relación teoría ↔ sistema (15%) | Es la aplicación industrial de lo que el profe acaba de enseñar |
| Presentación (10%) | Demo en vivo con grafos animados y contraejemplos visibles |

---

## Arquitectura general

El proyecto tiene **dos capas** diseñadas desde el inicio para complementarse:

```
┌─────────────────────────────────────────────────────┐
│  CAPA 1: Jupyter Notebook (Python)                  │
│  · Definición formal                                │
│  · Implementación del motor                         │
│  · Visualización estática con networkx/matplotlib   │
│  · Ejemplos pre-construidos con texto explicativo   │
└─────────────────────────────────────────────────────┘
         ↓ misma lógica, re-implementada en JS
┌─────────────────────────────────────────────────────┐
│  CAPA 2: HTML/JS Visualizador (index.html)          │
│  · Animación paso a paso de cada etapa              │
│  · Todas las iteraciones del algoritmo observables  │
│  · Grafos interactivos con D3.js o SVG              │
│  · Contraejemplo resaltado al final                 │
└─────────────────────────────────────────────────────┘
```

**Decisión de arquitectura:** los algoritmos se diseñan una vez en Python, luego se replican en JS. El HTML es completamente autónomo (sin servidor).

---

## El modelo formal

### Sistema (Estructura de Kripke)
```
M = (S, S₀, R, L)
  S   = conjunto de estados
  S₀  ⊆ S = estados iniciales
  R   ⊆ S×S = relación de transición
  L   : S → 2^AP = función de etiquetado
```

### Autómata de Büchi (no determinista)
```
A = (Q, Σ, δ, Q₀, F)
  Q   = estados finitos
  Σ   = alfabeto (conjuntos de proposiciones)
  δ   = función de transición
  Q₀  = estados iniciales
  F   = estados de aceptación (visitados infinitamente)
```

### LTL — operadores soportados
| Símbolo | Texto | Significado |
|---|---|---|
| `□ P` | `G P` | P es verdadero **siempre** |
| `◇ P` | `F P` | P es verdadero **eventualmente** |
| `○ P` | `X P` | P es verdadero en el **siguiente** paso |
| `P U Q` | `P U Q` | P es verdadero **hasta que** Q lo es |
| `¬ P` | `! P` | negación |
| `P ∧ Q` | `P & Q` | conjunción |
| `P ∨ Q` | `P \| Q` | disyunción |

### Criterio de aceptación de Büchi
```
Una ejecución ρ = q₀q₁q₂... es aceptante ⟺ Inf(ρ) ∩ F ≠ ∅
donde Inf(ρ) = {q ∈ Q : q aparece infinitas veces en ρ}
```

---

## Ejemplos pre-construidos (para la presentación)

### Ejemplo 1 — Semáforo (introductorio)
```
Sistema:  rojo → verde → amarillo → rojo → ...
Propiedad: G F verde   ("siempre eventualmente llega al verde")
Resultado: ✅ Verificado
```

### Ejemplo 2 — Exclusión mutua (clásico)
```
Sistema:  2 procesos compitiendo por sección crítica
Propiedad A: G !(critica_A & critica_B)  → ✅ seguridad
Propiedad B: G F critica_A               → ❌ vivacidad violada (con contraejemplo)
```

### Ejemplo 3 — Protocolo con bug (más impactante)
```
Sistema:  protocolo de handshake simplificado
Propiedad: G (request → F grant)  ("toda petición es eventualmente atendida")
Resultado: ❌ Violado — contraejemplo: s0→s1→s0→s1→... (petición ignorada para siempre)
```

---

## Estrategia de desarrollo: POC → Completo

**Principio:** primero hacer que todo funcione de punta a punta con lógica simplificada, luego extender cada componente.

```
POC (todo conectado, simplificado)
  ↓
Extender cada componente
  ↓
Notebook formal completo
  ↓
HTML animado
```

Esto garantiza tener algo demostrable en todo momento. Si el tiempo se acaba, el POC ya es un proyecto funcional.

---

## Plan de implementación

---

### FASE 0 — POC completo end-to-end (~45 min)
**Meta:** pipeline completo corriendo con los 3 ejemplos, aunque la lógica interna sea simplificada.

#### Qué hace el POC
- `KripkeStructure`: clase básica, sin visualización aún
- `LTLFormula`: representación del AST con soporte para `G`, `F`, `!`, `&`, `|`, átomos
- `ltl_to_buchi()`: conversión simplificada — maneja `G p`, `F p`, `G F p`, `! p`, `p & q` con construcciones hardcodeadas para estos patrones comunes
- `product_automaton()`: construcción del producto completa
- `nested_dfs()`: detección de ciclos completa
- `verify()`: función principal que conecta todo y retorna ✅ o ❌ + contraejemplo

**Output esperado del POC:**
```python
result = verify(semaforo, "G F verde")
# ✅ Propiedad verificada

result = verify(exclusion_mutua, "G F critica_A")
# ❌ Propiedad violada
# Contraejemplo: idle → request → idle → request → ...
```

---

### FASE 1 — Extender cada componente (~1h)
**Meta:** hacer cada parte robusta y correcta, no solo los casos del POC.

#### Tarea 1.1 — KripkeStructure completa
- Validación de la estructura (estados referenciados existen, inicial válido)
- Función `visualizar()` con networkx: nodos coloreados, etiquetas, layout automático
- Los 3 sistemas ejemplo con documentación

#### Tarea 1.2 — Parser LTL completo
- Parser real con gramática: tokenizer + parser recursivo descendente
- Soporte completo: `G`, `F`, `X`, `U`, `!`, `&`, `|`, paréntesis, átomos
- Mostrar AST parseado como árbol en el notebook
- Manejo de errores con mensajes claros

#### Tarea 1.3 — LTL → Büchi completo
- Normalización a NNF (negación hacia adentro) para cualquier fórmula
- Construcción general por tableau (no solo patrones hardcodeados)
- Visualizar el Büchi con networkx: estados de aceptación con doble círculo

#### Tarea 1.4 — Producto completo
- Construcción on-the-fly (solo estados alcanzables)
- Visualización con estados de aceptación resaltados

#### Tarea 1.5 — Nested DFS completo
- Extracción correcta de prefijo + ciclo del contraejemplo
- Visualización del contraejemplo resaltado en rojo sobre el grafo del producto

---

### FASE 2 — Notebook formal (~30 min)
**Meta:** el notebook como documento de presentación.

- Introducción: qué es model checking, motivación real
- Definición formal de cada estructura (LaTeX con MathJax)
- Código organizado con texto explicativo entre celdas
- Los 3 ejemplos con análisis de resultados
- Sección de discusión: limitaciones, conexión con herramientas reales (SPIN, NuSMV)

---

### FASE 3 — Visualizador HTML (~1.5h)
**Meta:** `index.html` autónomo con animación completa.

#### Estructura
- Misma paleta oscura (variables CSS del proyecto anterior)
- Layout: panel izquierdo (selector + fórmula + info) / panel derecho (visualización)
- Pestañas: Definición Formal | Verificador animado

#### Animación — 8 etapas observables
```
Etapa 1: Sistema          → grafo aparece nodo a nodo
Etapa 2: Fórmula LTL      → árbol de sintaxis construido
Etapa 3: Negación de φ    → árbol transformado (¬φ)
Etapa 4: Büchi de ¬φ      → grafo del Büchi aparece
Etapa 5: Producto         → estados del producto aparecen uno a uno
Etapa 6: DFS externo      → camino marcado avanzando en el grafo
Etapa 7: DFS interno      → ciclo buscado, encontrado o no
Etapa 8: Resultado        → ✅ verificado / ❌ contraejemplo resaltado en rojo
```

Controles: `← Paso anterior` / `Siguiente →` / `▶ Auto` / slider de velocidad

#### Selector de ejemplos
- Dropdown: Semáforo / Exclusión mutua / Protocolo con bug
- Fórmula LTL renderizada con MathJax
- Al cambiar ejemplo: reinicia animación desde etapa 1

---

## Pruebas por etapa

Cada componente tiene pruebas propias antes de pasar al siguiente. Si una prueba falla, se corrige antes de continuar.

### Pruebas FASE 0 — POC
| Prueba | Qué verifica |
|---|---|
| `test_kripke_basico` | Sistema semáforo tiene 3 estados, transiciones correctas, etiquetas correctas |
| `test_ltl_parse_simple` | `"G F verde"` genera AST con nodo G → nodo F → átomo `verde` |
| `test_ltl_to_buchi_GF` | `G F p` genera Büchi con al menos un estado de aceptación y ciclo posible |
| `test_product_estados` | Producto semáforo × Büchi tiene estados alcanzables esperados |
| `test_dfs_ciclo` | Grafo con ciclo aceptante conocido → nested DFS lo encuentra |
| `test_verify_ok` | Semáforo + `"G F verde"` → ✅ verificado |
| `test_verify_falla` | Exclusión mutua + `"G F critica_A"` → ❌ con contraejemplo no vacío |

### Pruebas FASE 1 — Extensión
| Prueba | Qué verifica |
|---|---|
| `test_kripke_validacion` | Sistema con estado inexistente en transiciones lanza error |
| `test_parser_operadores` | `"G !(a & b)"` parsea correctamente con precedencia correcta |
| `test_parser_parentesis` | `"(F a) U (G b)"` parsea sin error |
| `test_nnf_negacion_G` | `! G p` se normaliza a `F ! p` |
| `test_nnf_negacion_F` | `! F p` se normaliza a `G ! p` |
| `test_buchi_general` | Büchi generado para `F (a & b)` tiene estructura correcta |
| `test_contraejemplo_formato` | Contraejemplo retorna prefijo + ciclo separados |

### Pruebas FASE 2 — Notebook
- Todas las celdas corren sin error de principio a fin (Restart & Run All)
- Los 3 grafos se generan sin error
- Los 3 resultados de verificación son los esperados

### Pruebas FASE 3 — HTML
- Los 3 ejemplos cargan y animan sin error en consola
- La animación avanza y retrocede correctamente
- El contraejemplo se resalta en rojo en el ejemplo que falla

---

## Checklist de entrega

### POC
- [ ] KripkeStructure básica
- [ ] LTLFormula AST básica
- [ ] ltl_to_buchi() simplificada
- [ ] product_automaton()
- [ ] nested_dfs()
- [ ] verify() conectando todo
- [ ] 3 ejemplos corriendo
- [ ] Pruebas POC pasando

### Fase 1 — Extensión
- [ ] KripkeStructure con visualización
- [ ] Parser LTL completo
- [ ] LTL → Büchi general (NNF + tableau)
- [ ] Producto on-the-fly
- [ ] Contraejemplo con visualización
- [ ] Pruebas Fase 1 pasando

### Fase 2 — Notebook
- [ ] Texto formal con LaTeX
- [ ] 3 ejemplos con análisis
- [ ] Discusión y conclusiones
- [ ] Restart & Run All sin errores

### Fase 3 — HTML
- [ ] Grafos SVG interactivos
- [ ] Animación 8 etapas
- [ ] Selector 3 ejemplos
- [ ] Controles de navegación
- [ ] Pruebas HTML pasando

---

## Orden de ejecución

```
FASE 0  →  POC end-to-end funcionando        (~45 min)
FASE 1  →  Extender KripkeStructure          (~15 min)
        →  Extender Parser LTL               (~20 min)
        →  Extender LTL→Büchi               (~25 min)
        →  Extender Producto + DFS           (~15 min)
─── CHECKPOINT: notebook completo y funcional ───
FASE 2  →  Texto formal del notebook         (~30 min)
FASE 3  →  HTML base + grafos SVG            (~45 min)
        →  Animación 8 etapas                (~30 min)
        →  Selector + controles              (~15 min)
─── ENTREGA ───
```
