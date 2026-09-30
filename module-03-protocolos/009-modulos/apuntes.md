

# CLASE 9 — Organizing Python Projects Into Modules

> **Nivel objetivo:** Senior Software Engineer / Senior AI Engineer / Tech Lead / Staff Engineer

## Parte 1 — Contenido reorganizado + explicación técnica

> **Objetivo de esta parte:** entender la modularización como una herramienta de diseño, no como una simple técnica para repartir código entre archivos. La meta profesional es poder responder: **¿por qué este código pertenece a este módulo, de quién depende, qué expone y qué impacto tendría cambiarlo?**

### 0. Contenido del material fuente

El material fuente en inglés parte de un proyecto de biblioteca donde `main.py` mezclaba lógica sin relación y `usuarios.py` incluso ejecutaba código al importarse — describe esto explícitamente como una violación del principio "un archivo, una responsabilidad". La refactorización propuesta mueve las clases de libros a `libros.py`, la clase `Biblioteca` a `biblioteca.py`, y limpia `usuarios.py` para que solo contenga definiciones (clases y Protocols), sin código de ejecución.

Dos beneficios prácticos se destacan: encontrar código se vuelve trivial (si necesitas `Libro`, abres `libros.py`), y los archivos más cortos reducen el costo de tokens al pasar contexto a un LLM — el material señala explícitamente que pasar ocho líneas en vez de veinte reduce directamente el costo al generar código con asistentes de IA. También introduce una convención de nombres en plural (`libros.py`, `usuarios.py`) cuando el archivo agrupa varias clases del mismo dominio.

Tras el refactor, `main.py` se convierte en el entry point: solo importa y orquesta (crea la biblioteca, instancia estudiantes y profesores, construye los libros, conecta todo). El material explica la sintaxis de imports (`from usuarios import Estudiante, Profesor, SolicitanteProtocol`), el orden recomendado por PEP 8 (librería estándar, terceros, módulos propios — cada bloque separado por línea en blanco y ordenado alfabéticamente), y menciona que herramientas como Ruff automatizan este orden y eliminan imports no usados al guardar. Cierra con la diferencia entre módulo (`.py` individual) y package (carpeta con múltiples módulos, normalmente con `__init__.py`), y anticipa que el siguiente paso del curso será manejo de excepciones para el sistema de préstamos.

El contenido es una introducción correcta y práctica, pero — como en las clases anteriores — insuficiente por sí sola para actuar como Tech Lead/Staff: no cubre el import system internamente, import side effects peligrosos en sistemas de IA, circular imports, packaging moderno (`pyproject.toml`, `src/` layout) ni change coupling. Esta clase magistral parte de ese ejemplo y lo eleva a criterio arquitectónico completo.

### 1. Qué problema resuelve la modularización

Un proyecto pequeño puede comenzar así:

```text
main.py
├── Libro
├── LibroFisico
├── LibroDigital
├── Usuario
├── Estudiante
├── Profesor
├── Biblioteca
├── Protocols
├── lógica de préstamos
└── ejecución
```

Mientras el proyecto es pequeño, puede funcionar. El problema aparece cuando crece:

```text
más clases
+ más dependencias
+ más funcionalidades
+ más integraciones
+ más tests
+ más desarrolladores
        ↓
main.py demasiado grande
        ↓
alto acoplamiento
        ↓
difícil localizar responsabilidades
        ↓
difícil modificar sin efectos secundarios
```

La modularización busca introducir **boundaries** claros:

```text
                    Aplicación
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
     usuarios.py    libros.py      biblioteca.py
        │               │               │
     usuarios        libros          biblioteca
```

Pero hay una precisión importante:

> **Modularizar no significa simplemente crear muchos archivos.**

Un proyecto puede tener 50 archivos y seguir estando mal diseñado. La pregunta correcta es:

> **¿La división reduce el acoplamiento y aumenta la cohesión de cada módulo?**

### 2. Contenido del curso: qué cambia en el proyecto

El curso parte de un proyecto donde `main.py` concentra demasiadas responsabilidades. La refactorización propuesta separa el código aproximadamente así:

```text
proyecto/
├── main.py
├── libros.py
├── usuarios.py
└── biblioteca.py
```

Conceptualmente:

```text
libros.py    → Libro, LibroFisico, LibroDigital
usuarios.py  → Usuario, Estudiante, Profesor, SolicitanteProtocol
biblioteca.py → Biblioteca
main.py      → ensamblaje + ejecución
```

```python
# libros.py

class Libro:
    ...

class LibroFisico(Libro):
    ...

class LibroDigital(Libro):
    ...
```

```python
# biblioteca.py

class Biblioteca:
    ...
```

```python
# main.py

from biblioteca import Biblioteca
from libros import LibroDigital, LibroFisico
from usuarios import Estudiante, Profesor


biblioteca = Biblioteca("Biblioteca Principal")

libro_fisico = LibroFisico("Cien años de soledad", "Gabriel García Márquez")
libro_digital = LibroDigital("Fahrenheit 451", "Ray Bradbury")

biblioteca.agregar_libro(libro_fisico)
biblioteca.agregar_libro(libro_digital)
```

La idea central: `Módulos → definen` / `Entry point → conecta y ejecuta`.

### 3. Qué es realmente un módulo en Python

Un **módulo** es, conceptualmente, una unidad de código Python que puede ser importada. Normalmente `libros.py` es un módulo. Cuando hacemos `import libros`, Python obtiene el módulo y permite acceder a sus definiciones (`libros.Libro`, `libros.LibroFisico`). También podemos importar nombres concretos: `from libros import LibroFisico`.

### 4. Módulo ≠ clase ≠ instancia

```text
libros.py        → módulo
class Libro:      → clase
libro = Libro(...) → instancia / objeto
```

Son niveles diferentes: el **módulo** es unidad de organización/importación; la **clase** define estructura y comportamiento; la **instancia** es el objeto concreto creado a partir de la clase.

> **Un archivo Python puede contener múltiples clases, y una clase puede generar múltiples instancias.** No existe una regla de Python que diga "1 archivo = 1 clase". La decisión correcta depende de cohesión, acoplamiento y razones de cambio.

### 5. Modularización ≠ encapsulación

**Modularización** pregunta: **¿cómo divido el sistema en unidades manejables?** (`usuarios.py`, `libros.py`, `biblioteca.py`).

**Encapsulación** pregunta: **¿cómo controlo qué estado y comportamiento puede manipular directamente otro código?**

```python
class Libro:
    def __init__(self, titulo: str) -> None:
        self.titulo = titulo  # accesible directamente
```

```python
class Libro:
    def __init__(self, titulo: str) -> None:
        self._titulo = titulo

    @property
    def titulo(self) -> str:
        return self._titulo
```

La modularización organiza **unidades de código**; la encapsulación controla **acceso y exposición de implementación**. Una buena arquitectura necesita ambas, pero resolver una no significa resolver automáticamente la otra.

### 6. La regla "un archivo, una responsabilidad" necesita una precisión Senior

El curso presenta una idea útil: "un archivo debería tener una responsabilidad clara." Como regla pedagógica funciona, pero no debe interpretarse literalmente como "una clase = un archivo siempre" ni "cada función = un módulo". La interpretación profesional:

> **Un módulo debería agrupar código con alta cohesión y una razón de cambio suficientemente relacionada.**

```python
# libros.py

class Libro: ...
class LibroFisico: ...
class LibroDigital: ...
```

Tiene sentido si esas clases evolucionan como parte del mismo boundary conceptual. Pero si `LibroDigital` empieza a contener integración con almacenamiento, procesamiento de PDF, licencias, cifrado, delivery y telemetría, probablemente el módulo dejó de tener una responsabilidad cohesiva.

### 7. Cohesión: el criterio que realmente importa

Un módulo altamente cohesivo (`libros.py` con `Libro`, `LibroFisico`, `LibroDigital`) tiene una relación conceptual fuerte. Un módulo poco cohesivo podría ser:

```text
utils.py
├── calcular_interes()
├── enviar_email()
├── parsear_json()
├── crear_usuario()
├── generar_embedding()
└── conectar_postgres()
```

Aunque técnicamente todo "funcione", el módulo se convierte en un contenedor arbitrario. El problema es que distintas partes cambian por razones diferentes.

### 8. Reason for Change

Pregunta útil: **¿por qué tendría que cambiar este módulo?** `libros.py` debería cambiar cuando cambian las reglas relacionadas con libros. Pero si también cambia cuando cambia PostgreSQL, OpenAI, autenticación, email o Docker, probablemente contiene demasiadas responsabilidades — conecta directamente con el **Single Responsibility Principle**: una responsabilidad no significa "una sola función"; significa una razón coherente de cambio.

### 9. Change Coupling

Dos módulos pueden estar físicamente separados (`usuarios.py`, `libros.py`) pero si prácticamente cada cambio en uno obliga a modificar el otro, existe un acoplamiento de cambio importante. La modularización debería intentar reducir este fenómeno cuando no representa una dependencia legítima.

> La pregunta Senior no es "¿cuántos archivos tengo?" Es **"¿cuántos cambios independientes puedo hacer sin propagar modificaciones innecesarias por el sistema?"**

