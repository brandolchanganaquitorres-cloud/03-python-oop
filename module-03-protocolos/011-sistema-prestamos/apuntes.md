# CLASE MAGISTRAL — SISTEMA DE PRÉSTAMOS EN PYTHON
## Excepciones, validaciones y flujo de negocio: de ejercicio a sistema de producción y AI Engineering

> **Nivel objetivo:** Senior AI Engineer · Senior Software Engineer · Tech Lead · Staff Engineer
> **Anclaje de versión:** Python 3.14 (estable) · estado del arte revisado al **30 de agosto de 2026**
> **Idioma:** español; APIs, keywords y términos oficiales se conservan en su forma original.
> **Fuente:** clase del curso (módulos, `Biblioteca`, `buscar_usuario`, `UsuarioNoEncontrado`, flujo de préstamo) + Partes 1–4 previas, fusionadas sin eliminar contenido y sin repetir explicaciones.

---

## Cómo leer este documento

| Etiqueta | Significado |
|---|---|
| **Corrección técnica** | Algo del material fuente que conviene corregir o precisar |
| **Ampliación profesional** | Profundización necesaria para comprender bien el tema |
| **Actualización 2026** | Práctica o cambio del lenguaje/industria vigente a agosto 2026 |
| **[Verificar]** | Dato sensible al tiempo: confirmar en la fuente oficial antes de citarlo en una entrevista o decisión |

**Mapa del documento**

```text
PARTE I    Fundamentos y explicación técnica (contenido del curso, reorganizado)
PARTE II   Código de referencia (PEP 8, type hints, pedagógico → profesional)
PARTE III  Producción: persistencia, concurrencia, transacciones, idempotencia, errores por capa
PARTE IV   Estado del arte 2026: Python 3.14, async, tooling, AI/Agents/MCP, seguridad, regulación
PARTE V    Arquitectura, System Design, decisiones, coste, escalabilidad, Tech Lead / Staff
PARTE VI   Entrevistas: por concepto, banco 3+3+3, Senior/Staff, System Design, Leadership
PARTE VII  Ejercicios prácticos
PARTE VIII Recursos oficiales, glosario, checklists, mapa conceptual, principios finales
```

---

# PARTE I — FUNDAMENTOS Y EXPLICACIÓN TÉCNICA

## 1. Objetivo de la clase

Integrar lo estudiado (POO, módulos, listas, `for`, `input`, excepciones) en un **flujo de préstamos coherente**:

```text
Identificar usuario → Buscar por cédula → Solicitar libro → Verificar existencia/disponibilidad
→ Validar límite de préstamos → Registrar préstamo → Confirmación
```

Si una precondición falla, el sistema produce un **error específico y comprensible**:

```text
Usuario no encontrado → UsuarioNoEncontrado(Error)
Libro no disponible   → LibroNoDisponibleError
Límite alcanzado      → LimitePrestamosError
```

> **Idea central:** no se trata de "usar `try/except`", sino de **modelar explícitamente los estados inválidos del dominio y controlar dónde se recuperan**.

## 2. Arquitectura modular resultante

```text
proyecto/
├── main.py          # entrada y composición del flujo
├── biblioteca.py    # coordinación de usuarios y libros
├── usuarios.py      # modelos/comportamiento de usuarios
├── libros.py        # modelos/comportamiento de libros
└── exceptions.py    # excepciones propias del dominio
```

| Módulo | Responsabilidad |
|---|---|
| `main.py` | Entrada de la aplicación y composición del flujo |
| `biblioteca.py` | Coordinación de usuarios y libros |
| `usuarios.py` | Modelos/comportamiento de usuarios |
| `libros.py` | Modelos/comportamiento de libros |
| `exceptions.py` | Excepciones propias del dominio |

Continúa la clase de modularización: **separar por responsabilidad reduce acoplamiento y facilita localizar dónde cambiar algo**.

## 3. Estado de `Biblioteca` (atributos de instancia)

```python
class Biblioteca:
    def __init__(self):
        self.usuarios = []
        self.libros = []
```

```text
Biblioteca
├── usuarios ──→ [Usuario, Usuario, ...]
└── libros ────→ [Libro, Libro, ...]
```

`usuarios` y `libros` son **atributos de instancia**: `Biblioteca()` dos veces produce dos objetos con estado independiente (el estado pertenece a la instancia, no a la clase globalmente).

> **Corrección técnica (clásica):** nunca usar listas mutables como *atributo de clase* compartido ni como *default mutable* (`def __init__(self, libros=[])`): todas las instancias compartirían la misma lista.

## 4. Asignar las listas creadas en `main`: referencias y aliasing

```python
biblioteca.usuarios = usuarios
biblioteca.libros = libros
```

En Python las variables guardan **referencias a objetos**. No se copia la lista: ambas referencias apuntan al **mismo** objeto.

```text
usuarios ───────────┐
                    ↓
                [u1, u2]
                    ↑
biblioteca.usuarios ┘
```

`usuarios.append(estudiante3)` también es visible vía `biblioteca.usuarios`.

**Implicación de diseño:** esto es *aliasing mutable*. Válido en un ejercicio pequeño; en software profesional se controla quién posee y modifica el estado (constructor injection, repositorios, servicios de aplicación, métodos de dominio, colecciones encapsuladas, copias defensivas o tipos inmutables). No hace falta esa arquitectura aquí, pero sí **reconocer el trade-off**.

## 5. Excepción `UsuarioNoEncontrado`: jerarquía y semántica

```python
# exceptions.py
class UsuarioNoEncontrado(Exception):
    pass
```

```text
Exception
└── UsuarioNoEncontrado
```

**Mejora recomendada:** raíz común del dominio para poder capturar por categoría o por caso específico.

```python
class BibliotecaError(Exception):
    pass

class UsuarioNoEncontrado(BibliotecaError):
    pass

class LibroNoDisponibleError(BibliotecaError):
    pass

class LimitePrestamosError(BibliotecaError):
    pass
```

```python
except BibliotecaError:      # cualquier error de negocio de la biblioteca
except LibroNoDisponibleError:  # tratamiento diferenciado
```

**¿Qué significa `pass` aquí?** No que la excepción "no haga nada": no agrega comportamiento propio, pero hereda todo el de `Exception`. Lo que se agrega es **semántica**: `raise UsuarioNoEncontrado(...)` comunica el problema; `raise Exception(...)` no.

**Nombres (corrección técnica):** el material mezcla `UsuarioNoEncontrado` con `TituloInvalidoError`. PEP 8 recomienda el sufijo `Error` para excepciones que son errores. Elegir **una convención** (`UsuarioNoEncontradoError`, `LibroNoDisponibleError`, `LimitePrestamosError`, `TituloInvalidoError`) mejora búsqueda, lectura, logging, documentación y testing. Si el proyecto ya fijó una, se respeta.

**Regla:** las excepciones de aplicación derivan de `Exception`, **no** de `BaseException` (esta se reserva para `KeyboardInterrupt`, `SystemExit`, `GeneratorExit` y, notablemente, `asyncio.CancelledError`; ver §IV.3).

## 6. `buscar_usuario` por cédula

```python
class Biblioteca:
    def __init__(self):
        self.usuarios = []
        self.libros = []

    def buscar_usuario(self, cedula: str):
        for usuario in self.usuarios:
            if usuario.cedula == cedula:
                return usuario

        raise UsuarioNoEncontrado(
            f"El usuario con la cédula {cedula} no fue encontrado."
        )
```

```text
buscar_usuario(cedula)
   └─ recorrer self.usuarios
        ├─ ¿usuario.cedula == cedula? ── Sí → return usuario
        └─ No → continuar … fin del for → raise UsuarioNoEncontrado
```

### 6.1 ¿Por qué `return usuario`?
Devuelve **el objeto** encontrado (una referencia), no la cédula ni el nombre. Por eso luego se accede a `usuario.cedula`, `usuario.nombre`, `usuario.libros_prestados`.

### 6.2 ¿Por qué `raise` va fuera del `for`?
El `raise` se ejecuta **después de revisar todos** los usuarios:

```python
# ✅ correcto
for usuario in self.usuarios:
    if usuario.cedula == cedula:
        return usuario
raise UsuarioNoEncontrado(...)

# ❌ bug lógico: falla tras el PRIMER usuario que no coincide
for usuario in self.usuarios:
    if usuario.cedula == cedula:
        return usuario
    raise UsuarioNoEncontrado(...)
```

### 6.3 Complejidad
Búsqueda lineal: **O(n)** en tiempo. Razonable en un ejercicio; con cientos de miles o millones de usuarios, no. Alternativa en memoria: índice por cédula (dict) → **O(1) promedio**:

```python
usuarios_por_cedula = {"12345678": usuario1, "87654321": usuario2}
usuario = usuarios_por_cedula[cedula]   # KeyError si no existe → traducir a UsuarioNoEncontrado
```

En producción la búsqueda probablemente vive detrás de un repositorio/base de datos con índice (Parte III).

> **Principio:** la estructura de datos debe responder al **patrón de acceso dominante**.

## 7. Entrada de la cédula: `input()` devuelve `str`

```python
cedula = input("Digite el número de cédula: ")
```

`input()` devuelve siempre `str` (`"12345678"`, no `12345678`). Un identificador no es una cantidad sobre la que se opere aritméticamente: tratarlo como `str` es la decisión correcta (además preserva ceros iniciales y formatos). Ver también §III.7.

## 8. El `try/except` y la frontera de responsabilidades

```python
try:
    usuario = biblioteca.buscar_usuario(cedula)
    print(f"Cédula: {usuario.cedula}, Nombre: {usuario.nombre}")
except UsuarioNoEncontrado:
    print("El usuario que estás buscando no existe.")
```

```text
biblioteca.py  → detecta la condición inválida → raise UsuarioNoEncontrado
      ↓ propagación
main.py        → decide cómo responder al usuario → mensaje
```

> **El componente que detecta el problema no necesariamente decide cómo presentarlo.**

Ejecución con una cédula inexistente: `buscar_usuario` recorre todos, no coincide ninguno, hace `raise`, la excepción **propaga** hasta el `except` compatible. En este diseño la función **no devuelve `None`**: éxito → `return usuario`; ausencia → `raise`.

### 8.1 ¿`None` o excepción?
No hay regla universal. Ambas son correctas si el contrato es claro:

| Opción | Semántica | Cuándo |
|---|---|---|
| `Usuario \| None` | "No encontré un usuario" (ausencia = resultado normal) | Búsqueda opcional; ausencia frecuente |
| `raise UsuarioNoEncontrado` | "Esta operación esperaba encontrarlo; su ausencia rompe el contrato" | Caso de uso que requiere el usuario para continuar |
| `Result` (`Success`/`Failure`) | El fallo es parte visible del tipo | Composición explícita, estilo funcional (ver §III.16) |

Criterio: **¿la ausencia es un resultado esperado o una violación del contrato de la operación?** Usar excepciones como `if` (esperando que la mitad de las búsquedas fallen) es un olor de diseño.

### 8.2 Capturar la excepción correcta
`except UsuarioNoEncontrado` expresa exactamente el caso conocido. `except Exception` puede tragarse `TypeError`, `AttributeError`, `ValueError`, `RuntimeError`… y **ocultar bugs de programación**.

> **Regla:** captura donde puedas tomar una decisión útil y con la categoría mínima necesaria.

### 8.3 El `print` va tras el éxito
El acceso a `usuario` debe estar en la rama de éxito. Este código es conceptualmente incorrecto:

```python
try:
    usuario = biblioteca.buscar_usuario(cedula)
except UsuarioNoEncontrado:
    print("Usuario inexistente")
print(usuario.nombre)   # ❌ si falló, `usuario` no quedó asignado (NameError / estado inválido)
```

Flujo correcto: **éxito → usar resultado; error → manejar error** (o usar `else:` del `try`).

### 8.4 Qué es realmente `e` en `except ... as e` *(ampliación; el checklist original lo pedía sin explicarlo)*

```python
try:
    biblioteca.buscar_usuario("999")
except UsuarioNoEncontrado as e:
    e              # el objeto excepción (instancia)
    str(e)         # el mensaje (args formateados)
    type(e)        # la clase: <class 'UsuarioNoEncontrado'>
    e.args         # tupla con los argumentos pasados al constructor
    e.__cause__    # causa explícita (raise ... from)
    e.__context__  # excepción que se estaba manejando al lanzarse esta
    e.__traceback__
```

`e` solo existe dentro del bloque `except` (Python lo elimina al salir). Para registrar con traza: `logger.exception(...)` o `logger.error(..., exc_info=True)`.

## 9. Mostrar libros disponibles (encapsulación)

```python
print("Bienvenido a Platzi Biblioteca.")
print("Libros disponibles:")
for titulo in biblioteca.libros_disponibles():
    print(f"  - {titulo}")
```

