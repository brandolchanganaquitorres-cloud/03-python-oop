
# CLASE 8 — Composition Over Inheritance in Python

> **Nivel objetivo:** Senior Software Engineer / Senior AI Engineer / Tech Lead
> **Enfoque:** diseño orientado a objetos, composición, Protocol, dependency inversion y evolución arquitectónica.

---

### 0. Contenido del material fuente

El material fuente en inglés introduce composition mediante la analogía de construir una casa: en lugar de tallar cada ladrillo desde cero, se ensamblan piezas ya existentes. Presenta la relación `has-a` frente a `is-a`, y construye un ejemplo práctico: una clase `Biblioteca` que contiene una lista de libros y una lista de usuarios, construida sobre el `LibroProtocol` (`prestar`, `devolver`, `calcular_duracion`) y sus dos implementaciones ya vistas en la clase anterior — `LibroFisico` (préstamo de 7 días) y `LibroDigital` (préstamo de 14 días).

El curso implementa el método `libros_disponibles()`, que filtra los libros por el atributo `disponible` sin que `Biblioteca` necesite conocer los detalles internos de cada tipo de libro. La lección cierra con la regla práctica: usar herencia cuando la relación es "is-a" (ej. `Profesor` es un `Usuario`) y composición cuando es "has-a" (ej. `Biblioteca` tiene muchos `Usuario`), señalando además que ambas técnicas pueden convivir en el mismo sistema.

El contenido es correcto como introducción, pero — igual que ocurrió con la clase de `Protocol` — insuficiente por sí solo para el nivel Senior/Tech Lead. Esta clase magistral parte de ese ejemplo y lo eleva a criterio arquitectónico completo.

#### 0.1 ¿Qué es composition en Python y por qué importa?

Composition es un enfoque de diseño donde una clase contiene instancias de otras clases como parte de su propio estado. En lugar de decir que una clase **es un tipo de** otra (herencia), decimos que **tiene una o varias**.

```text
Inheritance
    ↓
"is-a"
    ↓
Estudiante IS-A Usuario


Composition
    ↓
"has-a"
    ↓
Biblioteca HAS-A Libro
Biblioteca HAS-A Usuario
```

Ejemplo mínimo, tal como lo construye el material original:

```python
class Biblioteca:
    def __init__(self) -> None:
        self.libros = []
        self.usuarios = []
```

`Biblioteca` no necesita heredar de `Libro` ni de `Usuario`. Los **compone**.

---

### 1. Composition no significa simplemente "tener una lista"

El ejemplo del curso es correcto pero minimalista. La idea arquitectónica real no es la lista en sí, sino:

> **Una clase puede construir comportamiento delegando responsabilidades a objetos especializados, en lugar de obtenerlas mediante una jerarquía de herencia.**

```text
                   Biblioteca
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
       Libros       Usuarios     Préstamos
          │            │            │
          ▼            ▼            ▼
 LibroFisico      Estudiante    Prestamo
 LibroDigital     Profesor
```

`Biblioteca` coordina estos componentes sin convertirse en subclase de ninguno de ellos.

---

### 2. Composition vs Inheritance

La regla inicial del curso es útil como punto de partida:

```text
"is-a"  → inheritance
"has-a" → composition
```

```text
Estudiante IS-A Usuario
Profesor IS-A Usuario

Biblioteca HAS-A Libro
Biblioteca HAS-A Usuario
Biblioteca HAS-A Prestamo
```

Un Senior va un paso más allá:

> **"Is-a" y "has-a" son señales iniciales, no reglas absolutas de diseño.**

La decisión real depende de: responsabilidades, acoplamiento, variabilidad, reutilización, ciclo de vida, sustitución, evolución esperada, testabilidad y dependencia entre componentes. La herencia también introduce **acoplamiento fuerte entre la clase hija y la clase padre** — no es automáticamente "gratis" solo porque el enunciado suene a "is-a".

---

### 3. El ejemplo de Biblioteca, completo

Partimos del `LibroProtocol` definido en la Clase 7:

```python
from typing import Protocol


class LibroProtocol(Protocol):

    def prestar(self) -> str:
        ...

    def devolver(self) -> str:
        ...

    def calcular_duracion(self) -> int:
        ...
```

Con dos implementaciones — exactamente como plantea el material original en inglés (7 días para físico, 14 para digital):

```python
class LibroFisico:
    def __init__(self, titulo: str, autor: str) -> None:
        self.titulo = titulo
        self.autor = autor
        self.disponible = True

    def prestar(self) -> str:
        self.disponible = False
        return (
            f"'{self.titulo}' prestado. "
            f"Devolver en {self.calcular_duracion()} días."
        )

    def devolver(self) -> str:
        self.disponible = True
        return f"'{self.titulo}' devuelto."

    def calcular_duracion(self) -> int:
        return 7


class LibroDigital:
    def __init__(self, titulo: str, autor: str) -> None:
        self.titulo = titulo
        self.autor = autor
        self.disponible = True

    def prestar(self) -> str:
        return (
            f"'{self.titulo}' prestado digitalmente. "
            f"Disponible durante {self.calcular_duracion()} días."
        )

    def devolver(self) -> str:
        return f"'{self.titulo}' finalizó su préstamo digital."

    def calcular_duracion(self) -> int:
        return 14
```

Ambos cumplen estructuralmente el contrato:

```text
                  LibroProtocol
                  /            \
                 /              \
                ▼                ▼
         LibroFisico       LibroDigital
```

---

### 4. La Biblioteca utiliza composición

```python
class Biblioteca:

    def __init__(self, nombre: str) -> None:
        self.nombre = nombre
        self.libros: list[LibroProtocol] = []
        self.usuarios: list["Usuario"] = []
```

```text
Biblioteca
    │
    ├── libros
    │     ├── LibroFisico
    │     └── LibroDigital
    │
    └── usuarios
          ├── Estudiante
          └── Profesor
```

La biblioteca no necesita conocer todos los detalles internos de cada implementación.

---

### 5. Agregar comportamiento sobre los objetos compuestos

```python
class Biblioteca:

    def __init__(self, nombre: str) -> None:
        self.nombre = nombre
        self.libros: list[LibroProtocol] = []
        self.usuarios: list["Usuario"] = []

    def agregar_libro(self, libro: LibroProtocol) -> None:
        self.libros.append(libro)

    def libros_disponibles(self) -> list[str]:
        return [
            libro.titulo
            for libro in self.libros
            if libro.disponible
        ]
```

Aquí aparece una cuestión técnica importante: el `LibroProtocol` anterior **no declaraba `titulo` ni `disponible`**. Este código —`libro.titulo`, `libro.disponible`— no está completamente representado por el contrato tal como se definió en la Clase 7.

#### Corrección técnica

**Error detectado:** el `LibroProtocol` de la Clase 7 solo declara `prestar`, `devolver` y `calcular_duracion`. `Biblioteca.libros_disponibles()` accede a `titulo` y `disponible`, atributos que el contrato no garantiza.

**Corrección:**

```python
class LibroProtocol(Protocol):

    @property
    def titulo(self) -> str:
        ...

    @property
    def disponible(self) -> bool:
        ...

    def prestar(self) -> str:
        ...

    def devolver(self) -> str:
        ...

    def calcular_duracion(self) -> int:
        ...
```

**Justificación:** un type checker (`mypy`/`pyright`) no puede detectar el uso de `libro.titulo` como un error si el Protocol no declara ese miembro — en la práctica el chequeo pasaría silenciosamente para cualquier objeto con esos atributos, pero el contrato dejaría de documentar honestamente lo que `Biblioteca` realmente necesita.

**Impacto:** si mañana se crea un tercer tipo de libro que no expone `titulo` como atributo público (por ejemplo, lo expone como método `obtener_titulo()`), el fallo aparecerá como `AttributeError` en producción, no como error de tipos en CI — exactamente el mismo patrón de riesgo visto en la Clase 7.

**Regla Senior:**

> **Si un consumidor utiliza una capacidad, esa capacidad debe formar parte del contrato que declara necesitar.**

---

### 6. Composición + Protocol

```python
class Biblioteca:

    def __init__(self, nombre: str) -> None:
        self.nombre = nombre
        self.libros: list[LibroProtocol] = []

    def agregar_libro(self, libro: LibroProtocol) -> None:
        self.libros.append(libro)

    def prestar_libro(self, indice: int) -> str:
        libro = self.libros[indice]
        return libro.prestar()
```

`Biblioteca` no necesita:

```python
if isinstance(libro, LibroFisico):
    ...
elif isinstance(libro, LibroDigital):
    ...
```

Simplemente usa el contrato: `libro.prestar()`.

```text
Biblioteca
     │
     │ depende de
     ▼
LibroProtocol
     ▲
     │
 ┌───┴─────────────┐
 │                 │
 ▼                 ▼
Fisico           Digital
```

Esto combina tres conceptos: **Composition + Protocol + Polymorphism** — la combinación central de esta clase magistral.

---

### 7. ¿Por qué Composition puede ser preferible?

La herencia puede producir una jerarquía:

```text
               Base
                │
          ┌─────┴─────┐
          ▼           ▼
       ChildA       ChildB
          │
          ▼
       ChildC
```

Conforme crece el sistema, aparecen problemas: dependencias entre clases padre e hijas, cambios en la base que afectan a múltiples descendientes, jerarquías difíciles de modificar, comportamiento heredado que una subclase no necesita, dificultad para combinar capacidades independientes.

Composition permite construir el objeto a partir de componentes:

```text
                Service
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
   Repository   Validator   Notifier
```

Cada componente tiene una responsabilidad más específica.

---

### 8. Ejemplo cercano a AI Engineering

```python
class AIService:

    def __init__(
        self,
        llm: LLMProvider,
        retriever: Retriever,
        validator: Validator,
    ) -> None:
        self.llm = llm
        self.retriever = retriever
        self.validator = validator
```

Esto es composición: `AIService` **tiene** `LLMProvider`, `Retriever`, `Validator`. No hacemos:

```python
class AIService(OpenAIProvider, PineconeRetriever, ...):
    ...
```

porque estaríamos usando herencia para representar dependencias que conceptualmente son componentes.

```text
                    AIService
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     LLMProvider    Retriever    Validator
          │            │            │
      ┌───┴───┐     ┌──┴──┐       ...
      ▼       ▼     ▼     ▼
   OpenAI  Gemini  VectorDB
```

---

### 9. Composition permite cambiar implementaciones

```python
class OpenAIProvider:
    def generate(self, prompt: str) -> str:
        return "respuesta OpenAI"


class GeminiProvider:
    def generate(self, prompt: str) -> str:
        return "respuesta Gemini"
```

Si ambos cumplen `LLMProvider(Protocol)`:

```python
service = AIService(llm=OpenAIProvider(), retriever=retriever, validator=validator)
```

y luego:

```python
service = AIService(llm=GeminiProvider(), retriever=retriever, validator=validator)
```

`AIService` no cambia:

> **La implementación puede cambiar sin modificar el componente que consume el contrato.**

---

### 10. Composition no elimina la herencia

Un error conceptual sería pensar: "ahora que aprendí composition, nunca debo usar inheritance". Incorrecto. Podemos tener simultáneamente:

```python
class Estudiante(Usuario):
    ...
```

y:

```python
class Biblioteca:
    def __init__(self) -> None:
        self.usuarios: list[Usuario] = []
```

```text
Estudiante
    │
    │ inheritance
    ▼
Usuario


Biblioteca
    │
    │ composition
    ▼
Estudiante
```

No existe contradicción. Un sistema real puede combinar `Inheritance + Composition + Protocol + Dependency Injection` según el problema.

---

### 11. Composition vs Aggregation: precisión importante

El material original usa "composition" para una biblioteca que mantiene libros y usuarios. Conceptualmente, hay una distinción de OOP que un Senior debe saber articular:

**Composition fuerte:** el objeto contenido depende fuertemente del ciclo de vida del contenedor. `Order → OrderLine`: si desaparece el `Order`, las `OrderLine` normalmente pierden sentido en ese dominio.