### 10. Acoplamiento

```python
# biblioteca.py
from usuarios import Usuario
from libros import Libro
```

`biblioteca.py` tiene dependencias explícitas — eso no es necesariamente malo; es necesario si `Biblioteca` realmente necesita esos conceptos. El problema aparece cuando las dependencias son innecesarias, bidireccionales, difíciles de sustituir, demasiado concretas, difíciles de testear, o provocan una cadena de cambios excesiva.

> **El objetivo no es eliminar todo acoplamiento. El objetivo es controlar el acoplamiento.**

### 11. La dirección de las dependencias importa

```text
main.py → biblioteca.py → libros.py
```

Dirección clara. Pero:

```text
biblioteca.py → usuarios.py → biblioteca.py
```

genera un ciclo: `Biblioteca → Usuario → Biblioteca`. Eso puede provocar **circular imports** y, más importante, revelar un problema arquitectónico. La modularización no consiste solo en separar archivos — también consiste en diseñar la **dependency direction**.

### 12. El sistema de imports de Python

```python
from libros import Libro
```

Python necesita localizar, cargar e inicializar el módulo:

```text
from libros import Libro
        ↓
resolver módulo → localizar libros → cargar módulo →
ejecutar código de nivel módulo → registrar módulo → obtener Libro
```

`import` no es simplemente "leer otro archivo". El import system tiene reglas de resolución y carga — comprender esto resulta importante ante circular imports, packages, plugins, tests, packaging, import side effects, configuración y aplicaciones AI.

### 13. `sys.path`: ¿dónde busca Python?

```python
import sys
print(sys.path)
```

```text
import libros → ¿dónde está libros? → rutas de búsqueda → encuentra módulo → lo carga
```

Esto explica por qué un import puede funcionar desde un directorio y fallar desde otro, y anticipa problemas con packages, virtual environments, installation, `src/` layout y `PYTHONPATH`.

> **Python necesita un mecanismo de resolución para determinar qué módulo corresponde a un import.**

### 14. `sys.modules` y module caching

```python
import sys
print(sys.modules)
```

`sys.modules` funciona como un registro/cache de módulos cargados:

```text
import libros → ¿libros está en sys.modules?
    ├── Sí → usar módulo
    └── No → cargar módulo → sys.modules
```

Esto ayuda a entender por qué el sistema de imports no ejecuta ingenuamente el archivo completo cada vez que aparece `import libros` — `sys.modules` es una pieza central del proceso.

### 15. Import side effects

Este es un problema real que el curso apenas menciona, y adquiere mucha importancia en sistemas profesionales.

```python
# usuarios.py
print("Conectando a la base de datos...")
database = connect_to_database()
```

```python
import usuarios  # produce un efecto secundario
```

En una aplicación AI podría ser mucho peor:

```python
# ❌ Evitar
client = OpenAI()
response = client.responses.create(model="...", input="...")
```

Si ese código está a nivel de módulo, simplemente importar el archivo puede ejecutar una llamada externa: llamadas inesperadas, coste de API, lentitud durante imports, problemas en tests, errores de inicialización, dependencia de secretos, problemas de observabilidad, comportamiento difícil de predecir.

> **Importar un módulo debería, en general, establecer definiciones; no ejecutar operaciones externas inesperadas.**

Esto no significa que ningún código a nivel de módulo pueda ejecutarse — definiciones de clases, funciones, constantes y estructuras necesarias para inicializar el módulo son normales. El problema son los **side effects inesperados**.

### 16. `__name__ == "__main__"`

```python
def main() -> None:
    biblioteca = Biblioteca("Biblioteca Principal")
    ...


if __name__ == "__main__":
    main()
```

Permite que el archivo se ejecute directamente o se importe sin ejecutar automáticamente la aplicación. Al ejecutar `python main.py`, Python establece `__name__ == "__main__"`; cuando otro módulo hace `import main`, `__name__` es normalmente `"main"`. Por tanto, `if __name__ == "__main__": main()` protege el código de ejecución.

### 17. `main.py` no debería convertirse en otro God Object

Después de modularizar, un error común es trasladar todo a `main.py`:

```python
# ❌
def main():
    # configuración / conexión DB / creación de repositorios
    # reglas de negocio / llamadas al LLM / autenticación
    # logging / ejecución
```

Eso no es modularización real. Un `main.py` profesional normalmente actúa como **Entry Point** y, dependiendo de la aplicación, puede participar también en el **Composition Root**:

```text
Entry Point → Composition Root → [implementaciones, dependencias, configuración] → aplicación
```

La lógica de negocio no debería terminar acumulándose allí.

### 18. Entry Point vs Composition Root

**Entry Point:** lugar donde comienza la ejecución de la aplicación (`if __name__ == "__main__": main()`).

**Composition Root:** lugar donde se decide qué implementación concreta se utiliza.

```python
def build_application() -> AIService:
    provider = OpenAIProvider()
    repository = PostgresConversationRepository()
    return AIService(provider=provider, repository=repository)
```

```text
Entry Point → Composition Root → construcción de dependencias → Application
```

No debemos confundir "ejecutar la aplicación" con "decidir cómo se ensamblan sus dependencias".

### 19. Imports absolutos vs relativos

```python
from usuarios import Usuario           # absoluto
from .usuarios import Usuario          # relativo, dentro de un package
```

La diferencia está relacionada con cómo se resuelve el módulo dentro del package. La elección no debe hacerse arbitrariamente: en aplicaciones profesionales, especialmente con una estructura de package clara, hay que ser consistente con la estrategia de imports definida por el proyecto.

### 20. ¿Qué es un package?

Un módulo normalmente corresponde a un archivo (`libros.py`). Un package permite organizar múltiples módulos dentro de un namespace:

```text
biblioteca/
├── __init__.py
├── libros.py
├── usuarios.py
└── biblioteca.py
```

```python
from biblioteca.libros import Libro
```

### 21. `__init__.py`: qué hace y qué NO hace

Históricamente se usó para marcar directorios como packages tradicionales. Pero afirmar que "`__init__.py` siempre es obligatorio" es incorrecto en Python moderno: existen **namespace packages** que pueden funcionar sin él. Además, `__init__.py` puede ejecutar código cuando el package es importado, así que no debe llenarse arbitrariamente. Puede usarse para definir una API pública cuidadosamente seleccionada, pero introducir demasiada lógica ahí puede generar import side effects y acoplamiento innecesario.

### 22. API pública vs implementación interna

```text
          módulo
┌───────────────────────────┐
│   API pública              │
│   Biblioteca                │
│                            │
│   implementación interna   │
│   helpers, detalles,        │
│   algoritmos                │
└───────────────────────────┘
```

Un consumidor debería depender solo de `from biblioteca import Biblioteca`, no de detalles internos que no forman parte del contrato del módulo. Esto reduce el acoplamiento entre consumidores e implementación.

### 23. Modularización y Protocol

La Clase 8 introdujo `class LibroProtocol(Protocol): ...`. La modularización permite colocar ese contrato en un boundary apropiado:

```text
biblioteca/
├── protocols.py
├── libros.py
├── biblioteca.py
└── usuarios.py
```

Pero cuidado: **no significa que debamos crear un `Protocol` para cada clase**. El módulo y el contrato deben existir porque hay una razón arquitectónica. Si `Biblioteca` necesita `libro.titulo`, `libro.disponible`, `libro.prestar()`, el contrato debe representar las capacidades que realmente consume — exactamente la corrección técnica ya vista en la Clase 8.

```text
Protocol → contrato → Dependency Inversion → Dependency Injection → Composition
```

La modularización agrega otro nivel: `módulos → organizan boundaries → contratos cruzan boundaries → implementaciones quedan detrás`.

### 24. La modularización no elimina las dependencias

```text
biblioteca.py → libros.py
```

Existe dependencia. La modularización no busca conseguir componentes sin conexiones — eso sería imposible en un sistema real. Busca conseguir: dependencias explícitas + dirección controlada + alta cohesión + bajo acoplamiento innecesario.

> **Una arquitectura profesional no elimina dependencias; administra sus direcciones y sus boundaries.**

### 25. ¿Por qué dividir el código ayuda también con AI Engineering?

El curso menciona que archivos más pequeños facilitan trabajar con asistentes de IA. La idea es válida, pero debe entenderse correctamente: no se trata simplemente de "archivo pequeño = menos tokens = mejor". El beneficio más importante es:

```text
boundary claro → contexto relevante → menos ruido →
mejor comprensión por humano y LLM → menor probabilidad de cambios accidentales
```

Es mucho más fácil pedirle a un LLM "modifica el comportamiento del `Retriever` sin alterar el contrato de `RAGService`" que entregarle un `main.py` de 3.000 líneas.

> **No debemos modularizar artificialmente solo para reducir tokens.** El diseño correcto viene primero.

### 26. Un módulo debe optimizar el contexto humano, no solamente el contexto del LLM

Un buen módulo debería permitir que un ingeniero responda rápidamente: ¿qué hace? ¿qué expone? ¿de qué depende? ¿qué puede cambiar? ¿qué no debería conocer? Si necesitamos abrir diez archivos para comprender una función trivial, posiblemente hemos fragmentado demasiado.