`main` pregunta "¿qué libros están disponibles?"; `Biblioteca` resuelve la lógica. **El consumidor usa una capacidad, no la implementación interna.**

## 10. `Biblioteca` como coordinador… y el riesgo de God Object

Al agregar `buscar_usuario()`, `libros_disponibles()`, `prestar_libro()`, `registrar_usuario()`, `generar_reportes()`, `enviar_emails()`, `guardar_en_BD()`… la clase tiende a un **God Object**. `Biblioteca` debe coordinar responsabilidades **propias del dominio de biblioteca**, no ser un contenedor universal. Esto se retoma en la evolución a servicios de aplicación (§III.12).

## 11. El flujo completo del préstamo

```text
Usuario da cédula → buscar_usuario()
   ├─ no existe → UsuarioNoEncontrado → mensaje
   └─ existe → solicitar libro → verificar libro
        ├─ no disponible → LibroNoDisponibleError
        └─ disponible → validar límite
             ├─ excedido → LimitePrestamosError
             └─ OK → registrar préstamo → confirmación
```

No es una secuencia de `if`: es **un proceso de negocio con precondiciones**.

## 12. Precondiciones y orden de validación

```text
P1: el usuario existe
P2: el libro existe
P3: el libro está disponible
P4: el usuario no superó su límite
→ solo entonces: registrar préstamo
```

**Validar antes de mutar estado.** Este orden es un bug:

```python
registrar_prestamo()
if not libro.disponible:
    raise LibroNoDisponibleError(...)
```

### 12.1 `LibroNoDisponibleError`
```python
class LibroNoDisponibleError(BibliotecaError):
    pass

raise LibroNoDisponibleError(f"El libro '{libro.titulo}' no está disponible.")
```
Comunica *qué ocurrió* y *a qué dominio pertenece*, y permite tratar cada error de forma distinta:

```python
try:
    biblioteca.prestar(...)
except LibroNoDisponibleError: ...
except LimitePrestamosError: ...
except UsuarioNoEncontrado: ...
```

## 13. Las excepciones son parte del contrato de la función

```text
Entrada:  cedula
Éxito:    Usuario
Fallo esperado: UsuarioNoEncontrado
```

Una API interna bien diseñada permite a otro ingeniero saber: ¿qué le paso?, ¿qué devuelve?, ¿qué puede fallar?, ¿qué errores puedo manejar? (Documentar en docstring: sección `Raises:`.)

## 14. Estado duplicado: una sola fuente de verdad

El material indica eliminar `disponible = True` redundante en la subclase cuando la base ya lo define:

```python
class Libro:
    def __init__(self, titulo):
        self.titulo = titulo
        self.disponible = True

class LibroFisico(Libro):
    def __init__(self, titulo):
        super().__init__(titulo)
        self.disponible = True   # ❌ redundante: dos lugares definen el mismo estado inicial
```

> **Una pieza de estado debe tener una fuente de verdad clara.**

## 15. Limpieza de código como parte del diseño
Código muerto/ejemplos obsoletos generan más superficie de lectura, más confusión, más mantenimiento y más contexto inútil para herramientas automáticas y LLMs. Se limpia **sin** eliminar comportamiento que aún tenga consumidores.

## 16. Separación dominio / presentación

```python
# ❌ el dominio depende de una forma concreta de presentación
class Biblioteca:
    def buscar_usuario(self, cedula):
        ...
        print("Usuario no encontrado")
```

Preferible: `Dominio → raise UsuarioNoEncontrado → Aplicación/Interfaz → mensaje`. El mismo error de dominio podrá terminar como salida de CLI, JSON de FastAPI, respuesta GUI, mensaje de chatbot o resultado estructurado de una tool de agente.

## 17. Qué se aprende realmente

`input/print/for/raise/try/except` son el vehículo. El valor está en la **composición**:

```text
POO → módulos → estado → búsqueda → validación → excepciones específicas
→ propagación → manejo en el nivel apropiado → flujo de negocio
```

Y el patrón es transferible: `Request → validación → Application Service → Domain → Repository → DB`; ante fallo: `DB exception → Infrastructure error → Application error → API error contract → HTTP response`.

## 18. Checklist técnico de la Parte I

**Python**
- [ ] atributo de instancia · `self.usuarios` · asignar `biblioteca.usuarios = usuarios`
- [ ] referencia a objeto · aliasing mutable · cómo funciona `for` · `return usuario` · `raise` · `try/except`
- [ ] qué representa `e` en `except ... as e` (`e`, `str(e)`, `type(e)`)
- [ ] por qué excepción específica > `Exception` genérico

**Diseño**
- [ ] por qué `UsuarioNoEncontrado` es de dominio · jerarquías · detectar ≠ presentar
- [ ] validar antes de mutar · `O(n)` y cuándo indexar · evitar estado duplicado · `main` coordina, no contiene toda la lógica

> **Regla de diseño:** *No se trata de llenar el código de `try/except`. Se trata de modelar los estados inválidos del dominio, propagarlos con errores semánticos y manejarlos en el nivel que realmente puede decidir.*

---

# PARTE II — CÓDIGO DE REFERENCIA (PEP 8 + type hints)

> Distinción: **código pedagógico** (el del curso, arriba) vs. **implementación profesional** (abajo). No es sobreingeniería: sigue en memoria, sin frameworks; solo añade tipado, errores con contrato y docstrings.

### `exceptions.py`
```python
"""Errores de dominio del sistema de biblioteca."""


class BibliotecaError(Exception):
    """Base de todos los errores de negocio de la biblioteca."""

    code: str = "LIBRARY_ERROR"
    retryable: bool = False


class UsuarioNoEncontradoError(BibliotecaError):
    code = "USER_NOT_FOUND"

    def __init__(self, cedula: str) -> None:
        super().__init__(f"El usuario con la cédula {cedula} no fue encontrado.")
        self.cedula = cedula


class LibroNoDisponibleError(BibliotecaError):
    code = "BOOK_UNAVAILABLE"

    def __init__(self, titulo: str) -> None:
        super().__init__(f"El libro '{titulo}' no está disponible.")
        self.titulo = titulo


class LimitePrestamosError(BibliotecaError):
    code = "LOAN_LIMIT_REACHED"

    def __init__(self, limite: int) -> None:
        super().__init__(f"Se alcanzó el límite de {limite} préstamos.")
        self.limite = limite
```

### `libros.py` / `usuarios.py`
```python
class Libro:
    def __init__(self, titulo: str) -> None:
        self.titulo = titulo
        self.disponible = True          # única fuente de verdad del estado inicial


class Usuario:
    def __init__(self, cedula: str, nombre: str, limite_prestamos: int = 3) -> None:
        self.cedula = cedula
        self.nombre = nombre
        self.limite_prestamos = limite_prestamos
        self.libros_prestados: list[Libro] = []
```

### `biblioteca.py`
```python
from exceptions import (
    LibroNoDisponibleError,
    LimitePrestamosError,
    UsuarioNoEncontradoError,
)
from libros import Libro
from usuarios import Usuario


class Biblioteca:
    def __init__(
        self,
        usuarios: list[Usuario] | None = None,
        libros: list[Libro] | None = None,
    ) -> None:
        self.usuarios: list[Usuario] = usuarios if usuarios is not None else []
        self.libros: list[Libro] = libros if libros is not None else []

    def buscar_usuario(self, cedula: str) -> Usuario:
        """Devuelve el usuario.

        Raises:
            UsuarioNoEncontradoError: si no existe un usuario con esa cédula.
        """
        for usuario in self.usuarios:
            if usuario.cedula == cedula:
                return usuario
        raise UsuarioNoEncontradoError(cedula)

    def libros_disponibles(self) -> list[str]:
        return [libro.titulo for libro in self.libros if libro.disponible]

    def prestar(self, cedula: str, titulo: str) -> Libro:
        """Registra un préstamo validando TODAS las precondiciones antes de mutar.

        Raises:
            UsuarioNoEncontradoError, LibroNoDisponibleError, LimitePrestamosError
        """
        usuario = self.buscar_usuario(cedula)
        libro = next(
            (l for l in self.libros if l.titulo == titulo and l.disponible), None
        )
        if libro is None:
            raise LibroNoDisponibleError(titulo)
        if len(usuario.libros_prestados) >= usuario.limite_prestamos:
            raise LimitePrestamosError(usuario.limite_prestamos)

        # Solo aquí se muta el estado (todas las precondiciones ya pasaron)
        libro.disponible = False
        usuario.libros_prestados.append(libro)
        return libro
```

### `main.py`
```python
from biblioteca import Biblioteca
from exceptions import BibliotecaError, UsuarioNoEncontradoError


def main() -> None:
    biblioteca = Biblioteca(...)  # composición con datos iniciales
    print("Bienvenido a Platzi Biblioteca.\nLibros disponibles:")
    for titulo in biblioteca.libros_disponibles():
        print(f"  - {titulo}")

    cedula = input("Digite el número de cédula: ").strip()
    titulo = input("Título del libro: ").strip()
    try:
        libro = biblioteca.prestar(cedula, titulo)
    except UsuarioNoEncontradoError:
        print("El usuario que estás buscando no existe.")
    except BibliotecaError as exc:          # resto de errores de negocio
        print(f"No fue posible el préstamo: {exc}")
    else:
        print(f"Préstamo registrado: {libro.titulo}")


if __name__ == "__main__":
    main()
```

**Variante con índice O(1)** (misma interfaz pública):
```python
self._por_cedula: dict[str, Usuario] = {u.cedula: u for u in usuarios or []}

def buscar_usuario(self, cedula: str) -> Usuario:
    try:
        return self._por_cedula[cedula]
    except KeyError as exc:
        raise UsuarioNoEncontradoError(cedula) from exc   # traducción + causa preservada
```

---

# PARTE III — PRODUCCIÓN: PERSISTENCIA, CONCURRENCIA Y ERRORES POR CAPA

## 1. Curso vs. producción

```text
EJERCICIO:   "¿Funciona el flujo?"
PRODUCCIÓN:  "¿Sigue siendo correcto con concurrencia, persistencia, errores parciales,
              múltiples clientes, reintentos, observabilidad y fallos?"
```

| Curso | Producción |
|---|---|
| `list[Usuario]` | Repositorio / base de datos |
| Búsqueda lineal | Índices y consultas |
| `input()` / `print()` | API / CLI / frontend · HTTP/JSON/UI/eventos |
| `raise` / `try/except` en `main` | Error de dominio/aplicación · error boundary |
| Estado en memoria | Estado persistente |
| Una ejecución, un proceso | Múltiples clientes, workers e instancias |
| `libro.disponible` | Invariante transaccional |
| Préstamo secuencial | Operaciones concurrentes |
| Error textual | Error contract |
| Ejemplo local | Observabilidad distribuida |

Objetivo: entender **qué problemas aparecen cuando el mismo concepto pasa de ejercicio a sistema real**, no reemplazarlo por arquitectura innecesaria.

## 2. Estado en memoria y fuente de verdad

`biblioteca.libros` vive en RAM: si el proceso termina, el estado se pierde; si hay varias instancias, cada una tendría su "verdad". En producción la **base de datos** es la fuente persistente de verdad; la app puede mantener cache/objetos/sesiones temporales, pero no asumir que representan el estado global.

```text
PostgreSQL ← Repository ← Application Service ← API
```

## 3. Búsqueda: mover la responsabilidad, no solo el algoritmo

```sql
CREATE UNIQUE INDEX idx_usuario_cedula ON usuarios(cedula);
```
`SELECT ... WHERE cedula = $1` resuelto por el índice. *Optimiza dónde vive la responsabilidad de resolver el problema.* (100 usuarios → 100 comparaciones máx.; 10 M → hasta ~10 M.)

## 4. La cédula no es solo un campo

Preguntas de producción: ¿puede repetirse? ¿es obligatoria? ¿formato válido? ¿puede cambiar? ¿se almacena como texto? ¿espacios? ¿ceros iniciales (`"00123456"` ≠ `123456`)? ¿índice UNIQUE? Como identificador se preserva como `str`.

**Validación de formato ≠ existencia:**

```text
Formato inválido → ValidationError (400/422)
Formato correcto → buscar → UsuarioNoEncontrado (404)
```

> **Ampliación (contexto Perú):** el identificador nacional habitual es el DNI; la "cédula" del curso es equivalente conceptual. Es **PII**: aplica minimización, enmascarado y retención acotada (ver §III.20 y §IV.10).

## 5. "Disponible" no es solo un `bool`

Un dominio real: `Libro(id, título, ejemplares_totales, ejemplares_disponibles)` o, mejor, **`Libro` + `Ejemplar`** (cada copia física con su estado). `disponible = True` se queda corto cuando el dominio evoluciona (varias copias, reservas, mantenimiento, pérdidas).

## 6. Problema crítico: el último ejemplar (race condition)