**Aggregation:** el contenedor mantiene referencias a objetos que pueden existir independientemente. `Biblioteca → Libro`: un `Libro` puede existir aunque desaparezca una biblioteca concreta.

Estrictamente, una biblioteca que almacena referencias a libros ya existentes se modela mejor como **aggregation** que como composition fuerte. En Python y en la práctica cotidiana, ambos casos suelen englobarse informalmente bajo "composition" — pero saber la distinción es una señal de seniority en entrevista.

> **Respuesta de alto impacto para entrevista:** "En Python solemos usar composition en sentido amplio para referirnos a construir objetos a partir de otros objetos. Si queremos ser rigurosos con UML, conviene distinguir composition de aggregation según el ownership y el ciclo de vida."

---

### 12. El verdadero objetivo: reducir acoplamiento

La razón para preferir composition no es "porque es más moderno", sino:

> **permite controlar mejor el acoplamiento entre componentes.**

```text
Alta dependencia

AIService
    │
    ├── OpenAIClient
    ├── PineconeClient
    └── PydanticValidator
```

vs.

```text
AIService
    │
    ├── LLMProvider
    ├── Retriever
    └── Validator
             ▲
             │
      implementaciones
```

```text
Composition
     ↓
Dependency Injection
     ↓
Dependency Inversion
     ↓
Low Coupling
     ↓
Testability
     ↓
Maintainability
```

---

### 13. Composition y Dependency Injection

```python
class AIService:
    def __init__(self, llm: LLMProvider) -> None:
        self.llm = llm
```

```python
service = AIService(OpenAIProvider())
```

El constructor está haciendo **Dependency Injection**. Comparar:

```python
# Más acoplado
class AIService:
    def __init__(self) -> None:
        self.llm = OpenAIProvider()
```

```python
# Menos acoplado
class AIService:
    def __init__(self, llm: LLMProvider) -> None:
        self.llm = llm
```

Esto facilita: testing, sustitución de proveedores, configuración, model routing, fallbacks, evolución del sistema.

---

### 14. Composition y testing

```python
class AIService:
    def __init__(self, llm: LLMProvider) -> None:
        self.llm = llm

    def answer(self, prompt: str) -> str:
        return self.llm.generate(prompt)
```

```python
class FakeLLM:
    def generate(self, prompt: str) -> str:
        return "respuesta de prueba"


service = AIService(FakeLLM())
assert service.answer("Hola") == "respuesta de prueba"
```

No necesitamos Internet, API key, modelo real, latencia real ni costo real. Esto demuestra por qué composition + Protocol es tan útil en sistemas reales.

---

### 15. Una arquitectura pequeña pero profesional

```text
                    Application
                         │
                         ▼
                    AIService
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
     LLMProvider     Retriever       Validator
          │              │              │
     ┌────┼────┐         │              │
     ▼    ▼    ▼         ▼              ▼
 OpenAI Claude Gemini  Vector DB      Pydantic
```

Cada dependencia puede cambiar independientemente — mucho más escalable que una jerarquía enorme de clases.

---

### 16. Regla de decisión Senior

No uses inheritance simplemente porque "las clases tienen cosas en común". Pregunta:

1. **¿Existe una verdadera relación "is-a"?** Si sí, inheritance puede ser válida.
2. **¿Necesito reutilizar comportamiento o solo compartir capacidades?** Si es solo un contrato, considera `Protocol`.
3. **¿Una clase necesita utilizar otros componentes?** Considera `Composition`.
4. **¿Necesito poder cambiar esas dependencias?** Considera `Composition + Dependency Injection`.
5. **¿Necesito garantizar un contrato estático?** Considera `Protocol + mypy/pyright`.

---

### 17. Modelo mental definitivo

```text
                    DISEÑO OO
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
      HERENCIA                 COMPOSICIÓN
          │                         │
       "is-a"                    "has-a"
          │                         │
          ▼                         ▼
   reutilización             colaboración
   especializada             entre objetos
          │                         │
          ▼                         ▼
      jerarquía                 componentes
                                    │
                                    ▼
                             Dependency Injection
                                    │
                                    ▼
                                Protocol
                                    │
                                    ▼
                              bajo acoplamiento
```

---

### 18. Implementación final recomendada

```python
from typing import Protocol


class LibroProtocol(Protocol):

    @property
    def titulo(self) -> str:
        ...

    @property
    def disponible(self) -> bool:
        ...

    def prestar(self) -> str:
        ...

    def devolver(self) -> str:
        ...

    def calcular_duracion(self) -> int:
        ...


class Biblioteca:

    def __init__(self, nombre: str) -> None:
        self.nombre = nombre
        self.libros: list[LibroProtocol] = []

    def agregar_libro(self, libro: LibroProtocol) -> None:
        self.libros.append(libro)

    def libros_disponibles(self) -> list[str]:
        return [
            libro.titulo
            for libro in self.libros
            if libro.disponible
        ]

    def prestar_libro(self, indice: int) -> str:
        libro = self.libros[indice]
        return libro.prestar()
```

Uso:

```python
biblioteca = Biblioteca("Biblioteca Principal")

biblioteca.agregar_libro(
    LibroFisico("Cien años de soledad", "Gabriel García Márquez")
)
biblioteca.agregar_libro(
    LibroDigital("Fahrenheit 451", "Ray Bradbury")
)

for libro in biblioteca.libros:
    print(libro.prestar())
```

Observa qué **no** existe: ningún `isinstance(libro, LibroFisico)` ni `isinstance(libro, LibroDigital)`. La biblioteca simplemente usa el contrato — **Composition + Protocol + Polymorphism + Dependency Inversion**.

#### Testing del ejemplo

```python
class LibroFalso:
    """Test double mínimo que cumple LibroProtocol."""

    titulo = "Libro de prueba"
    disponible = True

    def prestar(self) -> str:
        return f"'{self.titulo}' prestado (fake)."

    def devolver(self) -> str:
        return f"'{self.titulo}' devuelto (fake)."

    def calcular_duracion(self) -> int:
        return 1


def test_libros_disponibles_filtra_por_disponibilidad() -> None:
    biblioteca = Biblioteca("Test")
    disponible = LibroFalso()
    no_disponible = LibroFalso()
    no_disponible.disponible = False

    biblioteca.agregar_libro(disponible)
    biblioteca.agregar_libro(no_disponible)

    assert biblioteca.libros_disponibles() == ["Libro de prueba"]
```

---

### 19. Preguntas de razonamiento sobre el ejemplo

**¿Qué pasaría si `Biblioteca` necesitara iterar 10,000 libros?** El diseño actual (lista en memoria) seguiría funcionando funcionalmente, pero `libros_disponibles()` con una list comprehension en memoria dejaría de ser la estrategia correcta — señal de que `Biblioteca` debería evolucionar hacia un `CatalogRepository` respaldado por una base de datos (ver Parte 6, sección sobre Repository Pattern).

**¿Por qué elegirías Protocol en vez de una clase abstracta `Libro` de la que hereden `LibroFisico` y `LibroDigital`?** Porque no existe comportamiento compartido real entre ambos (7 vs 14 días, mensajes distintos) que justifique una base común — solo un contrato de forma. Introducir una ABC agregaría una jerarquía sin beneficio de reutilización de código.

**¿Qué cambiarías si apareciera un tercer tipo, `LibroAudioLibro`?** Solo se necesita una nueva clase que implemente `LibroProtocol`; `Biblioteca` no requiere ningún cambio — es exactamente el Open/Closed Principle en acción.

---

### 20. Checklist de dominio — Parte 1

- [ ] Puedo explicar qué es composition y qué significa "has-a".
- [ ] Puedo diferenciar composition e inheritance con criterio, no solo con la regla "is-a vs has-a".
- [ ] Puedo explicar cómo una clase se compone de múltiples objetos.
- [ ] Puedo combinar composition con Protocol.
- [ ] Puedo combinar composition con Dependency Injection.
- [ ] Puedo explicar por qué composition mejora la testabilidad.
- [ ] Sé que composition no elimina inheritance.
- [ ] Puedo diferenciar rigurosamente composition de aggregation.
- [ ] Puedo aplicar el patrón a un sistema de AI Engineering.
- [ ] Detecté y corregí el desajuste entre el contrato (`LibroProtocol`) y lo que el consumidor (`Biblioteca`) realmente necesita.

---

### 21. Relación directa con las clases anteriores y siguientes

```text
Clase 5 — Herencia
    ↓
Clase 6 — Polimorfismo
    ↓
Clase 7 — Protocol / Structural Typing
    ↓
Clase 8 — Composition
    ↓
Dependency Injection
    ↓
Dependency Inversion
    ↓
Arquitectura de aplicaciones
    ↓
AI Systems
```

La evolución importante es pasar de "¿cómo reutilizo código?" a "¿cómo diseño componentes intercambiables con responsabilidades y contratos claros?".

---

### 22. Regla final de la Parte 1

> **No se trata de reemplazar herencia por composición.**
>
> **Se trata de elegir correctamente entre especialización y colaboración.**
>
> **Inheritance dice "soy un tipo de". Composition dice "trabajo con / tengo un".**
>
> **En sistemas que deben evolucionar, composition + contratos + dependency injection suele permitir cambiar implementaciones sin modificar al consumidor.**

---

## PARTE 2 — Estado del arte y problemas reales en producción

> Objetivo de esta parte: pasar del concepto académico de composition a los criterios que utilizaría un Senior/Staff Engineer al diseñar, revisar y evolucionar un sistema real.

### 23. Estado del arte: ¿qué tan actual es el enfoque del curso?

El curso introduce correctamente la idea fundamental (`is-a` / `has-a`), pero en ingeniería profesional moderna esta regla es insuficiente para tomar una decisión arquitectónica. La pregunta de un Senior no es "¿esta clase tiene una relación has-a?", sino:

> **"¿Qué diseño minimiza el acoplamiento y permite que las partes que cambian independientemente puedan evolucionar independientemente?"**

En sistemas modernos de Python, especialmente backend y AI Engineering, es frecuente encontrar:

```text
Composition
+
Protocols / interfaces
+
Dependency Injection
+
Dependency Inversion
+
Small cohesive components
+
Static typing
+
Automated tests
```

La composición deja de ser simplemente una técnica de OOP y pasa a formar parte de una estrategia para controlar dependencias.

### 24. Curso vs. práctica profesional

| Tema | Curso | Práctica profesional |
|---|---|---|
| Composition | Una clase contiene otros objetos | Se utiliza para controlar colaboración y acoplamiento |
| "has-a" | Regla principal | Señal inicial, no criterio suficiente |
| Inheritance | "is-a" | También implica acoplamiento con la jerarquía |
| Protocol | Contrato | Se combina con DI para desacoplar implementaciones |
| Biblioteca | Contiene libros y usuarios | Podría coordinar múltiples servicios/repositorios |
| `list` de objetos | Ejemplo educativo | Puede convertirse en repositorio, aggregate o colección especializada |
| Testing | Implícito | Composition facilita test doubles e inyección de dependencias |
| AI Engineering | No aparece explícitamente | Fundamental para LLM providers, retrievers, vector stores, evaluadores |
| Arquitectura | OOP básica | Se busca cohesión, bajo acoplamiento y límites claros |

La evolución profesional consiste en dejar de pensar únicamente en **clases** y empezar a pensar en **dependencias entre componentes**.

### 25. Composition no es automáticamente mejor que inheritance

Una mala interpretación común: "composition siempre es mejor". No. Un ejemplo del uso formalmente correcto pero arquitectónicamente pobre de composition:

```python
class Biblioteca:

    def __init__(
        self,
        libro_service,
        usuario_service,
        prestamo_service,
        reserva_service,
        notificacion_service,
        auditoria_service,
        pago_service,
        reporte_service,
        analytics_service,
    ):
        ...
```

Formalmente es composition. Arquitectónicamente es un **God Object**. El problema no era inheritance; el problema es demasiadas responsabilidades + demasiadas dependencias + baja cohesión.

### 26. El problema de sobre-componer