> **La modularización tiene un coste cognitivo.** Más módulos no significa automáticamente mejor arquitectura.

### 27. Abstraction explosion también puede ocurrir al modularizar

```text
interfaces/ protocols/ adapters/ services/ repositories/
factories/ builders/ utils/ helpers/ managers/ handlers/ providers/
```

y terminar con un sistema difícil de entender. Pregunta Senior: **¿qué problema concreto resuelve este boundary?** Si no podemos responderlo, probablemente estamos introduciendo estructura sin necesidad.

### 28. Modularización correcta vs fragmentación

**Modularización saludable:**

```text
biblioteca/
├── libros.py
├── usuarios.py
├── biblioteca.py
└── main.py
```

**Fragmentación excesiva:**

```text
biblioteca/
├── libro.py
├── libro_factory.py
├── libro_service.py
├── libro_manager.py
├── libro_helper.py
├── libro_utils.py
├── libro_validator.py
├── libro_mapper.py
└── libro_constants.py
```

No concluir que la segunda es más profesional porque tiene más archivos — puede ser exactamente lo contrario.

### 29. El criterio correcto para crear un módulo

Antes de mover una clase a un archivo: (1) ¿tiene una responsabilidad clara? (2) ¿tiene una relación fuerte con el resto del módulo? (3) ¿tiene razones de cambio relacionadas? (4) ¿reduce acoplamiento? (5) ¿mejora la navegabilidad? (6) ¿establece un boundary útil? (7) ¿evita dependencias circulares? (8) ¿hace más fácil testear o sustituir componentes? Si la respuesta es "no" a prácticamente todo: probablemente no necesitas otro módulo.

### 30. Refactorización del proyecto de biblioteca

**Antes:**

```text
main.py
├── Protocols, Libro, LibroFisico, LibroDigital
├── Usuario, Estudiante, Profesor
├── Biblioteca
└── ejecución
```

**Después:**

```text
project/
├── main.py
├── libros.py
├── usuarios.py
└── biblioteca.py
```

```text
                main.py
        ┌──────────┼──────────┐
    biblioteca   libros    usuarios
        └──────────┴──────────┘
```

La ventaja no es estética: cada parte del sistema tiene un lugar más claro.

### 31. Pero la arquitectura real puede evolucionar

Un proyecto pequeño puede empezar con la estructura plana anterior y evolucionar hacia:

```text
project/
├── src/
│   └── biblioteca/
│       ├── domain/
│       ├── application/
│       ├── infrastructure/
│       └── ...
└── tests/
```

Ese salto **no debe hacerse solo porque "así se ve más Senior"**. La estructura debe aparecer cuando el sistema tenga necesidades reales que justifiquen nuevos boundaries.

> **La arquitectura debe evolucionar con la complejidad del sistema, no adelantarse artificialmente a ella.**

Los detalles modernos de packaging, `pyproject.toml`, `src/` layout y organización arquitectónica se desarrollan en la Parte 2.

### 32. El error más común al aprender módulos

**Junior:** "La modularización consiste en mover cada clase a un archivo." → `archivo → clase`.

**Senior:** "La modularización consiste en definir boundaries que controlen cohesión, acoplamiento, dependencia y cambio." → `responsabilidad → cohesión → boundary → dependencias → dirección → cambio`.

### 33. Modelo mental definitivo de la Parte 1

```text
              ¿Qué está creciendo?
          ┌──────────┼──────────┐
    responsabilidades dependencias cambios
          └──────────┼──────────┘
                     ▼
              definir módulos
          ┌──────────┼──────────┐
       cohesión   boundaries   imports
          └──────────┼──────────┘
                     ▼
             sistema mantenible
```

La modularización no es una operación mecánica de mover código. Es una decisión de diseño.

### 34. Checklist de comprensión — Parte 1

- [ ] Qué diferencia existe entre archivo, módulo, package, clase e instancia.
- [ ] Por qué modularización y encapsulación no son lo mismo.
- [ ] Qué significa cohesión y qué significa acoplamiento.
- [ ] Por qué no debemos intentar eliminar todo acoplamiento.
- [ ] Qué significa dependency direction.
- [ ] Por qué los circular imports pueden ser un síntoma arquitectónico.
- [ ] Qué ocurre conceptualmente cuando Python ejecuta un `import`.
- [ ] Qué papel desempeñan `sys.path` y `sys.modules`, y qué es module caching.
- [ ] Qué es un import side effect y por qué un módulo no debería realizar llamadas externas inesperadas al importarse.
- [ ] Qué diferencia existe entre Entry Point y Composition Root.
- [ ] Qué función cumple `__name__ == "__main__"`.
- [ ] Qué diferencia existe entre API pública e implementación interna.
- [ ] Por qué `__init__.py` no es universalmente obligatorio.
- [ ] Por qué "un archivo = una clase" no es una regla profesional.
- [ ] Qué significa reason for change y qué es change coupling.
- [ ] Por qué modularizar no significa simplemente crear más archivos.
- [ ] Cómo conecta la modularización con Protocol, DI y Dependency Inversion de la Clase 8.
- [ ] Por qué una arquitectura más grande no es automáticamente una arquitectura mejor.

### 35. Principio de diseño para recordar

> **No se trata de repartir código entre archivos.**
>
> **Se trata de crear boundaries que hagan explícitas las responsabilidades, controlen las dependencias y reduzcan el impacto del cambio.**
>
> Un buen módulo no es simplemente pequeño: **es cohesivo, comprensible y tiene una razón clara para existir.**

## Parte 2 — Estado del arte + práctica profesional + problemas reales de producción

> **Objetivo de esta parte:** pasar de "sé separar Python en módulos" a "sé decidir una estructura de proyecto profesional y defenderla técnicamente". La pregunta ya no es **"¿en qué archivo pongo esta clase?"**, sino: **"¿qué boundary arquitectónico necesito, qué dependencia estoy introduciendo y cómo evolucionará este código cuando el sistema crezca?"**

### 36. Curso vs. práctica profesional actual

La propuesta del curso (`main.py`, `libros.py`, `usuarios.py`, `biblioteca.py`) es **correcta para enseñar modularización**, pero no debe confundirse con una arquitectura universal para producción. En un proyecto profesional real, la estructura depende de: tamaño del sistema, número de desarrolladores, dominio, tipo de aplicación, estrategia de despliegue, cantidad de integraciones, necesidades de testing, límites entre dominio e infraestructura, frecuencia y tipo de cambios.

> **No existe una estructura de carpetas "Senior" universal.** Una estructura correcta es aquella que hace explícitos los límites que el sistema realmente necesita.

### 37. Evolución razonable de un proyecto Python

```text
project/                          project/
├── main.py                       ├── biblioteca/
├── libros.py         ────►       │   ├── __init__.py
├── usuarios.py                   │   ├── libros.py
└── biblioteca.py                 │   ├── usuarios.py
                                   │   └── biblioteca.py
                                   └── main.py
```

Y cuando aparecen verdaderos boundaries:

```text
project/
├── src/
│   └── biblioteca/
│       ├── domain/
│       ├── application/
│       └── infrastructure/
└── tests/
```

```text
archivos → módulos → package → boundaries arquitectónicos
```

Pero **no todos los proyectos deben recorrer necesariamente todas estas etapas.**

### 38. `pyproject.toml`: el centro moderno de configuración del proyecto

El curso se concentra en imports y módulos. En un proyecto profesional moderno también hay que distinguir código fuente, configuración del proyecto, dependencias, herramientas, build, testing, linting, type checking. Una pieza central:

```toml
[project]
name = "biblioteca"
version = "0.1.0"
description = "Library application"
requires-python = ">=3.12"

dependencies = [
    "fastapi",
    "pydantic",
]

[dependency-groups]
dev = [
    "pytest",
    "ruff",
    "mypy",
]
```

La importancia arquitectónica es que el proyecto deja de depender de configuraciones dispersas — `pyproject.toml` funciona como un punto estandarizado de configuración para múltiples herramientas del ecosistema Python moderno.

> **Actualización profesional 2026:** `pyproject.toml` es hoy el estándar consolidado; herramientas como `uv` (gestor de paquetes y entornos escrito en Rust) han ganado adopción rápida por su velocidad frente a `pip`/`Poetry` tradicionales, aunque todas convergen en leer/escribir `pyproject.toml`.

### 39. `requirements.txt` no es lo mismo que la definición del proyecto

Un error común: "`requirements.txt` = proyecto Python". No exactamente. `requirements.txt` puede representar un conjunto de dependencias para instalación, pero el proyecto necesita además metadatos, configuración de build y configuración de tooling — por eso el ecosistema moderno usa `pyproject.toml` como archivo de configuración estándar. La elección exacta del gestor (`pip`, `uv`, Poetry, etc.) es una decisión adicional.

### 40. `src/` layout: por qué existe

```text
project/
├── pyproject.toml
├── src/
│   └── biblioteca/
│       ├── __init__.py
│       ├── libros.py
│       ├── usuarios.py
│       └── biblioteca.py
└── tests/
```