```text
Libro: ejemplares_disponibles = 1
Usuario A                 Usuario B
 consulta → 1 disponible   consulta → 1 disponible
 "sí" → presta             "sí" → presta
        → MISMO LIBRO PRESTADO 2 VECES
```

### 6.1 Check-then-act
```python
if libro.disponible:          # CHECK
    libro.disponible = False  # ACT  ← otro actor pudo cambiar el estado entre ambos
    registrar_prestamo()
```
Es una variante de *check-then-act race condition*: la operación completa no es atómica. **Ampliación 2026:** incluso con GIL hay carreras entre hilos (el GIL protege el intérprete, no tu lógica de negocio), y en builds **free-threaded** (Python 3.14, PEP 779) el paralelismo real las hace aún más visibles. No confiar nunca en `if` sobre estado compartido en memoria.

### 6.2 Regla profesional
Validación y mutación dependiente deben estar protegidas por una **operación atómica** o sincronización correcta:

```text
BEGIN
  verificar disponibilidad + reservar/decrementar + registrar préstamo
COMMIT     (si algo falla → ROLLBACK)
```

## 7. Transacción y atomicidad

Un préstamo modifica varias entidades (Libro, Préstamo, opcionalmente contadores/historial). No puede quedar `Libro actualizado ✓ / Préstamo creado ✗`.

> **Atomicidad:** una operación de negocio indivisible no queda parcialmente aplicada — *todo o nada*. Es mucho más importante que saber escribir `try/except`.

## 8. Estrategias de control de concurrencia (elegir por invariante, no por moda)

| Estrategia | Idea | Cuándo | Costo / riesgo |
|---|---|---|---|
| **Locking pesimista** `SELECT … FOR UPDATE` | Bloquea la fila dentro de la transacción | Alta contención, sección crítica corta | Espera/contención; deadlocks si el orden de locks es inconsistente |
| **Actualización condicional atómica** | `UPDATE … SET disp = disp-1 WHERE id=$1 AND disp>0`; verificar filas afectadas | Contadores simples | 0 filas = no disponible/conflicto |
| **Restricción de unicidad parcial** | El esquema hace *imposible* el estado inválido | Un ejemplar = un préstamo activo | Traducir `UniqueViolation` → error de dominio |
| **Locking optimista** (columna `version`) | `UPDATE … WHERE id=$1 AND version=$2`; reintentar si 0 filas | Baja contención | Reintentos; lógica de retry |
| **Aislamiento `SERIALIZABLE`** | El motor detecta anomalías y aborta | Reglas complejas multi-fila | La app **debe** reintentar `serialization_failure` (SQLSTATE 40001) |
| **`FOR UPDATE SKIP LOCKED`** | Saltar filas bloqueadas | Colas/asignación de trabajo, no para "el último ejemplar" | Semántica distinta (no espera) |
| **Advisory locks** | Lock lógico por clave arbitraria | Coordinación fuera del modelo de filas | Fácil de olvidar liberar; acoplado a la conexión/sesión |

Ejemplos:

```sql
-- Pesimista
BEGIN;
SELECT * FROM libros WHERE id = $1 FOR UPDATE;
-- verificar, decrementar, insertar préstamo
COMMIT;

-- Condicional atómico
UPDATE libros
SET ejemplares_disponibles = ejemplares_disponibles - 1
WHERE id = $1 AND ejemplares_disponibles > 0;
-- filas afectadas: 1 → ok · 0 → no disponible
```

**Invariantes reforzadas por el esquema** (defensa en profundidad; el `if` de Python no basta):

```sql
CREATE TABLE prestamos (
  id          uuid PRIMARY KEY,
  usuario_id  uuid NOT NULL REFERENCES usuarios(id),
  ejemplar_id uuid NOT NULL REFERENCES ejemplares(id),
  prestado_en timestamptz NOT NULL DEFAULT now(),
  devuelto_en timestamptz
);
-- Un ejemplar no puede tener dos préstamos activos:
CREATE UNIQUE INDEX uq_prestamo_activo_por_ejemplar
  ON prestamos (ejemplar_id) WHERE devuelto_en IS NULL;

-- Contador y límite del usuario protegidos por CHECK:
ALTER TABLE usuarios
  ADD CONSTRAINT ck_limite CHECK (prestamos_activos BETWEEN 0 AND limite_prestamos);
```

> **Un Staff no dice "pongo un lock".** Pregunta: ¿qué invariante protejo? ¿cuál es la unidad atómica? ¿quién es dueño de la consistencia? ¿cuánto tiempo bloqueo? ¿qué nivel de aislamiento? ¿qué throughput necesito? ¿qué ocurre en conflicto? Los locks son una herramienta, no una arquitectura. **[Verificar]** semántica exacta por nivel de aislamiento en la documentación de la versión de PostgreSQL en uso.

## 9. Invariantes del dominio

```text
I1: ejemplares_disponibles >= 0
I2: una cédula identifica como máximo a un usuario
I3: un préstamo activo corresponde a un libro/ejemplar válido
I4: no se presta un ejemplar no disponible
I5: un usuario no supera su límite de préstamos
I6: un ejemplar no tiene dos préstamos activos
```

El código profesional no solo implementa operaciones: **protege invariantes**.

### 9.1 El límite de préstamos también es concurrente
`Límite=3, actual=2`, dos solicitudes leen `2 < 3` y ambas continúan → 4. Solución: incluir la validación en la misma transacción con lock sobre la fila del usuario (o contador con `CHECK`), no en una lectura aislada en memoria.

## 10. Unit of Work y capas

```text
        API / CLI / MCP
              ↓
     Application Service   ← orquesta el caso de uso y la política transaccional
              ↓
           Domain          ← reglas, invariantes, errores de dominio
              ↓
         Repository
              ↓
   PostgreSQL / Redis / APIs externas
```

Dirección de dependencia controlada: infraestructura depende del dominio, no al revés.

## 11. Excepciones por capa

| Capa | Ejemplos | Regla |
|---|---|---|
| **Dominio** | `LibroNoDisponibleError`, `LimitePrestamosError`, `UsuarioNoEncontradoError` | Sin conocer HTTP/CLI/agent |
| **Infraestructura** | `TimeoutError`, `UniqueViolation`, `ConnectionRefused` | No se filtran al usuario |
| **Aplicación** | `PrestamoPersistenceError` (traducción) | Agrega contexto, preserva causa |
| **Interfaz** | HTTP 409 / JSON / mensaje CLI / tool result | Único lugar que conoce el transporte |

### 11.1 ¿Por qué el dominio no lanza `HTTPException`?
Acopla el dominio a FastAPI/HTTP y bloquea la reutilización desde CLI, worker, GraphQL, gRPC, consumidor de eventos, agente o batch. Preferible: `Domain → LibroNoDisponibleError → API adapter → HTTP 409`.

### 11.2 Traducción de errores y chaining (PEP 3134)
```python
try:
    repo.guardar(prestamo)
except UniqueViolation as exc:
    raise LibroNoDisponibleError(titulo) from exc      # semántica + causa técnica
```
`from exc` → `__cause__`; `from None` suprime el contexto (usar con criterio). Así se evita exponer `psycopg.errors.UniqueViolation` al cliente y se conserva el diagnóstico.

## 12. Evolución del diseño: servicio de aplicación

```python
class PrestarLibroService:
    def ejecutar(self, usuario_id: UUID, libro_id: UUID, idempotency_key: str) -> Prestamo:
        with self._uow:                          # begin/commit/rollback
            usuario = self._usuarios.obtener_para_actualizar(usuario_id)
            ejemplar = self._ejemplares.reservar_disponible(libro_id)   # atómico
            prestamo = usuario.prestar(ejemplar)  # invariantes del dominio
            self._prestamos.agregar(prestamo)
            return prestamo
```
`main.py` deja de acumular `usuario = … if … if … if …`. Se introduce **cuando el problema lo justifica** (ver §V.9 sobre sobreingeniería).

## 13. De `main.py` a API

`POST /prestamos` con `{"cedula": "12345678", "libro_id": "book-123"}`:

```text
HTTP → validación (Pydantic) → Application Service → Domain → Repository → DB → resultado/error → HTTP response
```

El `input()` desaparece; **el problema no**: el problema real es *identificar de manera confiable al usuario que solicita un préstamo*, sea por CLI, web, móvil, chatbot o agente.

## 14. Error contract: excepción ≠ contrato externo

```text
LibroNoDisponibleError → Application Error → Error Contract → HTTP 409
```
```json
{ "code": "BOOK_UNAVAILABLE", "message": "El libro solicitado no está disponible." }
```
- `code` para máquinas (estable, versionado); `message` para humanos (puede cambiar). Los clientes no deben depender de `type(exc).__name__` ni del texto.
- **Actualización 2026:** formalizar con **RFC 9457 (Problem Details for HTTP APIs)**, que obsoleta el RFC 7807: `application/problem+json` con `type`, `title`, `status`, `detail`, `instance` y extensiones (`code`, `retryable`, `trace_id`).

```python
@app.exception_handler(BibliotecaError)
async def biblioteca_error_handler(request: Request, exc: BibliotecaError) -> JSONResponse:
    status = {"USER_NOT_FOUND": 404, "BOOK_UNAVAILABLE": 409, "LOAN_LIMIT_REACHED": 409}.get(exc.code, 400)
    return JSONResponse(
        status_code=status,
        media_type="application/problem+json",
        content={
            "type": f"https://api.ejemplo.com/errors/{exc.code.lower()}",
            "title": exc.code, "status": status, "detail": str(exc),
            "code": exc.code, "retryable": exc.retryable,
            "trace_id": getattr(request.state, "trace_id", None),
        },
    )
```
No existe una regla universal de HTTP status por dominio: el contrato concreto se define y documenta (OpenAPI).

## 15. Clasificación de errores y matriz de decisión

| Error | ¿Esperado? | ¿Retry? | Respuesta |
|---|---|---|---|
| Usuario no encontrado | Sí | No | 404 / dominio |
| Libro no disponible | Sí | No automático | 409 / dominio |
| Límite alcanzado | Sí | No | 409 / dominio |
| Input inválido | Sí | No | 400 / 422 |
| Rate limit del proveedor | Infra | Sí, respetando `Retry-After` | 429 |
| DB timeout | Infra | Posiblemente | 503 |
| Serialization failure (40001) | Infra/concurrencia | Sí, con política | retry interno |
| Bug (`AttributeError`) | No | No | 500 + alerta |
| Credenciales inválidas | No recuperable | No | 500/config + alerta |

La clasificación determina **respuesta, retry, nivel de log, métrica, alerta y rollback**. Distinguir: error recuperable vs. que requiere intervención.

## 16. Excepciones vs `None` vs `Result`

Tres diseños válidos (ver §I.8.1). Ejemplo de `Result` explícito:

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class Ok[T]:
    value: T

@dataclass(frozen=True, slots=True)
class Err[E]:
    error: E

type Result[T, E] = Ok[T] | Err[E]      # sintaxis PEP 695 (Python 3.12+)

match buscar_usuario_result("999"):
    case Ok(usuario): ...
    case Err(UsuarioNoEncontradoError() as e): ...
```
Decisión según frecuencia del caso, profundidad de propagación, claridad del contrato, estilo arquitectónico y necesidad de composición. Python no obliga a uno.

## 17. Antipatrones

- `except Exception: print("Algo salió mal")` como estrategia: mezcla bugs, fallos de infraestructura y reglas de negocio. Aceptable **solo** en un *error boundary* de nivel superior para logging/métrica/respuesta segura.
- Convertir todo en mensaje amigable: exponer `Connection refused`, stack traces, SQL, hosts o credenciales es fuga de información; mensaje seguro al usuario, detalle técnico a logs.
- Reportar "libro no disponible" cuando la BD está caída: un fallo técnico no se disfraza de estado de negocio (semánticamente falso).
- Excepciones como control de flujo normal.

## 18. Dónde capturar

> **Captura cuando tienes contexto suficiente para hacer algo útil.** El repositorio no muestra "Base de datos caída" al usuario: `DatabaseError → ApplicationError → boundary API → 503`. Cada capa agrega la información que le corresponde.

## 19. Reliability: retry, backoff, jitter, timeouts, circuit breaker

Toda dependencia externa (DB, API, LLM, MCP server) puede tardar, fallar, rechazar, degradarse o responder parcialmente. Política explícita de **timeout · retry · backoff · circuit breaker · fallback · observabilidad**.

- Reintentar solo errores **transitorios** y operaciones **seguras de repetir o idempotentes**. Retry sin límite amplifica caídas (**retry storm**).
- **Backoff exponencial + jitter** (evita sincronización de clientes) y presupuesto máximo de reintentos/tiempo.
- No reintentar: usuario inexistente, libro no disponible, input inválido, credenciales inválidas.
- **Circuit breaker:** `CLOSED → OPEN → HALF-OPEN`.

```python
import random, time

