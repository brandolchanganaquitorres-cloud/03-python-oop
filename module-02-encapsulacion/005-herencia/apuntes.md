# Herencia en Python — Guía de Preparación Senior AI Engineer / Tech Lead

> **Objetivo:** dominar la herencia no como sintaxis, sino como herramienta de diseño: cuándo usarla, cuándo evitarla, y cómo defender la decisión frente a un Tech Lead.

---

## 1. El problema real que resuelve la herencia

En un sistema de biblioteca hay distintos tipos de usuario que comparten datos y comportamiento (`nombre`, `cedula`, `solicitar_libro()`) pero también tienen reglas propias (un estudiante tiene límite de préstamos, un profesor depende de un departamento).

```
                       Usuario
                 ┌───────────────┐
                 │ nombre        │
                 │ cedula        │
                 │ solicitar_    │
                 │ libro()       │
                 └───────┬───────┘
               ┌─────────┴─────────┐
               ▼                   ▼
        Estudiante            Profesor
        carrera               departamento
        límite préstamos
```

**Idea central:** la clase base concentra comportamiento compartido; las derivadas especializan lo que cambia.

**Corrección clave de nivel senior:** la herencia no se justifica *solo* porque reduce duplicación. Una jerarquía puede reducir líneas de código y, al mismo tiempo, producir un diseño peor si la relación entre clases no representa el dominio correctamente. La pregunta correcta no es "¿cómo reutilizo esto?" sino **"¿qué relación contractual quiero modelar y cuánto acoplamiento estoy dispuesto a aceptar?"**

---

## 2. Mecánica del lenguaje (una sola vez, bien explicada)

### 2.1 Sintaxis base

```python
class Usuario:
    def __init__(self, nombre, cedula):
        self.nombre = nombre
        self.cedula = cedula

    def solicitar_libro(self, titulo):
        return f"Solicitud de '{titulo}' registrada para {self.nombre}."


class Estudiante(Usuario):
    def __init__(self, nombre, cedula, carrera=None, limite_libros=3, libros_prestados=0):
        super().__init__(nombre, cedula)
        self.carrera = carrera
        self.limite_libros = limite_libros
        self.libros_prestados = libros_prestados

    def solicitar_libro(self, titulo):
        if not self.carrera:
            return "No se puede prestar: carrera inactiva o no registrada."
        if self.libros_prestados >= self.limite_libros:
            return "No se puede prestar: alcanzó el límite de libros."
        return super().solicitar_libro(titulo)


class Profesor(Usuario):
    def __init__(self, nombre, cedula, departamento=None):
        super().__init__(nombre, cedula)
        self.departamento = departamento

    def solicitar_libro(self, titulo):
        if not self.departamento:
            return "No se puede prestar: departamento no asignado."
        return super().solicitar_libro(titulo)
```

Terminología: `Usuario` es clase base/padre/superclase. `Estudiante` es subclase/clase hija/clase derivada.

### 2.2 `super()` — la explicación correcta (y por qué la simplificada falla)

❌ **Simplificación incompleta:** "`super()` llama al método del padre."

✅ **Definición técnica correcta:** `super()` devuelve un proxy que continúa la búsqueda de atributos/métodos **siguiendo el MRO (Method Resolution Order) desde la posición de la clase actual**. En herencia simple esto coincide casi siempre con "llamar al padre", por eso la simplificación parece funcionar — pero deja de ser cierta en herencia múltiple.

### 2.3 MRO — Method Resolution Order

Es el orden en que Python busca métodos/atributos en una jerarquía. Se construye con el algoritmo **C3 linearization** (no necesitas implementarlo, sí entender su propósito: producir un orden consistente incluso con múltiples rutas de herencia).

```python
print(Estudiante.__mro__)
# (Estudiante, Usuario, object)
```

### 2.4 Herencia múltiple — por qué el orden de las bases importa