Una razón importante para introducir `src/` es evitar que los tests ejecuten accidentalmente el código directamente desde el directorio del repositorio en lugar de probar el paquete tal como sería importado después de instalarlo:

```text
repository → src/ → package
           → tests/ → import package → package instalado
```

Esto ayuda a detectar errores de packaging/importación que una estructura plana puede ocultar. Pero **`src/` no convierte automáticamente un proyecto en "Senior"** — es una herramienta para resolver problemas concretos de packaging y aislamiento del código fuente.

### 41. La diferencia crítica entre ejecutar desde el repositorio e instalar el paquete

```text
project/
├── biblioteca/
└── tests/
```

Puede ocurrir que `pytest` funcione porque el directorio actual está disponible para imports. Pero al instalar el proyecto en otro entorno aparece `ModuleNotFoundError`. Eso revela una diferencia entre "funciona desde mi checkout" y "el paquete está correctamente distribuido e importable". Los proyectos profesionales deben considerar ambos escenarios.

### 42. `main.py` vs aplicación real

En el curso, `main.py` es el entry point — correcto pedagógicamente. Pero una aplicación profesional puede tener múltiples entry points: CLI, API HTTP, worker, scheduled job, migration, consumer.

```text
src/
└── app/
    ├── api/
    ├── workers/
    ├── domain/
    └── infrastructure/
```

La arquitectura ya no gira alrededor de un único `main.py`: puede existir `API → application`, `Worker → application`, `CLI → application`, compartiendo el mismo núcleo.

### 43. El problema de "organizar por tipo"

```text
app/
├── models/
├── services/
├── repositories/
├── controllers/
├── schemas/
└── utils/
```

Puede funcionar, pero en sistemas grandes puede producir un problema: para implementar una funcionalidad hay que saltar entre `models/`, `services/`, `repositories/`, `controllers/`, `schemas/` — aumenta el coste cognitivo.

### 44. Organización por feature

```text
app/
├── users/
│   ├── models.py
│   ├── service.py
│   ├── repository.py
│   └── api.py
├── books/
│   └── (misma estructura)
└── loans/
    └── (misma estructura)
```

Ahora el cambio "agregar funcionalidad de préstamos" puede estar localizado principalmente en `loans/` — reduce el **change surface** de determinadas funcionalidades.

### 45. Organización por capas vs. organización por feature

No existe un ganador universal.

**Por capas:** `domain/ application/ infrastructure/ presentation/`. Ventaja: separación arquitectónica clara. Riesgo: una feature atraviesa muchas carpetas.

**Por feature:** `users/ books/ loans/`. Ventaja: cambios de una feature más localizados. Riesgo: puede duplicarse estructura.

**Combinación** (sistemas grandes):

```text
app/
├── users/
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── api/
├── books/
│   └── (misma estructura)
```

La decisión debe depender del dominio y de los patrones de cambio.

### 46. El criterio Senior: Change Surface

Pregunta: **¿cuántos archivos y módulos tengo que tocar para realizar un cambio normal?** Si "añadir reserva de libros" requiere modificar 12 módulos no relacionados, puede existir un problema de boundaries. Si principalmente afecta `reservations/`, el diseño puede estar mejor alineado con el dominio. No es una regla matemática; es una herramienta para detectar acoplamiento.

### 47. Import cycles: problema real de producción

```python
# biblioteca.py
from usuarios import Usuario
```

```python
# usuarios.py
from biblioteca import Biblioteca
```

```text
biblioteca → usuarios → biblioteca
```

Python puede producir `ImportError: cannot import name ...` o comportamiento relacionado con módulos parcialmente inicializados. Pero el problema profundo no es el mensaje de Python — la pregunta arquitectónica es: **¿por qué dos módulos necesitan conocerse mutuamente?** Muchas veces el ciclo revela un boundary incorrecto.

### 48. Soluciones a circular imports

**Opción 1 — corregir dependencia arquitectónica:** la mejor opción cuando el ciclo revela un diseño incorrecto (`A → B` en lugar de `A ↔ B`).

**Opción 2 — extraer un contrato:** `A → Protocol`, `B → Protocol`.

**Opción 3 — mover un concepto compartido a `common/`:** solo si existe una razón real para ese boundary.

**Opción 4 — import local:**

```python
def funcion():
    from usuarios import Usuario
```

Puede solucionar ciertos problemas de inicialización, pero **un import local no debería utilizarse como parche automático para ocultar una dependencia arquitectónica incorrecta.**

### 49. `utils.py` es una señal de alerta

```python
# utils.py
def send_email(): ...
def calculate_price(): ...
def generate_embedding(): ...
def parse_date(): ...
def connect_database(): ...
```

Esto destruye la capacidad de razonar sobre responsabilidades. Un `utils.py` gigantesco puede convertirse en un **God Module**. La solución no es prohibir todos los helpers; la pregunta es: **¿qué concepto posee realmente esta función?** `datetime_utils.py`, `embedding.py`, `pricing.py`, `email.py` pueden representar boundaries mucho más claros.

### 50. Import side effects en sistemas AI

Especialmente peligroso en AI Engineering. Evitar:

```python
# ❌
from openai import OpenAI
client = OpenAI()
model_response = client.responses.create(model="...", input="...")
```

a nivel de módulo. Una importación podría desencadenar: `import → crear cliente → configuración → llamada externa → coste → latencia`. El diseño correcto separa definición y ejecución:

```python
from openai import OpenAI


class OpenAIProvider:
    def __init__(self, client: OpenAI) -> None:
        self.client = client

    def generate(self, prompt: str) -> str:
        response = self.client.responses.create(model="...", input=prompt)
        return response.output_text
```

```python
# composition root
client = OpenAI()
provider = OpenAIProvider(client)
```

Conecta directamente con la Clase 8: `Composition + Protocol + Dependency Injection + Composition Root`.

### 51. Modularización aplicada a AI Engineering

```text
src/
└── app/
    ├── domain/
    │   ├── conversations.py
    │   └── documents.py
    ├── application/
    │   ├── rag_service.py
    │   └── agent_service.py
    ├── infrastructure/
    │   ├── llm/
    │   ├── embeddings/
    │   ├── vector_store/
    │   └── persistence/
    └── api/
        └── routes.py
```

El objetivo no es "tener carpetas bonitas" — es separar dominio → casos de uso → infraestructura → frameworks/proveedores externos.

### 52. Ejemplo: RAG

```text
RAGService
    ├── Embedder
    ├── Retriever
    ├── Reranker
    └── LLM
```

```text
app/
├── application/
│   └── rag_service.py
├── domain/
│   └── retrieval.py
└── infrastructure/
    ├── embeddings/
    │   └── openai_embedder.py
    ├── retrieval/
    │   └── pgvector_retriever.py
    └── llm/
        └── openai_provider.py
```

Cambiar los embeddings de OpenAI por otro proveedor no debería obligar a reescribir el caso de uso completo — modularización con una razón arquitectónica real.

### 53. Ejemplo: Agentic AI

```text
agent/
├── agent.py
├── tools.py
├── state.py
├── policies.py
└── memory.py
```

o, cuando el sistema crece: `domain/ application/ infrastructure/ api/`. El boundary debe responder a responsabilidades reales — un `Tool` no debería necesariamente conocer el LLM provider, la implementación de base de datos ni el framework HTTP si no los necesita.

### 54. El principio más importante: Dependency Direction

```text
application/ → infrastructure/
```

Puede ser aceptable en una aplicación pequeña. Pero si quieres sustituir PostgreSQL por MongoDB, o OpenAI por Anthropic, puede ser conveniente invertir la dependencia mediante contratos:

```text
              Protocol
              ▲      ▲
      Application  Infrastructure
```

```text
Application → Protocol ← OpenAIProvider
```

Conecta directamente con `Dependency Inversion / Dependency Injection / Ports & Adapters` de la Clase 8.

### 55. No todo debe depender de Protocol

```python
class AProtocol: ...
class BProtocol: ...
class CProtocol: ...
```

para absolutamente todo crea **abstraction explosion**. La abstracción debe existir porque hay múltiples implementaciones, necesitas sustituir infraestructura, existe un boundary importante, necesitas aislar infraestructura, testing requiere una implementación alternativa, o existe una evolución prevista razonable. No porque "un Senior usa Protocol."

### 56. Tooling profesional

```text
Ruff          → linting + formatting
Pyright/mypy  → static type checking
pytest        → testing
CI/CD         → automatización
```

```text
developer → commit → lint → type check → tests → build
```

La modularización se vuelve mucho más efectiva cuando el proyecto puede verificar automáticamente que sus boundaries siguen funcionando.

### 57. PEP 8 vs tooling moderno

PEP 8 sigue siendo referencia fundamental de estilo Python. Pero un equipo profesional no debería depender de que cada desarrollador recuerde manualmente orden de imports, formato, espaciado y estilo — Ruff automatiza gran parte de esto.

> **PEP 8 define convenciones; tooling las convierte en una política ejecutable.** Eso es mucho más importante en equipos grandes.

### 58. Type hints y boundaries

```python
class CatalogoProtocol(Protocol):
    def buscar(self, titulo: str) -> list[str]: ...
```

