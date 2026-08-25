# Clase 06 — Herencia Simple en Python: Guía Senior / Tech Lead

> **Objetivo:** entender la herencia simple no como sintaxis, sino como decisión de modelado — qué representa, cómo funciona `__init__`/`super()`/MRO internamente, y cuándo el diseño deja de escalar.
>
> Todo el código de esta guía sigue **PEP 8**: 2 líneas en blanco entre definiciones de nivel módulo, 1 línea en blanco entre métodos, líneas ≤ 79 caracteres, `snake_case` para funciones/atributos y `PascalCase` para clases.

---

## 1. El problema real (no solo duplicación)

Dos clases relacionadas comparten estado y comportamiento:

```python
# ❌ Sin herencia — dos fuentes de verdad
class Estudiante:
    def __init__(self, nombre, cedula):
        self.nombre = nombre
        self.cedula = cedula
        self.libros_prestados = []


class Profesor:
    def __init__(self, nombre, cedula):
        self.nombre = nombre
        self.cedula = cedula
        self.libros_prestados = []
```

Si mañana cambia cómo se inicializa un usuario, hay que tocar N clases. La herencia centraliza esa responsabilidad:

```
                    Usuario
                   /       \
          Estudiante       Profesor
```

**Precisión clave:** la duplicación de código **no es, por sí sola**, razón suficiente para heredar. La pregunta correcta no es "¿cómo evito repetir esto?" sino:

> ¿Un `Estudiante` es realmente un `Usuario` dentro del dominio, y puede usarse donde el sistema espera un `Usuario` sin romper sus expectativas?

Si la respuesta es sí → evaluar herencia. Si es "solo quiero reutilizar código" → composición, no herencia.

> **Error conceptual común:** *"Si dos clases repiten código, creo una clase padre."*
> Es la versión débil del razonamiento. La versión senior siempre pasa primero por la pregunta `is-a` + sustituibilidad — la reutilización es una consecuencia, no el criterio.

---

## 2. Mecánica interna: `__init__`, `__new__` y creación de instancias

```python
class Usuario:
    def __init__(self, nombre, cedula):
        self.nombre = nombre
        self.cedula = cedula
        self.libros_prestados = []

    def solicitar_libro(self, titulo):
        return f"Solicitud del libro {titulo} realizada"
```

```
Usuario("Luis", "12345")
        │
        ▼
   __new__()  ← crea la instancia (Python lo maneja internamente)
        │
        ▼
   __init__() ← inicializa el estado de la instancia ya creada
        │
        ▼
   objeto inicializado
```

**Precisión que distingue a un Senior:** `__init__` **no crea** la instancia — la inicializa. La creación real ocurre en `__new__`. Para trabajo cotidiano basta con `__init__`, pero el modelo mental correcto importa cuando trabajas con metaclases, singletons, o frameworks que sobrescriben `__new__` (ORMs, serializadores).

---

## 3. Herencia simple: extender `Usuario`

```python
class Estudiante(Usuario):
    def __init__(self, nombre, cedula, carrera):
        super().__init__(nombre, cedula)
        self.carrera = carrera
        self.limite_libros = 3


class Profesor(Usuario):
    def __init__(self, nombre, cedula):
        super().__init__(nombre, cedula)
        self.limite_libros = None
```

```
Usuario
├── nombre, cedula, libros_prestados
│
├── Estudiante  → + carrera, limite_libros=3
└── Profesor    → + limite_libros=None
```

### `super()` — la definición que realmente importa

❌ *"`super()` llama al método del padre."* — funciona como intuición en herencia simple, pero es incompleta.

✅ **`super()` devuelve un proxy que continúa la resolución de atributos/métodos según el MRO (Method Resolution Order), desde la posición de la clase actual.** En herencia simple el MRO es lineal (`Estudiante → Usuario → object`), por eso "llamar al padre" *parece* correcto — pero deja de serlo en herencia múltiple, donde `super()` puede saltar a una tercera clase que no es literalmente el padre declarado.

**Por qué usar `super()` en vez de repetir código:** si `Usuario.__init__` cambia (ej. se agrega `self.activo = True`), las subclases que llamaron a `super().__init__(...)` reciben el cambio automáticamente; las que duplicaron la lógica quedan inconsistentes.