```python
class ProfesorEstudiante(Profesor, Estudiante):
    pass

print(ProfesorEstudiante.__mro__)
# ProfesorEstudiante → Profesor → Estudiante → Usuario → object
```

`Profesor` se resuelve antes que `Estudiante`. La explicación informal "Profesor tiene prioridad" es aceptable como intuición, pero técnicamente lo que ocurre es que **el MRO determina el orden de resolución**, no una regla de "prioridad del padre".

**super() en herencia múltiple — el caso que realmente importa:**

```python
class A:
    def ejecutar(self):
        print("A")

class B(A):
    def ejecutar(self):
        print("B")
        super().ejecutar()

class C(A):
    def ejecutar(self):
        print("C")
        super().ejecutar()

class D(B, C):
    def ejecutar(self):
        print("D")
        super().ejecutar()

# MRO: D → B → C → A → object
D().ejecutar()  # imprime D, B, C, A
```

Esto prueba que `super()` **no** significa "ejecutar directamente la clase padre escrita en la declaración" — significa "continuar la cadena cooperativa según el MRO". Para que esto funcione, **las clases deben diseñarse para cooperar** (todas usando `super()` consistentemente); no puedes tomar dos jerarquías independientes y esperar que `super()` resuelva mágicamente la incompatibilidad.

### 2.5 `ProfesorEstudiante(Profesor, Estudiante)` — por qué es un mal ejemplo de producción

Es sintácticamente legal, pero antes de aceptarlo hay que resolver:
- ¿Cómo se inicializan todos los atributos si ambos padres tienen `__init__` propio?
- ¿Qué implementación de `solicitar_libro()` debe tener prioridad y por qué?
- ¿`Profesor` y `Estudiante` son realmente subtipos de la misma jerarquía, o son **roles independientes** que un `Usuario` puede tener simultáneamente?

> Que Python *permita* una jerarquía no significa que sea una buena decisión de arquitectura.

### 2.6 `_atributo` vs `__atributo`

- `_atributo`: convención — "esto es interno", Python no restringe el acceso.
- `__atributo`: activa **name mangling** (`self.__saldo` se almacena como `self._Cuenta__saldo`). Evita colisiones de nombres accidentales en jerarquías — **no es un mecanismo de seguridad**. Para secretos/credenciales usa Secrets Manager, variables de entorno, IAM/KMS/Vault, nunca name mangling.

---

## 3. Principios de diseño

### 3.1 La relación "es un tipo de" (heurística inicial, no suficiente)

`Estudiante` es un `Usuario` → sintácticamente razonable. Pero la pregunta senior es más exigente:

> ¿La subclase puede usarse donde el sistema espera la clase base **sin romper las expectativas de comportamiento**?

### 3.2 Liskov Substitution Principle (LSP) — el criterio real

No basta con `isinstance(estudiante, Usuario)` — eso solo prueba una relación nominal. Hace falta **compatibilidad de comportamiento**: precondiciones no más estrictas, postcondiciones no más débiles, sin excepciones inesperadas que el contrato base no prometía.

```python
class Usuario:
    def solicitar_libro(self, titulo):
        return True

class Estudiante(Usuario):
    def solicitar_libro(self, titulo):
        raise RuntimeError("No permitido")  # ⚠️ viola LSP: rompe la expectativa del consumidor
```

**Diagnóstico rápido:** si una subclase necesita agregar precondiciones, debilitar postcondiciones o lanzar excepciones que el contrato base no contemplaba, la jerarquía probablemente está mal modelada.

### 3.3 Override, polimorfismo y herencia no son sinónimos

| Concepto | Qué define |
|---|---|
| **Herencia** | Relación estructural entre clases (`Estudiante → Usuario`) |
| **Override** | Una subclase redefine un método ya existente en la base |
| **Polimorfismo** | Usar objetos distintos mediante una interfaz común, sin que el consumidor conozca el tipo concreto |