def con_reintentos(operacion, *, intentos: int = 4, base: float = 0.2, tope: float = 5.0):
    for n in range(intentos):
        try:
            return operacion()
        except TransientError:                     # solo transitorios
            if n == intentos - 1:
                raise
            time.sleep(min(tope, base * 2**n) * random.uniform(0.5, 1.5))
```
Testing sin esperar tiempo real: inyectar `sleep` (o parchear) y verificar número/orden de reintentos.

## 20. Idempotencia

`POST /prestamos` se procesa pero la respuesta se pierde; el cliente reintenta → 2 préstamos. Solución: **`Idempotency-Key`**.

```text
misma key → ya procesada → devolver el resultado anterior
misma key + payload distinto → 422/409 (uso indebido de la key)
```
```sql
CREATE TABLE idempotency_keys (
  scope text NOT NULL, key text NOT NULL, request_hash text NOT NULL,
  response_status int, response_body jsonb,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (scope, key)
);
-- Patrón: INSERT de la key en la MISMA transacción que el efecto; conflicto = ya procesado.
```
Definir TTL de claves, alcance por usuario/tenant y comportamiento ante procesamiento en curso (`409` o esperar). **[Verificar]** estado del borrador IETF `Idempotency-Key` header antes de citarlo como estándar. Es propiedad fundamental de sistemas distribuidos y **agentic AI**: un agente que reintenta una tool no debe duplicar el efecto.

## 21. Observabilidad

`print("Usuario no encontrado")` no es observabilidad. Necesitas responder: ¿cuándo, dónde, en qué request, con qué usuario, qué operación, cuánto tardó, qué dependencia falló, con qué frecuencia?

- **Logs estructurados** (JSON) con `trace_id`, `request_id`, `operation`, `user_id` interno, `book_id`, `exception_type`, `error_code`, `duration`, `service`, `environment`.
- **Métricas:** tasa de error por `code`, latencia p50/p95/p99, conflictos de concurrencia, reintentos, **`duplicate_active_loans = 0`**.
- **Trazas:** OpenTelemetry (estándar de facto) con propagación de contexto.
- Errores esperados (dominio) → `INFO/WARNING` y métrica; bugs → `ERROR` + alerta.
- **PII:** no registrar cédula completa, tokens, passwords, API keys, ni devolver stack traces al cliente. Enmascarar (`cedula=******78`), definir retención y control de acceso. La observabilidad también es seguridad.

```python
logger.warning("prestamo_rechazado", extra={"code": exc.code, "book_id": libro_id, "trace_id": tid})
```

## 22. Seguridad

- **Identificación ≠ autenticación ≠ autorización.** Conocer una cédula no demuestra ser ese usuario. Autenticación = ¿quién eres?; autorización = ¿puedes hacer esta operación?
- **Validación de entrada:** normalización (`strip`), longitud máxima, formato, encoding.
- **SQL parametrizado, nunca concatenado:**
```python
cursor.execute("SELECT * FROM usuarios WHERE cedula = %s", (cedula,))   # ✅
query = f"SELECT * FROM usuarios WHERE cedula = '{cedula}'"             # ❌ inyección SQL
```
- Secretos fuera del código (secret manager), mínimo privilegio en DB/tools, auditoría de operaciones de escritura, aislamiento por tenant.
- Enumeración de usuarios: en endpoints públicos, evitar revelar por diferencia de mensaje si una cédula existe.

## 23. Testing profesional del flujo

Casos: usuario existente / inexistente / libro disponible / no disponible / límite alcanzado / **fallo durante persistencia → rollback sin estado parcial**.

```python
import pytest

def test_usuario_no_encontrado(biblioteca):
    with pytest.raises(UsuarioNoEncontradoError) as exc_info:
        biblioteca.buscar_usuario("99999999")
    assert exc_info.value.cedula == "99999999"
    assert exc_info.value.code == "USER_NOT_FOUND"

def test_no_muta_libro_si_usuario_supera_limite(biblioteca_con_usuario_al_limite, libro):
    with pytest.raises(LimitePrestamosError):
        biblioteca_con_usuario_al_limite.prestar("123", libro.titulo)
    assert libro.disponible is True          # ← la propiedad importante: sin estado parcial

def test_preserva_causa_al_traducir(...):
    with pytest.raises(LibroNoDisponibleError) as exc_info:
        servicio.prestar(...)
    assert isinstance(exc_info.value.__cause__, UniqueViolation)
```

**Modelo de madurez del testing de errores:**

```text
N1 ¿excepción correcta? → N2 ¿datos correctos? → N3 ¿causa preservada? → N4 ¿estado consistente?
→ N5 ¿bajo concurrencia? → N6 ¿bajo retry? → N7 ¿qué observa el sistema? → N8 ¿qué recibe el consumidor?
```

**Test de invariante bajo concurrencia** (integración con DB real, p. ej. Testcontainers):

```python
from concurrent.futures import ThreadPoolExecutor

def test_ultimo_ejemplar_maximo_un_exito(servicio, libro_id, usuarios_ids):
    def intentar(uid):
        try:
            servicio.prestar(uid, libro_id, idempotency_key=f"k-{uid}")
            return "ok"
        except LibroNoDisponibleError:
            return "conflicto"

    with ThreadPoolExecutor(max_workers=8) as pool:
        resultados = list(pool.map(intentar, usuarios_ids))

    assert resultados.count("ok") == 1
    assert contar_prestamos_activos(libro_id) == 1     # invariante verificada en persistencia
```
Herramientas de alto valor: **property-based testing (Hypothesis)** para invariantes; mocks de excepciones de SDK; tests de retry con reloj/`sleep` falso. Coverage es una señal, no una garantía de corrección.

Pirámide: **Unit** (dominio, validaciones, límite) → **Integration** (Application Service + Repository + DB) → **E2E** (HTTP → DB → respuesta). No todo al nivel E2E.

## 24. Fallos parciales y sistemas distribuidos

Si la reserva y el préstamo viven en la **misma BD**, una transacción ACID local basta. Si cruzan servicios (Biblioteca → Notificaciones → servicio externo): considerar **Outbox pattern**, **Saga/compensación** o mensajería con consumidores idempotentes. No introducir Saga para dos tablas de la misma base (sobreingeniería). Palabra clave: **unidad de consistencia**.

## 25. Escalabilidad (cuando crece a millones de usuarios)

No responder "Kubernetes". Primero identificar cuello de botella: búsqueda, DB, locks, CPU, latencia, conexiones, cache, red. Luego: índices, optimización de queries, **connection pooling**, cache, réplicas de lectura, particionado, escalado horizontal. Kubernetes es operativa; no arregla un `O(n)` ni una transacción mal diseñada.

**Cache:** sí para metadata de libros/catálogo/consultas frecuentes; con extremo cuidado para disponibilidad, límites y estado transaccional (*stale data*: cache dice "disponible", DB dice "prestado"). La decisión de escritura crítica se toma contra la fuente de verdad transaccional.

## 26. Resumen: qué cambió entre curso y producción

```text
Data structure → Algorithm → Domain rule → Exception taxonomy → Application boundary
→ Persistence → Transaction → Concurrency → Idempotency → Observability → API → AI Agent / MCP
```

> Un Senior no se queda en la sintaxis que resuelve el ejemplo: identifica **qué propiedades deben mantenerse cuando el sistema crece**.

---

# PARTE IV — ESTADO DEL ARTE (revisado al 30 de agosto de 2026)

## 1. Material del curso vs. práctica actual

| Tema | En el curso | Estado 2026 | Vigencia |
|---|---|---|---|
| `raise` / `try/except` / excepciones custom | Base | Sin cambios de fondo | Vigente |
| `except` sobre categoría específica | Sí | Reforzado; sintaxis simplificada en 3.14 (§IV.2) | Vigente |
| Estado en `list` | Sí | Solo pedagógico | Parcialmente vigente |
| `print` para errores | Sí | Logging estructurado + OpenTelemetry | Desactualizado en producción |
| Excepciones secuenciales | Sí | Concurrencia estructurada: `ExceptionGroup`/`except*` | Ampliado |
| Backend sin IA | Sí | Backends consumidos por agentes vía tools/MCP | Ampliado |

## 2. Python 3.14 y lo relevante para esta clase

Ancla de versión: **Python 3.14** (estable desde octubre de 2025; documentación oficial es la referencia). **[Verificar]** el estado de 3.15 (ciclo de release de otoño 2026) antes de adoptar novedades.

- **PEP 758 — `except` y `except*` sin paréntesis:** en 3.14 es válido `except TimeoutError, ConnectionError:` cuando **no** se usa `as`. Con `as` siguen requiriéndose paréntesis: `except (A, B) as e:`. Cuidado: en versiones ≤3.13 esa forma sin paréntesis era sintaxis de Python 2 / error. Mantener paréntesis por legibilidad y compatibilidad si el equipo soporta versiones anteriores.
- **PEP 765 — prohibir `return`/`break`/`continue` que salgan de un bloque `finally`:** 3.14 emite `SyntaxWarning`. Un `return` dentro de `finally` **traga la excepción en vuelo** (bug clásico de manejo de errores).
- **PEP 654 (3.11) — `ExceptionGroup` y `except*`**, y **PEP 678 — `add_note()`** para enriquecer excepciones sin cambiar su tipo.
- **PEP 3134 — chaining** (`from`, `__cause__`, `__context__`).
- **PEP 649/749 — anotaciones diferidas (3.14):** las anotaciones se evalúan de forma perezosa; menos `from __future__ import annotations` y menos referencias hacia adelante como strings. Relevante para type hints de modelos/repositorios.
- **PEP 779 — free-threaded Python soportado oficialmente (fase II)** y **PEP 734 — múltiples intérpretes en la stdlib**: el paralelismo real en CPU pasa a ser opción; **empeora** las consecuencias de estado compartido mutable sin sincronización (§III.6).
- **PEP 750 — template strings (t-strings):** útiles para construir SQL/HTML/logs de forma segura si la librería procesa el template (no sustituyen la parametrización de consultas; verificar que el driver/librería lo soporte).
- Otras mejoras de 3.14: mejoras de introspección de `asyncio` (`python -m asyncio ps` / `pstree`), interfaz de depuración externa segura (PEP 768), módulo `compression.zstd`. **[Verificar]** detalles concretos en *What's New in Python 3.14*.
- Características previas ya asentadas: `match/case` (3.10), sintaxis de parámetros de tipo y `type` alias (PEP 695, 3.12), `@override` (3.12), `Self`, `StrEnum`, `dataclass(slots=True, frozen=True)`.

## 3. Async, concurrencia estructurada y errores

```python
import asyncio

async def tool_ok() -> str:
    return "ok"

async def tool_timeout() -> None:
    raise TimeoutError("proveedor lento")

async def tool_invalida() -> None:
    raise ValueError("argumentos inválidos")

async def main() -> None:
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(tool_ok())
            tg.create_task(tool_timeout())
            tg.create_task(tool_invalida())
    except* TimeoutError as eg:
        print("timeouts:", eg.exceptions)
    except* ValueError as eg:
        print("inválidas:", eg.exceptions)

asyncio.run(main())
```

```text
Secuencial:  A → error → propagación (una excepción)
Concurrente: A ok · B TimeoutError · C ValueError → ExceptionGroup
```

Puntos clave:
- `async def` define una **función asíncrona** (no una clase); invocarla devuelve una **coroutine**, que se ejecuta con `await` o dentro del event loop. Es la base de FastAPI, clientes HTTP, APIs de LLM, streaming, agentes y tool calls paralelos.
- **`asyncio.TaskGroup` (3.11)** = concurrencia estructurada: si una tarea falla, cancela a las hermanas y agrega los errores en un `ExceptionGroup`.
- **`asyncio.timeout()` (3.11)** como context manager para plazos; lanza `TimeoutError`.
- **Cancelación ≠ fallo de negocio.** `asyncio.CancelledError` hereda de `BaseException`, por eso `except Exception` **no** la captura (correcto). Capturar `BaseException` o silenciar la cancelación rompe timeouts y apagados ordenados. Regla: si se captura `CancelledError` para limpiar, **re-lanzarla**.
- Testing: `with pytest.raises(ExceptionGroup) as exc_info:` + `exc_info.value.exceptions`; o `pytest.RaisesGroup` en versiones recientes de pytest **[Verificar]**.
- Patrón para clasificar y agregar fallos: `except*` por tipo → decidir qué es reintentable, qué degrada la respuesta parcialmente y qué aborta.

## 4. Tooling profesional Python (2026)

- **uv** (gestión de entornos/dependencias/lockfile, muy extendido) y **ruff** (lint + format) como estándar de facto.
- **Type checkers:** mypy y pyright consolidados; **[Verificar]** madurez de `ty` (Astral) antes de adoptarlo como gate de CI.
- **pytest** + **Hypothesis** + **Testcontainers** para integración con Postgres real.
- **pre-commit**, CI con lint/type/test, escaneo de dependencias y SBOM.
- **Pydantic v2** para validación en la frontera (entrada/salida, tool schemas, structured outputs).

## 5. Bases de datos: lo que un Senior debe saber hoy

- **PostgreSQL** sigue siendo la elección por defecto para sistemas transaccionales; con `pgvector` cubre muchos casos de embeddings sin una vector DB aparte. **[Verificar]** versión estable vigente (PostgreSQL 18 publicada en 2025; 19 en ciclo 2026) y notas de release para cambios en concurrencia/índices.
- Restricciones e índices parciales como mecanismo de invariantes (§III.8) son preferibles a confiar en lógica de aplicación.
- **Durable execution** (Temporal, Restate, DBOS y similares) cobra relevancia para flujos largos con reintentos e idempotencia gestionados por el motor. Evaluarlo cuando el flujo cruza servicios, no para dos tablas.

## 6. Agentes, tools y MCP (estado 2026)

Principio rector, sin cambios: **el modelo propone acciones; el sistema autoriza y ejecuta las reglas.**

```text
Usuario → LLM (interpreta intención, elige tool)
        → Tool/MCP (frontera de integración)
        → Application Service → Domain → Repository → DB
        → resultado estructurado → LLM → respuesta natural