```text
Class A
   │
   ├── B
   │    └── C
   │         └── D
   │              └── E
   │
   ├── F
   │    └── G
   │
   └── H
```

Entender una operación aparentemente sencilla puede requerir navegar por muchos objetos: indirection excesiva, dificultad para rastrear el flujo, debugging más complejo, interfaces innecesarias, mayor coste cognitivo. Un Senior no optimiza el número de clases; optimiza **cohesión, acoplamiento, claridad y capacidad de evolución.**

### 27. Abstraction explosion

Crear un `Protocol` para absolutamente todo no hace el diseño más profesional:

```python
class UserRepositoryProtocol(Protocol): ...
class BookRepositoryProtocol(Protocol): ...
class LoanRepositoryProtocol(Protocol): ...
class NotificationServiceProtocol(Protocol): ...
class ValidatorProtocol(Protocol): ...
class LoggerProtocol(Protocol): ...
class ConfigurationProtocol(Protocol): ...
```

Una abstracción tiene un coste: interfaz + implementación + inyección + testing + documentación + mantenimiento. Por eso: **no abstraigas porque puedas; abstrae cuando existe una razón de diseño** (múltiples implementaciones, necesidad de sustituir infraestructura, testing, desacoplamiento entre capas, boundary arquitectónico, integración con terceros, evolución prevista).

### 28. El error de crear abstracciones demasiado pronto

```python
class EmailSenderProtocol(Protocol):
    def send(self, message: str) -> None: ...
```

Si solo existe `EmailSender` y no hay necesidad real de sustitución ni un boundary que lo justifique, la abstracción es complejidad accidental. Un Senior preguntaría: "¿qué problema estamos resolviendo con esta abstracción?" — "porque en arquitectura limpia siempre hay que usar interfaces" **no es respuesta suficiente**.

### 29. Composition y Dependency Injection

```python
class AIService:
    def __init__(self, llm: LLMProvider) -> None:
        self.llm = llm
```

vs. dependencia creada internamente:

```python
class AIService:
    def __init__(self) -> None:
        self.llm = OpenAIProvider()
```

```text
                 LLMProvider
                      ▲
              ┌───────┴───────┐
              │               │
          OpenAI           Gemini
              │               │
              └───────┬───────┘
                      ▼
                  AIService
```

El consumidor depende del contrato, no de una implementación concreta.

### 30. Composition + Protocol + Dependency Injection en AI

```python
from typing import Protocol


class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...


class AIService:
    def __init__(self, provider: LLMProvider) -> None:
        self.provider = provider

    def answer(self, prompt: str) -> str:
        return self.provider.generate(prompt)


class OpenAIProvider:
    def generate(self, prompt: str) -> str:
        return "Respuesta de OpenAI"


class AnthropicProvider:
    def generate(self, prompt: str) -> str:
        return "Respuesta de Anthropic"
```

```python
service = AIService(OpenAIProvider())
# luego, sin modificar AIService:
service = AIService(AnthropicProvider())
```

### 31. Aplicación real en sistemas LLM, RAG, Agentic AI y MCP

```text
                    AIService
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
   LLMProvider       Retriever        Validator
        │               │                │
        ▼               ▼                ▼
     OpenAI          Vector DB        Schema
```

El caso real de cambiar de proveedor LLM: requisitos que aparecen meses después de un lanzamiento (reducir costes, fallback, routing por tarea, evitar vendor lock-in, A/B testing, proveedores por región) hacen que un diseño acoplado —

```python
class AIService:
    def answer(self, prompt: str) -> str:
        client = OpenAI(...)
        return client.responses.create(...)
```

— obligue a que el cambio atraviese muchas partes del sistema. Con una abstracción adecuada, el cambio queda localizado; no elimina la complejidad, pero **contiene dónde vive**.

La misma arquitectura aparece en **RAG**:

```text
RAGService
    │
    ├── Embedder
    ├── Retriever
    ├── Reranker
    ├── LLM
    └── AnswerValidator
```

en **Agentic AI**:

```text
Agent
 │
 ├── Model
 ├── Tools
 ├── Memory
 ├── Planner
 ├── Guardrails
 └── State Store
```

y en sistemas que usan **MCP**:

```text
AI Agent
   │
   ├── Model Provider
   ├── MCP Client
   │       ├── Server A
   │       ├── Server B
   │       └── Server C
   └── Memory
```

El agente no necesita heredar de cada servidor; los usa como componentes/servicios externos — el caso donde composition resulta más natural que una jerarquía de herencia.

### 32. Problema real #1 — God Object

**Síntoma:** una clase termina con 20+ dependencias y métodos para usuarios, libros, préstamos, notificaciones, auditoría, pagos, reportes, analytics. **Causa:** se interpretó "composition = una clase coordina todo" — incorrecto. **Diagnóstico:** si cambiar una funcionalidad no relacionada obliga a modificar la misma clase, probablemente hay baja cohesión. **Solución:** separar responsabilidades (`LibraryService`, `LoanService`, `UserService`, `NotificationService`, `AuditService`). **Prevención:** High Cohesion + Single Responsibility + Bounded Dependencies.

### 33. Problema real #2 — Constructor con demasiadas dependencias

```python
class Service:
    def __init__(self, a, b, c, d, e, f, g, h): ...
```

Puede indicar demasiadas responsabilidades, mala separación de capas, falta de una abstracción intermedia, u orchestration excesiva. Pero cuidado: **el número de dependencias no determina por sí solo que el diseño sea malo** — una clase de orquestación puede coordinar legítimamente varios componentes. La pregunta correcta: "¿estas dependencias pertenecen coherentemente a la responsabilidad de este componente?"

### 34. Problema real #3 — Service Locator disfrazado

```python
class AIService:
    def __init__(self, container):
        self.container = container
# ...
self.container.get("llm")
self.container.get("retriever")
```

Parece flexible, pero oculta las dependencias reales. Comparar con:

```python
class AIService:
    def __init__(self, llm: LLMProvider, retriever: Retriever): ...
```

El primero comunica explícitamente qué necesita `AIService`; el segundo lo oculta. Regla: **preferir dependencias explícitas sobre dependencias ocultas.**

### 35. Problema real #4 — Dependency Injection excesiva

```python
class UserService:
    def __init__(
        self, user_repository, validator, logger, clock, config,
        metrics, tracer, serializer, formatter, mapper,
        feature_flags, cache,
    ): ...
```

Técnicamente correcto, pero probablemente requiere revisar los límites del componente. Un Staff preguntaría: "¿por qué este objeto necesita conocer todos estos servicios?" La solución no es eliminar DI necesariamente — puede requerir rediseñar responsabilidades.

### 36. Problema real #5 — Abstracción que no abstrae nada

```python
class OpenAIProviderProtocol(Protocol):
    def generate(self, prompt: str) -> str: ...

class OpenAIProvider: ...
```

Si solo existe OpenAI y no hay razón arquitectónica para ocultarlo, la abstracción no aporta beneficio inmediato. Tiene valor cuando desacopla `consumer → contract → implementation`; si el contrato solo duplica la implementación, cuestionarlo.

### 37. Problema real #6 — La abstracción no representa las necesidades del consumidor

```python
class BookProtocol(Protocol):
    def prestar(self) -> str: ...
    def devolver(self) -> str: ...
    def calcular_duracion(self) -> int: ...
```

pero `Biblioteca` necesita `libro.titulo` y `libro.disponible` — capacidades ausentes en el contrato (este es exactamente el problema que se corrigió en la sección 5 de la Parte 1). Solución: revisar el contrato preguntando "¿qué necesita realmente el consumidor?" y definir el contrato mínimo necesario.

### 38. Interface Segregation aplicada a Protocol

En lugar de:

```python
class MassiveBookProtocol(Protocol):
    # 20 métodos
    ...
```

separar:

```python
class Borrowable(Protocol):
    def prestar(self) -> str: ...

class Returnable(Protocol):
    def devolver(self) -> str: ...
```

Un consumidor depende solo de lo que necesita — reduce coupling, facilita testing, maintenance y evolution.

### 39. Problema real #7 — Confundir reutilización con composición

```python
class C:
    def __init__(self):
        self.a = A()
        self.b = B()
```

No es automáticamente buen diseño. La pregunta: "¿por qué `C` necesita `A` y `B`?" Composition tiene sentido cuando existe una relación real de colaboración o dependencia significativa, no como mecanismo automático para "reutilizar todo".

### 40. Problema real #8 — Acoplamiento oculto a implementaciones concretas

```python
class AIService:
    def __init__(self, provider: LLMProvider):
        self.provider = provider

    def answer(self, prompt: str):
        return self.provider.generate(prompt)
```

Hasta aquí: `AIService → LLMProvider`. Pero si después aparece:

```python
if isinstance(self.provider, OpenAIProvider):
    ...
```

se reintroduce dependencia concreta y el diseño se degrada: `Contrato → Implementación concreta → if/else por proveedor`, reduciendo el beneficio de polymorphism y Protocol.

### 41. Problema real #9 — Composition donde inheritance expresa mejor el dominio

```text
Shape
 ├── Circle
 └── Rectangle
```

Si todas las formas comparten un contrato y existen operaciones polimórficas naturales, inheritance puede ser razonable: `Circle IS-A Shape`, `Rectangle IS-A Shape` tiene sentido semántico. No eliminar inheritance simplemente por seguir una moda arquitectónica.

### 42. Problema real #10 — Herencia usada para representar capacidades

```python
class AIService(OpenAIProvider):
    ...
```

Implica `AIService IS-A OpenAIProvider` — semánticamente incorrecto. Un servicio de IA **usa** un proveedor LLM: `AIService → has-a → LLMProvider` es la relación adecuada. Regla útil: **no uses inheritance simplemente para obtener acceso a métodos de otra clase** — suele señalar diseño incorrecto.

### 43. Composition y SOLID

- **SRP:** la composición facilita dividir responsabilidades (`Retriever`, `Validator`, `LLMProvider`, `Notifier`) en lugar de una clase gigantesca.
- **OCP:** nuevas implementaciones (`OpenAIProvider`, `GeminiProvider`, `AnthropicProvider`) sin modificar el consumidor.
- **LSP:** las implementaciones deben cumplir correctamente el contrato esperado.
- **ISP:** los Protocols pueden mantenerse pequeños y orientados al consumidor.
- **DIP:** los componentes de alto nivel dependen de abstracciones/contratos, no de implementaciones concretas.

### 44. Pero cuidado con SOLID

"Si aplico todos los principios SOLID a todo, mi arquitectura será mejor" — no. SOLID son heurísticas, no reglas mecánicas; aplicarlas indiscriminadamente produce más clases, más interfaces, más indirection, más complejidad. El objetivo sigue siendo: claridad + cohesión + bajo acoplamiento + testabilidad + evolución.

### 45. Qué revisaría un Tech Lead en un Pull Request

```python
class Biblioteca:
    def __init__(self):
        self.libros = []
        self.usuarios = []
```

No se aprueba ni rechaza automáticamente. Se pregunta: (1) ¿Quién es dueño del ciclo de vida — la biblioteca crea los libros o recibe libros existentes? (2) ¿Quién modifica el estado — puede cualquier código hacer `biblioteca.libros = []`? (3) ¿Existe razón para exponer directamente la lista, o sería mejor `biblioteca.agregar_libro(libro)`? (4) ¿Qué contrato necesita `Biblioteca` — `LibroProtocol` representa realmente todas las capacidades usadas? (5) ¿La biblioteca tiene demasiadas responsabilidades? (6) ¿Cómo se prueba? (7) ¿Qué cambia si aparece un nuevo tipo de libro? Estas preguntas importan más que verificar si el código "usa composition".

### 46. Evolución del diseño

```text
Biblioteca (lista en memoria)
    ↓
biblioteca.agregar_libro(libro)
    ↓
CatalogRepository
    ↓
Database
    ↓
Application Service
       ├── CatalogRepository
       ├── LoanService
       └── NotificationService
```

> **No debemos diseñar hoy una arquitectura de 50 componentes para resolver un problema que actualmente necesita 3.** La arquitectura debe evolucionar con la complejidad real.