El polimorfismo **puede lograrse sin herencia**: con `ABC`, `Protocol`, duck typing o composición + dependency injection. La herencia es *una* herramienta posible para lograr polimorfismo, no es sinónimo de él.

```python
usuarios = [Estudiante("Ana", "123", carrera="Ingeniería"),
            Profesor("Carlos", "456", departamento="Sistemas")]

for u in usuarios:
    print(u.solicitar_libro("Python"))  # el cliente no pregunta isinstance()
```

**Señal de diseño problemático:** encontrar `isinstance()` repetido por todo el código (`Controller`, `Service`, `Repository`, `Worker`...) suele indicar que se está perdiendo polimorfismo y el consumidor "conoce" demasiadas implementaciones concretas.

> Excepción legítima: `isinstance()` en capas de infraestructura, adaptación o integración con tipos externos no es automáticamente un code smell — el criterio no es "nunca usarlo" sino "¿por qué necesito conocer el tipo concreto aquí?"

### 3.4 Herencia vs. Composición

| | Herencia (`is-a`) | Composición (`has-a`) |
|---|---|---|
| Relación | Estructural fuerte, subtipo | El objeto usa otros objetos |
| Acoplamiento | Alto (clase base ↔ derivadas) | Bajo, componentes intercambiables |
| Cuándo | Relación de subtipo real y estable | Capacidades independientes que varían por separado |

```python
# Composición
class Usuario:
    def __init__(self, nombre, historial):
        self.nombre = nombre
        self.historial = historial  # Usuario "tiene un" HistorialPrestamos
```

No tendría sentido decir "`HistorialPrestamos` es un `Usuario`" → por tanto herencia no aplica ahí; composición sí.

### 3.5 ABC vs. Protocol — nominal typing vs. structural typing

```python
# ABC — jerarquía nominal explícita, permite compartir implementación
from abc import ABC, abstractmethod

class LLMProvider(ABC):
    @abstractmethod
    def generate(self, prompt: str) -> str: ...

class OpenAIProvider(LLMProvider):
    def generate(self, prompt: str) -> str:
        ...

# Protocol — contrato estructural, sin herencia obligatoria (PEP 544)
from typing import Protocol

class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...

class AnthropicProvider:  # no hereda, pero satisface el Protocol
    def generate(self, prompt: str) -> str:
        ...
```

| Usa **ABC** cuando... | Usa **Protocol** cuando... |
|---|---|
| Quieres jerarquía nominal explícita | Te interesa el contrato, no la jerarquía |
| Quieres compartir implementación | Trabajas con adapters / múltiples SDKs externos |
| Necesitas imponer la relación | Quieres reducir acoplamiento y facilitar testing/mocks |

No hay respuesta universal — el criterio senior es: **contrato + acoplamiento + reutilización + testabilidad + evolución del sistema**.

---

## 4. Problemas reales de producción (patrones a reconocer en code review)

### 4.1 Fragile Base Class Problem
Un cambio "seguro" en la clase base altera comportamiento heredado que las subclases asumían implícitamente. Síntoma: modificar una base obliga a revisar/testear todas sus derivadas. **Mitigación:** bases pequeñas y estables, contract tests, mover responsabilidades a componentes independientes.

### 4.2 Herencia usada solo para reutilizar código
Dos clases con métodos utilitarios similares → se crea una "BaseService" que termina acumulando logging, retries, cache, auth, HTTP, serialización... Se convierte en un punto de acoplamiento central ("Síndrome BaseEverything"). **Solución:** composición de capacidades independientes en vez de una base gigante.

### 4.3 Explosión combinatoria de herencia
Modelar roles independientes (`Profesor`, `Investigador`, `Premium`) como herencia produce combinaciones inmanejables (`ProfesorInvestigadorPremium`...). **Solución:** composición — `Usuario` con `Roles`, `Permissions`, `Subscription` como atributos combinables.