Un consumidor tiene información explícita sobre entrada, salida y contrato — ayuda a IDEs, type checkers, refactorizaciones, documentación, revisión de código, y agentes de IA que analizan el repositorio. El typing no sustituye el diseño, pero hace que el diseño sea más verificable.

### 59. Modularización y testing

```text
tests/
├── test_libros.py
├── test_usuarios.py
└── test_biblioteca.py
```

En aplicaciones grandes conviene que los tests sigan también los boundaries reales:

```text
tests/
├── unit/         → componente aislado
├── integration/  → varios componentes reales
└── e2e/          → sistema completo
```

La modularización facilita decidir qué parte necesita qué nivel de prueba.

### 60. Problema real: import funciona localmente pero falla en producción

**Causa:** el entorno de desarrollo tiene una ruta de importación que producción no tiene. **Síntoma:** `ModuleNotFoundError` solo después del deployment. **Diagnóstico:** verificar package instalado, working directory, Python interpreter, virtual environment, `sys.path`, build configuration. **Solución:** definir correctamente package, `pyproject.toml`, build, installation, y ejecutar tests en un entorno limpio, idealmente en CI. **Prevención:** no depender accidentalmente del checkout local.

### 61. Problema real: circular imports

**Causa:** `A → B`, `B → A`. **Síntomas:** `ImportError`, `AttributeError`, "partially initialized module". **Diagnóstico:** mapear el grafo de dependencias. **Solución:** revisar dependency direction, boundary, responsabilidad; extraer contratos o mover conceptos cuando corresponda. **Prevención:** evitar que los módulos se conozcan innecesariamente entre sí.

### 62. Problema real: módulo con side effects

**Causa:** código ejecutable a nivel de módulo (`database = connect()`, `client = ExternalClient()`, `load_configuration()`). **Síntomas:** tests lentos, imports frágiles, errores de inicialización, llamadas externas inesperadas, problemas de configuración. **Solución:** separar `definition` de `execution/initialization`, centralizando la construcción de infraestructura cuando corresponda.

### 63. Problema real: módulo demasiado grande

**Síntomas:** cualquier cambio toca el mismo archivo, o nadie sabe dónde agregar una nueva funcionalidad. **Causa:** el módulo acumula múltiples razones de cambio. **Solución:** identificar responsabilidades, cohesión, change coupling, y dividir solamente cuando exista un boundary útil.

### 64. Problema real: demasiados módulos pequeños

El extremo contrario: 100 archivos no significa arquitectura excelente. **Síntomas:** navegar el proyecto requiere demasiados saltos, clases de una sola función sin necesidad, wrappers innecesarios, interfaces artificiales, abstracciones sin consumidores alternativos. **Solución:** consolidar unidades que siempre cambian juntas, tienen la misma razón de cambio, forman un concepto cohesivo, o no necesitan un boundary independiente.

### 65. Problema real: `main.py` como God Object

**Síntomas:** `main.py` con negocio + DB + API + LLM + configuración + logging + orchestration. **Solución:** extraer responsabilidades — el entry point debe iniciar la aplicación, no contener toda la aplicación.

### 66. Problema real: dependencia directa de SDKs

```python
class AIService:
    def answer(self, prompt: str):
        client = OpenAI()
        ...
```

```text
AIService → OpenAI SDK
```

El consumidor conoce infraestructura concreta. Alternativa:

```python
class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...

class AIService:
    def __init__(self, provider: LLMProvider) -> None:
        self.provider = provider
```

```text
AIService → LLMProvider ← OpenAIProvider
```

Combina modularización + Protocol + Dependency Inversion + Dependency Injection.

### 67. Qué cambia realmente respecto al curso

| Curso | Práctica profesional |
|---|---|
| Separar clases en archivos | Definir boundaries con intención |
| `main.py` como entry point | Entry points pueden ser múltiples |
| Imports organizados | Imports + dependency direction |
| Módulos | Módulos + packages + packaging |
| `__init__.py` | Comprender packages y namespace packages |
| Archivos pequeños | Cohesión y cambio |
| PEP 8 | PEP 8 + tooling automatizado |
| Separar responsabilidades | Controlar change surface |
| Evitar código en `usuarios.py` | Controlar import side effects |
| Proyecto pequeño | Estructura que evoluciona con el sistema |

La diferencia entre ambos niveles no es "más sintaxis". Es **criterio de diseño**.

### 68. Regla de decisión para un Senior

Antes de crear **un nuevo archivo**: ¿qué responsabilidad estoy separando? Antes de crear **un nuevo package**: ¿qué boundary arquitectónico estoy creando? Antes de crear **un `Protocol`**: ¿qué dependencia quiero desacoplar? Antes de crear **`service.py`**: ¿qué responsabilidad tiene realmente este servicio? Antes de crear **`utils.py`**: ¿de qué concepto es realmente esta función? Este patrón de preguntas importa mucho más que memorizar estructuras de carpetas.

### 69. Checklist profesional de Parte 2

- [ ] Sé explicar por qué `pyproject.toml` es relevante en proyectos Python modernos.
- [ ] Sé diferenciar código fuente, package y configuración del proyecto.
- [ ] Entiendo por qué existe `src/` layout.
- [ ] Sé explicar el riesgo de depender accidentalmente del checkout local.
- [ ] Distingo Entry Point de Composition Root.
- [ ] Entiendo por qué `main.py` no debe convertirse en God Object.
- [ ] Sé comparar organización por capas y por feature.
- [ ] Sé utilizar change surface como criterio de diseño.
- [ ] Entiendo qué revela un circular import y cómo se corrige (sin esconderlo con un import local).
- [ ] Entiendo por qué `utils.py` puede convertirse en un God Module.
- [ ] Sé explicar los riesgos de import side effects.
- [ ] Puedo relacionar modularización con Dependency Inversion.
- [ ] Puedo explicar cuándo un Protocol aporta valor y cuándo genera abstraction explosion.
- [ ] Entiendo cómo la modularización afecta testing, RAG y Agentic AI.
- [ ] Entiendo que PEP 8 debe complementarse con tooling automatizado.
- [ ] Puedo evaluar una estructura de proyecto sin asumir que existe una única arquitectura correcta.

### 70. Regla final de la Parte 2

> **No se trata de tener una estructura de carpetas que parezca profesional.**
>
> **Se trata de diseñar módulos cuyos límites correspondan a responsabilidades, dependencias y cambios reales del sistema.**
>
> Un proyecto Senior no es el que tiene más carpetas: es el que permite **predecir el impacto de un cambio antes de hacerlo**.

## Parte 3 — Entrevista Senior/Staff + ecosistema AI + casos reales

> En una entrevista Senior, el entrevistador no busca que recuerdes `from x import y`. Busca saber si puedes controlar **acoplamiento, cohesión, dependency direction, change surface, testabilidad y evolución arquitectónica**.

### 71. Banco de preguntas de entrevista Senior/Staff sobre modularización

### 71.1 ¿Por qué dividir un proyecto Python en módulos?

**Qué evalúa:** si entiendes cohesión, acoplamiento, responsabilidades, change surface, mantenibilidad y dependency direction — no si sabes crear archivos `.py`.

**Respuesta de alto impacto:**
> "Dividir un proyecto en módulos permite establecer boundaries explícitos entre responsabilidades. El objetivo no es reducir el número de líneas por archivo, sino aumentar la cohesión y controlar el acoplamiento. Una buena modularización hace que los cambios tengan un alcance predecible, facilita testing y permite sustituir implementaciones sin propagar cambios innecesarios por el sistema."

Respuesta aún más fuerte:
> "No considero que 'un archivo por clase' sea una regla arquitectónica. Prefiero agrupar código por cohesión y razón de cambio. Si dos componentes siempre cambian juntos y representan el mismo concepto, separarlos artificialmente puede aumentar el coste cognitivo."

**Red flag:** "Porque los archivos grandes son malos" (demasiado superficial).

### 71.2 ¿Un archivo debe contener una sola clase?

**Respuesta Senior:**
> "No. Un módulo debe representar una responsabilidad o conjunto cohesivo de conceptos. Puede contener varias clases relacionadas. Separaría clases cuando exista una razón de diseño: diferente responsabilidad, diferentes dependencias, diferente ciclo de cambio o necesidad de un boundary independiente."

**Red flag:** "Sí, siempre una clase por archivo" — no es una regla de Python ni de arquitectura.

### 71.3 ¿Qué diferencia existe entre modularización y encapsulación?

**Respuesta de alto impacto:**
> "La modularización organiza el sistema en unidades con boundaries y dependencias explícitas. La encapsulación controla qué detalles internos quedan expuestos por una unidad. Puedo modularizar sin encapsular correctamente, por ejemplo creando muchos archivos pero exponiendo todos sus detalles internos."

### 71.4 ¿Qué es un módulo en Python?

**Respuesta:**
> "Un módulo es un archivo Python que puede ser importado como una unidad. Contiene definiciones y puede mantener estado a nivel de módulo. Cuando Python importa un módulo, ejecuta su código de nivel superior y mantiene el módulo cargado en `sys.modules`, lo que permite reutilizar esa instancia del módulo durante la vida del proceso."