### 47. Qué significa "production-ready" en este contexto

No se evalúa por cantidad de clases, sino por preguntas: ¿puedo cambiar una implementación sin modificar consumidores? ¿puedo testear el consumidor sin infraestructura real? ¿las dependencias están explícitas? ¿los contratos son pequeños y estables? ¿las responsabilidades están claramente separadas? ¿puedo localizar el impacto de un cambio? ¿la abstracción realmente resuelve un problema? ¿el diseño sigue siendo entendible para otro ingeniero?

### 48. La conexión más importante con AI Engineering

En AI Engineering las implementaciones cambian constantemente (LLM, embedding model, vector database, retriever, reranker, memory, tool, provider, evaluator, guardrail). Un diseño excesivamente acoplado a una implementación concreta se vuelve costoso:

```text
                    AI Application
                          │
             ┌────────────┼─────────────┐
             ▼            ▼             ▼
          LLMProvider   Retriever    Evaluator
             │            │             │
       ┌─────┼─────┐      │             │
       ▼     ▼     ▼      ▼             ▼
    OpenAI Claude Gemini  VectorDB    Custom Eval
```

El beneficio no es "usar muchas clases" — es **aislar puntos de variabilidad del sistema**.

### 49. Checklist de revisión Senior

```text
[ ] ¿Existe una relación real de colaboración?
[ ] ¿La composición reduce acoplamiento?
[ ] ¿Cada componente tiene una responsabilidad clara?
[ ] ¿Las dependencias están explícitas?
[ ] ¿Se necesita realmente una abstracción?
[ ] ¿El Protocol representa exactamente el contrato necesario?
[ ] ¿Se puede sustituir la implementación?
[ ] ¿El componente es fácil de testear?
[ ] ¿Evita dependencia innecesaria de implementaciones concretas?
[ ] ¿No estamos creando un God Object?
[ ] ¿No estamos introduciendo abstraction explosion?
[ ] ¿La arquitectura sigue siendo comprensible?
[ ] ¿La complejidad añadida está justificada?
```

### 50. Criterio de Staff Engineer

- **Junior:** "Usé composition porque es mejor que inheritance."
- **Senior:** "Elegí composition porque estas dependencias representan colaboradores independientes, necesito sustituir algunas implementaciones y quiero que el consumidor dependa de contratos en lugar de detalles concretos."
- **Staff:** "Además, limité el contrato al comportamiento que necesita el consumidor, mantuve las dependencias explícitas, evité introducir abstracciones sin variabilidad real y verifiqué que la composición no estuviera convirtiendo el servicio en un God Object."

### 51. Resumen de la Parte 2

```text
Composition ≠ "crear muchas clases".

Composition es una herramienta para estructurar colaboración
entre componentes y controlar dependencias.

Protocol permite definir contratos sin imponer una jerarquía.

Dependency Injection permite proporcionar esas dependencias
desde fuera.

Composition + Protocol + Dependency Injection + Dependency Inversion
es especialmente útil en backend y AI Engineering.
```

Pero: `demasiada composition → demasiadas dependencias → demasiada abstracción → complejidad accidental`.

> **La mejor arquitectura no es la que tiene más abstracciones; es la que hace explícitas las decisiones importantes y mantiene bajo el coste de cambiar el sistema.**

---

## PARTE 3 — Entrevista Senior/Staff + ecosistema de IA + casos reales

### 52. Banco de preguntas de entrevista Senior/Staff sobre Composition

### 52.1 ¿Cuándo elegirías composición sobre herencia?

**Qué evalúa realmente el entrevistador:** si puedes tomar una decisión de diseño basada en acoplamiento, extensibilidad y responsabilidades, en lugar de repetir la regla superficial "is-a vs. has-a".

**Respuesta de alto impacto:**
> "Prefiero composición cuando una clase necesita coordinar o delegar responsabilidades a otros objetos y no existe una relación semántica fuerte de especialización. La composición reduce el acoplamiento estructural de una jerarquía de herencia y permite sustituir componentes con mayor facilidad. Usaría herencia cuando existe una relación estable de subtipo y necesito que la subclase pueda utilizarse donde se espera el tipo base."

```python
class Biblioteca:
    def __init__(self, libros, usuarios):
        self.libros = libros
        self.usuarios = usuarios
```

`Biblioteca` no es un `Libro` ni un `Usuario` — coordina esos objetos. En cambio `class Profesor(Usuario)` expresa relación de subtipo.

**Trade-off:** herencia → reutilización y polimorfismo, pero mayor acoplamiento a la jerarquía. Composición → mayor flexibilidad y menor acoplamiento, pero requiere definir explícitamente las colaboraciones.

**Red flag:** "Siempre hay que usar composición." — también es mala respuesta; el principio no dice que la herencia sea mala, dice que no debe usarse únicamente para reutilizar código.

### 52.2 ¿Qué diferencia existe entre composición y agregación?

**Qué evalúa:** si entiendes que "has-a" no implica necesariamente propiedad exclusiva ni el mismo ciclo de vida.

**Respuesta de alto impacto:**
> "Ambas representan relaciones entre objetos, pero la composición normalmente modela una relación más fuerte de ownership: el objeto compuesto controla o contiene una parte de su estructura. La agregación es más débil: un objeto utiliza o referencia otros objetos que pueden existir independientemente."

**Punto Senior:** no existe una palabra clave de Python que diga `composition` o `aggregation` — es una decisión del modelo de dominio.

**Red flag:** "Composición significa que el objeto se destruye automáticamente cuando se destruye el padre" — demasiado simplista para Python.

### 52.3 ¿Por qué composición puede reducir el acoplamiento frente a una jerarquía de herencia?

**Respuesta de alto impacto:**
> "La herencia crea una dependencia estructural entre la subclase y la implementación o contrato de la clase base. Cambios en la jerarquía pueden afectar múltiples descendientes. Con composición, una clase depende de una colaboración explícita y puede reemplazar ese colaborador sin modificar necesariamente su propia jerarquía."

```text
Antes:                          Después:
Biblioteca                          ┌── CatalogoMemoria
     │                              │
     ├── BibliotecaSQL     Biblioteca──┼── CatalogoSQL
     ├── BibliotecaMemoria           │
     └── BibliotecaVectorial         └── CatalogoVectorial
```

**Red flag:** "Composición siempre elimina el acoplamiento" — no; puede reducir ciertos tipos de acoplamiento, pero la clase puede seguir acoplada a la interfaz, ciclo de vida o implementación de sus colaboradores.

### 52.4 ¿Cómo combinarías composición con Protocol para diseñar un sistema extensible?

**Respuesta de alto impacto:**
> "Definiría un contrato mediante Protocol y haría que la clase principal compusiera una implementación de ese contrato, separando la dependencia del comportamiento concreto."

```python
from typing import Protocol


class CatalogoProtocol(Protocol):
    def buscar(self, titulo: str) -> list[str]: ...


class CatalogoMemoria:
    def __init__(self, libros: list[str]) -> None:
        self.libros = libros

    def buscar(self, titulo: str) -> list[str]:
        return [l for l in self.libros if titulo.lower() in l.lower()]


class Biblioteca:
    def __init__(self, catalogo: CatalogoProtocol) -> None:
        self.catalogo = catalogo

    def buscar_libro(self, titulo: str) -> list[str]:
        return self.catalogo.buscar(titulo)
```

Esto combina composición, polimorfismo, structural typing, dependency inversion y testabilidad.

**Red flag:** "Protocol reemplaza completamente a las clases" — no; define contratos principalmente para el sistema de tipos estático, no implementa comportamiento automáticamente.

### 52.5 ¿Cómo probarías una clase que utiliza composición sin depender de una implementación real?

**Respuesta de alto impacto:**

```python
class CatalogoFalso:
    def buscar(self, titulo: str) -> list[str]:
        return ["Libro de prueba"]


def test_biblioteca_busca_libro() -> None:
    biblioteca = Biblioteca(CatalogoFalso())
    assert biblioteca.buscar_libro("Python") == ["Libro de prueba"]
```

No necesitamos base de datos, API externa, infraestructura real ni implementación completa. Se relaciona directamente con Dependency Injection: la dependencia entra desde afuera (`Biblioteca(catalogo)`) en lugar de crearse internamente.

**Red flag:** "Siempre usaría Mock" — un test double explícito puede ser más claro y detectar mejor si realmente se modela el contrato correcto.

### 52.6 ¿Qué problema aparece si `Biblioteca` conoce demasiado sobre `Libro`?

**Respuesta de alto impacto:** si `Biblioteca` manipula detalles internos (`libro.estado_interno = ...`, `libro._otra_variable`), la clase depende de la representación interna de `Libro`, aumentando el acoplamiento. Mejor exponer comportamiento (`libro.esta_disponible()`) que el consumidor utiliza sin conocer la implementación:

> "Un objeto debería exponer las operaciones que otros objetos necesitan, no obligarlos a conocer cómo mantiene internamente su estado." — conecta directamente con encapsulación.

**Red flag:** confundir composición con acceso irrestricto al estado interno de los objetos compuestos.

### 52.7 ¿Composición significa que nunca deberíamos usar herencia?

**Respuesta de alto impacto:**
> "No. Composición y herencia resuelven problemas diferentes. La composición es una excelente estrategia para ensamblar comportamiento y reducir acoplamiento, mientras que la herencia sigue siendo apropiada cuando existe una relación de subtipo estable y el contrato del tipo base realmente aplica a la subclase."

```text
                 Usuario
                    ▲
             ┌──────┴──────┐
        Estudiante      Profesor
             └──────┬──────┘
                     ▼
              Biblioteca
             /           \
         usuarios       libros
                        /   \
                 LibroFisico LibroDigital
```

La arquitectura real rara vez es "solo herencia" o "solo composición".

### 52.8 ¿Por qué preferirías composición sobre herencia? (variante Staff)

**Red flag:** "Siempre composición porque la herencia es mala" — dogmatismo, no seniority.

### 52.9 ¿Cuál es la diferencia entre Dependency Injection y Dependency Inversion?

**Respuesta de alto impacto:**
> "Dependency Inversion es un principio arquitectónico: el código de alto nivel debe depender de abstracciones y no de detalles concretos. Dependency Injection es una técnica para proporcionar esas dependencias desde afuera. Puedo aplicar DI sin haber diseñado correctamente la inversión de dependencias, por lo que no son sinónimos."

**Red flag:** "DI significa usar interfaces" — no distingue principio de mecanismo.

### 52.10 ¿Por qué usarías Protocol en un sistema con múltiples proveedores LLM?

**Respuesta de alto impacto:**
> "Porque necesito expresar una capacidad requerida por el consumidor sin acoplarlo a la jerarquía de clases de cada proveedor. Protocol permite structural subtyping, por lo que OpenAI, Anthropic o un fake de testing pueden satisfacer el contrato sin heredar de una clase común. El type checker valida compatibilidad estáticamente, mientras que los adapters aíslan las diferencias específicas de cada SDK."

**Red flag:** "Porque Protocol es una clase abstracta" — incorrecto conceptualmente.

### 52.11 ¿Crearías un Protocol para cada clase?

**Respuesta de alto impacto:**
> "No. Crearía una abstracción cuando existe una necesidad real de sustitución, aislamiento, evolución o testing. Una abstracción prematura agrega indirección y coste cognitivo."

**Red flag:** "Sí, porque así desacoplamos todo" — desacoplamiento indiscriminado puede aumentar la complejidad.

### 52.12 ¿Qué diferencia existe entre un Protocol y un contract test?

**Respuesta de alto impacto:**
> "Protocol expresa principalmente un contrato estructural verificable mediante análisis estático: nombres, parámetros, retornos, compatibilidad de tipos. Un contract test verifica comportamiento observable. Una implementación puede satisfacer la forma del Protocol y aun así comportarse incorrectamente."

**Red flag:** "Protocol garantiza que el código funciona correctamente" — no lo garantiza.

### 52.13 ¿Qué problema solucionaría un Adapter en una arquitectura AI?