```

- **MCP (Model Context Protocol):** protocolo abierto para conectar modelos con herramientas y contexto. Evoluciona rápido (autorización basada en OAuth, structured tool output, elicitation, tareas de larga duración, anotaciones de tools) y su gobierno pasó a una fundación neutral bajo la Linux Foundation (Agentic AI Foundation). **[Verificar]** la revisión vigente de la especificación en modelcontextprotocol.io antes de decisiones de arquitectura.
- **A2A (Agent2Agent) y otros protocolos de interoperabilidad entre agentes:** conocer el problema que resuelven (agente↔agente) frente a MCP (agente↔herramienta). **[Verificar]** estado de adopción.
- **SDKs y frameworks de agentes:** Claude Agent SDK, OpenAI Agents SDK, Google ADK, LangGraph, Pydantic AI, Semantic Kernel, CrewAI, AutoGen/Microsoft Agent Framework, entre otros. Regla del curso: **problema → primitiva → patrón → arquitectura → framework**; el conocimiento debe sobrevivir a la herramienta. **[Verificar]** APIs concretas: cambian frecuentemente.
- **Diseño de tools:** menos tools, más coherentes y orientadas al caso de uso (`prestar_libro`, no seis pasos manuales); descripciones y schemas estrictos; **errores accionables** que permitan al modelo recuperarse (`code`, `message`, `retryable`, `suggested_action`); resultados compactos (coste de contexto); idempotencia y `Idempotency-Key`.

```python
def prestar_libro_tool(usuario_id: str, libro_id: str, idempotency_key: str) -> dict:
    """Tool expuesta al agente. Sin lógica de negocio propia: delega al caso de uso."""
    try:
        prestamo = servicio.ejecutar(UUID(usuario_id), UUID(libro_id), idempotency_key)
        return {"ok": True, "prestamo_id": str(prestamo.id)}
    except BibliotecaError as exc:
        return {"ok": False, "error": {"code": exc.code, "message": str(exc), "retryable": exc.retryable}}
```

- **Context engineering** (qué entra a la ventana de contexto, cuándo y con qué formato) sustituye a "prompt engineering" como disciplina central; incluye memoria, recuperación, resultados de tools y compactación.
- **Evaluation Engineering:** datasets de evaluación, baselines, graders (humanos y basados en modelo), regresión offline/online, evaluación de trayectorias y de tools; "funciona" no es evidencia suficiente.
- **Observabilidad de LLMs/agentes:** trazas por paso (prompt, retrieval, tool call, resultado, tokens, coste, latencia); **OpenTelemetry GenAI semantic conventions** (en evolución **[Verificar]**).
- **Costo/latencia/calidad:** prompt caching, *model routing* (modelo pequeño→grande), fallbacks entre proveedores, límites de tokens y presupuestos por request/tenant.

## 7. Seguridad específica de agentes

- **Prompt injection e *indirect* prompt injection** (instrucciones dentro de documentos/páginas/resultados de tools); **excessive agency**; exfiltración de datos; uso inseguro de tools; aislamiento entre tenants. Referencias: **OWASP Top 10 for LLM Applications (edición 2025)** y el trabajo de OWASP sobre aplicaciones agénticas **[Verificar]**.
- **"Lethal trifecta"** (acceso a datos privados + contenido no confiable + capacidad de comunicación externa): si coinciden en un agente, el riesgo de exfiltración es estructural; romper al menos una pata.
- Mitigaciones: **mínimo privilegio por tool**, credenciales acotadas y de corta vida, **human-in-the-loop** para efectos irreversibles, validación determinista de argumentos, listas de permitidos, auditoría y límites de tasa. **Ningún control crítico debe depender de que el LLM "obedezca".**
- El LLM **no** es autoridad sobre invariantes: disponibilidad, límites, permisos y transacciones viven en código determinista.

## 8. Fiabilidad de tools con reintentos y duplicados

Un orquestador o SDK puede reintentar una tool por timeout, error de red o mensaje duplicado. Si la tool crea un préstamo, envía un email o cobra un pago, el reintento duplica el efecto. Cadena de protección: `Agent → Tool → Application Service → Idempotency → Database`. La protección **no** puede depender de que el LLM recuerde si ya ejecutó la acción.

## 9. Sobreingeniería: qué NO hacer todavía

Conocer Clean Architecture, DDD, CQRS, Event Sourcing, microservicios, Kafka, Kubernetes, service mesh o agentes **no** implica meterlos en el ejercicio. Progresión correcta: **simple → medir → identificar problema real → introducir solución → medir de nuevo.** La complejidad debe estar justificada.

## 10. Regulación y datos personales (contexto profesional)

- **[Verificar]** cronograma vigente del **EU AI Act** (obligaciones por fases; ajustes en discusión) si el producto sirve a la UE.
- **[Verificar]** marco peruano: Ley de Protección de Datos Personales (Ley 29733) y su reglamento, y el marco de IA vigente, si operas en Perú.
- Práctica transversal: minimización de datos, finalidad, retención, trazabilidad de decisiones automatizadas y auditoría.

---

# PARTE V — ARQUITECTURA, SYSTEM DESIGN Y TECH LEAD / STAFF

## 1. Arquitectura de referencia

```text
                      ┌────────────┐
                      │   User     │
                      └─────┬──────┘
          ┌─────────────────┼─────────────────┐
          ↓                 ↓                 ↓
        REST              CLI               MCP / Agent
          └─────────────────┼─────────────────┘
                            ↓
                ┌────────────────────────┐
                │  Application Service   │  PrestarLibro (transacción, idempotencia)
                └───────────┬────────────┘
                            ↓
                ┌────────────────────────┐
                │  Domain                │  Usuario · Libro/Ejemplar · Préstamo · Domain Errors
                └───────────┬────────────┘
                            ↓
                ┌────────────────────────┐
                │  Repositories / UoW    │
                └───────────┬────────────┘
                            ↓
                ┌────────────────────────┐
                │  PostgreSQL            │  transacciones · constraints · índices
                └────────────────────────┘
   Transversal: Observability · Security · Idempotency · Retries · Timeouts · Testing