### 71.5 ¿Qué ocurre internamente cuando haces `import usuarios`?

**Respuesta Senior:**

```text
import usuarios → Python busca el módulo → encuentra/carga el módulo →
crea su objeto módulo → ejecuta código top-level →
registra módulo en sys.modules → lo deja disponible para imports posteriores
```

Por eso un módulo no debe ejecutar accidentalmente operaciones peligrosas al importarse (ej. `client = OpenAI(); response = client.responses.create(...)` a nivel de módulo).

### 71.6 ¿Por qué existe `sys.modules`?

**Respuesta de alto impacto:**
> "`sys.modules` es el registro de módulos que el intérprete ya ha cargado. Python lo utiliza para evitar volver a cargar y ejecutar innecesariamente un módulo en cada importación."

**Implicación importante:** el estado mutable a nivel de módulo puede persistir durante la vida del proceso — por eso los globals de módulo deben utilizarse conscientemente.

### 71.7 ¿Qué es un circular import y qué significa arquitectónicamente?

**Respuesta Senior:**
> "El problema inmediato es de resolución e inicialización de módulos, pero arquitectónicamente un circular import suele indicar una dependency direction problemática. No me limitaría a esconderlo con imports locales; primero investigaría por qué ambos módulos necesitan conocerse."

**Red flag:** "Lo arreglo moviendo el import dentro de la función" (puede funcionar técnicamente, pero no necesariamente corrige el diseño).

### 71.8 ¿Cuándo usarías un import local?

**Respuesta:**
> "Puede ser válido cuando existe una razón concreta, por ejemplo evitar una dependencia costosa o diferir una dependencia opcional. Pero no lo utilizaría como mecanismo predeterminado para esconder circular dependencies. Si el ciclo representa una dependencia arquitectónica incorrecta, prefiero corregir el boundary."

### 71.9 ¿Qué problema resuelve `src/` layout?

**Respuesta:**
> "El `src` layout ayuda a evitar que el código se importe accidentalmente directamente desde el checkout del repositorio. Obliga a que el paquete se trate como un paquete instalado, lo que permite detectar antes ciertos errores de packaging e importación."

No responder: "`src` es la estructura profesional obligatoria" — no lo es.

### 71.10 ¿`src/` mejora el rendimiento?

**Respuesta:**
> "No es su propósito. `src/` es principalmente una decisión de layout y packaging. Su valor está en hacer más explícita la diferencia entre el código fuente del repositorio y el paquete instalado."

### 71.11 ¿Qué es `pyproject.toml` y por qué importa?

**Respuesta Senior:**
> "Es el estándar moderno de configuración del proyecto Python. Puede centralizar metadatos del proyecto, dependencias y configuración utilizada por tooling y sistemas de build. Su valor arquitectónico está en hacer reproducible y explícita la definición del proyecto."

**Red flag:** "`pyproject.toml` reemplaza completamente a todos los demás archivos de Python" (demasiado absoluto).

### 71.12 ¿Cómo decidirías entre organización por capas y por features?

**Respuesta de alto impacto:**
> "Observaría principalmente cómo cambia el sistema. Si una funcionalidad requiere modificar muchas capas distribuidas, una organización por feature puede reducir el change surface. Si existen boundaries arquitectónicos fuertes entre dominio, aplicación e infraestructura, una estructura por capas puede ser más apropiada. También pueden combinarse."

### 71.13 ¿Qué es cohesion y qué es coupling?

**Respuesta:**
> "Cohesión mide qué tan relacionadas están las responsabilidades dentro de una unidad — busco alta cohesión. Coupling representa cuánto depende una unidad de otras unidades; un buen diseño no intenta eliminar todo coupling —eso sería imposible— sino controlar su dirección, naturaleza y coste de cambio."

> **Senior insight:** no buscas cero acoplamiento; buscas acoplamiento controlado y en la dirección correcta.

### 71.14 ¿Qué es change coupling?

**Respuesta:**
> "Change coupling ocurre cuando componentes diferentes tienden a modificarse juntos. Si para implementar una sola funcionalidad tengo que cambiar constantemente cinco módulos aparentemente independientes, puede existir una mala separación de responsabilidades o un boundary incorrecto."

### 71.15 ¿Qué es un God Module?

**Respuesta:**
> "Es un módulo que acumula demasiadas responsabilidades y se convierte en punto central de cambios, dependencias y conocimiento — un ejemplo típico es un `utils.py` que termina conteniendo lógica de negocio, acceso a base de datos, integración con APIs y helpers no relacionados."

**Solución:** no consiste simplemente en convertir `utils.py` en 10 archivos — la solución correcta es identificar responsabilidades y boundaries reales.

### 71.16 ¿Qué es abstraction explosion?

**Respuesta:**
> "Es la proliferación de abstracciones que no aportan un beneficio real. Cada interface o Protocol añade coste de diseño, testing, documentación, wiring y mantenimiento. Abstraería cuando existe una razón concreta: múltiples implementaciones, boundary arquitectónico, sustitución de infraestructura o necesidad real de desacoplamiento."

### 71.17 ¿Dónde pondrías un `Protocol`?

**Respuesta Senior:** no existe una única ubicación — depende de quién sea dueño del contrato:

```python
# application/ports.py
from typing import Protocol

class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...
```

```text
Application → LLMProvider ← OpenAIProvider (infrastructure/openai_provider.py)
```

La idea importante es que la capa que necesita la capacidad no tenga que conocer el SDK concreto.

### 71.18 ¿Por qué no crear un `Protocol` para cada clase?

**Respuesta:**
> "Porque una abstracción también es una dependencia que debemos mantener. Si solamente existe una implementación y no hay una razón arquitectónica para sustituirla o aislarla, el Protocol puede introducir más complejidad que valor."

### 71.19 ¿Qué relación tiene modularización con Dependency Inversion?

**Respuesta:**
> "La modularización define unidades y límites. Dependency Inversion ayuda a controlar hacia dónde apuntan las dependencias. Una arquitectura modular no necesariamente tiene dependency direction correcta; puedo tener veinte módulos perfectamente separados que siguen dependiendo directamente de infraestructura concreta."

### 71.20 Caso de entrevista: SDK de OpenAI

> "Tu servicio utiliza directamente el SDK de OpenAI. Mañana producto quiere soportar otro proveedor. ¿Qué harías?"

**Respuesta débil:** `if provider == "openai": ... elif provider == "anthropic": ...`

**Respuesta Senior:**
> "Primero determinaría si realmente existe una necesidad de múltiples proveedores. Si la existe, definiría una capacidad estable requerida por la aplicación, por ejemplo `LLMProvider`, y aislaría los SDKs concretos detrás de adapters o providers. La aplicación dependería del contrato y el composition root decidiría la implementación concreta."

```text
                 Composition Root
              ┌──────────┴──────────┐
      OpenAIProvider        AnthropicProvider
              └──────────┬──────────┘
                    LLMProvider
                         ▲
                     AIService
```

### 71.21 Caso de entrevista: modularizar un RAG production-ready

**Respuesta de alto impacto:** no responder simplemente `rag.py` — considerar:

```text
application/rag_service.py
domain/retrieval.py
infrastructure/embeddings/ | vector_store/ | reranking/ | llm/
```

```text
              RAG Application
       ┌────────────┼────────────┐
   Embedder      Retriever     LLM
       ▲            ▲            ▲
   Provider      Vector DB     Provider
```

La decisión importante es aislar los detalles que probablemente cambien: modelo de embeddings, vector database, reranker, LLM provider.

### 71.22 Caso de entrevista: Agent sin acoplarse a todos los proveedores

```text
Agent
 ├── Model
 ├── Tool interface
 ├── State interface
 ├── Memory interface
 └── Policy / Guardrails
```

```text
infrastructure/ → llm/ | tools/ | persistence/ | telemetry/
```

El agente debería depender de las capacidades que necesita, no necesariamente del SDK de OpenAI, del cliente de PostgreSQL, del cliente de Redis, ni de un SDK de telemetría específico.

### 71.23 ¿Modularizarías por clase, por dominio o por infraestructura?

**Respuesta:**
> "Por el boundary que reduzca mejor el acoplamiento y represente una unidad cohesiva. En sistemas pequeños puede ser suficiente separar módulos por concepto. En sistemas grandes probablemente utilizaría boundaries de dominio, aplicación e infraestructura. No empezaría con una arquitectura compleja si el problema todavía no existe." (YAGNI + evolutividad + criterio arquitectónico)

### 71.24 ¿Qué harías si una refactorización modular rompe 40 imports?

**Respuesta Senior:** no mover archivos manualmente hasta que los errores desaparezcan. Primero: (1) identificar dependency graph, (2) identificar dependencias públicas, (3) detectar circular dependencies, (4) identificar imports internos vs API pública, (5) establecer nuevo boundary, (6) migrar progresivamente, (7) ejecutar tests + type checking + linting. La refactorización debe ser una operación controlada.

### 71.25 ¿Cómo sabes que tu modularización mejoró realmente el sistema?