**Respuesta de alto impacto:**
> "Aislaría las diferencias entre APIs externas y el contrato interno de nuestra aplicación. Cada proveedor LLM tiene modelos, respuestas, errores y mecanismos de autenticación distintos. El Adapter transforma esas diferencias a un contrato interno estable, evitando que los detalles del SDK se propaguen por el dominio."

**Red flag:** "El Adapter sirve para que el código quede más ordenado" — demasiado superficial.

### 52.14 ¿Cuándo NO utilizarías composición, Protocol y DI?

**Respuesta de alto impacto:**
> "Cuando el problema no justifica la complejidad adicional: única implementación, sin necesidad de sustitución, componente estable, testing que no requiere aislarlo. Seniority implica optimizar no solo desacoplamiento, sino también complejidad total del sistema."

**Red flag:** "Siempre usaría DI para mantener buenas prácticas" — indica cargo culto de patrones.

### 52.15 ¿Cómo diseñarías un sistema que permita cambiar de OpenAI a Anthropic?

**Respuesta de alto impacto:**
> "Primero definiría qué capacidades necesita realmente la aplicación, no qué métodos expone cada SDK. Crearía contratos internos mínimos, implementaría adapters para cada proveedor y realizaría la composición desde el Composition Root. Añadiría contract tests para asegurar que cada adapter respeta el comportamiento esperado. También evaluaría diferencias funcionales, latencia, costo, rate limits y capacidades antes de asumir que los proveedores son intercambiables."

**Red flag:** `if provider == 'openai': ... elif provider == 'anthropic': ...` esparcido por todo el código — propaga el detalle de infraestructura.

### 53. Relación con el ecosistema de IA

```python
from typing import Protocol


class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...


class AIService:
    def __init__(self, provider: LLMProvider) -> None:
        self.provider = provider

    def answer(self, prompt: str) -> str:
        return self.provider.generate(prompt)
```

```text
AIService
    │
    │ compone / recibe
    ▼
LLMProvider
    ▲
 ┌──┴───────────────┐
OpenAIProvider   AnthropicProvider
```

`AIService` no necesita heredar de cada proveedor ni conocer todos sus detalles internos.

**Conexión con RAG:** `RAGService → Embedder, Retriever, LLM, DocumentStore` — en lugar de construir todo dentro de `RAGService`, cada responsabilidad se convierte en un colaborador reemplazable, permitiendo cambiar `OpenAI → Anthropic → Gemini` o `FAISS → pgvector → Azure AI Search` sin reconstruir toda la aplicación.

> **No estás simplemente evitando herencia; estás diseñando componentes intercambiables.**

### 54. Caso real de diseño: proveedor de LLM intercambiable

**Problema:** aplicación acoplada directamente a un único SDK:

```python
class AIService:
    def answer(self, prompt: str):
        client = OpenAIClient(...)
        return client.generate(prompt)
```

**Diseño mejorado:**

```python
class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...

class AIService:
    def __init__(self, provider: LLMProvider) -> None:
        self.provider = provider

    def answer(self, prompt: str) -> str:
        return self.provider.generate(prompt)
```

**Beneficio:** el dominio deja de depender directamente del proveedor concreto — facilita testing, sustitución de proveedores, evaluación de modelos, fallback, experimentación y reducción de vendor lock-in.

### 55. Regla de entrevista para recordar

Cuando pregunten "¿por qué composición?", no respondas solo "porque es has-a" — demuestra conocimiento académico, no seniority. La respuesta senior:

> **"Uso composición para ensamblar responsabilidades mediante colaboradores explícitos, reducir acoplamiento estructural y mantener componentes sustituibles. Si además defino contratos con Protocol, puedo combinar composición, polimorfismo y structural typing para construir componentes testeables y extensibles."**

### 56. Checklist de dominio — Parte 3

- [ ] ¿Cuándo elegir composición sobre herencia?
- [ ] ¿Cuál es la diferencia entre composición y agregación?
- [ ] ¿Por qué composición puede reducir acoplamiento?
- [ ] ¿Cómo combinar composición con Protocol?
- [ ] ¿Cómo probar una clase que utiliza composición?
- [ ] ¿Qué es Dependency Injection y cómo se relaciona con composición?
- [ ] ¿Por qué no conviene que una clase conozca los detalles internos de sus colaboradores?
- [ ] ¿Cuándo sigue siendo apropiada la herencia?
- [ ] ¿Cómo aplicarías composición en un sistema RAG?
- [ ] ¿Cómo diseñarías un servicio que permita cambiar de proveedor de LLM?

> **Regla final:** no se trata de eliminar la herencia; se trata de elegir conscientemente cómo ensamblar comportamiento. La herencia modela especialización y subtipos; la composición modela colaboración entre componentes. En sistemas de IA extensibles, composición + contratos explícitos + Dependency Injection suele producir componentes más fáciles de sustituir, probar y evolucionar.

---

## PARTE 4 — Recursos oficiales + glosario técnico

### 57. Recursos oficiales — prioridad profesional

**`super()` — Prioridad ALTA.** https://docs.python.org/3/library/functions.html#super — Su comportamiento está relacionado con el MRO (Method Resolution Order); no debe entenderse simplemente como "llamar al padre". Dominar: búsqueda de métodos mediante MRO, cooperación entre clases, herencia múltiple, relación entre `super()` y diseño extensible.

**Python Data Model — Prioridad MUY ALTA.** https://docs.python.org/3/reference/datamodel.html — Explica cómo funcionan internamente las clases y objetos de Python: métodos especiales, atributos, herencia, protocolos del lenguaje, resolución de atributos. Para un Senior AI Engineer no basta con saber usar clases: hay que entender cómo Python resuelve el comportamiento de los objetos.

**`typing.Protocol` — Prioridad MUY ALTA.** https://docs.python.org/3/library/typing.html#typing.Protocol — Conecta directamente esta clase con la Clase 7: `Protocol → Structural Subtyping → Duck Typing + Static Typing → Contratos → Dependency Injection → Composición`.

**`typing` — Prioridad ALTA.** https://docs.python.org/3/library/typing.html — Referencia para type annotations, `Protocol`, `TypeVar`, tipos genéricos, `Callable`, `Optional`, `Union`, colecciones, variance. No hace falta memorizar toda la librería; el objetivo profesional es saber qué existe, cuándo usarlo y dónde consultar su contrato exacto.

**PEP 544 — Protocols: Structural subtyping — Prioridad MUY ALTA.** https://peps.python.org/pep-0544/ — Conecta `Duck Typing → Structural Subtyping → Protocol → Static Type Checking`. Idea central: un tipo puede satisfacer un contrato por su estructura sin heredar explícitamente del tipo que define dicho contrato.

**PEP 484 — Type Hints — Prioridad ALTA.** https://peps.python.org/pep-0484/ — Base conceptual del sistema moderno de anotaciones de tipos de Python; interesa comprender cómo las anotaciones permiten que herramientas externas analicen el código antes de ejecutarlo.

**SOLID — Prioridad MUY ALTA.** La composición de esta clase conecta directamente con `Composition → Dependency Injection → Dependency Inversion Principle → Open/Closed Principle → Testability`. Especialmente el **Dependency Inversion Principle**: los componentes de alto nivel no deberían depender directamente de implementaciones concretas cuando pueden depender de abstracciones o contratos estables — en Python, `Protocol` puede expresar ese contrato.

**Refactoring — Martin Fowler — Prioridad ALTA.** https://martinfowler.com/books/refactoring.html — No estudiarlo como teoría aislada; usarlo para desarrollar criterio sobre cuándo extraer responsabilidades, cuándo eliminar duplicación, cuándo una jerarquía empieza a ser problemática, cuándo introducir composición, y cómo modificar código existente sin destruir su comportamiento.

**Design Patterns — Gang of Four — Prioridad MEDIA-ALTA.** No es necesario memorizar los 23 patrones. Para la trayectoria hacia AI Engineering, prestar especial atención a **Strategy, Adapter, Factory, Decorator y Composite** — muchos sistemas de IA modernos usan variantes de estos conceptos aunque no siempre se presenten explícitamente como "Design Patterns".

> **Actualización profesional 2026:** el ecosistema de análisis estático se mantiene sobre `mypy` y `pyright`, con herramientas más nuevas como `ty`/`pyrefly` (escritas en Rust) ganando adopción por velocidad en repos grandes. La referencia conceptual (PEP 544, PEP 484) sigue siendo la misma.

### 58. Glosario técnico

**Composition.** Relación en la que un objeto mantiene otros objetos como parte de su estado y delega responsabilidades en ellos. La clave arquitectónica no es que exista un atributo, sino que **un objeto colabora con otro para cumplir una responsabilidad**.

**Aggregation.** Relación "has-a" más débil en la que un objeto mantiene referencias hacia otros objetos que pueden existir independientemente. No debe confundirse automáticamente con composición fuerte; en Python la diferencia es de modelado y ownership, no una keyword del lenguaje.

**Delegation.** Un objeto recibe una solicitud y delega la ejecución a otro objeto (`AIService.answer` delega en `provider.generate`). Patrón central en arquitectura de servicios.

**Dependency Injection.** Técnica mediante la cual una dependencia se proporciona desde fuera del objeto en lugar de que el objeto la cree internamente. Facilita testing, sustitución, configuración, evolución, separación de responsabilidades.

**Structural Subtyping.** Un tipo puede considerarse compatible con otro si posee la estructura requerida, aunque no exista herencia explícita — base conceptual de `Protocol`. No pregunta "¿hereda de X?"; pregunta "¿tiene la interfaz compatible con X?".

**Nominal Typing.** Sistema de tipos donde la relación entre tipos depende de su identidad o jerarquía declarada explícitamente mediante herencia. Contrasta con structural typing.

**Duck Typing.** Principio dinámico de Python: si un objeto proporciona las operaciones necesarias, puede utilizarse independientemente de su clase concreta.

**Protocol.** Contrato estructural utilizado por el sistema de tipado de Python; una implementación no necesita heredar explícitamente, debe proporcionar una interfaz compatible.

**Dependency Inversion.** Principio arquitectónico que evita que componentes de alto nivel dependan directamente de implementaciones concretas: `AIService → LLMProvider ← OpenAIClient / AnthropicClient / GeminiClient`, en vez de `AIService → OpenAIClient` directamente. La dirección de la dependencia se diseña alrededor del contrato.

**Coupling (acoplamiento).** Grado de dependencia entre componentes. Un diseño altamente acoplado hace que cambios en un componente produzcan cambios en muchos otros. La composición puede reducir determinados tipos de acoplamiento, pero no lo elimina automáticamente.

**Cohesion (cohesión).** Qué tan relacionadas están las responsabilidades dentro de un componente. Un buen diseño busca alta cohesión + acoplamiento controlado; una clase que coordina demasiadas responsabilidades puede convertirse en un God Object aunque use composición.

**Subtype.** Tipo que puede utilizarse donde se espera su tipo base sin romper las expectativas del consumidor — relevante para herencia, Liskov Substitution Principle, Protocol, variance y generic types.

**MRO — Method Resolution Order.** Orden utilizado por Python para determinar dónde buscar atributos y métodos dentro de una jerarquía de clases; crítico al estudiar herencia múltiple y `super()`.

**Abstraction.** Representación de una interfaz o comportamiento relevante ocultando detalles de implementación innecesarios para el consumidor.

**Interface.** Contrato que especifica qué operaciones puede utilizar un consumidor. Python no tiene keyword `interface` como Java; se expresa mediante `Protocol`, clases abstractas, convenciones o duck typing. Para Python moderno, `Protocol` es especialmente relevante al combinar interfaces estructurales con static typing.

**Test Double.** Objeto utilizado durante una prueba para sustituir una dependencia real: Stub, Fake, Mock, Spy. Permite probar el consumidor sin levantar infraestructura real.

**Vendor Lock-in.** Dependencia excesiva de un proveedor que hace costoso cambiar a otra implementación. En AI Engineering aparece cuando toda la aplicación depende directamente de un proveedor específico, SDK específico, API específica y modelo específico. Composition y contratos ayudan a reducir ese acoplamiento, aunque no eliminan por completo las diferencias entre proveedores.