```

## 2. Decisiones arquitectónicas (formato de ADR)

| Decisión | Opción A | Opción B | Elección y condiciones de cambio |
|---|---|---|---|
| Errores | Excepciones de dominio | `Result` explícito | Excepciones para el flujo principal (idiomático en Python); `Result` en pipelines donde el fallo es resultado normal. Reconsiderar si el equipo necesita composición explícita |
| Persistencia de reglas | `if` en Python | Constraints + transacción | Constraints + transacción como garantía, validación en Python como UX/rapidez |
| Concurrencia | `FOR UPDATE` | Update condicional / índice único parcial | Preferir la garantía más simple que preserve el invariante; pesimista si hay lógica multi-fila |
| Repository | Sí | No | Solo si aporta frontera real (volatilidad, testing, complejidad). No es obligatorio |
| Saga/Outbox | Sí | Transacción local | Saga solo si la operación cruza límites no transaccionables |
| Exposición a agentes | Tools granulares | Caso de uso único `prestar_libro` | Caso de uso único: reduce la ventana entre lectura y escritura |

Plantilla: *Opción A (ventajas/desventajas) · Opción B · Decisión y por qué · Trade-offs (qué sacrificamos) · Condiciones de cambio (cuándo reconsiderar).*

## 3. Trade-offs clave
- **Consistencia vs. throughput:** bloqueo/serialización garantizan correctitud a costa de contención.
- **Simplicidad vs. desacople:** Repository/UoW cuestan código; solo si pagan.
- **Excepciones vs. Result:** ergonomía idiomática vs. explicitud del fallo.
- **Cache vs. frescura:** velocidad vs. datos obsoletos en decisiones de escritura.
- **Granularidad de tools:** flexibilidad del agente vs. superficie de error.
- **Reintentos:** resiliencia vs. amplificación de carga y duplicados.
- **Más barato ≠ mejor:** equilibrar coste, calidad, latencia, fiabilidad y escala.

## 4. Coste
`Coste + Calidad + Latencia + Fiabilidad + Escala`. Fuentes de coste en este sistema: conexiones/locks en DB, almacenamiento de logs/trazas, llamadas a LLM (tokens, reintentos), infra de idempotencia, cache. Controles: presupuestos por request, límites de tasa, muestreo de trazas, TTL de claves, modelos más pequeños para tareas simples.

## 5. Preguntas que un Staff Engineer haría a este diseño

```text
1. ¿Cuál es la fuente de verdad del estado?
2. ¿Qué garantiza que una cédula sea única?
3. ¿Qué ocurre si dos usuarios solicitan el último ejemplar simultáneamente?
4. ¿Qué garantiza que el límite de préstamos no se exceda?
5. ¿Qué ocurre si se actualiza el libro pero falla el registro del préstamo?
6. ¿Qué ocurre si el cliente reintenta la misma solicitud?
7. ¿Dónde se traducen los errores de dominio a HTTP?
8. ¿Cómo sabemos cuántos préstamos fallan y por qué?
9. ¿Qué información sensible estamos registrando?
10. ¿Qué parte de esta lógica reutilizan una API, un worker y un agente sin duplicarla?
```
Responder "lo soluciono con `try/except`" = pensar en **sintaxis**. Responder con invariante, unidad transaccional, política de concurrencia, idempotencia y error boundary = pensar como **Senior/Staff**.

## 6. Tech Lead / Staff Thinking

| Nivel | Pregunta que se hace |
|---|---|
| Developer | ¿Cómo lo implemento? |
| Senior | ¿Por qué lo implementaría así y qué puede fallar? |
| Tech Lead | ¿Cómo afecta esta decisión al sistema y al equipo? |
| Staff | ¿Cómo afecta a múltiples sistemas/equipos y a la evolución futura? |

Acciones concretas del Tech Lead sobre este caso: documentar el **error contract** y la taxonomía de errores como estándar de equipo; fijar la convención de nombres; imponer tests de invariantes en CI; definir SLOs (p. ej. `duplicate_active_loans = 0`, tasa de 5xx, p95 de préstamo); revisar PRs con el checklist de Design Review (§VIII.5); decidir cuándo NO añadir arquitectura; mentorear sobre "detectar vs. manejar vs. presentar".

## 7. Casos de ingeniería (escenarios hipotéticos de entrenamiento; no son casos reales verificables)

| # | Problema | Solución | Métrica | Lección |
|---|---|---|---|---|
| 1 | Último ejemplar, 2 solicitudes simultáneas | Transacción + control de concurrencia + índice único parcial | `duplicate active loans = 0` | Validar no basta; garantizar la propiedad bajo concurrencia |
| 2 | Retry duplica un préstamo (respuesta perdida) | `Idempotency-Key` + resultado almacenado | duplicados por retry = 0 | Diseñar suponiendo que los mensajes pueden repetirse |
| 3 | Timeout de DB al crear préstamo | Error de infraestructura (503), log+métrica+alerta; **no** "libro no disponible" | tasa de 5xx | No convertir un fallo técnico en estado de negocio falso |
| 4 | Agente pide "Préstame El Principito" y el libro no está disponible | Tool → Application Service → `LibroNoDisponibleError` → error estructurado → LLM explica | tasa de tool errors por `code` | El backend protege las reglas; el LLM no conoce locks/transacciones |
| 5 | El caso de uso se expone por MCP | MCP delega al Application Service (sin duplicar reglas) | divergencias de reglas = 0 | MCP es frontera de integración, no sustituto del dominio |

Formato de debugging aplicado: **síntoma → hipótesis → experimento → evidencia → diagnóstico → corrección → prevención** (p. ej. síntoma: 2 préstamos activos del mismo ejemplar; hipótesis: check-then-act; experimento: test concurrente que lo reproduce; corrección: índice único parcial + transacción; prevención: test de invariante en CI + alerta).

## 8. System Design Interview aplicado

```text
Clarificar requisitos → Estimar escala → Restricciones → Arquitectura → Bottlenecks
→ Fallos → Seguridad → Observabilidad → Coste → Trade-offs → Evolución
```
Ejemplo de apertura de nivel Staff (ver §VI.6).

---

# PARTE VI — ENTREVISTAS SENIOR / STAFF

## 1. Modelo mental

```text
REQUERIMIENTO → INVARIANTE → RIESGO → DISEÑO → TRADE-OFF → FALLO → OBSERVABILIDAD → TEST
```
Ejemplo: *"Un libro no puede prestarse dos veces"* → invariante: ningún ejemplar con >1 préstamo activo → riesgo: requests concurrentes → diseño: transacción + control de concurrencia → trade-off: más coordinación a cambio de consistencia → test: dos solicitudes simultáneas → observabilidad: registrar conflictos.

## 2. Preguntas por concepto (formato completo)

### P1. ¿Por qué crear `UsuarioNoEncontrado` en lugar de `raise Exception(...)`?
- **Nivel:** Básico-Intermedio · **Evalúa:** semántica de errores, contratos, jerarquías, mantenibilidad.
- **NO responder:** "Porque es más profesional." → no explica qué capacidad técnica se gana; suena a memorización.
- **Responder:** permite manejar el caso semánticamente, sin depender de texto ni capturar categorías amplias; se puede registrar, mapear a contrato y agrupar en una jerarquía.
- **Alto impacto:** *"El error es parte del contrato del dominio. Con una excepción específica el consumidor reacciona por tipo, no parseando mensajes ni capturando `Exception`; además puedo mapearla a un contrato externo y agruparla bajo `BibliotecaError`."*
- **Follow-up:** ¿capturarías `BibliotecaError` o la específica? → específica si el tratamiento difiere; la base para un boundary común.
- **Red flags:** capturar `Exception`; comparar mensajes de texto.

### P2. ¿Qué opinas de `try: prestar() / except Exception: print("Error")`?
- **Evalúa:** criterio de captura, error boundaries. **Nivel:** Intermedio.
- **NO responder:** "Está bien, así no se rompe el programa." (oculta bugs).
- **Alto impacto:** *"No como estrategia general: mezcla reglas de negocio, fallos de infraestructura y bugs. Capturo excepciones específicas donde puedo decidir algo útil; un `except Exception` solo en un error boundary de nivel superior para logging, métricas y respuesta segura, no para continuar como si nada."*
- **Follow-up:** ¿y `CancelledError`? → hereda de `BaseException`; nunca silenciarla, re-lanzar tras limpiar.

### P3. Dos usuarios piden el último ejemplar. ¿Cómo garantizas que solo uno lo obtiene?
- **Evalúa:** concurrencia, invariantes, transacciones. **Nivel:** Avanzado (Senior/Staff).
- **NO responder:** "Con un `if libro.disponible`" / "Pongo un lock."
- **Por qué NO:** el `if` en memoria es check-then-act; "lock" sin invariante ni alcance no es diseño.
- **Alto impacto:** *"Defino el invariante —un ejemplar no tiene dos préstamos activos— y muevo la autoridad a la persistencia: reserva del ejemplar y registro del préstamo en una transacción, reforzados por un índice único parcial; según contención uso `FOR UPDATE` o un update condicional atómico. El perdedor recibe un conflicto de dominio (409) y lo pruebo con solicitudes concurrentes."*
- **Follow-up:** ¿y con `SERIALIZABLE`? → el motor aborta una; la app reintenta 40001 con política acotada.
- **Red flags:** confiar en Redis/cache o en el LLM para esto.

### P4. ¿Devolver `None` o lanzar excepción?
- **Alto impacto:** *"Depende del contrato: si la ausencia es resultado normal de una búsqueda opcional, `None` o `Result`; si el caso de uso exige que exista, excepción de dominio."* Criterio: ¿resultado esperado o violación del contrato?

### P5. ¿Dónde capturarías `UsuarioNoEncontrado`?
- **Alto impacto:** *"Donde hay contexto para decidir: el repositorio la deja propagar; la aplicación decide que es un caso de negocio fallido; la API la traduce a 404 con un contrato de error."* Evitar capturar demasiado pronto.

### P6. ¿Diferencia entre excepción y error contract?
- **Alto impacto:** *"La excepción es un mecanismo interno de Python; el contrato es la representación estable para consumidores externos (`code`, `message`, `retryable`, `trace_id`). Puedo cambiar la excepción sin romper clientes."*

### P7. ¿Por qué el dominio no lanza `HTTPException`?
- **Alto impacto:** *"Acopla el dominio al transporte. El dominio lanza un error semántico y el adaptador de interfaz lo traduce, lo que permite reutilizar el caso de uso desde REST, CLI, workers, MCP o agentes."*

### P8. ¿Qué invariante estás protegiendo?
- Un Senior las enumera (I1–I6 en §III.9): cambia la conversación de "¿qué código escribes?" a "¿qué propiedad debe preservar el sistema?".

### P9. ¿Qué pasa si falla a mitad del préstamo?
- **Alto impacto:** *"Las operaciones que deben ocurrir juntas forman una unidad atómica; con una transacción, un fallo intermedio revierte todo. Pruebo explícitamente el rollback."* Si cruza servicios: outbox/saga, según el caso.

### P10. ¿Cómo probarías la concurrencia?
- **NO responder:** "Un unit test de `prestar()`."
- **Alto impacto:** *"Test de invariante: N solicitudes concurrentes sobre 1 ejemplar ⇒ como máximo un éxito, y verifico el estado persistente."*

## 3. Preguntas por concepto (formato compacto — cubre el resto)

| # | Pregunta | Qué NO responder | Alto impacto |
|---|---|---|---|
| 11 | ¿Por qué `input()` no pertenece al dominio? | "Porque es feo." | Es tecnología de interacción; el dominio necesita `cedula`/`libro_id` sin saber si vienen de terminal, HTTP, Kafka, MCP o un agente. |
| 12 | ¿Dónde vive `buscar_usuario()`? | "Siempre en `Biblioteca`." | En el ejercicio, en la colección en memoria; en producción tras un repositorio sobre una fuente persistente consultable con índice. |
| 13 | ¿Repository Pattern es obligatorio? | "Siempre hay que usarlo." | No; útil cuando desacopla de la persistencia o facilita tests; sobreingeniería si no aporta frontera real. Depende de volatilidad, complejidad y testing. |
| 14 | ¿Límite de préstamos bajo concurrencia? | "Con el `if len(...) < limite`." | La validación aislada no es garantía; va en la misma transacción con lock o restricción (`CHECK`), según el nivel de aislamiento. |
| 15 | ¿Qué es check-then-act? | "Un `if` normal." | Separar validación y mutación dependiente, dejando ventana a otro actor: race condition. Se resuelve con operación atómica/sincronizada. |
| 16 | ¿Cuándo usarías retry? | "Cuando hay un error." | Solo errores transitorios y operaciones idempotentes, con backoff+jitter y presupuesto; retry sin límite genera retry storms. |
| 17 | ¿Relación de idempotencia con este sistema? | "Es para pagos." | Repetir la operación no produce efectos adicionales; `Idempotency-Key` evita 2 préstamos ante un reintento. Clave en sistemas distribuidos y agentes. |
| 18 | ¿Por qué importa para AI Agents? | "El agente recuerda lo que ejecutó." | Los agentes reintentan tools; la protección va en el backend (idempotencia), no en la memoria del LLM. |
| 19 | ¿Un LLM debería decidir si el libro está disponible? | "Sí, con buen prompt." | No como autoridad: interpreta intención y elige tool; el backend determinista decide existencia, disponibilidad, permisos y límite. |
| 20 | ¿Cómo expondrías esto a un agente? | "Una tool por cada operación interna." | Un caso de uso `prestar_libro` con validaciones, transacción, concurrencia e idempotencia dentro del backend. |
| 21 | ¿Por qué no exponer cada operación interna como tool? | "Da más control al agente." | Tools demasiado granulares permiten secuencias inconsistentes y abren ventanas lectura→escritura; el agente solicita la intención, el backend controla la invariabilidad. |
| 22 | ¿Cómo manejas `LibroNoDisponibleError` en FastAPI? | "`HTTPException` en el dominio." | Excepción de dominio → exception handler → 409 con contrato (`code`, `message`, RFC 9457). El status pertenece a la API. |
| 23 | ¿Qué registrarías en logs? | "Todo, incluida la cédula." | `trace_id`, operación, IDs internos, `error_code`, duración, servicio; nada de PII completa, tokens o stack traces al cliente. |
| 24 | ¿Cómo distinguir error esperado de bug? | "Todo es 'error al prestar'." | `LibroNoDisponibleError` es dominio (métrica/INFO); `AttributeError` sobre `None` es bug (ERROR+alerta). No colapsarlos. |
| 25 | ¿Y si PostgreSQL está caído? | "Devuelvo 'libro no disponible'." | Es un fallo de infraestructura (503) con log/métrica/traza/alerta; no se disfraza de regla de negocio. |
| 26 | ¿Y si el error ocurre tras modificar el libro? | "Hago try/except y lo revierto a mano." | Defino la unidad transaccional; local → transacción; distribuido → outbox/saga/compensación. Primero el problema, luego el patrón. |
| 27 | ¿Cuándo usarías una Saga? | "Siempre en microservicios." | Cuando la operación cruza límites sin transacción ACID viable; para dos tablas en la misma DB sería sobreingeniería. |
| 28 | ¿Excepción de dominio vs. infraestructura? | "Es lo mismo." | Dominio = reglas de negocio; infraestructura = fallos de integración; se traducen entre capas sin contaminar el dominio con el proveedor. |
| 29 | ¿Cómo probarías `ExceptionGroup`? | "No se puede." | `pytest.raises(ExceptionGroup)` + verificar `.exceptions` por tipo; el valor es probar cómo se agregan/clasifican múltiples fallos concurrentes. |
| 30 | ¿Conexión con `asyncio.TaskGroup`? | "Solo sirve para velocidad." | Fan-out concurrente de tools: varios errores ⇒ `ExceptionGroup`; cambia el modelo de error secuencial→concurrente. |
| 31 | ¿Qué ocurre con la cancelación? | "La capturo con `except Exception`." | Cancelación ≠ fallo de negocio; `CancelledError` es `BaseException`; se re-lanza tras limpiar; capturar todo rompe timeouts y shutdown. |
| 32 | ¿Cómo diseñarías los tests? | "Todo E2E." | Pirámide: unit (dominio) → integración (servicio+repo+DB) → E2E (flujo completo), con tests de invariantes. |
| 33 | ¿Qué test tiene más valor? | "El de mayor coverage." | El que protege una propiedad crítica ("nunca dos préstamos activos por ejemplar"); coverage es una señal, no garantía. |
| 34 | ¿Y si crece a millones de usuarios? | "Kubernetes." | Identifico el cuello de botella (búsqueda, DB, locks, conexiones…) y aplico índices, pooling, cache, réplicas, particionado. |
| 35 | ¿Qué cachearías? | "Todo." | Metadata/catálogo; con cuidado disponibilidad/límites (stale data); escrituras críticas contra la fuente transaccional. |

## 4. Banco general (3 básicas + 3 intermedias + 3 avanzadas)

### Básicas

**B1. ¿Qué diferencia hay entre `raise` y `try/except`?**
- Evalúa: fundamentos. **NO:** "son alternativas." **Responder:** `raise` señala/provoca un error; `try/except` define dónde reaccionar; trabajan juntos.
- **Alto impacto:** *"`raise` crea y lanza la condición de fallo; `except` es el punto donde decido qué hacer con ella; entre ambos, la excepción se propaga por la pila."* **Error común:** creer que `except` "evita" el error sin decidir nada.

**B2. ¿Qué significa `pass` en una excepción personalizada?**
- **NO:** "que no hace nada." **Responder:** no agrega comportamiento, hereda el de `Exception`; agrega semántica.
- **Alto impacto:** *"La clase hereda todo de `Exception`; lo que aporto es un tipo distinto que comunica el problema."* **Error común:** pensar que la excepción queda "vacía/inútil".

**B3. ¿Por qué `raise` va fuera del `for` en `buscar_usuario`?**
- **NO:** "por estilo." **Responder:** solo tras revisar todos los elementos se sabe que no existe; dentro del `for` falla en el primer no coincidente.
- **Alto impacto:** *"Ausencia solo se concluye tras recorrer toda la colección; un `raise` dentro del bucle es un bug lógico."* **Error común:** poner el `else`/`raise` en el nivel equivocado.

### Intermedias

**I1. ¿Cuándo `None` y cuándo excepción?** *(ver P4)*
- **Trade-offs:** `None` fuerza chequeos en cada llamada; excepciones ocultan el fallo en la firma. **Follow-up:** ¿y `Result`? **Red flags:** "siempre excepciones".

**I2. ¿Por qué validar antes de mutar estado?**
- **NO:** "por orden." **Alto impacto:** *"Si muto y luego falla la validación, dejo estado parcial e inconsistente; las precondiciones se comprueban antes, y la mutación va agrupada al final o dentro de una transacción."* **Follow-up:** ¿y si falla la mutación a la mitad? → atomicidad/rollback. **Red flags:** "ya lo revertiré a mano".

**I3. ¿Por qué separar dominio y presentación?**
- **Alto impacto:** *"El dominio lanza errores semánticos; la interfaz decide cómo mostrarlos. Así el mismo caso de uso sirve a CLI, REST, MCP o agentes."* **Trade-off:** más capas a cambio de reutilización; en un script de 50 líneas no compensa. **Red flags:** `print` en el dominio.

### Avanzadas

**A1. Diseña la garantía de no-doble-préstamo bajo múltiples instancias.** *(ver P3 y §V.5)*
- **Follow-up Staff:** ¿cómo cambia con sharding por `libro_id`? → el invariante queda local a un shard si se enruta por ejemplar; el límite por usuario cruza shards ⇒ replantear (contador por usuario en su propio shard, reservas/saga).

**A2. ¿Cómo evitas duplicados cuando un agente reintenta la tool?** *(idempotencia)*
- **Alto impacto:** *"Clave de idempotencia por intención, persistida en la misma transacción que el efecto; misma clave + mismo payload devuelve el resultado original; payload distinto se rechaza. TTL y scope por usuario."* **Follow-up:** ¿qué pasa si la primera petición sigue en curso? → 409/espera con estado `in_progress`. **Red flags:** confiar en la memoria del agente.

**A3. ¿Cómo diseñas el error handling de un agente que llama varias tools en paralelo?**
- **Alto impacto:** *"TaskGroup con timeouts; los fallos se agregan en `ExceptionGroup`, los clasifico con `except*` en transitorios (reintento acotado), permanentes (informar al modelo con error estructurado) y bugs (abortar+alerta). La cancelación se propaga, no se traga."* **Follow-up:** degradación parcial: ¿respondes con resultados parciales? → depende del contrato; marcar explícitamente qué faltó.

## 5. Evolución de una misma respuesta (Junior → Staff)

Pregunta: *"¿Cómo evitas que dos personas presten el último libro?"*

| Nivel | Respuesta |
|---|---|
| Junior | "Pongo `disponible = False` cuando lo presten." |
| Mid | "Uso una transacción y un lock en la fila del libro." |
| Senior | "El invariante es un préstamo activo por ejemplar; lo garantizo con transacción y un índice único parcial, traduzco la violación a `LibroNoDisponibleError` y pruebo con solicitudes concurrentes." |
| Staff | "Además de lo anterior: defino SLO y métrica `duplicate_active_loans`, política de retry para 40001, idempotencia para reintentos de clientes y agentes, y un plan de evolución (sharding/reservas) si la contención por título populares se vuelve un cuello de botella." |

## 6. System Design Interview — respuesta modelo

> **"Diseña un sistema de préstamos usado por web, móvil y un AI Agent. Debe impedir préstamos duplicados, respetar el límite por usuario, soportar reintentos y sobrevivir a múltiples instancias del backend."**

*"Primero defino invariantes y fuente de verdad: no confío en estado en memoria porque hay varias instancias. Disponibilidad y límite se garantizan bajo concurrencia en la capa persistente con transacción y el mecanismo adecuado (índice único parcial + lock/update condicional). Expongo el caso de uso como servicio de aplicación reutilizable por REST, móvil y MCP. Los errores de dominio se traducen a un contrato externo estable (RFC 9457). Las escrituras son idempotentes. Añado observabilidad (logs estructurados, métricas, trazas) y limito los retries a fallos transitorios sobre operaciones seguras de repetir. El agente pide la acción; las reglas viven en el backend."*

Cierre con: cuellos de botella (contención por título popular), fallos (DB caída → 503), seguridad (authn/authz, PII), coste, y evolución (outbox para notificaciones, réplicas de lectura para catálogo).

## 7. Tech Lead / Leadership Interview (respuestas con escenarios hipotéticos)

| Pregunta | Marco de respuesta |
|---|---|
| ¿Cómo resolverías un desacuerdo arquitectónico (Repository sí/no)? | Aclarar el problema real; comparar opciones con criterios explícitos (volatilidad, testing, coste); prototipo/spike acotado; ADR; decisión reversible con condición de revisión |
| ¿Cómo defenderías la decisión de añadir idempotencia? | Riesgo (duplicados) × impacto × coste; evidencia (incidente o test que lo reproduce); alternativa más barata descartada y por qué |
| ¿Cómo priorizarías deuda técnica (estado en memoria, excepciones genéricas)? | Impacto en invariantes y en incidentes > estética; deuda que bloquea seguridad/consistencia primero |
| ¿Cómo manejarías un incidente de doble préstamo? | Contener → diagnosticar con trazas/logs → corregir datos afectados → fix + test de invariante → postmortem sin culpables → prevención |
| ¿Cómo convencerías a otros equipos de adoptar el error contract? | Mostrar coste actual (parseo de mensajes), piloto con un consumidor, guía y librería compartida, versionado |
| ¿Cómo equilibrarías velocidad y calidad? | Calidad no negociable en invariantes/seguridad; velocidad en lo reversible; medir |
| ¿Cómo decidirías con información incompleta? | Decisiones reversibles primero, explicitar supuestos, definir qué métrica te haría cambiar |
| ¿Cómo mentorearías a otro Senior? | Preguntas guía sobre invariantes y fallos; revisión de diseño, no solo de código |
| ¿Cómo construirías consenso? | Doc corto con opciones, trade-offs y recomendación; sesión de revisión; ADR |
| ¿Cuándo escalarías un problema? | Cuando el riesgo excede mi autoridad/plazo o cruza equipos; llevar datos y opciones, no solo el problema |

*(No inventar experiencias propias; usar escenarios hipotéticos declarados como tales.)*

## 8. Red flags en entrevista

```text
"Uso try/except para todos los errores."          "Le pongo un booleano disponible."
"Si hay concurrencia uso un lock."                "Uso Kubernetes para escalar."
"Uso Redis para solucionar la concurrencia."      "El agente AI decide si puede prestar."
"El LLM valida el límite."                        "Siempre uso Repository Pattern."
"Siempre uso microservicios."                     "Siempre hago retry."
"Si falla hago except Exception."                 "Si no existe devuelvo un mensaje."
"Con 100% de coverage ya está probado."
```
Problema común: **presentan una herramienta como solución sin definir primero el problema que debe resolver.**

## 9. Banco rápido (respuestas de una línea)

1. *¿Excepción específica vs mensaje?* → Permite manejo semántico sin depender del texto.
2. *¿Por qué no `Exception` en todos lados?* → Oculta imprevistos y mezcla categorías con estrategias de recuperación distintas.
3. *¿Qué protege una transacción?* → Atomicidad y consistencia de cambios que se confirman o revierten como unidad.
4. *¿Race condition?* → Resultado dependiente del orden/interleaving de operaciones concurrentes.
5. *¿Check-then-act?* → Validación separada de la mutación dependiente; ventana para otro actor.
6. *¿Idempotencia?* → Repetir la operación no produce efectos adicionales tras el primer éxito.
7. *¿Dominio y HTTP?* → El dominio no conoce el transporte.
8. *¿Error Contract?* → Representación estable y explícita de errores para consumidores externos.
9. *¿LLM y reglas de negocio?* → El LLM es probabilístico; las invariantes críticas son deterministas.
10. *¿MCP y lógica del préstamo?* → MCP es frontera de integración; la lógica vive en el caso de uso.

---

# PARTE VII — EJERCICIOS PRÁCTICOS

**Ejercicio 1 (base del curso).** Implementa `buscar_usuario`, `UsuarioNoEncontrado`, `libros_disponibles` y el flujo de `main.py`. Verifica: `raise` fuera del `for`, `print` solo en éxito, `except` específico, sin estado duplicado.

**Ejercicio 2 (jerarquía y convención).** Refactoriza a `BibliotecaError` + sufijo `Error`, con `code` y `retryable`. Escribe tests con `pytest.raises` que verifiquen tipo, `code` y datos.

**Ejercicio 3 (validar antes de mutar).** Añade `prestar()` con las 4 precondiciones. Test: si falla el límite, `libro.disponible` sigue `True`.

**Ejercicio 4 (índice y traducción).** Cambia a `dict` por cédula; traduce `KeyError` → `UsuarioNoEncontradoError` con `from exc` y comprueba `__cause__`.

**Ejercicio 5 (persistencia).** Modela `usuarios`, `ejemplares`, `prestamos` en PostgreSQL con `UNIQUE INDEX` parcial y `CHECK`. Implementa un `PrestamoRepository`; traduce `UniqueViolation` → `LibroNoDisponibleError`.

**Ejercicio 6 (concurrencia).** Test con `ThreadPoolExecutor` (8 hilos, 1 ejemplar) ⇒ exactamente 1 éxito. Cambia la estrategia (`FOR UPDATE` vs update condicional) y compara.

**Ejercicio 7 (idempotencia).** Añade `Idempotency-Key` con tabla `idempotency_keys`; test: mismo request dos veces ⇒ un solo préstamo y la misma respuesta.

**Ejercicio 8 (FastAPI).** Endpoint `POST /prestamos`, exception handler que devuelve `application/problem+json` (RFC 9457), y pruebas de 404/409/422.

**Ejercicio 9 (async).** Tres tools concurrentes con `TaskGroup`; una lanza `TimeoutError` y otra `ValueError`. Maneja con `except*` y clasifica reintentable/no reintentable. Añade `asyncio.timeout()`.

**Ejercicio 10 (agente).** Envuelve `prestar_libro` como tool (o servidor MCP) devolviendo error estructurado; escribe tests que simulen el reintento del orquestador y verifiquen que no duplica.

**Ejercicio 11 (observabilidad).** Sustituye `print` por logging estructurado con `trace_id`, enmascara la cédula y expón la métrica `prestamos_rechazados_total{code}`.

**Ejercicio 12 (Design Review).** Aplica el checklist de §VIII.5 al sistema y redacta un ADR sobre Repository sí/no.

---

# PARTE VIII — RECURSOS, GLOSARIO, CHECKLISTS Y CIERRE

## 1. Recursos oficiales (prioridad máxima)

| Tema | Recurso |
|---|---|
| Python — excepciones incorporadas | https://docs.python.org/3/library/exceptions.html |
| Python Tutorial — Errors and Exceptions | https://docs.python.org/3/tutorial/errors.html |
| PEP 3134 — Exception Chaining | https://peps.python.org/pep-3134/ |
| PEP 654 — Exception Groups y `except*` | https://peps.python.org/pep-0654/ |
| PEP 678 — `add_note()` | https://peps.python.org/pep-0678/ |
| PEP 758 — `except`/`except*` sin paréntesis | https://peps.python.org/pep-0758/ |
| PEP 765 — control de flujo en `finally` | https://peps.python.org/pep-0765/ |
| PEP 8 — estilo (imports, nombres, sufijo `Error`) | https://peps.python.org/pep-0008/ |
| asyncio — Tasks y TaskGroup | https://docs.python.org/3/library/asyncio-task.html |
| What's New in Python 3.14 | https://docs.python.org/3/whatsnew/3.14.html |
| pytest | https://docs.pytest.org/ |
| FastAPI — Handling Errors | https://fastapi.tiangolo.com/tutorial/handling-errors/ |
| PostgreSQL — Transactions | https://www.postgresql.org/docs/current/tutorial-transactions.html |
| PostgreSQL — Concurrency Control (MVCC) | https://www.postgresql.org/docs/current/mvcc.html |
| PostgreSQL — Transaction Isolation | https://www.postgresql.org/docs/current/transaction-iso.html |
| Python logging | https://docs.python.org/3/library/logging.html |
| typing / dataclasses | https://docs.python.org/3/library/typing.html · https://docs.python.org/3/library/dataclasses.html |
| RFC 9457 — Problem Details for HTTP APIs | https://www.rfc-editor.org/rfc/rfc9457 |
| Model Context Protocol | https://modelcontextprotocol.io |
| OpenTelemetry | https://opentelemetry.io |
| OWASP GenAI Security Project | https://genai.owasp.org |

> Las URLs de PEPs/RFCs y documentación se corresponden con las fuentes oficiales estándar; **[Verificar]** en cada caso la versión vigente (Python 3.14 es el ancla; PostgreSQL, MCP, OpenTelemetry GenAI y OWASP evolucionan por revisiones).

## 2. Glosario técnico de alto ROI

| Término | Definición |
|---|---|
| **Exception** | Objeto que representa una condición excepcional durante la ejecución; contiene información estructurada, no solo un mensaje |
| **Exception hierarchy** | Jerarquía de clases que permite capturar por categoría o por caso |
| **Exception propagation** | La excepción sube por la cadena de llamadas hasta hallar un handler compatible |
| **Error boundary** | Punto donde una excepción se captura *deliberadamente* (logging, mapping, respuesta, retry, fallback); no "capturar todo" |
| **Domain Error** | Error que representa una regla o condición del negocio |
| **Infrastructure Error** | Fallo de una dependencia técnica (DB, HTTP, filesystem, cola) |
| **Error Translation** | Convertir un error técnico en uno apropiado para la capa superior |
| **Error Contract** | Contrato estructurado y estable con el que un sistema comunica errores |
| **Problem Details (RFC 9457)** | Formato estándar `application/problem+json` para errores HTTP |
| **Invariant** | Propiedad que debe permanecer verdadera durante toda la ejecución |
| **Precondition / Postcondition** | Condición previa/posterior a una operación |
| **State transition** | Cambio controlado de estado (`AVAILABLE → BORROWED`) |
| **Race condition** | Resultado dependiente del orden de operaciones concurrentes |
| **Check-then-act** | Validación separada de la mutación que depende de ella |
| **Atomicity** | Todo o nada |
| **Transaction** | Unidad de trabajo con `commit`/`rollback` |
| **Isolation level** | Grado de aislamiento entre transacciones concurrentes |
| **Serialization failure (40001)** | Aborto por conflicto en aislamiento serializable; requiere retry |
| **Optimistic / Pessimistic locking** | Detectar conflictos al escribir vs. bloquear antes de escribir |
| **Partial unique index** | Índice único con `WHERE` que impone una invariante condicional |
| **Idempotency** | Repetir una operación no genera efectos adicionales |
| **Retry / Backoff / Jitter** | Reejecución tras fallo / espera creciente / variación aleatoria |
| **Retry storm** | Reintentos simultáneos que agravan una caída |
| **Timeout** | Límite máximo de espera |
| **Circuit breaker** | Corta el envío de requests a una dependencia que falla (`CLOSED/OPEN/HALF-OPEN`) |
| **Partial failure** | Una parte del sistema funciona y otra falla |
| **Outbox / Saga** | Publicación atómica de eventos / coordinación con compensaciones entre servicios |
| **ExceptionGroup / `except*`** | Contenedor de varias excepciones concurrentes / sintaxis para manejarlas por tipo |
| **Coroutine / `async def`** | Operación asíncrona esperable con `await` / función que la define (no es una clase) |
| **Task / TaskGroup** | Unidad de ejecución de asyncio / concurrencia estructurada |
| **Cancellation** | Solicitud de detener una operación asíncrona; ≠ fallo de negocio |
| **Free-threaded Python** | Build sin GIL con paralelismo real (PEP 779) |
| **Aliasing mutable** | Varias referencias al mismo objeto mutable |
| **God Object** | Clase que acumula responsabilidades ajenas |
| **Tool / Tool calling** | Capacidad invocable por un modelo / mecanismo por el que el modelo solicita ejecutarla |
| **MCP** | Protocolo abierto para conectar modelos con herramientas y contexto; frontera de integración |
| **Agent** | Modelo + capacidad de decidir, usar tools, observar resultados y continuar |
| **Context engineering** | Diseño de qué información entra al contexto del modelo, cuándo y cómo |
| **Prompt injection (indirecta)** | Instrucciones maliciosas dentro de contenido que el modelo procesa |
| **Excessive agency** | Dar a un agente más permisos/autonomía de los necesarios |
| **Deterministic boundary** | Parte del sistema donde las decisiones críticas las toma lógica determinista |
| **ADR** | Architecture Decision Record: decisión, contexto, alternativas y consecuencias |

## 3. Mapa conceptual completo

```text
PYTHON → funciones/objetos → EXCEPCIONES (raise · try/except) → PROPAGACIÓN
   → ERROR BOUNDARY → ERROR TRANSLATION → ERROR CONTRACT (RFC 9457)
   → API / Worker (HTTP · retry/DLQ) → APPLICATION → DOMAIN (reglas · estado · invariantes)
   → PERSISTENCIA (transacción · concurrencia · constraints) → IDEMPOTENCIA → OBSERVABILIDAD
   → AI (LLM · Tools · MCP · Agents) → EVALUACIÓN · SEGURIDAD · COSTE