### 4.4 Constructores no cooperativos en herencia múltiple
No asumas que `class C(A, B)` inicializa automáticamente los atributos de ambos padres. La herencia múltiple cooperativa exige que **todas** las clases usen `super().__init__(**kwargs)` de forma consistente — no es gratuito, hay que diseñarlo así desde el inicio.

### 4.5 Abstracción demasiado genérica ("falsa uniformidad")
Intentar mapear todas las capacidades de OpenAI, Anthropic y Gemini a una única interfaz `generate(prompt, temperature, tools, schema, reasoning, cache, provider_specific_option, ...)` termina ocultando diferencias semánticas reales entre proveedores, produciendo bugs difíciles de diagnosticar.

> Una buena abstracción elimina complejidad accidental; una mala abstracción oculta diferencias que sí importan.

### 4.6 Sobre-abstracción innecesaria
El extremo opuesto: crear `OpenAIWrapper`, `OpenAIService`, `OpenAIAdapter`, `OpenAIProvider`, `OpenAIManager`... para una app que solo necesita `client.responses.create(...)`. Abstrae cuando **cambio frecuente + múltiples implementaciones + testing + aislamiento** lo justifican, no por dogma arquitectónico.

---

## 5. Aplicación en AI Engineering

### 5.1 Providers de LLM — el patrón Adapter

```python
from typing import Protocol

class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...

class OpenAIProvider:
    def generate(self, prompt: str) -> str: ...

class AnthropicProvider:
    def generate(self, prompt: str) -> str: ...

class AIService:
    def __init__(self, provider: LLMProvider):
        self.provider = provider

    def answer(self, prompt: str) -> str:
        return self.provider.generate(prompt)

# service = AIService(OpenAIProvider())
# service = AIService(AnthropicProvider())  ← sustitución sin tocar AIService
```

```
Application → LLMProvider (contrato) → [OpenAI | Anthropic | Gemini] Adapter → SDK/API
```

### 5.2 RAG, Tools, Agents y MCP — el mismo patrón, sin jerarquías profundas

Evita esto (explosión de clases):
```
BaseAgent → RAGAgent → ToolAgent → WebRAGToolAgent → EnterpriseWebRAGToolAgent
```

Prefiere composición de capacidades:
```
Agent
 ├── Model        (LLMProvider)
 ├── Retriever     (Protocol: search(query))
 ├── ToolRegistry  (Protocol: execute(arguments))
 ├── Memory
 └── Observability
```

MCP se aísla igual que cualquier SDK externo: `internal tool contract ← MCP adapter ← MCP protocol`. El agente depende de la **capacidad**, no del transporte concreto.

### 5.3 Testabilidad como criterio arquitectónico

Con composición + DI, sustituir dependencias reales por fakes es trivial:

```python
class FakeProvider:
    def generate(self, prompt): return "respuesta de prueba"

service = AIService(FakeProvider())  # sin credenciales, sin costo, sin latencia real
```

**Contract testing:** correr la misma suite de tests contra `OpenAIProvider`, `AnthropicProvider`, `GeminiProvider` y `FakeProvider` garantiza que todas cumplen el mismo contrato mínimo, no solo que "compilan".

### 5.4 Vendor lock-in — evaluarlo, no asumirlo

No abstraigas automáticamente por miedo al lock-in. Evalúa: probabilidad real de cambiar de proveedor, cantidad de proveedores a soportar, costo de migración, diferencias semánticas entre modelos, necesidad real de portabilidad.

---

## 6. Checklist previo a escribir `class Hija(Padre)`

```
¿Existe una relación "es un tipo de" real?
        │
   ┌────┴────┐
  SÍ         NO → usa composición
   │
   ▼
¿Existe un contrato común y estable?
   ▼
¿La subclase cumple LSP (sustituible sin sorpresas)?
   ▼
¿El polimorfismo aporta valor real al consumidor?
   ▼
¿La jerarquía se mantendrá estable en el tiempo?
   ▼
¿Composición / Protocol / DI resolverían esto más simple?
```