### 59. Qué debes retener realmente de esta clase

No hace falta memorizar la definición textual de composición, listas de patrones ni terminología académica aislada. Hay que poder razonar sobre una arquitectura:

```text
¿Esto es especialización?
        │
        ├── Sí → ¿La herencia representa realmente un subtipo?
        │
        └── No
             ↓
       ¿Es colaboración?
             ↓
        Composición
             ↓
      ¿Necesito sustituir la implementación?
             ↓
       Dependency Injection
             ↓
      ¿Quiero contrato estático?
             ↓
          Protocol
```

Ese razonamiento importa mucho más que memorizar "composition = has-a".

### 60. Conexión directa con las próximas clases

```text
OOP
 │
 ├── Composition
 │      ├── Dependency Injection
 │      └── Delegation
 │
 ├── Protocol
 │      └── Structural Typing
 │
 └── Polymorphism
        └── Interchangeable Implementations
                    │
                    ▼
             AI Architecture
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
       LLM        Retriever    Storage
        │           │           │
        ▼           ▼           ▼
     OpenAI       Vector DB    SQL/Blob
     Anthropic    API          Cloud
     Gemini
```

Esta arquitectura conceptual reaparece en RAG, agentes, tool calling, MCP, proveedores de modelos, embeddings, vector databases, servicios externos, testing, MLOps, LLMOps.

### 61. Evaluación final de la clase

**Nivel del contenido original:** intermedio de Python/OOP. El curso introduce correctamente composición y su diferencia básica con herencia, pero por sí solo no representa aún conocimiento de nivel Senior AI Engineer.

**Nivel tras esta ampliación:** intermedio-avanzado en diseño Python, con conexiones relevantes hacia arquitectura de software y AI Engineering. La diferencia está en dejar de estudiar composición como "una clase tiene otra clase" y pasar a: `Composición → Delegación → Dependency Injection → Contratos → Sustituibilidad → Testabilidad → Arquitectura extensible`. Ese es el salto de **aprender Python** a **diseñar software profesional con Python**.

> **Regla final:** no se trata de evitar la herencia por principio; se trata de modelar correctamente las dependencias. Usa herencia cuando realmente estás modelando un subtipo. Usa composición cuando estás ensamblando colaboradores. Usa contratos y Dependency Injection cuando necesitas que esos colaboradores puedan evolucionar o sustituirse sin arrastrar cambios por todo el sistema.

---

## PARTE 5 — Criterio Senior: de Composition a Arquitectura de Producción

> Esta parte no repite teoría; incorpora los conceptos que faltaban para que `composition + Protocol + polymorphism` se conviertan en criterio de diseño de un Senior AI Software Engineer / Tech Lead.

### 62. Composition no es solamente "has-a"

Cuando una clase contiene otra, un Senior analiza además: quién crea la dependencia, quién es responsable de su ciclo de vida, quién puede reemplazarla, qué contrato necesita, si debe ser concreta o abstracta, y si existe una razón real para introducir una abstracción.

```python
class AIService:
    def __init__(self, provider):
        self.provider = provider
```

La pregunta profesional no es "`AIService` tiene un `provider`", sino "¿por qué `AIService` recibe el `provider` desde afuera?" — permite cambiar de proveedor, facilita testing, desacopla el servicio de infraestructura, permite configurar la implementación en el composition root.

### 63. Dependency Injection ≠ Dependency Inversion

**Dependency Injection** es una técnica para proporcionar una dependencia desde afuera (`AIService(provider)`).

**Dependency Inversion Principle** es un principio de diseño: los componentes de alto nivel no deberían depender directamente de detalles de bajo nivel; ambos deberían depender de abstracciones.

```text
AIService
   │ depende de
   ▼
LLMProvider
   ▲ implementan
OpenAIProvider / AnthropicProvider / GeminiProvider
```

```text
Dependency Inversion = principio
Dependency Injection  = técnica
```

Un Senior debe saber distinguirlos con precisión en entrevista.

### 64. Composition Root

Si las dependencias se inyectan, ¿dónde se decide qué implementación utilizar? Respuesta profesional: el **Composition Root**.

```python
def build_application() -> AIService:
    provider = OpenAIProvider()
    return AIService(provider)
```

```text
                 Composition Root
                       │
             decide implementaciones
                       ▼
              OpenAIProvider
                       ▼
                  AIService
                       ▼
                  Application
```

Esto evita que el dominio tenga código como:

```python
class AIService:
    def __init__(self):
        self.provider = OpenAIProvider()  # acoplado a infraestructura
```

### 65. Liskov Substitution Principle: herencia no significa automáticamente sustituibilidad

```python
class Bird:
    def fly(self) -> None:
        print("Flying")

class Penguin(Bird):
    def fly(self) -> None:
        raise RuntimeError("Penguins cannot fly")
```

Sintácticamente `Penguin → Bird`, pero si un consumidor recibe un `Bird` esperando ejecutar `bird.fly()`, un `Penguin` rompe esa expectativa. Una relación de herencia correcta debe **preservar las expectativas que el consumidor tiene sobre el tipo base** — esto es LSP.

```text
"is-a"  ≠  "puede sustituirlo correctamente"
```

Razón adicional por la que composición puede ser preferible a una jerarquía de herencia excesiva.

### 66. Composition + Strategy Pattern

```python
from typing import Protocol


class PricingStrategy(Protocol):
    def calculate(self, amount: float) -> float: ...


class RegularPricing:
    def calculate(self, amount: float) -> float:
        return amount


class PremiumPricing:
    def calculate(self, amount: float) -> float:
        return amount * 0.90


class Order:
    def __init__(self, pricing: PricingStrategy):
        self.pricing = pricing

    def total(self, amount: float) -> float:
        return self.pricing.calculate(amount)
```

```python
regular_order = Order(RegularPricing())
premium_order = Order(PremiumPricing())
```

```text
Herencia: "cambio el tipo de objeto"
Composition + Strategy: "cambio la capacidad utilizada"
```

En lugar de `Order → RegularOrder / PremiumOrder / VIPOrder / EnterpriseOrder...`, el comportamiento cambia mediante composición.

### 67. Adapter Pattern en AI Engineering

Cada SDK (OpenAI, Anthropic, Gemini, Azure) tiene APIs, modelos de respuesta y configuraciones diferentes. El dominio no debe conocer esas diferencias:

```python
class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...
```

```text
                    LLMProvider
                         ▲
             ┌───────────┼───────────┐
        OpenAIAdapter  AnthropicAdapter  GeminiAdapter
             ▼           ▼           ▼
          OpenAI      Anthropic     Gemini
           SDK          SDK          SDK
```

La aplicación consume `AIService(provider: LLMProvider)` sin conocer `openai.responses.create(...)` ni `anthropic.messages.create(...)` ni `gemini.generate_content(...)`. La diferencia queda encapsulada en los adapters.

> **Principio arquitectónico:** los detalles externos deberían adaptarse al contrato interno de la aplicación, no obligar a toda la aplicación a conocer sus APIs — especialmente valioso en AI Engineering porque los proveedores cambian rápidamente.

### 68. Anti-Corruption Layer (ACL)

El Adapter puede formar parte de una estrategia más amplia. Sin ACL, propagar la respuesta específica del proveedor por todo el código (`response["choices"][0]["message"]["content"]`) contamina el sistema. Con ACL:

```text
External System
      ▼
┌──────────────────────┐
│ Anti-Corruption Layer │
│ Adapter / Mapping /   │
│ Validation             │
└──────────┬───────────┘
           ▼
     Internal Model
```

```python
class LLMResponse:
    content: str
    input_tokens: int
    output_tokens: int
```

El resto del sistema trabaja con el contrato interno propio.

### 69. Interface Segregation: Protocols pequeños

Evitar interfaces gigantes:

```python
class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...
    def embed(self, text: str) -> list[float]: ...
    def moderate(self, text: str) -> bool: ...
    def upload_file(self, path: str) -> str: ...
    def stream(self, prompt: str): ...
```

Mejor separar por capacidad:

```python
class TextGenerator(Protocol):
    def generate(self, prompt: str) -> str: ...

class Embedder(Protocol):
    def embed(self, text: str) -> list[float]: ...

class Moderator(Protocol):
    def moderate(self, text: str) -> bool: ...
```

La pregunta correcta no es "¿qué métodos tiene mi proveedor?" sino "¿qué capacidad necesita este consumidor?"

### 70. `Protocol` no ejecuta validación completa en runtime

```text
Código Python
      ▼
Pyright / mypy
      ▼
Análisis estático
      ├── compatible
      └── incompatible → error
```

Python no ejecuta automáticamente una validación completa del protocolo durante la asignación:

```python
class BadProvider:
    def embed(self, text: str) -> list[float]:
        return []

provider: LLMProvider = BadProvider()  # el type checker puede marcarlo; Python en sí no bloquea
```

```text
Static type checking ≠ Runtime validation
```

`@runtime_checkable` permite ciertos checks con `isinstance()`/`issubclass()`, pero no convierte a `Protocol` en un sistema completo de validación runtime — mismo matiz que en la Clase 7.

### 71. Test doubles: Fake, Stub, Mock y Spy

```text
Test Double
├── Fake  — implementación simplificada pero funcional
├── Stub  — respuestas predeterminadas
├── Mock  — verifica interacciones esperadas
└── Spy   — registra llamadas para inspeccionarlas
```

```python
class FakeProvider:
    def generate(self, prompt: str) -> str:
        return "respuesta de prueba"

class StubProvider:
    def generate(self, prompt: str) -> str:
        return "OK"
```

> Composition mejora testability porque permite sustituir colaboradores sin modificar el objeto que estamos probando.

### 72. Contract Testing

Con múltiples implementaciones de un mismo Protocol, no basta comprobar que las clases tienen los métodos correctos — hay que comprobar que respetan el comportamiento esperado:

```text
                    LLMProvider
              ┌──────────┼──────────┐
           OpenAI     Anthropic    Gemini
              └──────────┼──────────┘
                         ▼
                  Contract Tests
```

`Protocol` expresa "qué forma debe tener"; un contract test verifica "cómo debe comportarse" — problemas diferentes.

### 73. No abstraer también es una decisión Senior

```python
def calculate_total(amount: float) -> float:
    return amount * 1.18
```

No necesita `Protocol + Factory + Strategy + Adapter + Repository + Dependency Injection`. Una abstracción introduce indirección, complejidad, archivos adicionales, más conceptos que mantener, mayor carga cognitiva.

> **La abstracción debe pagar su propio costo.** Buena razón para abstraer: necesito sustituir implementaciones + necesito aislar infraestructura + necesito facilitar testing + espero evolución real del comportamiento. Si ninguna existe, una implementación concreta puede ser la mejor decisión.

### 74. Decision Framework de un Senior

```text
¿Es realmente un subtipo?
        ├── Sí → considerar herencia
        └── No
             ▼
      ¿Es una colaboración?
             └── Sí → composición
                         ▼
               ¿Necesito sustituibilidad?
                 ┌───────┴───────┐
                No              Sí
                 ▼               ▼
           clase concreta     Protocol → DI
```

Después: ¿el comportamiento es intercambiable? → Strategy. ¿Estoy integrando sistemas externos? → Adapter / ACL. ¿La interfaz tiene demasiadas responsabilidades? → Interface Segregation. ¿La herencia rompe expectativas del padre? → revisar LSP. ¿La abstracción cuesta más de lo que aporta? → no abstraer.

Este proceso importa mucho más que memorizar patrones.

### 75. Aplicación directa a AI Engineering

```text
                         API / FastAPI
                               ▼
                         AIService
             ┌─────────────────┼─────────────────┐
        LLMProvider        Retriever         Telemetry
       ┌─────┼─────┐       ┌───┴────┐             │
    OpenAI Anthropic Gemini   PGVector AzureSearch   OTEL
```