```

## 4. Checklist de dominio — Senior AI Engineer

**Python:** [ ] qué es una excepción · [ ] `Exception` vs `BaseException` · [ ] `raise` · [ ] `try/except/else/finally` · [ ] propagación · [ ] `as e` (`e`, `str(e)`, `type(e)`) · [ ] excepciones personalizadas y jerarquías · [ ] chaining (`from`) · [ ] PEP 758/765 en 3.14 · [ ] `add_note()`.

**Diseño:** [ ] detectar vs manejar vs presentar · [ ] dónde capturar · [ ] `None` vs excepción vs `Result` · [ ] error boundary · [ ] error translation · [ ] error contract.

**Producción:** [ ] retry · [ ] cuándo es peligroso · [ ] backoff/jitter/retry storm · [ ] idempotencia · [ ] timeout · [ ] circuit breaker · [ ] partial failure.

**Sistemas distribuidos:** [ ] race condition · [ ] check-then-act · [ ] atomicidad · [ ] transacción · [ ] invariante · [ ] proteger una operación concurrente · [ ] fallo parcial · [ ] aislamiento y `40001`.

**Async:** [ ] `def` vs `async def` · [ ] coroutine · [ ] `await` · [ ] Task · [ ] `TaskGroup` · [ ] cancelación · [ ] `ExceptionGroup` y `except*` · [ ] `asyncio.timeout`.

**AI Engineering:** [ ] tool · [ ] tool calling · [ ] idempotencia en tools · [ ] el LLM no controla invariantes · [ ] dónde encaja MCP · [ ] agente ↔ backend determinista · [ ] propagar errores estructurados a un agente · [ ] prompt injection y excessive agency · [ ] evaluación y observabilidad de agentes.

**Meta (checklist estándar):** [ ] explico con mis palabras · [ ] explico cómo funciona · [ ] lo implemento · [ ] cuándo usarlo / cuándo NO · [ ] comparo alternativas · [ ] trade-offs · [ ] diagnostico problemas · [ ] lo relaciono con producción · [ ] lo relaciono con AI Engineering · [ ] defiendo una decisión técnica · [ ] respondo preguntas de entrevista.

## 5. Checklist de Design Review (para cualquier sistema futuro)

```text
 1. ¿Cuál es la operación?                     12. ¿Puede ocurrir concurrencia?
 2. ¿Cuál es el estado?                         13. ¿La operación es idempotente?
 3. ¿Cuál es la fuente de verdad?               14. ¿Qué pasa si se ejecuta dos veces?
 4. ¿Cuáles son las invariantes?                15. ¿Qué pasa si falla a mitad?
 5. ¿Qué puede fallar?                          16. ¿Cómo se hace rollback?
 6. ¿Qué errores son esperados?                 17. ¿Qué se registra (y qué NO)?
 7. ¿Cuáles son inesperados?                    18. ¿Cómo se prueba?
 8. ¿Dónde se manejan?                          19. ¿Qué ocurre bajo carga?
 9. ¿Cuáles deben propagarse?                   20. ¿Qué ocurre si una dependencia externa falla?