**Respuesta de alto impacto:** no "porque ahora se ve más ordenado" — buscar evidencia: menor change surface, menor acoplamiento, tests más localizados, menos dependencias circulares, menor conocimiento requerido para cambiar una feature, sustitución más sencilla de infraestructura, límites más claros, menor cantidad de regresiones.

### 71.26 Pregunta Staff: "¿Cuándo NO modularizar?"

**Respuesta:**
> "Cuando la separación no representa una responsabilidad o boundary real y solo aumenta el coste cognitivo. También evitaría abstraer o fragmentar anticipadamente por escenarios hipotéticos. La arquitectura debe responder a problemas actuales o a evolución razonablemente previsible."

Esta pregunta separa bastante bien a un Senior engineer de un developer que memoriza patrones.

### 72. Errores de candidatos que parecen Senior pero no lo son

1. "Siempre usaría Clean Architecture" — no toda aplicación necesita la misma complejidad.
2. "Siempre usaría Protocol" — abstraction explosion.
3. "Cada clase debe tener su archivo" — confunde organización con arquitectura.
4. "Si hay circular import, uso import local" — puede esconder el síntoma sin corregir el boundary.
5. "`src/` es obligatorio" — confunde una estrategia de packaging con una regla universal.
6. "Un buen proyecto tiene muchas carpetas" — la complejidad estructural también tiene coste.
7. "Desacoplamiento significa cero dependencias" — un sistema sin dependencias útiles no existe.

### 73. Mapa de conocimientos para entrevista

```text
                    Python Modules
          ┌──────────────┼──────────────┐
       Imports        Packages      Packaging
          ▼              ▼              ▼
    sys.modules      Boundaries    pyproject.toml
          ▼
    Import Side Effects
          ▼
    Circular Imports
          ▼
 Dependency Direction
     ┌────┴────┐
  Protocol     DI
     └────┬────┘
 Composition Root
          ▼
 Ports & Adapters
     ┌────┴─────────────┐
    RAG              Agents
     ▼                  ▼
Providers          Tools / Memory
Vector DB          State / Models
LLM                Guardrails
```

### 74. Relación directa con el ecosistema AI

**Python modules → FastAPI:** una API profesional separa `api/ application/ domain/ infrastructure/`; FastAPI pertenece al boundary de entrada HTTP, no al dominio.

**Python modules → RAG:** `application/rag_service.py` + `infrastructure/{embeddings, vector_store, llm}/` — la modularización aísla proveedores concretos.

**Python modules → Agents:** `application/agent_service.py` + `domain/agent_state.py` + `infrastructure/{models, tools, persistence}/`.

**Python modules → MCP:** `agent/client.py` + `mcp/protocol.py` + `infrastructure/servers/` — mantiene aislados los detalles del protocolo y de cada integración.

**Python modules → LLM Providers:** `application/ai_service.py` + `domain/contracts.py` + `infrastructure/{openai_provider, anthropic_provider, gemini_provider}.py` — la aplicación conoce `LLMProvider` pero no necesariamente los SDKs de OpenAI, Anthropic o Google.

### 75. Caso real de diseño — sustitución de proveedor

**Problema:** `AIService → OpenAI SDK`, distribuido por múltiples módulos; cambiar proveedor requiere modificar `AIService`, RAG, Agent, API y tests.

**Solución:** crear un boundary `AIService → LLMProvider ← OpenAIProvider`.

**Beneficio:** el cambio de infraestructura queda localizado.

**Condición:** el contrato debe representar una capacidad común real — si OpenAI y otro proveedor tienen capacidades radicalmente diferentes, forzar ambos dentro de un contrato artificial puede producir una abstracción defectuosa.

### 76. Caso real — modularización de un sistema RAG

**Problema:** un único `rag.py` contiene PDF parsing, chunking, embedding, vector DB, reranking, prompt construction, LLM call, logging, evaluation. **Síntoma:** cambiar el vector database rompe partes no relacionadas.

**Refactor:**

```text
rag/
├── application/service.py
├── domain/retrieval.py
└── infrastructure/{embeddings, vector_store, reranking, llm}/
```

**Beneficio:** cada componente tiene un boundary más claro y los cambios pueden localizarse.

### 77. Caso real — Agent con múltiples entry points

```text
HTTP ─┐
CLI ──┼──► AgentService ──► [Model, Tools, State]
Worker┘
```

Todos invocan `AgentService`, evitando duplicar lógica de negocio en cada entry point.

### 78. El criterio Staff Engineer al revisar un proyecto

No preguntar solo "¿está bien organizado?" — hacer preguntas más difíciles: ¿qué cambia junto? ¿qué debería poder cambiar independientemente? ¿quién conoce a quién? ¿hacia dónde apuntan las dependencias? ¿dónde está el boundary? ¿quién es dueño del contrato? ¿qué ocurre durante import? ¿hay side effects? ¿hay ciclos? ¿la estructura refleja el dominio? ¿estoy introduciendo abstracciones sin necesidad? ¿cuál es el coste cognitivo de esta estructura?

### 79. Checklist de dominio para entrevista

- [ ] Qué es un módulo y qué ocurre durante `import`.
- [ ] Qué papel cumple `sys.modules`.
- [ ] Por qué existen los import side effects.
- [ ] Qué problema representa un circular import.
- [ ] Cuándo un import local es válido.
- [ ] Qué problema intenta resolver `src/`.
- [ ] Qué papel tiene `pyproject.toml`.
- [ ] Diferencia entre módulo, package y proyecto.
- [ ] Diferencia entre modularización y encapsulación.
- [ ] Diferencia entre cohesión y coupling.
- [ ] Qué significa change coupling y qué es change surface.
- [ ] Qué es un God Module y por qué `utils.py` puede convertirse en uno.
- [ ] Por qué no todo necesita un `Protocol`.
- [ ] Cómo modularizar un RAG y un Agent.
- [ ] Cómo aislar SDKs de proveedores.
- [ ] Cómo relacionar módulos con Dependency Inversion.
- [ ] Cómo defender una decisión de arquitectura ante un Staff Engineer.

### 80. Regla final de la Parte 3

> **No se trata de saber dónde colocar archivos.**
>
> **Se trata de saber dónde colocar responsabilidades y cómo controlar las dependencias entre ellas.**
>
> Un ingeniero Senior no pregunta únicamente **"¿dónde pongo esta clase?"**.
>
> Pregunta: **"¿qué cambio quiero aislar, quién debe conocer esta decisión y cuál debe ser la dirección de la dependencia?"**

## Parte 4 — Recursos oficiales + glosario técnico

> **Objetivo:** cerrar la clase con recursos que aportan profundidad profesional real y un glosario que prepare el terreno para Python avanzado, APIs, RAG, Agents, MCP y LLMOps. No se incluyen recursos por cantidad — se priorizan fuentes primarias.

### 81. Recursos oficiales — prioridad máxima

**Python — Modules (tutorial oficial) — Prioridad ★★★★★.** https://docs.python.org/3/tutorial/modules.html — Explica el modelo de módulos de Python, imports y el mecanismo de reutilización de código entre archivos. Tras estudiarlo debes poder responder: ¿qué es un módulo? ¿qué significa importarlo? ¿qué sucede cuando Python encuentra un `import`? ¿diferencia entre `import x` y `from x import y`? ¿qué papel cumple `__name__`? ¿qué significa ejecutar un módulo como script?

**Python — The import system (referencia del lenguaje) — Prioridad ★★★★★.** https://docs.python.org/3/reference/import.html — Una de las lecturas más importantes de toda la clase para superar el nivel intermedio: `import → module discovery → module loading → module initialization → sys.modules`. Un Senior no debería pensar en `from usuarios import Usuario` como una instrucción mágica, sino entender que existe un sistema de búsqueda, carga, inicialización y caching.

**Python — `sys.modules` — Prioridad ★★★★★.** https://docs.python.org/3/library/sys.html#sys.modules — Registro de módulos cargados; fundamental para entender module caching, circular imports y estado global a nivel de módulo. **Conocimiento Senior:** un import no es simplemente "leer otro archivo" — Python administra módulos como objetos dentro del proceso.

**Python — Packages — Prioridad ★★★★★.** https://docs.python.org/3/tutorial/modules.html#packages — Dominar `module / package / subpackage / import path`. La diferencia entre módulo y package será importante cuando el proyecto pase de ejercicio educativo a aplicación real.

**Python Packaging User Guide — Prioridad ★★★★★.** https://packaging.python.org/ — Fuente clave porque eventualmente los módulos deben convertirse en software instalable y distribuible: `package → distribution package → dependencies → build → installation → pyproject.toml`. No es necesario convertir esta clase en un curso completo de packaging; ese conocimiento pertenece a una capa posterior.

**`pyproject.toml` — Prioridad ★★★★★.** https://packaging.python.org/en/latest/specifications/pyproject-toml/ — Comprender el papel arquitectónico del archivo (`project metadata`, `dependencies`, `build configuration`, `tool configuration`), sin memorizar todas las claves.

**PEP 8 — Style Guide for Python Code — Prioridad ★★★★☆.** https://peps.python.org/pep-0008/ — Para esta clase interesan especialmente: import organization, naming, module structure, whitespace, readability, consistency. **Importante:** PEP 8 no es una ley del lenguaje ni una regla absoluta para cualquier proyecto — es una guía de estilo.