```python
class AIService:
    def __init__(self, provider: LLMProvider, retriever: Retriever) -> None:
        self.provider = provider
        self.retriever = retriever
```

Prepara directamente: `Composition → Dependency Injection → Protocols → Adapters → RAG → LLM Providers → Agents → MCP`.

> El consumidor depende de capacidades; los detalles concretos se conectan desde afuera.

### 76. Composición con código asíncrono

```python
from typing import Protocol


class LLMProvider(Protocol):
    async def generate(self, prompt: str) -> str: ...


class AIService:
    def __init__(self, provider: LLMProvider) -> None:
        self.provider = provider

    async def answer(self, prompt: str) -> str:
        return await self.provider.generate(prompt)
```

La diferencia está en el modelo de ejecución, no en el principio arquitectónico — relevante para FastAPI, async SDKs, concurrencia, streaming, llamadas paralelas y RAG.

### 77. Observabilidad como dependencia

```python
class Telemetry(Protocol):
    def record_latency(self, milliseconds: float) -> None: ...

class AIService:
    def __init__(self, provider: LLMProvider, telemetry: Telemetry) -> None:
        self.provider = provider
        self.telemetry = telemetry
```

Las implementaciones concretas de latencia, tokens, costo, modelo, proveedor, errores, retries y trace_id pueden cambiar sin modificar la lógica principal.

### 78. Regla crítica: abstraer por razón de cambio

> **¿Qué cambio futuro estoy aislando?**

Si realmente necesitamos alternar entre OpenAI, Anthropic y Gemini, `LLMProvider` tiene razón de existir. Si una aplicación pequeña usará una única implementación durante toda su vida útil, introducir múltiples capas puede ser innecesario. La abstracción debe responder a una necesidad arquitectónica concreta, no a "porque los Seniors usan Protocol". Los Seniors no acumulan patrones: **controlan complejidad.**

### 79. Preguntas de entrevista Senior/Staff — segunda tanda

**¿Por qué preferirías composición sobre herencia?** Qué evalúa: si sabes elegir una relación arquitectónica y no repetir "has-a vs is-a". *Red flag:* "Siempre composición porque la herencia es mala" (dogmatismo).

**¿Cuál es la diferencia entre Dependency Injection y Dependency Inversion?** *Red flag:* "DI significa usar interfaces" (no distingue principio de mecanismo).

**¿Por qué usarías Protocol en un sistema con múltiples proveedores LLM?** *Red flag:* "Porque Protocol es una clase abstracta" (incorrecto conceptualmente).

**¿Crearías un Protocol para cada clase?** *Red flag:* "Sí, porque así desacoplamos todo" (desacoplamiento indiscriminado aumenta la complejidad).

**¿Qué diferencia existe entre un Protocol y un contract test?** *Red flag:* "Protocol garantiza que el código funciona correctamente" (no lo garantiza).

**¿Qué problema solucionaría un Adapter en una arquitectura AI?** *Red flag:* "El Adapter sirve para que el código quede más ordenado" (demasiado superficial).

**¿Cuándo NO utilizarías composición, Protocol y DI?** *Respuesta de alto impacto:* "Cuando el problema no justifica la complejidad adicional... Seniority implica optimizar no solo desacoplamiento, sino también complejidad total del sistema." *Red flag:* "Siempre usaría DI para mantener buenas prácticas" (cargo culto).

### 80. Lo que debes ser capaz de explicar en una entrevista

Ante este código:

```python
class AIService:
    def __init__(self, provider: LLMProvider) -> None:
        self.provider = provider
```

No basta con decir "es composición". Hay que poder explicar:

```text
AIService
   │ composition
   ▼
provider
   │ structural contract
   ▼
LLMProvider
   ├── OpenAIAdapter
   ├── AnthropicAdapter
   └── GeminiAdapter
```

y luego: `Protocol → Structural typing → Static checking → Dependency Inversion → Dependency Injection → Testability → Provider substitution`.

Finalmente, la pregunta que distingue al Senior: **¿realmente necesitamos toda esta abstracción?** Si sí, justificarla; si no, tener la capacidad técnica de eliminarla.

### 81. Regla final de la Parte 5

> **No se trata de reemplazar herencia por composición.**
>
> **Se trata de elegir correctamente dónde debe vivir cada responsabilidad y qué dependencias pueden cambiar.**
>
> **Un Senior no abstrae todo: abstrae aquello cuyo cambio, sustitución, testing o evolución justifica el costo de la abstracción.**
>
> **La verdadera arquitectura empieza cuando puedes explicar por qué una abstracción NO debería existir.**

---

## PARTE 6 — Arquitectura Python Profesional: de OOP a sistemas mantenibles

> Objetivo: dar el siguiente salto natural después de Composition, Protocol, Dependency Injection, Adapter, Strategy y testing — de "aprender más OOP" a diseñar componentes que evolucionen, se prueben y se reemplacen sin propagar cambios por todo el sistema.

### 82. El salto conceptual: de clases a componentes

```text
Clase → Objeto → Composición → Protocol → Polimorfismo
```

Un sistema real no está compuesto solamente por clases; está compuesto por **componentes con responsabilidades diferentes**:

```text
                    Application
          ┌─────────────┼─────────────┐
       Domain       Application   Infrastructure
                                       ├── OpenAI
                                       ├── PostgreSQL
                                       └── Redis
```

El objetivo de un Senior no es que cada clase tenga muchas abstracciones — es conseguir que **un cambio localizado produzca el menor impacto posible en el resto del sistema.**

### 83. Separación de responsabilidades

```python
class AIService:
    def answer(self, prompt: str) -> str:
        # validar prompt / llamar OpenAI / registrar logs
        # guardar respuesta / manejar errores / calcular costos
        ...
```

Funciona, pero concentra demasiadas responsabilidades. Mejor:

```text
AIService
   ├── PromptValidator
   ├── LLMProvider
   ├── ConversationRepository
   └── Telemetry
```

La ventaja no es "tener más clases" — es controlar **las razones de cambio**: cambio de proveedor LLM → `LLMProvider/Adapter`; cambio de base de datos → `Repository`; cambio de observabilidad → `Telemetry`; cambio de reglas de validación → `Validator`. El cambio deja de atravesar todo el sistema.

### 84. Cohesión y acoplamiento

**Cohesión:** qué tan relacionadas están las responsabilidades dentro de un componente. Alta: `UserRepository` con operaciones relacionadas con persistencia de `User`. Baja: `UserService` mezclando DB, emails, LLM, PDF, logging.

**Acoplamiento:** cuánto depende un componente de otros componentes concretos.

```text
Mala situación:              Mejor:
AIService                    AIService
   ├── OpenAI SDK                ├── LLMProvider
   ├── PostgreSQL driver         ├── Repository
   ├── Redis client               └── NotificationService
   └── Slack SDK
```

> **Regla Senior:** alta cohesión + bajo acoplamiento es una heurística de diseño, no una religión arquitectónica. Reducir acoplamiento a cualquier costo también puede producir sistemas innecesariamente complejos.

### 85. Dependency Injection aplicada correctamente

```python
# Acoplado
class AIService:
    def __init__(self) -> None:
        self.provider = OpenAIProvider()

# Inyectado
class AIService:
    def __init__(self, provider: LLMProvider) -> None:
        self.provider = provider
```

Permite `AIService(OpenAIProvider())`, `AIService(AnthropicProvider())` o, en testing, `AIService(FakeProvider())` — el consumidor no necesita cambiar.

### 86. Constructor Injection como opción predeterminada

```python
class AIService:
    def __init__(
        self,
        provider: LLMProvider,
        repository: ConversationRepository,
    ) -> None:
        self.provider = provider
        self.repository = repository
```

Hace explícito qué necesita `AIService`. Alternativa problemática — setter injection:

```python
class AIService:
    def set_provider(self, provider):
        self.provider = provider

service = AIService()
service.answer(...)  # provider todavía no existe → estado inválido
```

Con constructor injection, la dependencia obligatoria forma parte del contrato de construcción.

### 87. Composition Root aplicado a una aplicación real

```python
def build_application() -> AIService:
    provider = OpenAIProvider()
    repository = PostgresConversationRepository()
    return AIService(provider=provider, repository=repository)
```

```text
                 Composition Root
          ┌────────────┼────────────┐
      OpenAI        PostgreSQL     Telemetry
          └────────────┼────────────┘
                       ▼
                   AIService
```

Esto evita que el dominio decida "¿utilizo OpenAI o Anthropic?" — la decisión pertenece a la configuración/composición de la aplicación.

### 88. Dependency Injection no significa usar un framework

`AIService(provider)` ya es Dependency Injection sin ningún contenedor. Un framework de DI puede aportar lifecycle management, scopes, configuración, resolución automática, integración con web frameworks — pero introduce complejidad. Muchas aplicaciones Python funcionan perfectamente con **manual dependency injection**.

> **Regla profesional:** empieza con DI explícita. Introduce un container cuando la complejidad real lo justifique.

### 89. El error de la abstracción prematura

```python
class UserRepositoryProtocol(Protocol):
    def save(self, user: User) -> None: ...

class UserRepository(UserRepositoryProtocol):
    ...
```

Puede parecer profesional, pero si hay una aplicación pequeña, una única implementación, ningún requisito de sustitución y testing que no necesita aislamiento, puede ser innecesario. El costo de una abstracción incluye más conceptos, más archivos, más navegación, más contratos, más mantenimiento.

> **No diseñes una abstracción porque puedas. Diseñala porque existe una razón de cambio que necesitas aislar.**

### 90. ¿Dónde colocar los Protocols?

```text
src/
├── domain/
│   ├── models.py
│   └── protocols.py
├── application/
│   └── services.py
├── infrastructure/
│   ├── llm/
│   │   ├── openai.py
│   │   └── anthropic.py
│   └── persistence/
│       └── postgres.py
└── main.py
```

```text
Domain/Application → define contratos → Protocols ← implementan ← Infrastructure
```

La infraestructura depende del contrato que necesita cumplir, en lugar de que todo el sistema dependa de la infraestructura.

### 91. Dependency direction

```text
Mala dirección:                    Dirección limpia:
Domain                             Application
   ↓                                   ↓
OpenAI SDK                         Protocol
   ↓                                   ▲
External API                    Infrastructure
```

```text
                     Application
                          ▼
                     LLMProvider
                    ▲          ▲
              OpenAIAdapter  GeminiAdapter
                    ▼          ▼
                 OpenAI      Gemini
```

### 92. Dependency Inversion llevado a AI (RAG)

```python
# Acoplado
class RAGService:
    def __init__(self):
        self.vector_db = Pinecone(...)

# Invertido
class Retriever(Protocol):
    def search(self, query: str, top_k: int) -> list[str]: ...

class RAGService:
    def __init__(self, retriever: Retriever) -> None:
        self.retriever = retriever
```

```text
RAGService → Retriever ← PineconeRetriever / PgVectorRetriever / AzureSearchRetriever
```

### 93. Repository Pattern: dónde aporta valor

```python
class ConversationRepository(Protocol):
    def save(self, conversation: Conversation) -> None: ...
    def get(self, conversation_id: str) -> Conversation: ...

class ConversationService:
    def __init__(self, repository: ConversationRepository) -> None:
        self.repository = repository
```

La implementación puede usar PostgreSQL, DynamoDB, Cosmos DB o MongoDB sin contaminar la lógica de aplicación. **Pero cuidado:** Repository no significa que todo acceso a datos deba pasar obligatoriamente por él — si introduce una abstracción artificial sobre consultas complejas que realmente necesitan las capacidades de la base de datos, puede empeorar el diseño.

### 94. Ports and Adapters (Hexagonal Architecture)

```text
                  ┌───────────────────┐
                  │      Domain       │
                  │   Business Logic  │
                  └─────────┬─────────┘
                          Ports
               ┌────────────┴────────────┐
            Adapter                   Adapter
               ▼                         ▼
             OpenAI                   PostgreSQL
```

Los **ports** representan capacidades que necesita la aplicación; los **adapters** conectan esas capacidades con tecnologías concretas.

### 95. FastAPI como punto de entrada, no como dominio