| Escenario | ¿`super()` = "llamar al padre"? |
|---|---|
| Herencia simple (`Estudiante → Usuario`) | Coincide en la práctica — funciona como intuición |
| Herencia múltiple (`class D(B, C)`) | **No coincide** — sigue el MRO, puede resolver a una clase que no es el padre inmediato |
| Regla a memorizar | `super()` = "siguiente paso del MRO", nunca "mi padre" |

---

## 4. Method overriding y a qué implementación llama Python

```python
class Estudiante(Usuario):
    def solicitar_libro(self, titulo):
        if len(self.libros_prestados) < self.limite_libros:
            self.libros_prestados.append(titulo)
            return f"Préstamo del libro {titulo} autorizado"

        return (
            f"No puedes prestar más libros. "
            f"Límite alcanzado: {self.limite_libros}"
        )
```

```
estudiante.solicitar_libro("Python")
        │
        ▼
Python busca en Estudiante primero
        │
        ▼
   encontrado → ejecuta Estudiante.solicitar_libro()
   (NO continúa buscando en Usuario automáticamente)
```

Si `Estudiante` no tuviera el método, Python seguiría buscando en `Usuario` (siguiente paso del MRO).

**Dos estrategias de override:**

```python
# A. Reemplazo completo — Profesor no reutiliza nada de Usuario
class Profesor(Usuario):
    def solicitar_libro(self, titulo):
        self.libros_prestados.append(titulo)
        return f"Préstamo del libro {titulo} autorizado"


# B. Especialización + delegación — Estudiante valida y reutiliza
class Estudiante(Usuario):
    def solicitar_libro(self, titulo):
        if len(self.libros_prestados) >= self.limite_libros:
            return "Límite alcanzado"

        self.libros_prestados.append(titulo)
        return super().solicitar_libro(titulo)  # reutiliza la lógica común
```

| Estrategia | Cuándo usarla |
|---|---|
| A — Reemplazo completo | El comportamiento de la base no aplica en absoluto a la subclase |
| B — Delegación con `super()` | Existe lógica común real que vale la pena centralizar |

---

## 5. Polimorfismo — por qué el consumidor no debe conocer el tipo concreto

```python
usuarios = [
    Estudiante("Luis", "12345", "Ingeniería"),
    Profesor("Ana", "67890"),
]

for usuario in usuarios:
    print(usuario.solicitar_libro("Python"))  # mismo mensaje, distinto comportamiento
```

El consumidor usa una única operación (`solicitar_libro`) sin preguntar `isinstance()`. Esto es lo que reduce acoplamiento: **el cliente conoce una interfaz, no una implementación concreta.**

```
Herencia → relación de subtipo (is-a)
Contrato común → solicitar_libro()
Polimorfismo → distintas implementaciones detrás de la misma llamada
```

**Importante:** la herencia no es requisito para lograr polimorfismo. Duck typing, `Protocol` y composición también lo logran — la herencia es *una* vía, no la única (ver §8).

---

## 6. Dónde debe vivir el estado compartido

El curso mueve `libros_prestados` de `Estudiante` a `Usuario` cuando descubre que `Profesor` también lo necesita. La regla introductoria —"lo común va al padre, lo específico al hijo"— es un buen punto de partida, pero insuficiente. La regla senior:

> El estado debe vivir en la abstracción que **conceptualmente lo posee** y cuyo contrato sea estable — no simplemente donde "ambas clases lo necesitan hoy".

Antes de mover estado hacia arriba, pregúntate:
```
¿A y B representan el mismo concepto base?
        ↓
¿Comparten invariantes y comportamiento, no solo el nombre del atributo?
        ↓
¿La relación de subtipo es estable en el tiempo?
```

---

## 7. Liskov Substitution Principle aplicado

> Si `Estudiante` es subtipo de `Usuario`, código que trabaja con `Usuario` debe poder recibir un `Estudiante` sin romper lo que ese código espera.

```python
def procesar_usuario(usuario: Usuario):
    return usuario.solicitar_libro("Python")

procesar_usuario(Estudiante(...))  # debe funcionar sin que el llamador sepa el tipo concreto
```

**Violación típica de LSP:** una subclase que lanza excepciones o impone restricciones que el contrato base no prometía, rompiendo las expectativas de cualquier código que ya confiaba en `Usuario`. El criterio no es "¿hereda?" sino **"¿puede sustituirse sin sorpresas?"**

---

## 8. Encapsulación del estado heredado