**Señales de alerta en code review (varias juntas → revisar arquitectura):**
- [ ] Clase base con demasiadas responsabilidades no relacionadas
- [ ] Más de 2-3 niveles de profundidad en la jerarquía
- [ ] Muchos `override` o `isinstance()` dispersos por el código
- [ ] Subclases que deshabilitan o rechazan comportamiento del padre
- [ ] `super()` difícil de rastrear en herencia múltiple
- [ ] Cambiar la base rompe múltiples subclases sin tocarlas directamente
- [ ] Clases creadas solo para compartir helpers, sin relación de subtipo real
- [ ] Explosión de clases por combinaciones de roles

---

## 7. Preguntas de entrevista Senior/Tech Lead (curadas, sin duplicados)

**1. ¿Cuándo usarías herencia y cuándo composición?**
> "Herencia cuando existe una relación de subtipo real y estable, con un contrato común que las subclases pueden cumplir sin romper LSP. Composición cuando necesito combinar capacidades independientes — reduce acoplamiento y permite sustituir implementaciones con más libertad. La decisión no la tomo por cantidad de código reutilizado, sino por el modelo de dominio y el acoplamiento que estoy dispuesto a aceptar."
> ❌ Red flag: "Herencia para reutilizar, composición cuando no." (reduce la decisión a duplicación de código, ignora subtipado).

**2. Explica qué hace realmente `super()`.**
> "No es 'llamar al padre'. Devuelve un proxy que continúa la resolución de atributos/métodos siguiendo el MRO desde la posición de la clase actual. En herencia simple coincide con llamar al padre, pero en herencia múltiple permite que las clases cooperen en una cadena definida por el MRO."
> Seguimiento típico del entrevistador: *"¿Y con dos clases padre?"* — si no puedes responder, queda expuesta la memorización superficial.

**3. ¿Qué problema ves en `class ProfesorEstudiante(Profesor, Estudiante): pass`?**
> "Es legal en Python, pero no asumiría que es buen modelo de dominio. Primero verificaría si `Profesor` y `Estudiante` son tipos de la misma jerarquía o roles independientes de un `Usuario`. Además hay que resolver el MRO, la inicialización cooperativa de ambos `__init__`, y qué implementación de `solicitar_libro()` debe prevalecer. Si son roles independientes, probablemente evaluaría composición o un modelo explícito de roles antes que herencia múltiple."

**4. ¿Cómo comprobarías si una subclase respeta correctamente su clase base (LSP)?**
> "Evaluando si una instancia de la subclase puede usarse donde se espera la base sin romper el contrato: precondiciones, postcondiciones, invariantes y comportamiento observable — no solo que tenga los mismos métodos."

**5. ¿Cuándo usarías ABC vs. Protocol en un sistema con múltiples LLM providers?**
> "Protocol cuando me interesa el contrato estructural y quiero reducir acoplamiento a una jerarquía concreta — ideal para adapters de OpenAI/Anthropic/Gemini donde no quiero forzar herencia. ABC cuando quiero una jerarquía nominal explícita y compartir implementación real entre las subclases."

**6. ¿Cómo diseñarías soporte para OpenAI, Anthropic y Gemini?**
> "Primero identifico qué capacidades son realmente comunes vs. específicas de cada proveedor. Defino un contrato interno pequeño (`Protocol`) para lo compartido, aíslo cada SDK con un adapter, y uso dependency injection para seleccionar el proveedor. No intento ocultar todas las diferencias — eso produce una abstracción artificial. Agrego contract tests para garantizar el comportamiento mínimo común."
> ❌ Red flag: "Crearía una `BaseLLM` y los tres heredarían de ella." — describe el mecanismo, no justifica la arquitectura ni contempla las diferencias semánticas entre proveedores.