```text
HTTP Request → FastAPI → Application Service → [LLMProvider, Retriever, Repository] → Response
```

Mala arquitectura: poner toda la lógica en el endpoint (validar, construir prompt, consultar vector DB, llamar LLM, guardar DB, calcular métricas, responder) → se convierte en un **God Function**. Mejor:

```python
@app.post("/ask")
def ask(request: AskRequest):
    return service.answer(request.prompt)
```

La ruta HTTP actúa principalmente como adaptador de entrada.

> **Actualización profesional 2026:** FastAPI sigue siendo el framework async dominante para exponer estos servicios como API; el principio de "endpoint delgado, servicio grueso" no ha cambiado.

### 96. Application Service vs Domain

Un `AIService` que coordina `validate → retrieve → generate → persist → return` es principalmente **orquestación de aplicación**. Una regla como "un estudiante no puede superar 3 préstamos" es una **regla de dominio**. Esta distinción evita tanto el `Controller` con 1000 líneas de negocio como el `God Service` que absorbe todo el sistema.

### 97. Testing de arquitectura

```python
class FakeLLMProvider:
    def generate(self, prompt: str) -> str:
        return "respuesta de prueba"

def test_ai_service():
    service = AIService(FakeLLMProvider())
    assert service.answer("Hola") == "respuesta de prueba"
```

Sin API key, sin Internet, sin proveedor externo, sin costo por tokens ni latencia de red — consecuencia directa de haber diseñado bien las dependencias.

### 98. Testing no significa mockear todo

Un sistema con cientos de mocks es difícil de mantener y puede quedar muy acoplado a la implementación interna: si cambia la implementación pero el comportamiento externo sigue correcto, muchos tests pueden romperse igual. Enfoque más robusto: `Unit Tests + Integration Tests + Contract Tests + End-to-End Tests`, en proporción según el sistema. El objetivo no es maximizar mocks, es maximizar **confianza al menor costo razonable**.

### 99. Qué cambia cuando llegamos a producción

```text
Desarrollo:              Producción:
AIService                AIService
    ↓                        ├── timeout
  OpenAI                     ├── retry
                              ├── rate limit
                              ├── circuit breaking
                              ├── observability
                              ├── tracing
                              ├── cost tracking
                              ├── authentication
                              ├── authorization
                              └── error handling
```

> **La arquitectura no termina cuando el código funciona.** El sistema debe seguir funcionando cuando el proveedor falla, aumenta el tráfico, expira una credencial, aparece latencia, cambia el modelo, aumenta el costo, o una dependencia devuelve datos inesperados.

### 100. Resiliencia como parte del diseño

Una implementación profesional debe definir timeout, retry policy, backoff, maximum attempts, fallback, logging, metrics. Pero **retry no siempre es correcto**: si una operación no es idempotente, repetirla puede producir efectos duplicados (`crear pago`, `crear pedido`, `enviar email` no se tratan igual que un `GET` o `consultar estado`). El diseño de resiliencia debe considerar la semántica de la operación.

### 101. AI Engineering: fallback entre proveedores

```python
class FallbackProvider:
    def __init__(self, primary: LLMProvider, fallback: LLMProvider) -> None:
        self.primary = primary
        self.fallback = fallback

    def generate(self, prompt: str) -> str:
        try:
            return self.primary.generate(prompt)
        except ProviderError:
            return self.fallback.generate(prompt)
```

```text
AIService → LLMProvider → FallbackProvider → [OpenAI, Anthropic]
```

`AIService` ni siquiera necesita conocer la estrategia — composición aplicada a resiliencia.

### 102. Pero fallback no significa "usar cualquier modelo"

Dos modelos pueden tener capacidades, context windows, calidad, costo, latencia y tool-calling distintos. `OpenAI → Anthropic` no significa automáticamente equivalencia funcional. Un fallback serio define requisitos mínimos: capability compatibility + quality threshold + latency budget + cost budget — diferencia clave entre una demo y un sistema de producción.

### 103. Diseño evolutivo y capability-based design

```text
OpenAIProvider
    ↓
OpenAIProvider + AnthropicProvider
    ↓
OpenAIProvider + AnthropicProvider + GeminiProvider + AzureOpenAIProvider
```

Si el contrato agrega continuamente métodos (`generate`, `embed`, `moderate`, `transcribe`, `generate_image`, `create_audio`) se termina con una interfaz artificialmente grande. Solución: separar capacidades — `TextGenerator`, `Embedder`, `Transcriber` — y preguntar "¿qué capacidad necesito?" en lugar de "¿qué proveedor es?". Encaja naturalmente con RAG, agentes, tool calling, MCP y proveedores múltiples.

### 104. Regla de diseño para las próximas clases

Ante una nueva clase, preguntar: (1) ¿qué responsabilidad tiene? (2) ¿de qué depende? (3) ¿quién crea esas dependencias? (4) ¿necesita una abstracción? (5) ¿puede sustituirse? (6) ¿cómo la voy a testear? (7) ¿qué ocurre si una dependencia falla? (8) ¿qué parte pertenece al dominio? (9) ¿qué parte pertenece a infraestructura? (10) ¿qué cambio futuro estoy aislando? Estas preguntas forman el **criterio arquitectónico**.

### 105. Conexión completa con el ecosistema AI

```text
OOP
 ├── Encapsulation
 ├── Inheritance
 └── Polymorphism
          ▼
      Protocol
          ▼
      Composition
          ▼
 Dependency Injection
          ▼
 Dependency Inversion
          ▼
 Ports & Adapters
          ├──────────────┐
       FastAPI          AI
          │        ┌─────┼─────────┐
          │        ▼     ▼         ▼
          │       LLM   RAG      Agents
          │        ▼     ▼         ▼
          │    Providers VectorDB Tools/MCP
          └────────── Application
```

Esto ya no es OOP aislado — es la transición hacia **Software Architecture aplicada a AI Engineering**.

### 106. Preguntas de entrevista de Arquitectura de Producción

**¿Por qué no inyectar directamente `OpenAIProvider` en `AIService`?**
> "Porque eso acopla el componente de aplicación a una implementación concreta. Si `AIService` depende de `LLMProvider`, puedo cambiar el proveedor, utilizar un fake durante testing o introducir un adapter sin modificar la lógica de negocio. La decisión concreta puede quedar en el Composition Root."
*Red flag:* "Porque usar interfaces es más limpio" (demasiado superficial).

**¿Dependency Injection requiere un framework?**
> "No. Pasar explícitamente una dependencia por constructor ya es Dependency Injection. Los frameworks pueden automatizar resolución y lifecycle management, pero introducen complejidad adicional y no son requisito conceptual."
*Red flag:* "Sí, necesitamos un contenedor DI" (incorrecto).

**¿Qué problema intenta resolver Ports and Adapters?**
> "Busca aislar la lógica central de los detalles externos mediante puertos que representan capacidades requeridas y adapters que conectan esos contratos con tecnologías concretas. Esto reduce el acoplamiento y facilita sustituir infraestructura."
*Red flag:* "Es una estructura de carpetas" (la estructura puede reflejarlo, pero el concepto es arquitectónico).

**¿Por qué no crear un Protocol gigante para todos los proveedores AI?**
> "Porque obligaría a consumidores e implementaciones a depender de capacidades que quizá no necesitan. Prefiero contratos pequeños orientados a capacidades, siguiendo Interface Segregation: `TextGenerator`, `Embedder`, `Transcriber`."
*Red flag:* "Porque muchos métodos son difíciles de leer" (no explica el problema arquitectónico).

**¿Cómo diseñarías un sistema que permita cambiar de OpenAI a Anthropic?**
> "Primero definiría qué capacidades necesita realmente la aplicación, no qué métodos expone cada SDK. Crearía contratos internos mínimos, implementaría adapters para cada proveedor y realizaría la composición desde el Composition Root. Añadiría contract tests para asegurar que cada adapter respeta el comportamiento esperado. También evaluaría diferencias funcionales, latencia, costo, rate limits y capacidades antes de asumir que los proveedores son intercambiables."
*Red flag:* `if provider == 'openai': ... elif provider == 'anthropic': ...` esparcido en todo el código.

**¿Qué pasa si tu Protocol es correcto pero una implementación devuelve datos incorrectos?**
> "Protocol verifica principalmente compatibilidad estructural y de tipos mediante análisis estático. No garantiza semántica de negocio. Para comportamiento necesito tests unitarios, contract tests o integration tests según el nivel de garantía requerido."
*Red flag:* "Mypy detectaría el error" (no necesariamente).

**¿Cuál es el mayor error que ves cuando un desarrollador aprende arquitectura?**
> "Confundir cantidad de abstracciones con calidad arquitectónica. Un diseño Senior no maximiza patrones, interfaces o capas; minimiza complejidad accidental y aísla los cambios que realmente pueden ocurrir. Cada abstracción debe tener una razón de negocio o evolución técnica que justifique su costo."
*Red flag:* "No usar SOLID" (SOLID es una herramienta de razonamiento, no un objetivo).

### 107. Nivel profesional alcanzado

```text
                PYTHON OOP
        ┌───────────┼───────────┐
 Encapsulation  Inheritance  Polymorphism
                                ▼
                             Protocol
                                ▼
                           Composition
                                ▼
                    Dependency Injection
                                ▼
                    Dependency Inversion
                                ▼
                       Ports & Adapters
                  ┌─────────────┼──────────────┐
               FastAPI          RAG          LLMs
                  └─────────────┼──────────────┘
                                ▼
                         AI Architecture
```

La diferencia fundamental es dejar de aprender Python como colección de sintaxis y empezar a aprender **cómo construir sistemas que puedan cambiar sin convertirse en una cadena de modificaciones impredecibles.**

---

### 108. Checklist final de dominio — Clase 8 completa

- [ ] Puedo explicar composition más allá de "has-a" y su distinción rigurosa frente a aggregation.
- [ ] Distingo Composition de Dependency Injection y de Dependency Inversion.
- [ ] Sé qué es el Composition Root y por qué concentra las decisiones de infraestructura.
- [ ] Entiendo por qué constructor injection suele ser la opción por defecto frente a setter injection.
- [ ] Puedo explicar cohesión y acoplamiento, y por qué "bajo acoplamiento a cualquier costo" no es la meta.
- [ ] Entiendo Dependency Direction y Ports & Adapters.
- [ ] Sé cuándo un Adapter o una Anti-Corruption Layer protege al dominio de un SDK externo.
- [ ] Puedo diseñar un Protocol pequeño basado en capacidades (Interface Segregation).
- [ ] Sé cuándo NO crear una abstracción y puedo justificarlo ("la abstracción debe pagar su propio costo").
- [ ] Puedo testear una dependencia mediante un Fake/Test Double, y distinguir Fake, Stub, Mock y Spy.
- [ ] Entiendo por qué Protocol no garantiza comportamiento correcto (structural check ≠ contract test).
- [ ] Puedo diferenciar unit, integration y contract testing.
- [ ] Puedo identificar problemas de resiliencia con dependencias externas (timeout, retry, idempotencia, fallback con requisitos mínimos de capacidad).
- [ ] Puedo aplicar estos principios a LLM providers, RAG y Agentic AI.
- [ ] Puedo explicar cómo esta arquitectura facilita cambiar de proveedor sin reescribir el negocio.
- [ ] Puedo defender estos trade-offs en una entrevista Senior/Staff, incluyendo qué NO responder.
- [ ] Puedo detectar y corregir un contrato (`Protocol`) que no representa fielmente lo que el consumidor necesita.
- [ ] Sé identificar God Objects, Service Locators disfrazados y abstraction explosion en revisión de código.

---

### 109. Regla final absoluta de la Clase 8

> **No se trata de llenar el proyecto de clases, Protocols y patrones.**
>
> **Se trata de controlar las dependencias y aislar las razones de cambio.**
>
> **Un buen diseño permite cambiar infraestructura sin reescribir el negocio.**
>
> **Un diseño Senior no elimina toda dependencia: hace explícitas las dependencias importantes y controla dónde impactan.**