```python
estudiante.libros_prestados.append("Python")   # cualquiera puede saltarse las reglas
estudiante.libros_prestados.clear()            # ...incluida esta
```

El problema no es que el atributo sea público — es que **el objeto no controla las transiciones válidas de su propio estado**. Un flujo real requiere: validar límite → validar disponibilidad → registrar → actualizar estado. Exponer la lista directamente permite saltarse todo eso.

```python
class Usuario:
    def __init__(self, nombre, cedula):
        self.nombre = nombre
        self.cedula = cedula
        self._libros_prestados = []   # convención, no barrera real

    def devolver_libro(self, titulo):
        if titulo not in self._libros_prestados:
            return (
                f"El libro '{titulo}' no está registrado como prestado."
            )

        self._libros_prestados.remove(titulo)
        return f"Libro '{titulo}' devuelto."
```

**Precisión:** `_atributo` es convención, no seguridad. Encapsulación real = estado + operaciones válidas + invariantes que el objeto hace cumplir — no el prefijo del nombre.

| Sintaxis | Bloquea acceso externo | Propósito |
|---|---|---|
| `atributo` | No | API pública |
| `_atributo` | No (convención) | Señala implementación interna |
| `__atributo` | No de forma absoluta (name mangling) | Evita colisiones accidentales |

---

## 9. `None` como decisión de modelado, no como atajo

```python
self.limite_libros = None   # Profesor: "sin límite"
```

Funciona para el ejercicio, pero `None` es ambiguo: puede significar *sin límite*, *no configurado*, *desconocido*, *no aplica* o *dato ausente* — cinco semánticas distintas bajo el mismo valor. En un dominio que crece, esta ambigüedad se paga como deuda técnica (validaciones que no saben distinguir "profesor sin límite" de "dato faltante por bug").

**Cuándo escalar el modelo:** si el negocio empieza a necesitar reglas de préstamo más ricas (límites por categoría, excepciones temporales, políticas por facultad), conviene extraer la regla a un objeto propio en vez de sobrecargar `None`:

```
LoanPolicy
├── StudentLoanPolicy   (límite = 3)
└── ProfessorLoanPolicy (sin límite)
```

No es necesario introducirlo aquí — pero un Senior debe **reconocer el punto en que el modelo actual deja de alcanzar**, no forzar la arquitectura desde el día uno ni ignorarla cuando ya hace falta.

---

## 10. Herencia vs. composición — la señal de alerta concreta

Con dos roles (`Estudiante`, `Profesor`) la herencia simple funciona bien. El problema aparece cuando el dominio empieza a pedir **combinaciones**:

```
ProfesorInvestigador
ProfesorInvestigadorVisitante
EstudianteInvestigadorBecado
...
```

Esto es herencia usada para modelar combinaciones de capacidades, no subtipos reales — produce explosión combinatoria de clases. La alternativa es tratar esas capacidades como componentes:

```python
class Usuario:
    def __init__(self, nombre, cedula, roles=None):
        self.nombre = nombre
        self.cedula = cedula
        self.roles = roles or []   # StudentRole, ProfessorRole, ResearcherRole...
```

No existe una regla universal ("composición siempre gana"). La regla útil:

> Elige la estructura que represente con **menor complejidad** las reglas que realmente existen en el dominio — no la que se ve más "orientada a objetos".

---

## 11. Conexión con AI Engineering

El mismo patrón — contrato común + implementaciones intercambiables — es la base de cómo se abstraen proveedores de LLM en producción:

```python
class AIService:
    def __init__(self, provider):
        self.provider = provider

    def answer(self, prompt):
        return self.provider.generate(prompt)
```

```
AIService → LLMProvider (contrato) → [OpenAI | Anthropic | Gemini]
```

El servicio no necesita conocer el SDK concreto. Y en Python moderno, ese contrato no requiere herencia obligatoria — puede expresarse de forma estructural:

```python
from typing import Protocol


class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...


class OpenAIProvider:          # no hereda de LLMProvider
    def generate(self, prompt: str) -> str: ...
```

**Antipatrón a evitar** (equivalente AI del `isinstance()` disperso):

```python
# ❌ Acopla el servicio a cada implementación concreta
class AIService:
    def answer(self, prompt):
        if isinstance(self.provider, OpenAIProvider):
            ...
        elif isinstance(self.provider, AnthropicProvider):
            ...
```