**PEP 420 — Implicit Namespace Packages — Prioridad ★★★☆☆.** https://peps.python.org/pep-0420/ — Corrige la simplificación frecuente "package = carpeta con `__init__.py`"; un package moderno no siempre lo requiere.

**Ruff — Prioridad ★★★★★.** https://docs.astral.sh/ruff/ — Tooling automatizado para linting, formatting e import sorting. Permite pasar de "yo ordeno los imports manualmente" a `CI/CD → Ruff → quality gate`. **Principio profesional:** las convenciones repetitivas deben automatizarse cuando una herramienta puede verificarlas de forma determinista.

**mypy — Prioridad ★★★★☆.** https://mypy.readthedocs.io/ — Los type hints no convierten automáticamente a Python en un lenguaje estáticamente tipado; son información que herramientas como mypy pueden analizar (`type hints → static analysis → early defect detection`).

**pytest — Prioridad ★★★★★.** https://docs.pytest.org/ — La modularización tiene un beneficio fundamental: módulos mejor separados → dependencias más localizadas → tests más focalizados. Esta clase no debería estudiarse aislada del testing.

**Git — Prioridad ★★★★★.** https://git-scm.com/docs — Una refactorización de `main.py` hacia módulos separados es una operación que debe poder revisarse mediante control de versiones; un Senior debe distinguir refactor estructural de cambio funcional y mantener commits y diffs suficientemente claros para revisar el cambio.

> **Actualización profesional 2026:** el trío `Ruff + mypy/pyright + pytest` sigue siendo el estándar de facto en proyectos Python profesionales; `uv` se ha consolidado como gestor de dependencias/entornos de referencia por velocidad, leyendo la misma configuración de `pyproject.toml`.

### 82. Recursos que NO necesitas estudiar todavía

Para mantener el principio de alto ROI, no conviertas esta clase en un curso completo de packaging. Todavía no hace falta profundizar exhaustivamente en: build backends, wheels, sdists, dependency resolvers, publishing a PyPI, plugins de packaging, internals completos de `importlib`, implementación del import machinery, ni namespace packages avanzados. Son conocimientos válidos, pero pertenecen a otras clases o a una profundización posterior.

### 83. Orden recomendado de estudio

```text
1. Python Modules
2. Python Import System
3. sys.modules
4. Packages
5. PEP 8
6. Ruff
7. Packaging / pyproject.toml
8. Mypy
9. pytest
```

No hace falta estudiar todo con la misma profundidad. La mayor prioridad para esta clase es: **Modules, Imports, Packages, Dependency Direction, Cohesion/Coupling.**

### 84. Glosario técnico

**Module.** Unidad de código Python importable, normalmente representada por un archivo `.py`. No confundir el archivo físico con el objeto módulo cargado por Python.

**Package.** Unidad organizativa que permite agrupar módulos relacionados dentro de una estructura importable.

**Subpackage.** Package contenido dentro de otro package (ej. `infrastructure/llm/` como subpackage de `infrastructure`).

**Import System.** Conjunto de mecanismos mediante los cuales Python encuentra, carga e inicializa módulos y packages.

**`sys.modules`.** Registro mantenido por Python de módulos cargados durante la ejecución del proceso.

**Module Caching.** Mecanismo por el cual Python reutiliza módulos ya cargados en lugar de tratarlos como una nueva carga completa cada vez.

**Import Side Effect.** Efecto producido simplemente por importar un módulo (ej. `connect_to_production_database()` a nivel de módulo, que se ejecuta con solo hacer `import database`).

**Circular Import.** Situación donde los módulos dependen directa o indirectamente unos de otros formando un ciclo (`A → B → A`). No es únicamente un problema sintáctico — puede revelar un problema de arquitectura.

**Cohesion.** Grado en que las responsabilidades de un módulo pertenecen conceptualmente juntas. Objetivo general: alta cohesión.

**Coupling.** Grado de dependencia entre componentes. El objetivo profesional no es "zero coupling" sino "coupling controlado".

**Change Coupling.** Situación donde componentes aparentemente separados necesitan modificarse juntos de manera recurrente — señal útil para evaluar boundaries.

**Change Surface.** Conjunto de componentes que deben modificarse para implementar un cambio. Un buen diseño evita que una modificación pequeña tenga una superficie de cambio innecesariamente grande.

**Dependency Direction.** Dirección en la que se propagan las dependencias entre módulos o capas. La dirección importa tanto como la existencia de la dependencia.

**Boundary.** Límite conceptual entre responsabilidades o componentes; permite controlar conocimiento, dependencias, cambios y contratos.

**Public API del módulo.** Conjunto de elementos que otros módulos pueden consumir como interfaz estable. No todo lo que existe dentro de un módulo debería considerarse parte de su API pública.

**God Module.** Módulo que acumula demasiadas responsabilidades y se convierte en un punto central de conocimiento y cambio (ej. un `utils.py` que termina conteniendo prácticamente todo).

**Abstraction Explosion.** Creación excesiva de abstracciones sin suficiente beneficio arquitectónico. El coste incluye interfaces + implementations + DI + tests + documentation + maintenance.

**Composition Root.** Lugar donde se construye el grafo de dependencias de la aplicación. La aplicación consume abstracciones; el composition root decide implementaciones.

**Protocol.** Mecanismo de typing estructural que permite definir un contrato basado en las operaciones que un objeto proporciona. No significa "todas las clases deben heredar de Protocol".

**Adapter.** Componente que adapta una interfaz externa a la interfaz que espera nuestra aplicación — útil para aislar SDKs, APIs externas, legacy systems, librerías de terceros.

**Port.** Contrato que representa una capacidad requerida o proporcionada por una parte de la aplicación (`Application → Port ← Adapter`).

### 85. Mapa mental final de la clase

```text
                    PYTHON PROJECT
                          ▼
                       MODULES
              ┌───────────┴───────────┐
           Imports                 Packages
              ▼                       ▼
       Import System             Boundaries
       ┌──────┼──────┐
 sys.modules  Side   Circular
              effects imports
              ▼
   Dependency Direction
              ▼
    Cohesion + Coupling
              ▼
       Change Surface
              ▼
        Architecture
     ┌─────┼──────────────┐
    RAG   Agents          APIs
     ▼      ▼              ▼
    LLM   Tools          FastAPI
    DB    Memory
          MCP
```

### 86. Lo que debes llevarte a las siguientes clases

```text
Clase 8: Composition → Dependency Injection → Protocol → Dependency Inversion
Clase 9: Modules → Packages → Imports → Dependency Direction → Boundaries
Próximas capas: Exceptions → APIs → Persistence → RAG → Agents → MCP → LLMOps
```

La conexión importante: estás aprendiendo progresivamente a construir sistemas donde responsabilidades → boundaries → contracts → dependencies → implementations están explícitamente controlados.

### 87. Regla final de la Clase 9

> **No se trata de dividir `main.py` en muchos archivos.**
>
> **Se trata de diseñar boundaries que hagan explícitas las responsabilidades y controlen la dirección del cambio.**
>
> Un proyecto modular no es el que tiene más carpetas.
>
> Es el que permite que un cambio tenga un impacto **predecible, localizado y revisable**.

---

### 88. Checklist final de dominio — Clase 9 completa

- [ ] Distingo archivo, módulo, package, clase e instancia con precisión.
- [ ] Sé por qué modularización y encapsulación son conceptos distintos y complementarios.
- [ ] Puedo explicar cohesión, acoplamiento, change coupling y change surface, y usarlos como criterio de diseño.
- [ ] Entiendo el proceso interno de un `import` (resolución, carga, `sys.modules`, module caching).
- [ ] Sé identificar y prevenir import side effects, especialmente en integraciones con proveedores de IA.
- [ ] Sé diagnosticar un circular import y elegir entre corregir el boundary, extraer un contrato, mover un concepto compartido o usar un import local con justificación.
- [ ] Distingo Entry Point de Composition Root y sé por qué `main.py` no debe convertirse en God Object.
- [ ] Sé comparar organización por capas, por feature, y su combinación, usando change surface como criterio.
- [ ] Entiendo el papel de `pyproject.toml`, `src/` layout y por qué ninguno es obligatorio universalmente.
- [ ] Sé cuándo un `Protocol` aporta valor en un boundary y cuándo genera abstraction explosion.
- [ ] Puedo aplicar estos principios a la modularización de sistemas RAG, Agentic AI y multi-entry-point.
- [ ] Puedo defender una decisión de estructura de proyecto ante un Staff Engineer, incluyendo qué NO responder.
- [ ] Sé identificar God Modules, dependencia directa de SDKs y fragmentación excesiva en revisión de código.

### 89. Regla final absoluta de la Clase 9

> **No se trata de saber usar `import`.**
>
> **Se trata de diseñar cómo las partes de un sistema se conocen entre sí, en qué dirección, y qué le cuesta al sistema cambiar mañana.**
>
> Un módulo no es una carpeta. Es una decisión sobre dónde vive una responsabilidad y quién tiene permiso de depender de ella.