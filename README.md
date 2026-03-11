# CDN Prompts Guía

Repositorio central de **prompts y guías** reutilizables para automatizar tareas de desarrollo. Los prompts aquí almacenados sirven como plantillas y referencias para agentes de IA, herramientas de automatización o flujos de trabajo asistidos por LLM.

**Aplica a cualquier lenguaje de programación y a cualquier tipo de proyecto.**

---

## Propósito

Este proyecto almacena prompts que pueden ser **útiles a la hora de automatizar ciertas tareas**. Cada prompt está diseñado para:

- Guiar a agentes o asistentes de IA en tareas concretas
- Establecer estándares y convenciones en flujos automatizados
- Servir como referencia reutilizable en distintos proyectos
- Reducir la configuración manual al copiar o referenciar estos prompts

---

## Estructura del repositorio

Los prompts se organizan por **lenguaje** y **dominio/tarea**:

```
cdn_prompts_guia/
├── python/
│   └── pruebas_unitarias/
│       ├── Pruebas_Unitarias_Guia.md    # Guía de estándares
│       └── Prompt Programador.md        # Instrucciones para crear pruebas
├── csharp/
│   └── pruebas_unitarias/
│       ├── Pruebas_Unitarias_Guia_CSharp.md  # Guía de estándares para C#
│       └── Prompt Programador.md             # Instrucciones para crear pruebas
├── [lenguaje]/
│   └── [dominio]/
│       ├── [Guia].md
│       └── [Prompt].md
└── README.md
```

---

## Contenido actual

### Python — Pruebas unitarias

| Archivo | Descripción |
|---------|-------------|
| `Pruebas_Unitarias_Guia.md` | Guía de mejores prácticas: AAA, mocks, fixtures, cobertura, estructura de tests |
| `Prompt Programador.md` | Instrucciones para un agente que crea pruebas unitarias módulo por módulo |

### C# — Pruebas unitarias

| Archivo | Descripción |
|---------|-------------|
| `Pruebas_Unitarias_Guia_CSharp.md` | Guía de mejores prácticas para C#: AAA, mocks, fixtures, cobertura, estructura de tests |
| `Prompt Programador.md` | Instrucciones para un agente que crea pruebas unitarias módulo por módulo |

---

## Cómo usar estos prompts

1. **Copiar y pegar** en tu conversación con un asistente de IA (Cursor, ChatGPT, Claude, etc.).
2. **Referenciar** desde archivos de contexto del proyecto (por ejemplo, `AGENTS.md`, `CLAUDE.md`).
3. **Integrar** en pipelines de CI/CD o scripts que invoquen modelos de lenguaje.
4. **Adaptar** a tu proyecto: los prompts son plantillas que puedes ajustar según tu stack y convenciones.

---

## Contribuir

Puedes añadir nuevos prompts siguiendo la estructura:

- **Por lenguaje**: `python/`, `javascript/`, `rust/`, etc.
- **Por dominio**: `pruebas_unitarias/`, `documentacion/`, `refactoring/`, etc.
- **Por tipo**: guías de estándares (`*_Guia.md`) y prompts operativos (`Prompt *.md`)

Los prompts deben ser:

- Reutilizables en distintos proyectos
- Claros y autocontenidos
- Fáciles de adaptar sin perder el propósito

---

## Licencia

Este repositorio es de uso libre. Adapta y reutiliza los prompts según tus necesidades.