**7. ¿Qué es el Fragile Base Class Problem y cómo lo mitigas?**
> "El riesgo de que cambios aparentemente seguros en la clase base alteren comportamiento inesperado en las subclases, porque estas dependen de detalles heredados además del contrato explícito. Se mitiga con contratos pequeños y estables, contract tests, y evitando bases con demasiadas responsabilidades."

**8. ¿Qué señales te harían migrar de herencia a composición en un sistema ya en producción?**
> "Si la relación de subtipo deja de ser clara, si las subclases necesitan cada vez más excepciones al contrato, si el consumidor empieza a depender de tipos concretos (`isinstance` disperso), si aparece explosión de clases por combinaciones de roles, o si testear una unidad exige levantar demasiadas dependencias heredadas."

---

## 8. Glosario técnico

| Término | Definición |
|---|---|
| **Subtyping** | Relación donde un tipo puede usarse donde se espera otro tipo compatible — más allá de heredar, exige compatibilidad de comportamiento |
| **MRO** | Method Resolution Order — orden en que Python resuelve métodos/atributos en una jerarquía (algoritmo C3 linearization) |
| **Name mangling** | Transformación de `__atributo` a `_Clase__atributo` para evitar colisiones — no es seguridad |
| **LSP** | Liskov Substitution Principle — una subclase debe poder sustituir a la base sin romper el contrato esperado |
| **Duck typing** | "Si el objeto tiene la capacidad necesaria, puede usarse como tal" — enfoque dinámico de Python |
| **Structural typing** | Compatibilidad basada en la estructura/capacidades del objeto (`Protocol`, PEP 544), no en una jerarquía declarada |
| **Nominal typing** | Compatibilidad basada en declaración explícita de herencia (`ABC`) |
| **Dependency Injection** | La dependencia se provee desde fuera del componente en vez de crearse internamente — facilita testing y sustitución |
| **Adapter** | Componente que traduce una interfaz externa (SDK, API) a un contrato interno propio |
| **Contract testing** | Ejecutar la misma suite de pruebas contra múltiples implementaciones del mismo contrato |
| **Tight / Loose coupling** | Alto acoplamiento = cambiar un componente obliga a revisar muchos otros; bajo acoplamiento = dependencias sobre contratos estables |
| **Vendor lock-in** | Dependencia fuerte de un proveedor (modelo, SDK, vector DB) que dificulta migrar |

---

## 9. Respuesta de cierre — nivel Tech Lead

> "Uso herencia cuando representa una relación de subtipo real y estable con un contrato claro que la subclase respeta según LSP. No la uso como mecanismo de reutilización porque introduce acoplamiento estructural entre base y derivadas. Cuando combino capacidades independientes prefiero composición y dependency injection; cuando necesito un contrato estructural sin jerarquía nominal, uso `Protocol`. En AI Engineering este criterio es directo: aíslo SDKs de proveedores LLM mediante adapters, mantengo contratos internos pequeños, y evito abstracciones universales que escondan diferencias semánticas reales entre modelos. La pregunta que me hago no es 'qué patrón uso', sino 'qué cambio necesito hacer posible sin romper el sistema, y qué acoplamiento estoy dispuesto a pagar por él'."

---

## 10. Recursos oficiales

- Python Classes: https://docs.python.org/3/tutorial/classes.html
- Python Data Model: https://docs.python.org/3/reference/datamodel.html
- `super()`: https://docs.python.org/3/library/functions.html#super
- `abc`: https://docs.python.org/3/library/abc.html
- `typing.Protocol`: https://docs.python.org/3/library/typing.html#typing.Protocol
- PEP 544 — Structural Subtyping: https://peps.python.org/pep-0544/
- PEP 8 — Style Guide: https://peps.python.org/pep-0008/
- mypy: https://mypy.readthedocs.io/
- Pyright: https://github.com/microsoft/pyright