Si esto se repite en `Controller`, `Service`, `Repository`, `Worker`, `Agent`, el consumidor terminó acoplado a cada implementación concreta en vez de depender del contrato (`generate()`). El objetivo del polimorfismo (§5) es exactamente evitar esto.

---

## 12. Código de referencia completo

```python
class Usuario:
    def __init__(self, nombre, cedula):
        self.nombre = nombre
        self.cedula = cedula
        self.libros_prestados = []

    def solicitar_libro(self, titulo):
        return f"Solicitud del libro {titulo} realizada"

    def devolver_libro(self, titulo):
        if titulo in self.libros_prestados:
            self.libros_prestados.remove(titulo)
            return f"Libro '{titulo}' devuelto."
        return f"El libro '{titulo}' no está registrado como prestado."


class Estudiante(Usuario):
    def __init__(self, nombre, cedula, carrera):
        super().__init__(nombre, cedula)
        self.carrera = carrera
        self.limite_libros = 3

    def solicitar_libro(self, titulo):
        if len(self.libros_prestados) < self.limite_libros:
            self.libros_prestados.append(titulo)
            return f"Préstamo del libro {titulo} autorizado"
        return f"No puedes prestar más libros. Límite alcanzado: {self.limite_libros}"


class Profesor(Usuario):
    def __init__(self, nombre, cedula):
        super().__init__(nombre, cedula)
        self.limite_libros = None

    def solicitar_libro(self, titulo):
        self.libros_prestados.append(titulo)
        return f"Préstamo del libro {titulo} autorizado"
```

---

## 13. Pregunta de entrevista — respuesta de cierre

### Pregunta

¿Por qué usarías herencia en este ejemplo (`Estudiante`/`Profesor` heredando de `Usuario`)?

### Error común

> "Para reutilizar código y evitar duplicación."

### Respuesta de alto impacto

> "La reutilización es un beneficio secundario, no el criterio. Uso herencia cuando existe una relación de subtipo estable y la subclase puede sustituir a la base sin romper su contrato (LSP). Si dos clases solo comparten implementación sin representar una relación semántica real, prefiero composición, un servicio compartido, o un `Protocol` — dependiendo de si necesito jerarquía nominal o solo contrato estructural."

### ¿Por qué la primera es insuficiente?

Reduce una decisión arquitectónica a una métrica de líneas de código. No menciona subtipado, LSP, ni alternativas — no demuestra criterio, solo memorización de sintaxis.

---

## 14. Checklist de dominio (para autoevaluarte, no para releer teoría)

- [ ] Puedo explicar la diferencia entre `__new__` y `__init__` sin dudar.
- [ ] Puedo explicar `super()` en términos de MRO, no de "llamar al padre".
- [ ] Puedo diseñar un caso de override con y sin delegación a `super()`.
- [ ] Puedo justificar dónde debe vivir un atributo compartido más allá de "ambas clases lo usan".
- [ ] Puedo identificar una violación de LSP en un ejemplo dado.
- [ ] Puedo reconocer cuándo `None` esconde una decisión de modelado no resuelta.
- [ ] Puedo detectar la señal de "herencia por combinación de roles" y proponer composición.
- [ ] Puedo trasladar este criterio a un `LLMProvider` con `Protocol` sin necesitar herencia.

---

## 15. Principio de Pareto de esta clase

El 20 % que sostiene el 80 % del criterio senior:

```text
1. La duplicación de código no justifica herencia por sí sola.
2. Herencia = relación de subtipo válida + contrato + LSP.
3. super() sigue el MRO, no "llama al padre" — solo coincide
   en herencia simple.
4. __init__ inicializa la instancia; __new__ la crea.
5. El estado compartido va donde el dominio lo posee
   conceptualmente, no donde "ambas clases lo necesitan hoy".
6. None que representa reglas de negocio es una decisión de
   modelado, no un atajo gratuito.
7. Explosión de subclases por combinación de roles → señal
   de que se necesita composición, no más herencia.
8. Polimorfismo no requiere herencia: Protocol, duck typing y
   composición también lo logran.
```

---

## 16. Regla final

> No se crea una clase padre porque dos clases repiten código — se modela una relación de subtipo válida. `super()` no es "llamar al padre", es continuar la resolución del MRO. Y cuando la jerarquía empieza a representar combinaciones de capacidades en vez de subtipos reales, es momento de evaluar composición.