10. ¿Hay traducción entre capas?                21. ¿Qué ocurre si un AI Agent ejecuta la operación?
11. ¿Existe un Error Contract?
```

## 6. Checklist de producción del caso de uso

**Datos:** [ ] formato de cédula · [ ] restricción de unicidad · [ ] índices · [ ] fuente persistente · [ ] integridad referencial.
**Dominio:** [ ] usuario no encontrado · [ ] libro no disponible · [ ] límite · [ ] invariantes explícitas · [ ] estado consistente.
**Persistencia:** [ ] transacciones · [ ] rollback · [ ] concurrencia considerada · [ ] estrategia ante conflictos.
**API:** [ ] validación de entrada · [ ] error contract (RFC 9457) · [ ] códigos estables · [ ] mensajes seguros · [ ] status coherentes.
**Reliability:** [ ] timeouts · [ ] retry donde corresponde · [ ] idempotencia · [ ] protección contra duplicados · [ ] fallos parciales.
**Observabilidad:** [ ] logs estructurados · [ ] trace ID · [ ] métricas · [ ] error rate · [ ] latencia · [ ] alertas.
**Seguridad:** [ ] authn · [ ] authz · [ ] PII · [ ] SQL parametrizado · [ ] sin stack traces al cliente · [ ] secretos fuera del código.
**AI:** [ ] tools sin lógica de negocio duplicada · [ ] agentes no autorizan operaciones críticas · [ ] tools idempotentes · [ ] errores estructurados · [ ] MCP como frontera · [ ] mínimo privilegio y HITL para efectos irreversibles.

## 7. Qué NO necesitas memorizar todavía

Todos los tipos de `Exception`, todos los métodos de `asyncio`, todos los códigos HTTP, todas las estrategias de locking, todos los patrones distribuidos, todas las clases de PostgreSQL. Objetivo actual: **razonar**. Ante un problema nuevo: identificar categoría → consultar documentación oficial → seleccionar mecanismo → probar → observar. Un Senior no memoriza la API: sabe encontrar la documentación correcta y evaluar la solución.

## 8. Conexión con las próximas clases

```text
EXCEPCIONES → FASTAPI → ERROR CONTRACTS → DATABASES → TRANSACTIONS → ASYNC → CONCURRENCY
→ LLM APIs → structured outputs → tool calling → RAG → AI AGENTS → MCP → LLMOPS
```
Un sistema AI real debe manejar timeouts, rate limits, fallos de proveedor, argumentos de tool inválidos, fallos de retrieval, fallos parciales de tools, tool calls duplicados, caídas de base de datos, fallos de red y cancelación.

## 9. Reglas finales para recordar

> **No se trata de capturar excepciones. Se trata de diseñar cómo falla el sistema.**

> **No se trata de mostrar un mensaje de error. Se trata de preservar semántica, contexto, seguridad y capacidad de recuperación.**

> **No se trata de validar que un libro está disponible. Se trata de garantizar que la disponibilidad siga siendo verdadera bajo concurrencia.**

> **No se trata de agregar IA al sistema. Se trata de permitir que un modelo solicite acciones sin entregarle el control de las invariantes críticas.**

> **No se trata de memorizar `ExceptionGroup`, `TaskGroup`, retries o transacciones. Se trata de reconocer el tipo de fallo que enfrentas y elegir el mecanismo adecuado para mantener el sistema correcto.**

> **No se trata de agregar arquitectura porque sí. Se trata de introducir complejidad solo cuando resuelve un problema real.**

> **No se trata de conocer patrones (Repository, Saga, Circuit Breaker). Se trata de reconocer el problema que justifica cada uno y saber cuándo NO utilizarlo.**

## 10. Cierre

La progresión alcanzada:

```text
Python básico → POO → Modularización → Excepciones → Diseño de errores → Testing
→ Concurrencia → Persistencia → APIs → AI Tools → Agents → MCP
```

La disciplina con la que proteges *"un libro no puede prestarse dos veces"* es la misma que protege *"un pago no puede ejecutarse dos veces"*, *"una tool no puede duplicar un efecto"*, *"un documento no puede indexarse incorrectamente"*, *"un agente no puede ejecutar una operación sin autorización"* y *"una transacción no puede dejar estado inconsistente"*. Ese es el salto entre **aprender Python** y **pensar como Software Engineer / AI Engineer**: construir sistemas que **fallan de forma controlada, son observables, testeables, escalables y seguros de integrar con IA**.

**Estado de dominio esperado:**
```text
✓ Exceptions  ✓ Custom Exceptions  ✓ Hierarchy  ✓ Propagation  ✓ Error Boundaries
✓ Error Translation  ✓ Error Contracts  ✓ Testing  ✓ Transactions  ✓ Concurrency
✓ Idempotency  ✓ Async  ✓ ExceptionGroup  ✓ TaskGroup  ✓ Tools  ✓ MCP  ✓ AI Agents  ✓ Observability
```

**Siguiente bloque:** Python → APIs/FastAPI → async/await → HTTP clients → LLM APIs → structured outputs → tool calling → RAG → AI Agents.

---
*Fin de la clase. Documento consolidado: contenido original preservado, redundancias fusionadas, actualizado al 30 de agosto de 2026. Los ítems marcados **[Verificar]** dependen de versiones y estados que cambian con frecuencia: confirmar en la fuente oficial antes de citarlos.*