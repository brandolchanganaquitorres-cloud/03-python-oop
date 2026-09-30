# CLASE 10 — MANEJO PROFESIONAL DE ERRORES Y EXCEPCIONES EN PYTHON
## Senior AI Engineer · Senior Software Engineer · Tech Lead · Staff Engineer

> **Nota de reorganización:** este documento consolida las 4 partes originales de la clase, elimina redundancias (el mismo diagrama de `try/except`, la misma advertencia sobre `except Exception`, y el mismo flujo de retry/backoff aparecían repetidos en varias partes) y verifica el estado del arte de Python a **septiembre de 2026**.
>
> **Verificación de vigencia (septiembre 2026):** Python 3.14 sigue siendo la serie estable actual (última bugfix: 3.14.7, agosto 2026); Python 3.15 está en fase de pre-lanzamiento con release planeado para octubre de 2026. Todo el contenido basado en `add_note()`, `ExceptionGroup`/`except*`, exception chaining y el comportamiento de `finally` sigue vigente sin cambios respecto al material original.
>
> **Ampliaciones de esta revisión:** testing ejecutable de excepciones con `pytest` (§14.1-§15), formalización del Error Contract como artefacto independiente de la clase de excepción (§16.1), separación explícita en cinco capas Exception → Application Error → API Contract → HTTP Status → User Message (§16.2), y ocho casos límite adicionales de Python (§22.1).
>
> **Segunda ronda de ampliaciones:** taxonomía unit/integration/contract test aplicada a errores con ejemplo de contract test (§15.6); conexión breve con MCP como transporte adicional de tool failures (§17); marcado explícito de §22.1 como "conocer, no memorizar" para no competir en prioridad con retry/idempotencia/observabilidad; y robustecimiento del Error Contract con resolución de mapeo por jerarquía (`__mro__`) y política explícita de estabilidad de campos (§16.1).

---

# ÍNDICE

```text
PARTE 1 — Fundamentos y explicación técnica
PARTE 2 — Profesionalización y producción
PARTE 3 — Arquitectura y Tech Lead thinking
PARTE 4 — Entrevistas, recursos, glosario y consolidación
```

---

# PARTE 1 — FUNDAMENTOS Y EXPLICACIÓN TÉCNICA

## 1. Objetivo de la clase

Dominar las excepciones no como sintaxis de `try/except`, sino como un **mecanismo de diseño** para:

```text
representar fallos
      +
controlar su propagación
      +
preservar causalidad
      +
definir límites de responsabilidad
      +
convertir errores técnicos en respuestas apropiadas
```

El criterio de éxito no es "sé usar `try/except`", sino:

> **Sé decidir dónde se detecta un error, quién tiene contexto para manejarlo, cómo se propaga, cómo se traduce entre capas y qué le cuesta al sistema manejarlo mal.**

---

## 2. Contenido del material fuente

El material parte de un caso mínimo de dominio (una biblioteca):

```python
estudiante.solicitar_libro(None)
```

Si no existe validación adecuada, el sistema puede terminar reportando un préstamo autorizado con un título inválido. El curso usa este caso para introducir `raise`, `try/except`, excepciones personalizadas, jerarquía de excepciones y traceback. Este documento conserva ese recorrido, reorganizado como una progresión única de causa → efecto.

---

## 3. Conceptos fundamentales

```text
Entrada inválida
      ↓
¿Dónde se detecta?
      ↓
¿Cómo se representa?
      ↓
¿Cómo se propaga?
      ↓
¿Quién tiene contexto para decidir?
      ↓
¿Se recupera, se traduce o se registra?
```

Este flujo es el hilo conductor de toda la clase. Cada mecanismo de Python (`raise`, `except`, jerarquías, `from`, `finally`) es una herramienta para responder una de estas preguntas, no un fin en sí mismo.

---

## 4. Explicación técnica

### 4.1 Fallo silencioso vs. fallo explícito

```text
titulo = None
    ↓
validación incorrecta o inexistente
    ↓
"Préstamo autorizado"   ← resultado incorrecto sin aviso
```

Esto es un **silent failure**: el sistema continúa como si nada hubiera ocurrido, produciendo un resultado que parece válido pero no lo es.

Alternativa:

```text
titulo = None
    ↓
validación
    ↓
raise TituloInvalidoError
    ↓
el flujo normal se interrumpe
    ↓
un nivel superior decide qué hacer
```

### Principio senior

> **Es preferible fallar explícitamente que producir un resultado aparentemente válido que en realidad es incorrecto.**

---

### 4.2 `raise`: producir una excepción deliberadamente

```python
def solicitar_libro(self, titulo: str) -> str:
    if not titulo:
        raise ValueError("Título inválido")

    print("Esta línea no se ejecutará")
    return "Préstamo autorizado"
```

Al ejecutarse `raise`, la función se interrumpe inmediatamente y Python busca un manejador compatible remontando la cadena de llamadas:

```text
main()
  │
  ▼
solicitar_libro()
  │
  ▼
validar_titulo()
  │
  ▼
raise TituloInvalidoError
  │
  ▼
¿hay except compatible aquí?
  │
  ├── sí → manejar
  │
  └── no → subir al caller y repetir la búsqueda
```

Si ningún nivel captura la excepción, se convierte en una excepción no manejada y el programa termina con un traceback. La documentación oficial de Python describe exactamente este comportamiento de propagación hasta encontrar un `except` compatible o terminar la ejecución. [1]

### Precisión importante

> `raise` **no** significa "terminar el programa". Significa "interrumpir el contexto actual y comenzar la propagación hasta encontrar un manejador". Si existe un `except` compatible, la ejecución continúa normalmente después del bloque `try/except`. [1]

```python
try:
    estudiante.solicitar_libro(None)
except BibliotecaError:
    print("Solicitud rechazada")

print("El programa continúa")   # esto sí se ejecuta
```

---

### 4.3 Detectar un error ≠ manejar un error

Esta es la distinción más importante de toda la clase, y la razón de ser de casi todas las reglas de diseño que siguen.

```text
Domain
  │
  │ detecta (sabe QUE algo es inválido)
  ▼
raise DomainError
  │
  ▼
Application layer
  │
  │ decide (sabe QUÉ hacer con eso)
  ▼
Interface / API
  │
  ▼
respuesta apropiada
```

`Estudiante` conoce la regla "el título debe ser válido", pero probablemente no debe decidir qué mensaje mostrar, qué HTTP status devolver, si reintentar o cómo registrar el error.

### Regla

> **La capa que detecta un error no necesariamente es la capa que tiene suficiente contexto para manejarlo.**

Este principio reaparece en la Parte 2 (producción) y en la Parte 4 (entrevistas) como el criterio central para decidir *dónde* poner un `except`.

---

### 4.4 `try` / `except`

```text
             ┌──────────────────┐
             │       try         │
             │ operación         │
             │ potencialmente    │
             │ fallida           │
             └────────┬──────────┘
                      │
                 ¿excepción?
                  /       \
                NO         SÍ
                │           │
                ▼           ▼
          continúa       except
                            │
                            ▼
                         manejo
```

La ventaja arquitectónica: el componente que **detecta** el problema no tiene que ser el que **decide** cómo recuperarse.

---

### 4.5 `ValueError`: cuándo es suficiente y cuándo no

`ValueError` es apropiado cuando el tipo del argumento es correcto pero su contenido no es válido para la operación:

```python
def set_age(age: int) -> None:
    if age < 0:
        raise ValueError("Age cannot be negative")
```

En una aplicación con reglas de dominio propias, `ValueError` puede quedarse corto semánticamente:

```python
raise ValueError("Libro no disponible")   # genérico, pierde significado de dominio
raise LibroNoDisponibleError(...)          # expresa el contrato del dominio
```

---

### 4.6 Excepciones personalizadas y jerarquía

```python
class BibliotecaError(Exception):
    """Base para errores del dominio de biblioteca."""


class TituloInvalidoError(BibliotecaError):
    """El título proporcionado no es válido."""


class LibroNoDisponibleError(BibliotecaError):
    """El libro no puede prestarse en su estado actual."""
```

```text
Exception
    │
    ▼
BibliotecaError
    │
    ├── TituloInvalidoError
    ├── LibroNoDisponibleError
    └── LimitePrestamosError
```

Un consumidor puede manejar una condición específica o toda la familia:

```python
except LibroNoDisponibleError:
    ofrecer_otro_libro()
```

```python
except BibliotecaError:
    registrar_error_de_negocio()
```

### Regla senior

> **Una jerarquía de excepciones debe representar categorías de comportamiento o recuperación, no simplemente nombres distintos para cada error imaginable.**

No es obligatorio crear `TituloVacioError`, `TituloNoneError`, `TituloStringInvalidoError` si las tres reciben exactamente el mismo tratamiento. Cada tipo nuevo cuesta: más documentación, más tests, más superficie de API. La pregunta correcta:

> **¿Este tipo comunica una distinción que algún consumidor realmente necesita?**

---

### 4.7 `pass` en una excepción — y cuándo transportar datos

```python
class BibliotecaError(Exception):
    pass
```

`pass` solo crea un tipo semánticamente significativo — ya es útil por sí mismo. Cuando el consumidor necesita datos estructurados para recuperación o logging, la excepción puede transportarlos:

```python
class LibroNoDisponibleError(BibliotecaError):
    def __init__(self, titulo: str) -> None:
        self.titulo = titulo
        super().__init__(f"El libro '{titulo}' no está disponible")
```

```python
except LibroNoDisponibleError as exc:
    print(exc.titulo)
```

---

### 4.8 Orden de los `except`: de específico a general

```python
except TituloInvalidoError:   # más específico primero
    ...
except BibliotecaError:       # más general después
    ...
```

Invertir el orden hace que la cláusula específica quede inalcanzable, porque `TituloInvalidoError` **es** una `BibliotecaError` y ya fue capturada por el primer `except` general. Python evalúa las cláusulas `except` en orden y solo ejecuta la primera que coincide. [1]

```text
más específico
      ↓
más general
```

Evitar además:

```python
except BibliotecaError as e:
    if type(e) == TituloInvalidoError:   # anti-patrón
        ...
```

La jerarquía existe para usarse por polimorfismo (`except TituloInvalidoError:`), no para reimplementarse con `if type(e) == ...`.

---

### 4.9 Por qué `except:` desnudo es una mala práctica

```python
try:
    ...
except:
    print("Algo salió mal")
```

Oculta indistintamente: bugs de programación, errores de configuración, errores de red, de validación, de infraestructura e incluso interrupciones del usuario. La documentación de Python recomienda ser lo más específico posible y dejar que las excepciones inesperadas se propaguen. [1]

`except Exception:` es más acotado que `except:`, pero sigue siendo peligroso si se usa como estrategia universal — se trata como problema de producción en la Parte 2.

---

### 4.10 `Exception` vs. `BaseException`

```text
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
     ├── ValueError
     ├── TypeError
     ├── OSError
     └── excepciones propias de aplicación
```

Las excepciones de aplicación deben heredar de `Exception`, no de `BaseException` directamente. Esto mantiene separadas las excepciones normales de mecanismos de control como `SystemExit` o `KeyboardInterrupt`. La documentación oficial mantiene esta jerarquía vigente en la serie 3.14. [2]

Consecuencia práctica: `except Exception:` **no** captura `KeyboardInterrupt` ni `SystemExit` — están fuera de esa rama.

---

## 5. Funcionamiento interno — Deep Dive: el traceback

Cuando una excepción no es manejada, Python emite un traceback que permite reconstruir:

```text
1. qué excepción ocurrió y con qué mensaje
2. dónde se lanzó (raise)
3. qué cadena de llamadas llevó hasta ahí
```

### Cómo leerlo profesionalmente

```text
Paso 1 — Ir al final
    ExceptionType: message

Paso 2 — Encontrar el punto de lanzamiento
    la línea donde aparece raise

Paso 3 — Reconstruir la cadena
    ¿quién llamó esta función?
    ¿quién llamó al caller?

Paso 4 — Buscar la causa real
    el último error del traceback no siempre es
    el primer problema conceptual (p. ej. un TypeError
    puede ser solo la consecuencia de un dato inválido
    varias capas antes)
```

El módulo `traceback` de la biblioteca estándar permite imprimir o inspeccionar programáticamente esta información; la documentación actual de Python 3.14 lo mantiene junto con el soporte de notas de excepción (`add_note()`) y grupos de excepciones. [3]

---

## 6. Código y ejemplos — diseño recomendado

```text
project/
├── main.py
├── biblioteca.py
├── libros.py
├── usuarios.py
└── exceptions.py
```

`exceptions.py`:

```python
class BibliotecaError(Exception):
    """Excepción base del dominio de biblioteca."""


class TituloInvalidoError(BibliotecaError):
    """Se lanza cuando el título de un libro no es válido."""


class LibroNoDisponibleError(BibliotecaError):
    """Se lanza cuando el libro no está disponible para préstamo."""
```

`usuarios.py`:

```python
from exceptions import TituloInvalidoError


class Estudiante:
    def solicitar_libro(self, titulo: str) -> str:
        if not titulo:
            raise TituloInvalidoError(
                f"El libro con el título {titulo!r} no es válido"
            )

        return "Préstamo autorizado"
```

`libros.py` — excepciones e invariantes del objeto:

```python
from exceptions import LibroNoDisponibleError


class Libro:
    def __init__(self, titulo: str) -> None:
        self.titulo = titulo
        self.disponible = True

    def prestar(self) -> None:
        if not self.disponible:
            raise LibroNoDisponibleError(
                f"El libro '{self.titulo}' no está disponible"
            )

        self.disponible = False
```

Flujo correcto — **validar antes de mutar**:

```text
estado inicial (disponible=True)
      ↓
prestar()
      ↓
validar
      ↓
válido → mutar estado (disponible=False)
```

Nunca mutar primero y fallar después: eso puede dejar al objeto en un estado inconsistente.

`main.py`:

```python
from exceptions import BibliotecaError

try:
    resultado = estudiante.solicitar_libro(None)
except BibliotecaError as exc:
    print(f"Error: {exc}")

resultado = estudiante.solicitar_libro("El Principito")
print(resultado)
```

### Excepciones como contrato

```text
prestar()
────────────────────────
Precondición:  libro disponible
Éxito:         disponible = False
Fallo:         LibroNoDisponibleError
```

Una función no es solo `input → output`: tiene precondiciones, postcondiciones, invariantes y errores esperados. La excepción `LibroNoDisponibleError` protege la invariante "un libro no disponible no puede prestarse de nuevo" — esto convierte el manejo de excepciones en parte del **diseño del dominio**, no solo en una técnica de debugging.

---

## 7. Correcciones técnicas detectadas

### Corrección técnica 1

**Detectado:** el material original usa `raise ValueError(...)` para todos los errores del dominio de biblioteca.

**Corrección:** reservar `ValueError` para validaciones genéricas de tipo/lenguaje, y usar excepciones de dominio (`TituloInvalidoError`, `LibroNoDisponibleError`) cuando el error tiene significado propio dentro del sistema.

**Justificación:** un consumidor no debería depender de `str(error)` o de comparar mensajes para diferenciar condiciones de negocio; la jerarquía de excepciones es una API más estable que el texto del mensaje.

**Impacto de no corregirlo:** cualquier cambio de redacción del mensaje de error rompe silenciosamente el código que dependa de él (p. ej. `if "no disponible" in str(e)`).

### Corrección técnica 2

**Detectado:** el material usa `except BibliotecaError as e: if type(e) == TituloInvalidoError: ...` como forma de inspección.

**Corrección:** usar cláusulas `except` separadas y ordenadas de específico a general.

**Justificación:** Python ya resuelve el despacho por tipo mediante las cláusulas `except`; reimplementarlo con `type(e) ==` ignora el propósito de la herencia y es frágil ante cambios en la jerarquía.

**Impacto:** código más difícil de extender — cada nueva subclase de error obliga a tocar la cadena de `if`.

### Checklist de la Parte 1

```text
[ ] Distingo fallo silencioso de fallo explícito.
[ ] Sé qué hace raise y qué NO hace (no termina el programa por sí mismo).
[ ] Distingo "detectar" de "manejar" un error.
[ ] Sé cuándo ValueError es suficiente y cuándo no.
[ ] Sé diseñar una jerarquía de excepciones con propósito, sin sobreingeniería.
[ ] Ordeno los except de específico a general.
[ ] Evito except: desnudo.
[ ] Entiendo Exception vs BaseException.
[ ] Sé leer un traceback siguiendo un método, no al azar.
[ ] Entiendo la excepción como parte del contrato de una función (pre/postcondición/invariante).
```

---

# PARTE 2 — PROFESIONALIZACIÓN Y PRODUCCIÓN

> El salto de nivel Senior consiste en pasar de conocer la sintaxis a diseñar una **política de errores**. La pregunta deja de ser "¿cómo capturo una excepción?" y pasa a ser:
>
> **"¿Qué tipo de fallo ocurrió, quién tiene autoridad para recuperarse, qué información debo preservar y qué consecuencias tiene manejarlo incorrectamente?"**

## 8. Curso vs. práctica profesional actual

```text
                 ERROR
                   │
                   ▼
             CLASIFICACIÓN
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
    esperado   recuperable  inesperado
        │          │          │
        ▼          ▼          ▼
     manejar    retry /      fail
                fallback     safely
                   │
                   ▼
             OBSERVABILIDAD
                   │
                   ▼
          MÉTRICAS / LOGS / TRACE
```

La Parte 1 enseñó `raise`, `try/except`, jerarquías y traceback. Eso es necesario pero insuficiente para producción: un sistema real necesita además clasificar el error, decidir su política de recuperación y hacerlo observable.

---

## 9. Buenas prácticas: cuatro antipatrones de producción

### 9.1 Capturar demasiado pronto

```python
def repository_get_user(user_id: str):
    try:
        return database.get_user(user_id)
    except Exception:
        return None
```

Parece robusto, pero si ocurrió un `DatabaseConnectionError`, el `None` devuelto hace que la capa superior interprete "usuario inexistente" cuando en realidad significaba "la base de datos está caída". Se ha producido una **pérdida semántica del error**: infraestructura caída se convierte en recurso inexistente.

> **Regla senior:** no captures una excepción en una capa si esa capa no tiene una estrategia correcta de recuperación o traducción.

### 9.2 Error swallowing

```python
try:
    process_payment()
except PaymentError:
    pass
```

El sistema continúa como si nada hubiera ocurrido. Un `pass` dentro de un `except` no es automáticamente incorrecto, pero requiere una justificación de diseño explícita; de lo contrario produce datos inconsistentes, operaciones parcialmente completadas y errores imposibles de reproducir.

### 9.3 Convertir cualquier error en éxito

```python
try:
    process_order()
except Exception:
    return "Order processed"
```

Esto viola el contrato semántico del sistema: una excepción no debe usarse para fabricar un resultado exitoso que no ocurrió. En un sistema AI, el equivalente sería que una generación fallida devuelva igualmente "respuesta generada".

### 9.4 `except Exception` como estrategia universal

```python
try:
    result = call_external_service()
except Exception:
    retry()
```

Distintos errores requieren distintas políticas — ver tabla de clasificación (§10). Tratar todo por igual es el antipatrón central de esta parte.

---

## 10. Clasificación profesional de errores y retryability

| Categoría      | Ejemplo                          |          ¿Retry? |
| -------------- | --------------------------------- | ----------------: |
| Validación     | input inválido                    |                No |
| Dominio        | libro no disponible               |                No |
| Auth           | API key inválida                  |                No |
| Rate limit     | HTTP 429                          |   Sí, con backoff |
| Timeout        | dependencia tardó demasiado       |      Posiblemente |
| Servicio caído | HTTP 503                          |      Posiblemente |
| Bug            | `AttributeError` inesperado       |                No |
| Configuración  | variable obligatoria ausente      |                No |

La clasificación —no la sintaxis de `except`— es la que determina la recuperación. Una excepción puede llevar esta información explícitamente:

```python
class ProviderTimeoutError(ProviderError):
    retryable = True


class ProviderAuthError(ProviderError):
    retryable = False
```

```python
if exc.retryable:
    retry()
else:
    fail()
```

> **Cuidado:** `retryable=True` no es una verdad absoluta. También depende de la operación, la idempotencia, el presupuesto de latencia y el número de intentos ya realizados (ver §11-§13).

---

## 11. Resiliencia: retry, timeout, backoff, jitter

Reintentar indiscriminadamente puede amplificar un incidente (**retry storm**):

```text
100 requests
    ↓
dependencia lenta
    ↓
cada request reintenta 3 veces → 300 requests
    ↓
más carga → dependencia más lenta → más retries
```

La resiliencia real combina varios mecanismos, no solo `except`:

```text
retry + timeout + backoff + jitter + límite de intentos
+ clasificación del error + idempotencia
```

**Timeout antes que retry.** Una llamada externa sin límite explícito puede esperar indefinidamente:

```text
request → timeout → ¿terminó? → sí: resultado / no: TimeoutError → ¿retry?
```

**Backoff exponencial.** Los reintentos se espacian progresivamente (`delay ≈ base × 2^attempt`). Se le añade **jitter** (variación aleatoria) para evitar que miles de clientes reintenten exactamente al mismo tiempo y vuelvan a golpear el servicio simultáneamente.

**Retry budget.** La pregunta profesional no es "¿cuántas veces reintento?" sino "¿cuánto tiempo y carga estoy dispuesto a gastar intentando recuperar esta operación, dado un deadline global?". Esto es crítico en cadenas como `API → Agent → RAG → Vector DB → LLM → Tool`, donde cada capa consume parte del presupuesto de latencia.

---

## 12. Idempotencia y efectos secundarios

Antes de reintentar: **¿es seguro ejecutar de nuevo esta operación?**

```text
charge_customer()
    ↓ pago realmente procesado
    ↓ respuesta perdida (timeout)
    ↓ cliente reintenta
    ↓ segundo cargo   ← efecto duplicado
```

`GET /users/123` es trivialmente reintentable; `POST /payments` no lo es. Los sistemas de pagos suelen usar **idempotency keys** para que el servidor reconozca que dos solicitudes representan la misma operación lógica.

> **Regla:** retry sin analizar idempotencia puede convertir un fallo temporal en una operación duplicada.

---

## 13. Circuit breaker

Cuando una dependencia falla continuamente, seguir llamándola empeora el incidente:

```text
             ┌──────────────┐
             │    CLOSED     │  tráfico normal
             └──────┬────────┘
                    │ demasiados fallos
                    ▼
             ┌──────────────┐
             │     OPEN      │  no llamar
             └──────┬────────┘
                    │ tras un tiempo
                    ▼
             ┌──────────────┐
             │  HALF-OPEN    │  probar
             └──────┬────────┘
                 /       \
             éxito       fallo
               │           │
               ▼           ▼
            CLOSED       OPEN
```

No es una característica de `try/except`: es una política de resiliencia construida encima del manejo de errores, normalmente vía una librería o un componente de infraestructura compartido.

---

## 14. Traducción de excepciones entre capas y error boundaries

### Dominio ≠ HTTP

```text
LibroNoDisponibleError   →  "el libro no puede prestarse según las reglas del sistema"
HTTP 409 Conflict        →  "la solicitud HTTP no puede completarse por el estado actual del recurso"
```

Son conceptos relacionados pero distintos. Un diseño limpio traduce en la frontera, en vez de que el dominio conozca HTTP:

```text
Domain → LibroNoDisponibleError → Application/API boundary → HTTP 409
```

### Regla de arquitectura

> **Las excepciones deben pertenecer al lenguaje de la capa que las produce; la frontera exterior las traduce.**

```python
try:
    client.responses.create(...)
except ExternalProviderError as exc:
    raise ProviderUnavailableError("LLM provider unavailable") from exc
```

`raise ... from exc` preserva la causalidad (`__cause__`) mientras el consumidor superior trabaja contra una abstracción estable — esto es especialmente importante en AI Engineering, donde OpenAI, Anthropic, Gemini o un modelo local pueden fallar con excepciones muy distintas.

### Error boundary global vs. handlers específicos

```text
Specific handlers  → errores conocidos     → recuperación concreta
Global boundary     → errores inesperados   → logging + respuesta segura
```

```python
def handle_request(request):
    try:
        return process(request)
    except Exception:
        logger.exception("Unhandled application error")
        return internal_server_error()
```

Este `except Exception` **no** significa "sé recuperarme de cualquier error"; significa "este es el último boundary de seguridad y observabilidad". Debe: registrarse siempre, nunca ocultarse, nunca devolver éxito, y nunca capturar `BaseException`.

### No filtrar información sensible

```python
raise Exception(f"Request failed with API key {api_key}")   # peligroso
```

Los mensajes de excepción pueden terminar en logs, traces o herramientas de monitoreo (APM/Sentry). La observabilidad no debe convertirse en un canal de fuga de secretos, credenciales, PII o prompts privados.

---

## 14.1 Testing ejecutable de excepciones

> No basta con decir "está testeado". Un test de excepciones debe verificar tres cosas distintas: **que se lanza el tipo correcto**, **que preserva la causalidad correcta** y **que la política de recuperación (retry/idempotencia/traducción) se comporta como se diseñó**. Estos tres niveles corresponden directamente a lo enseñado en §4, §14 y §11-§12.

### 15.1 Nivel 1 — Tipo y mensaje

```python
import pytest

from exceptions import TituloInvalidoError
from usuarios import Estudiante


def test_solicitar_libro_con_titulo_none_lanza_titulo_invalido():
    estudiante = Estudiante()

    with pytest.raises(TituloInvalidoError, match="no es válido"):
        estudiante.solicitar_libro(None)


@pytest.mark.parametrize("titulo_invalido", [None, "", "   "])
def test_solicitar_libro_rechaza_titulos_invalidos(titulo_invalido):
    estudiante = Estudiante()

    with pytest.raises(TituloInvalidoError):
        estudiante.solicitar_libro(titulo_invalido)
```

`pytest.raises(..., match=...)` verifica tipo **y** contenido del mensaje con una sola aserción — evita falsos positivos donde el test pasa porque se lanzó *alguna* excepción, no la correcta.

### 15.2 Nivel 2 — Causalidad y datos estructurados de la excepción

```python
def test_libro_no_disponible_expone_titulo_en_la_excepcion():
    libro = Libro("El Principito")
    libro.disponible = False

    with pytest.raises(LibroNoDisponibleError) as exc_info:
        libro.prestar()

    assert exc_info.value.titulo == "El Principito"


def test_provider_unavailable_preserva_la_causa_original():
    original = TimeoutError("connection timed out")

    def fake_call():
        try:
            raise original
        except TimeoutError as exc:
            raise ProviderUnavailableError("LLM provider unavailable") from exc

    with pytest.raises(ProviderUnavailableError) as exc_info:
        fake_call()

    assert exc_info.value.__cause__ is original
```

Este segundo test es el que realmente valida la §14 (traducción de excepciones): no basta con que se lance `ProviderUnavailableError`, hay que verificar que `__cause__` sigue apuntando al error original — de lo contrario, la traducción "silenciosa" perdió información de debugging sin que ningún test lo detecte.

### 15.3 Nivel 3 — Comportamiento de retry, backoff e idempotencia

```python
from unittest.mock import Mock, call


def test_retry_reintenta_solo_en_errores_transitorios(monkeypatch):
    sleeps = []
    monkeypatch.setattr("time.sleep", lambda s: sleeps.append(s))

    provider = Mock(
        side_effect=[ProviderTimeoutError(), ProviderTimeoutError(), "ok"]
    )

    resultado = call_with_retry(provider, max_attempts=3)

    assert resultado == "ok"
    assert provider.call_count == 3
    assert len(sleeps) == 2          # backoff antes del 2º y 3er intento
    assert sleeps[1] > sleeps[0]     # backoff creciente


def test_retry_no_reintenta_en_error_no_recuperable():
    provider = Mock(side_effect=ProviderAuthError())

    with pytest.raises(ProviderAuthError):
        call_with_retry(provider, max_attempts=3)

    assert provider.call_count == 1   # ni un solo retry


def test_operacion_idempotente_no_duplica_efecto_tras_retry():
    ledger = InMemoryPaymentLedger()

    # Simula: la primera llamada sí llegó al backend, pero el cliente
    # recibió timeout y reintenta con la MISMA idempotency_key.
    charge_customer(ledger, amount=100, idempotency_key="abc123")
    charge_customer(ledger, amount=100, idempotency_key="abc123")

    assert ledger.total_charged() == 100   # no 200
```

El tercer test es el más importante de esta sección: reproduce exactamente el escenario del CASO 2 (§12) — un retry que podría duplicar un cargo — y prueba la garantía de idempotencia, no solo que "no lanzó excepción".

### 15.4 Nivel 4 — Error boundary: verificar que NO se convierte fallo en éxito

```python
def test_error_boundary_no_convierte_excepcion_en_respuesta_exitosa(caplog):
    def process_order():
        raise DatabaseTimeoutError("db unreachable")

    response = handle_request_boundary(process_order)

    assert response.status_code == 500
    assert response.body != "Order processed"          # no fabrica éxito
    assert "Unhandled application error" in caplog.text  # sí se registró
```

Este test verifica directamente el antipatrón de §9.3 ("convertir cualquier error en éxito"): un boundary correcto debe producir una respuesta segura de fallo **y** dejar rastro en logs; un boundary roto pasaría el test de "no lanza excepción" pero fallaría este.

### 15.5 Testing de `ExceptionGroup` / `except*`

```python
import pytest


async def test_exception_group_agrupa_fallos_de_tools_paralelas():
    with pytest.raises(ExceptionGroup) as exc_info:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(tool_a())            # éxito
            tg.create_task(tool_b_timeout())     # TimeoutError
            tg.create_task(tool_c_invalid())     # ValueError

    group = exc_info.value
    assert len(group.exceptions) == 2
    assert any(isinstance(e, TimeoutError) for e in group.exceptions)
    assert any(isinstance(e, ValueError) for e in group.exceptions)
```

### Regla de testing senior

> **Un test de excepciones que solo verifica `pytest.raises(Exception)` genérico no prueba nada útil.** Debe fijar el tipo exacto, y —cuando la política de errores lo amerite— la causa, los datos estructurados, el número de reintentos y el efecto (o ausencia de efecto) sobre el estado del sistema.

### 15.6 Tres niveles de testing aplicados a errores: unit, integration, contract

Los niveles anteriores (§15.1-§15.5) son en su mayoría **unit tests**: prueban una función o una excepción en aislamiento. Para el pipeline completo de §16.2 hace falta distinguir explícitamente tres niveles de testing, porque cada uno protege contra un tipo de regresión distinto:

```text
UNIT TEST
    prueba una excepción o una función aislada
    (§15.1-§15.5: "¿esta función lanza TituloInvalidoError?")
        ↓
INTEGRATION TEST
    prueba el pipeline completo end-to-end
    (Exception real del SDK → ApplicationError → efecto en el sistema)
        ↓
CONTRACT TEST
    prueba que la SUPERFICIE PÚBLICA no cambió
    (dado un ApplicationError conocido, ¿el HTTP status y el
    error_code siguen siendo exactamente los mismos?)
```

Ninguno reemplaza a los otros: un **unit test** puede pasar mientras un **contract test** falla (la excepción se sigue lanzando correctamente, pero alguien cambió el `error_code` público sin darse cuenta). Y un **integration test** puede pasar mientras un **contract test** falla igualmente si el mapeo produce el JSON correcto pero el equipo consumidor esperaba un campo que ya no existe.

**Unit test — ya cubierto en §15.1-§15.5.**

**Integration test — el pipeline completo, no solo una función:**

```python
def test_pipeline_completo_provider_timeout_hasta_http(monkeypatch):
    """
    Exception (SDK)  →  ApplicationError  →  ErrorContract  →  HTTP response
    Prueba las cuatro transformaciones encadenadas, no cada una por separado.
    """
    monkeypatch.setattr(
        "sdk_client.generate",
        Mock(side_effect=SDKTimeoutError("timeout")),
    )

    response = api_client.post("/generate", json={"prompt": "hola"})

    assert response.status_code == 504
    body = response.json()
    assert body["error_code"] == "PROVIDER_TIMEOUT"
    assert body["retryable"] is True
    assert "stack" not in body            # el traceback interno NUNCA cruza a HTTP
```

**Contract test — protege la superficie pública del error, independientemente de la implementación interna:**

```text
Exception mapping
        ↓
ApplicationError
        ↓
ErrorContract
        ↓
HTTP response
```

```python
def test_error_contract_mapping():
    status, contract = to_error_contract(
        LibroNoDisponibleError("Dune"),
        correlation_id="abc",
    )

    assert status == 409
    assert contract.error_code == "LIBRO_NO_DISPONIBLE"
    assert contract.retryable is False
```

Este test no llama a ningún SDK, ni a la base de datos, ni monta la aplicación: prueba **exclusivamente** la tabla de mapeo (§16.2, `_CONTRACT_MAP`) como un artefacto propio. Debe existir uno de estos por cada `ApplicationError` que la API exponga, y —igual de importante— un test que cubra el caso por defecto:

```python
def test_error_contract_mapping_caso_no_listado_cae_en_default_seguro():
    class ErrorInternoNoMapeado(ApplicationError):
        code = "ALGO_QUE_NADIE_MAPEÓ"

    status, contract = to_error_contract(
        ErrorInternoNoMapeado("detalle interno sensible"),
        correlation_id="abc",
    )

    assert status == 500
    assert contract.message == "Ocurrió un error inesperado. Nuestro equipo fue notificado."
    assert "detalle interno sensible" not in contract.message   # no se filtra el mensaje crudo
```

### Por qué el contract test es el que más importa a nivel Staff

> **Un refactor interno de la jerarquía de excepciones puede pasar todos los unit tests y todos los integration tests, y aun así romper a un consumidor externo si cambió el `error_code`, el `http_status` o un campo del `ErrorContract` que ese consumidor ya lee en producción.**

El contract test es, en la práctica, el mecanismo que convierte el Error Contract de §16.1 de una intención de diseño en una garantía verificable en CI. Sin él, "el contrato es estable" es solo una promesa; con él, es una aserción que falla el build si alguien la rompe — exactamente el mismo principio que un *snapshot test* de un schema de API pública.

---

## Checklist de la Parte 2 — Producción

```text
[ ] Se captura únicamente lo que realmente se sabe manejar.
[ ] Se evita except: desnudo y except Exception como estrategia universal.
[ ] Existe una razón explícita para cada except Exception (boundary, no atajo).
[ ] Se preserva la excepción original (raise ... from).
[ ] La excepción pertenece a la capa correcta; se traduce en la frontera.
[ ] Se distinguen errores permanentes de transitorios (retryable).
[ ] Los retries tienen límite, timeout y backoff/jitter cuando corresponde.
[ ] Se evaluó idempotencia antes de reintentar.
[ ] Existe observabilidad (logs/metrics/traces) para cada fallo relevante.
[ ] No se filtran secretos ni PII mediante mensajes de excepción.
[ ] Los tests fijan el tipo exacto de excepción, no Exception genérica.
[ ] Existe al menos un test que verifica la causalidad (__cause__) tras una traducción.
[ ] Existe un test que verifica que un retry NO ocurre en errores no recuperables.
[ ] Existe un test que verifica idempotencia real (no solo ausencia de excepción).
[ ] Existe un test que verifica que un error boundary no fabrica una respuesta exitosa.
[ ] Distingo unit test, integration test y contract test aplicados a errores.
[ ] Existe al menos un contract test por cada ApplicationError expuesto por la API.
[ ] Existe un contract test para el caso por defecto (error no mapeado → 500 seguro, sin filtrar detalle interno).
```

---

# PARTE 3 — ARQUITECTURA Y TECH LEAD THINKING

> Esta parte eleva la perspectiva: ya no se trata de "¿cómo capturo esto?" sino de "¿cómo diseño la propagación de fallos en un sistema con múltiples componentes, equipos y dependencias externas —incluyendo proveedores LLM, RAG y agentes?".

## 15. System Design: el flujo de fallos como componente de arquitectura

```text
                 OPERATION
                     │
                     ▼
                   ERROR
                     │
                     ▼
                CLASSIFY
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      HANDLE       RETRY       PROPAGATE
        │            │            │
        │     timeout/backoff     │
        │     idempotency         │
        │            │            │
        └────────────┴────────────┘
                     │
                     ▼
                TRANSLATE
                     │
                     ▼
              OBSERVE / ALERT
                     │
                     ▼
              SAFE RESPONSE
```

Este diagrama sustituye al modelo mental de "`try` → `except`" cuando se diseña un sistema completo, no una función aislada.

---

## 16. Decisiones arquitectónicas: dónde vive cada excepción

```text
src/
└── app/
    ├── domain/
    │   ├── models.py
    │   └── exceptions.py        ← DomainError, LibroNoDisponibleError...
    │
    ├── application/
    │   ├── services.py
    │   └── exceptions.py        ← decide recovery / propagation
    │
    ├── infrastructure/
    │   ├── llm/
    │   │   ├── openai.py
    │   │   ├── anthropic.py
    │   │   └── exceptions.py    ← ProviderError, traduce SDK
    │   ├── vectorstore/
    │   └── database/
    │
    └── interfaces/
        └── api/
            ├── routes.py
            └── exception_handlers.py   ← traduce a HTTP
```

No es una plantilla obligatoria: la estructura correcta depende del tamaño y de los límites reales del sistema (mismo principio de organización "por responsabilidad, no por regla mecánica" de la clase de módulos y paquetes).

### Opción A vs. Opción B — ejemplo de decisión real

```text
Opción A — el dominio lanza HTTPException directamente
  Ventaja: menos código, más rápido de escribir
  Desventaja: el dominio queda acoplado a FastAPI

Opción B — el dominio lanza DomainError; la API la traduce
  Ventaja: dominio reutilizable fuera de HTTP (CLI, worker, otro framework)
  Desventaja: requiere una capa de traducción explícita

Decisión: Opción B para cualquier dominio con reglas de negocio no triviales.

Trade-off que se sacrifica: velocidad inicial de desarrollo.

Condición de cambio: si el dominio es trivial y de un solo uso
(un script interno de corta vida), la Opción A puede ser aceptable.
```

Nunca presentar una tecnología o patrón como solución universal: toda decisión arquitectónica debe declarar sus trade-offs y sus condiciones de reconsideración.

---

## 16.1 Formalización del Error Contract

Hasta aquí se ha hablado de "traducir excepciones entre capas" de forma general (§14). Un Staff Engineer va un paso más allá: define el **Error Contract** como un artefacto de diseño explícito, independiente de cualquier clase Python concreta.

### Qué es y qué no es un Error Contract

> **Un Error Contract NO es una clase de excepción.** Es la forma estable y versionable en la que un fallo se comunica a un consumidor —humano o programático— sin importar qué excepción de Python lo originó internamente.

Una excepción Python puede cambiar de nombre, de módulo o de jerarquía en un refactor interno sin romper nada externo, **siempre que el Error Contract que expone se mantenga estable**. Esa es precisamente la razón de separarlos.

### Campos mínimos de un Error Contract

| Campo             | Público / Interno | Propósito                                                              |
| ------------------ | ------------------ | ------------------------------------------------------------------------ |
| `error_code`       | Público            | Identificador estable y versionable (`LIBRO_NO_DISPONIBLE`), no el nombre de la clase Python |
| `category`         | Público            | Familia de error (`validation`, `domain`, `provider`, `infrastructure`)  |
| `retryable`        | Público            | Si el consumidor puede reintentar de forma segura                        |
| `message`          | Público            | Mensaje seguro para el consumidor (nunca el `str()` crudo de la excepción) |
| `correlation_id`   | Público            | Permite al consumidor reportar el incidente de forma trazable            |
| `http_status`      | Público (solo en API HTTP) | Se deriva del contrato, no de la excepción directamente          |
| `cause` / traceback | **Interno**        | Nunca se serializa hacia un consumidor externo                          |
| `raw_provider_error`| **Interno**        | Detalle del SDK/proveedor; solo para logs                               |
| `internal_metadata` | **Interno**        | Contexto de debugging (query, payload, host)                            |

### Regla de oro

> **Todo lo que cruza la frontera pública debe pasar por el Error Contract. Nada de la excepción interna (traceback, mensaje crudo, tipo de SDK) cruza esa frontera directamente.**

### Implementación de referencia

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone


class ApplicationError(Exception):
    """Excepción base de aplicación: transporta su propio contrato."""

    code: str = "APPLICATION_ERROR"
    category: str = "application"
    retryable: bool = False


class LibroNoDisponibleError(ApplicationError):
    code = "LIBRO_NO_DISPONIBLE"
    category = "domain"
    retryable = False

    def __init__(self, titulo: str) -> None:
        self.titulo = titulo
        super().__init__(f"El libro '{titulo}' no está disponible")


@dataclass(frozen=True)
class ErrorContract:
    """Representación estable y serializable de un fallo. NO es la excepción."""

    error_code: str
    category: str
    retryable: bool
    message: str
    correlation_id: str
    timestamp: str = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )
```

`ApplicationError` vive en el código; `ErrorContract` vive en la red. Son dos objetos distintos aunque el segundo se construya a partir del primero — ver §16.2 para el pipeline completo de construcción.

### Versionado del contrato

Un Error Contract, al ser público, debe versionarse como cualquier API:

```text
Cambio compatible (no rompe consumidores):
    agregar un nuevo error_code
    agregar un campo opcional nuevo

Cambio incompatible (rompe consumidores):
    renombrar un error_code existente
    cambiar el significado de retryable para un código existente
    eliminar un campo que un consumidor ya lee
```

> **Regla senior:** un refactor interno de la jerarquía de excepciones (dividir `LibroNoDisponibleError` en dos clases más específicas, por ejemplo) **no debe** cambiar el `error_code` público sin ser tratado como un cambio de contrato versionado.

### Resolución del mapeo por jerarquía, no por tipo exacto

El `_CONTRACT_MAP` de la implementación de referencia (§16.2) funciona bien como ejemplo pedagógico, pero un `dict` indexado por `type(exc)` exacto tiene una limitación real: si existe herencia entre errores de aplicación —muy común en `ProviderError` (§17)— una subclase no listada explícitamente cae directo al caso por defecto, aunque exista un ancestro sí mapeado:

```text
ProviderError                    ← mapeado en _CONTRACT_MAP
    ↓
ProviderTimeoutError             ← NO está en el dict → cae a 500 genérico,
                                    aunque ProviderError sí tenía un mapeo razonable
```

Una resolución por jerarquía recorre el `__mro__` (Method Resolution Order) de la excepción y usa el mapeo más específico disponible:

```python
def resolve_contract_entry(
    exc: ApplicationError,
    mapping: dict[type[ApplicationError], tuple[int, str]],
) -> tuple[int, str]:
    for klass in type(exc).__mro__:
        if klass in mapping:
            return mapping[klass]
    return 500, "Ocurrió un error inesperado. Nuestro equipo fue notificado."
```

Esto permite mapear `ProviderError` una sola vez con un status/mensaje razonable por defecto, y solo añadir entradas más específicas (`ProviderRateLimitError → 429`) cuando el comportamiento realmente deba diferenciarse — evitando que cada nueva subclase de excepción rompa silenciosamente el mapeo por omisión.

### Forma real del contrato en una API de producción

El `ErrorContract` mínimo de §16.1 es suficiente para enseñar el concepto, pero una API real típicamente necesita un campo adicional para datos estructurados específicos del error, y —sobre todo— una **política explícita** de qué es estable, qué es opcional y qué es exclusivamente interno:

```json
{
  "error_code": "PROVIDER_TIMEOUT",
  "category": "provider",
  "retryable": true,
  "message": "El servicio está tardando más de lo normal. Intenta de nuevo.",
  "correlation_id": "abc123",
  "details": {}
}
```

| Campo             | Estabilidad                          | ¿Puede consumirlo un cliente externo? |
| ------------------ | ------------------------------------- | ---------------------------------------- |
| `error_code`       | Estable (cambio = breaking change)    | Sí — es la clave para lógica condicional del cliente |
| `category`         | Estable                                | Sí — útil para agrupar métricas del lado del cliente |
| `retryable`        | Estable                                | Sí — determina si el cliente reintenta   |
| `message`          | Opcional (puede cambiar de redacción) | Sí, solo para mostrar — nunca parsear su contenido |
| `correlation_id`   | Estable                                | Sí — para reportar el incidente          |
| `details`          | Opcional, específico por `error_code`  | Sí, con cuidado — solo campos ya diseñados como públicos (p. ej. `{"retry_after_seconds": 30}`) |
| Traceback / SDK error | No forma parte del contrato          | **Nunca** — permanece solo en logs internos (§16.1) |

> **Regla:** un cliente externo puede tomar decisiones de código a partir de `error_code`, `category` y `retryable` — nunca a partir de `message` (texto libre, puede cambiar) ni de nada que no esté explícitamente documentado como parte del contrato. Esta distinción entre "campos accionables" y "campos informativos" es la que evita que un cambio de redacción del mensaje rompa la integración de un consumidor que, por error, estaba haciendo `if "no disponible" in error.message`.

Formalizar completamente esta política —incluyendo el diseño de `details` por `error_code`, versionado semántico del contrato y generación de esquemas (p. ej. con Pydantic/OpenAPI)— pertenece más naturalmente a una clase dedicada de diseño de APIs con FastAPI; aquí se deja establecida la base conceptual sobre la que esa clase futura construirá.

---

## 16.2 Separación explícita: Exception → Application Error → API Contract → HTTP Status → User Message

El material original (y buena parte de los tutoriales de Python) tiende a colapsar estos cinco conceptos en uno solo: "la excepción". En un sistema profesional son **cinco objetos distintos**, cada uno con su propio ciclo de vida y su propia responsabilidad.

```text
1. EXCEPTION                 objeto Python en tiempo de ejecución
        │                    (TimeoutError, ValueError, error del SDK...)
        ▼
2. APPLICATION ERROR         excepción tipada propia del dominio/aplicación
        │                    (LibroNoDisponibleError, ProviderTimeoutError...)
        │                    vive en el código, tiene comportamiento (§16.1)
        ▼
3. API CONTRACT               representación estable y serializada
        │                    (ErrorContract: error_code, category, retryable...)
        │                    independiente del transporte
        ▼
4. HTTP STATUS                 mapeo específico de transporte
        │                    (409, 429, 500...) — HTTP es solo UNO de los
        │                    transportes posibles (también CLI, evento, gRPC)
        ▼
5. USER MESSAGE                texto final mostrado a una persona
                             seguro, localizable, sin detalle interno
```

### Por qué separarlos en cinco y no en dos

| Si colapsas...                              | Pierdes...                                                                 |
| --------------------------------------------- | ----------------------------------------------------------------------------- |
| Exception = Application Error                | No puedes traducir errores de SDK sin acoplar tu dominio al proveedor (§14)   |
| Application Error = API Contract              | Un refactor interno de clases rompe silenciosamente a los consumidores externos |
| API Contract = HTTP Status                    | No puedes reutilizar el mismo contrato en un worker, una CLI o un evento — no todo transporte es HTTP |
| HTTP Status = User Message                    | Un `409` no te dice qué mostrarle a un usuario en español, con el tono correcto, sin exponer detalle técnico |

### Ejemplo completo, capa por capa

```python
# 1. EXCEPTION — objeto de bajo nivel, ajeno a tu aplicación
#    (ejemplo: el SDK de un proveedor LLM lanza esto)
class SDKTimeoutError(Exception):
    pass


# 2. APPLICATION ERROR — traducción en el adapter (ver §14)
class ProviderTimeoutError(ApplicationError):
    code = "PROVIDER_TIMEOUT"
    category = "provider"
    retryable = True


def call_provider(prompt: str) -> str:
    try:
        return sdk_client.generate(prompt)
    except SDKTimeoutError as exc:
        raise ProviderTimeoutError("El proveedor no respondió a tiempo") from exc


# 3. API CONTRACT — mapeo explícito, mantenido en la frontera, NO en el dominio
_CONTRACT_MAP: dict[type[ApplicationError], tuple[int, str]] = {
    LibroNoDisponibleError: (409, "Este libro no está disponible en este momento."),
    TituloInvalidoError:    (400, "El título proporcionado no es válido."),
    ProviderTimeoutError:   (504, "El servicio está tardando más de lo normal. Intenta de nuevo."),
}


def to_error_contract(exc: ApplicationError, correlation_id: str) -> tuple[int, ErrorContract]:
    # 4. HTTP STATUS se resuelve aquí, a partir del contrato — nunca desde
    #    la excepción original ni desde el SDK.
    http_status, user_message = _CONTRACT_MAP.get(
        type(exc), (500, "Ocurrió un error inesperado. Nuestro equipo fue notificado.")
    )

    contract = ErrorContract(
        error_code=exc.code,
        category=exc.category,
        retryable=exc.retryable,
        message=user_message,        # 5. USER MESSAGE — nunca str(exc) crudo
        correlation_id=correlation_id,
    )
    return http_status, contract
```

### El error que este diseño previene

Sin esta separación, es común encontrar código como:

```python
# ANTI-PATRÓN: colapsa las 5 capas en una línea
except Exception as e:
    return {"error": str(e)}, 500
```

Esto expone directamente el mensaje interno de la Exception (paso 1) como User Message (paso 5), sin pasar por Application Error, API Contract ni una decisión explícita de HTTP Status — es la causa raíz de la mayoría de fugas de información sensible mencionadas en §14 y de HTTP status incorrectos (p. ej. devolver 500 para un error de validación de negocio que debería ser 400/409).

### Regla senior

> **El HTTP Status y el User Message nunca se derivan directamente de la Exception original ni de `type(exc)` disperso por el código. Se derivan de una tabla de mapeo explícita y centralizada (`_CONTRACT_MAP`), mantenida en la frontera, testeable de forma independiente al resto de la aplicación.**

Esta tabla de mapeo es, en sí misma, un artefacto que debe tener sus propios tests: dado cada `ApplicationError` conocido, ¿produce el `http_status` y el `user_message` correctos? Y —igual de importante— dado un `ApplicationError` **no listado** en el mapeo, ¿cae de forma segura en el caso por defecto (`500`, mensaje genérico) en vez de fallar el propio manejador de errores?

---

## 17. Trade-offs: taxonomía de errores para AI Engineering

```text
AppError
├── ValidationError
├── DomainError
├── InfrastructureError
│   ├── DatabaseError
│   ├── CacheError
│   └── ExternalServiceError
├── ProviderError
│   ├── ProviderAuthError          (no retry)
│   ├── ProviderRateLimitError     (retry con backoff)
│   ├── ProviderTimeoutError       (retry limitado / fallback)
│   └── ProviderUnavailableError   (retry / fallback de proveedor)
├── RetrievalError
├── ToolError
└── ConfigurationError
```

No es obligatorio copiar esta jerarquía literalmente: es un **modelo de diseño**. El valor no está en la cantidad de nombres, sino en que cada rama habilite una política de recuperación distinta y explícita.

### Conexión breve con MCP

Cuando una tool de un agente se ejecuta a través de un servidor MCP en vez de una función local, la misma cadena de decisiones de §12 y §17 aplica sin cambios conceptuales — solo cambia dónde ocurre físicamente el fallo:

```text
MCP Server
   ↓
Tool execution
   ↓
external side effect
   ↓
tool failure
   ↓
retryability
   ↓
idempotency
   ↓
error contract
```

Es decir: un fallo de una tool MCP debe clasificarse igual que cualquier `ToolError` (§17) —distinguiendo si tuvo side effects antes de fallar (§12)— y debe traducirse al mismo `ErrorContract` (§16.1) que cualquier otro fallo de la aplicación, en vez de propagar el error crudo del protocolo MCP hacia el agente o hacia el usuario final. No es necesaria una sección aparte: MCP es simplemente otro transporte, sujeto a las mismas reglas de traducción de excepciones ya cubiertas en §14 y §16.2.

---

## 18. Escalabilidad: fallo parcial y concurrencia

En sistemas distribuidos no siempre ocurre "todo funciona" o "todo falla":

```text
Tool A → éxito
Tool B → éxito
Tool C → timeout
Tool D → éxito
```

Esto es **partial failure**, especialmente relevante en retrieval paralelo, workflows multi-agente y patrones fan-out/fan-in. El sistema debe decidir explícitamente: ¿falla toda la operación, se degrada parcialmente, se reintenta solo lo que falló, o se informa incertidumbre al usuario?

**Actualización profesional (vigente a septiembre 2026):** Python moderno representa esto mediante `ExceptionGroup` y `except*` (PEP 654), disponibles desde Python 3.11 y mantenidos en la serie 3.14 actual. [2]

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(tool_a())
        tg.create_task(tool_b())
        tg.create_task(tool_c())
except* TimeoutError:
    ...
except* ValueError:
    ...
```

Esto es relevante para agentes que ejecutan varias tools en paralelo, pipelines de RAG concurrentes y cualquier `asyncio.TaskGroup`.

**Async y cancelación:** en código concurrente, `CancelledError` es una señal de control, no necesariamente un incidente. Un `except Exception: pass` genérico puede destruir la semántica de cancelación según cómo se propague — distingue siempre fallo de negocio, fallo técnico y cancelación.

**Background jobs y colas:** cuando el error ocurre en un worker, no hay un usuario esperando en el mismo stack frame. La política pasa a ser `retry → backoff → max attempts → Dead Letter Queue (DLQ)`. Un mensaje que falla repetidamente por una condición que el retry no resolverá (p. ej. `{"user_id": null}`) es un **poison message**: debe clasificarse como error de validación y enviarse directo a la DLQ, no reintentarse indefinidamente.

---

## 19. Seguridad

```text
No filtrar secretos/PII mediante mensajes de excepción (ver §14).
No exponer tracebacks internos ni detalle de SDK al usuario final.
No convertir errores de autenticación en retry automático.
No permitir que un error de una tool con side effects se reintente
    sin verificar idempotencia (riesgo de acción duplicada, p. ej.
    reenvío de un correo o una cancelación de pedido).
```

No se añade aquí una lista genérica de seguridad: solo los puntos donde el manejo de excepciones es, específicamente, una superficie de riesgo.

---

## 20. Observabilidad

```text
Exception
    +
Logs (structured)
    +
Metrics (error rate, retry rate, fallback rate)
    +
Traces (correlation ID / trace ID)
```

Ejemplo de logging estructurado:

```json
{
  "event": "llm_request_failed",
  "provider": "openai",
  "operation": "embedding",
  "error_type": "TimeoutError",
  "retryable": true,
  "attempt": 2,
  "request_id": "..."
}
```

Un `request_id`/`trace_id` permite reconstruir una operación que atraviesa varios servicios (`API → Agent → RAG → Vector DB → LLM`) como una sola historia, en vez de logs aislados sin relación entre sí.

**Actualización profesional:** desde Python 3.11, `exception.add_note(...)` permite añadir contexto a una excepción sin reemplazar el error original, y esas notas aparecen en el traceback; la documentación actual de la serie 3.14 mantiene esta capacidad vigente. [2]

```python
try:
    ...
except ProviderError as exc:
    exc.add_note("provider=openai")
    exc.add_note("operation=embedding")
    raise
```

La pregunta que siempre debe poder responderse: ¿qué falló, dónde, para qué request, con qué dependencia, cuántas veces y desde cuándo? Sin esto, el manejo de excepciones es casi ciego operacionalmente.

---

## 21. Coste

```text
Coste + Calidad + Latencia + Fiabilidad + Escala
```

Un retry mal diseñado no es "gratis": multiplica llamadas a proveedores LLM facturados por token, aumenta la carga sobre dependencias y puede disparar rate limits en cascada. Un circuit breaker o un fallback a un proveedor más barato son decisiones de coste, no solo de disponibilidad.

> No afirmes que una arquitectura es mejor únicamente porque sea más barata: declara también qué fiabilidad o latencia se sacrifica a cambio.

---

## 22. Tech Lead / Staff Thinking

| Nivel      | Pregunta que se hace                                                                 |
| ---------- | -------------------------------------------------------------------------------------- |
| Developer  | ¿Cómo implemento este `try/except`?                                                    |
| Senior     | ¿Por qué lo implementaría así? ¿Qué trade-off asumo?                                   |
| Tech Lead  | ¿Cómo afecta esta decisión al resto del sistema y al equipo que lo mantiene?           |
| Staff      | ¿Cómo afecta esta decisión a múltiples sistemas, equipos y a la evolución futura?       |

### Ejemplo aplicado

Un Junior dice: *"Uso `try/except` para que el programa no se caiga."*

Un Mid dice: *"Creo excepciones personalizadas y las capturo según el tipo."*

Un Senior dice: *"Diseño una taxonomía de errores, capturo en el nivel que tiene suficiente contexto para recuperarse, traduzco entre boundaries, preservo causalidad y aplico políticas distintas para errores permanentes y transitorios."*

Un Staff dice: *"Primero defino la semántica de fallo y los límites de responsabilidad. Después diseño propagación, recuperación, idempotencia, timeouts, retries, observabilidad y error boundaries de acuerdo con los requisitos de consistencia, latencia y disponibilidad del sistema."*

### Principio central de esta parte

> **No se trata de evitar que el programa falle. Se trata de diseñar qué significa fallar, y garantizar que el sistema falle de forma observable, segura, recuperable cuando sea posible y sin corromper su estado.**

---

## 22.1 Precisiones adicionales — casos límite

> **Advanced Python edge cases — conocer, no memorizar.**
>
> Todo lo que sigue en esta sección tiene ROI real pero **menor** que el contenido de §11-§21. Si el tiempo de estudio es limitado, prioriza siempre `retry`, `idempotency`, `error boundary`, `error contract` y `observability` sobre lo que viene a continuación — son los conceptos que un entrevistador Senior/Staff evalúa con mucha más frecuencia, y los que realmente determinan si un sistema falla bien o mal en producción. Los casos de esta sección no cambian una decisión de arquitectura; explican comportamientos puntuales de Python que, si se ignoran, producen bugs raros y difíciles de reproducir. Conócelos para no sorprenderte en code review — no los memorices como si fueran el núcleo de la clase.

Estos son comportamientos reales de Python que no aparecen en el material original y que suelen generar bugs difíciles de reproducir precisamente porque son infrecuentes. No requieren una sección propia por tema, pero sí una mención explícita.

### Caso límite 1 — `raise` dentro de un `except` sin `from` no destruye la excepción original

```python
try:
    validar()
except ValidationError:
    raise ProcessingError("no se pudo procesar")   # sin "from"
```

Aunque no se use `raise ... from`, Python conserva automáticamente la excepción original en `__context__` (distinto de `__cause__`, que solo se llena con `from` explícito). El traceback mostrará ambas, con el texto *"During handling of the above exception, another exception occurred"*. Si realmente se quiere ocultar esa cadena implícita (por ejemplo, porque el detalle interno no aporta nada y confunde a quien lee el log), debe hacerse explícito con `raise ... from None` — nunca dejarlo como un efecto colateral no intencionado.

### Caso límite 2 — `return`/`break`/`continue` en `finally` descarta la excepción silenciosamente

```python
def process():
    try:
        raise ValueError("dato inválido")
    finally:
        return "ok"   # la excepción se pierde por completo, sin warning visible en runtime
```

Ya se mencionó en la Parte 1 (§28 del material original) como mala práctica; la precisión adicional es que **esto no lanza ningún error en tiempo de ejecución** — el programa simplemente continúa como si `raise` nunca hubiera ocurrido. Python 3.14 añadió un `SyntaxWarning` en tiempo de compilación para algunos de estos patrones, pero el código sigue ejecutándose igual. La única defensa real es un linter (`ruff`/`flake8` con la regla correspondiente) y revisión de código — no confiar en que "el intérprete avisará".

### Caso límite 3 — un `__exit__` que retorna `True` suprime la excepción como un `except` invisible

```python
class SuppressErrors:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        return True   # suprime CUALQUIER excepción dentro del bloque `with`


with SuppressErrors():
    raise ValueError("esto desaparece sin dejar rastro")
```

Es funcionalmente equivalente a un `except Exception: pass`, pero mucho menos visible en una revisión de código porque no usa la palabra `except`. Si un context manager suprime excepciones intencionalmente, debe documentarlo explícitamente en su docstring y, salvo casos muy justificados, registrar lo suprimido antes de descartarlo.

### Caso límite 4 — `raise` sin excepción activa

```python
def f():
    raise   # RuntimeError: No active exception to re-raise
```

Un `raise` sin argumentos solo es válido dentro de un bloque `except` activo (o algo que esté manejando la propagación de una excepción). Usarlo fuera de ese contexto no relanza "la última excepción vista en algún lugar del programa" — produce su propio `RuntimeError`.

### Caso límite 5 — `except Exception` genérico y el protocolo de cierre de generadores

```python
def leer_lote():
    try:
        yield linea_1
        yield linea_2
    except Exception:
        # esto también intercepta GeneratorExit en versiones donde
        # GeneratorExit hereda de BaseException, NO de Exception —
        # por eso un except Exception genérico NO debería interferir aquí.
        # El riesgo real es capturar Exception y no volver a propagar
        # cuando el generador está siendo cerrado explícitamente con close().
        raise
```

`GeneratorExit` hereda de `BaseException`, no de `Exception`, por lo que un `except Exception:` no debería interceptarlo — pero un `except BaseException:` (mucho menos común, aunque existe en código defensivo mal escrito) sí lo haría, y omitir el `raise` final rompería el protocolo de cierre del generador de forma silenciosa. Regla práctica: dentro de un generador, cualquier bloque `except` amplio debe terminar con `raise` salvo que exista una razón explícita para no propagar.

### Caso límite 6 — excepciones y `multiprocessing`/colas distribuidas

Una excepción personalizada que se envía entre procesos (`multiprocessing`, Celery, colas) debe ser **picklable**. Si el `__init__` de la excepción no acepta exactamente los mismos argumentos que se le pasan a `super().__init__(...)`, el unpickling puede fallar silenciosamente del otro lado y perderse el traceback original:

```python
# Riesgo de pickling roto: el __init__ no reconstruye igual que se creó
class LibroNoDisponibleError(BibliotecaError):
    def __init__(self, titulo: str) -> None:
        self.titulo = titulo
        super().__init__(f"El libro '{titulo}' no está disponible")
        # Al hacer pickle/unpickle, Python reconstruye llamando
        # ExceptionClass(*self.args) — verificar que eso siga siendo válido.
```

En sistemas con workers y colas (relevante para la Parte 2, §18, DLQ), esto debe verificarse explícitamente con una prueba de round-trip: `pickle.loads(pickle.dumps(exc))`.

### Caso límite 7 — comparar excepciones con `==` no compara contenido

```python
LibroNoDisponibleError("Dune") == LibroNoDisponibleError("Dune")   # False
```

Las excepciones no implementan `__eq__` por defecto: dos instancias con el mismo mensaje **no son iguales**, son objetos distintos. En tests, comparar por `type(exc)` y `exc.args` (o los atributos estructurados, como `exc.titulo`), nunca por `==` directo, salvo que la excepción implemente `__eq__` explícitamente.

### Caso límite 8 — excepciones dentro de `__del__` se ignoran

```python
class Recurso:
    def __del__(self):
        raise RuntimeError("esto no detiene nada, solo se imprime a stderr")
```

Python no propaga excepciones lanzadas durante la destrucción de un objeto (`__del__`); se limita a imprimir una advertencia. Nunca depender de que un fallo dentro de `__del__` interrumpa el flujo del programa o sea capturable por un `except` — para cleanup garantizado, usar context managers (`__exit__`) o `finally`, no `__del__`.

---

## Checklist de la Parte 3 — Arquitectura

```text
[ ] Sé decidir en qué capa vive cada tipo de excepción.
[ ] Sé justificar una decisión arquitectónica con opción A/B, trade-offs y condición de cambio.
[ ] Diseño taxonomías de error que habilitan políticas, no solo nombres.
[ ] Considero partial failure en operaciones concurrentes (ExceptionGroup/except*).
[ ] Distingo cancelación de fallo real en código async.
[ ] Sé cuándo un mensaje fallido debe ir a DLQ en vez de reintentarse indefinidamente.
[ ] No filtro secretos ni información interna mediante excepciones.
[ ] Diseño observabilidad estructurada (logs + metrics + traces + correlation ID).
[ ] Razono el coste de una política de retry, no solo su disponibilidad.
[ ] Puedo articular la misma decisión desde la perspectiva Developer → Senior → Tech Lead → Staff.
[ ] Puedo definir un Error Contract formal (error_code, category, retryable, message, correlation_id) separado de la clase de excepción.
[ ] El mapeo de errores resuelve por jerarquía (__mro__), no solo por type(exc) exacto.
[ ] Sé qué campos del contrato son estables/accionables (error_code, category, retryable) frente a informativos (message).
[ ] Distingo explícitamente Exception → Application Error → API Contract → HTTP Status → User Message como cinco objetos distintos.
[ ] El HTTP Status y el User Message se derivan de una tabla de mapeo centralizada, nunca de type(exc) disperso en el código.
[ ] Conozco al menos los casos límite de raise en finally, __exit__ que suprime excepciones, GeneratorExit, y pickling de excepciones entre procesos.
```

---

# PARTE 4 — ENTREVISTAS, RECURSOS, GLOSARIO Y CONSOLIDACIÓN

## 23. Preguntas por concepto aprendido

### ¿Dónde debería capturarse una excepción?

- **Nivel:** Senior/Staff
- **Qué evalúa:** propagación, límites arquitectónicos, separación de responsabilidades.
- **Qué NO responder:** "Donde aparezca el error" o "en cada función por si acaso".
- **Por qué no:** convierte el manejo de errores en ruido y oculta fallos inesperados (ver §9.1, error swallowing).
- **Qué responder:** en el punto donde existe suficiente contexto para tomar una decisión útil sobre el error; si la función no sabe si debe reintentar, hacer fallback, abortar o mandar a una DLQ, probablemente no debe capturarlo.
- **Respuesta de alto impacto:** *"Catch where you can handle, not where you can observe."*
- **Follow-up:** "¿Y si solo quieres loguear el error?" → capturar, añadir contexto (`add_note`/logging estructurado) y volver a lanzar (`raise`), no absorberlo.
- **Red flags:** `except Exception: pass`, `except Exception as e: print(e)`.

### ¿Por qué `except Exception` puede ser peligroso?

- **Nivel:** Intermedio/Senior
- **Qué evalúa:** distinción entre errores recuperables, inesperados, de programación, de infraestructura y señales de control del proceso.
- **Qué NO responder:** "Porque es una mala práctica" sin más.
- **Qué responder:** `Exception` es una categoría amplia; puede ser correcto en un error boundary (logging + respuesta segura), pero peligroso como estrategia dentro de cada función, porque oculta la clasificación necesaria para decidir retry, fallback o propagación.
- **Respuesta de alto impacto:** *"`except Exception` como boundary global es observabilidad; como atajo dentro de un servicio es pérdida semántica del error."*
- **Trade-off:** un boundary demasiado amplio puede ocultar bugs; uno demasiado estrecho deja errores sin manejar escapando a producción.
- **Follow-up:** "¿Captura `KeyboardInterrupt`?" → No, porque hereda de `BaseException`, no de `Exception`.
- **Red flags:** usar `except Exception` para "que no se caiga" sin logging ni re-propagación.

### ¿Cuándo usarías `ValueError` y cuándo una excepción personalizada?

- **Nivel:** Básico/Intermedio
- **Qué evalúa:** diseño de API estable y semánticamente expresiva.
- **Qué NO responder:** "Siempre uso `Exception` genérica, es más simple".
- **Qué responder:** `ValueError` para problemas genéricos de valor de argumento; excepción de dominio cuando la condición tiene significado y política propia (permite `except LibroNoDisponibleError:` en vez de parsear el mensaje).
- **Respuesta de alto impacto:** *"Uso excepciones estándar para errores genéricos del lenguaje; creo excepciones de dominio cuando una condición necesita significado y política propia."*
- **Follow-up:** "¿Crearías una excepción por cada validación de campo?" → No; solo si distintos consumidores necesitan reaccionar distinto.

### ¿Qué diferencia hay entre `raise`, `raise NuevaExcepcion(...)` y `raise ... from e`?

- **Nivel:** Intermedio/Senior
- **Qué evalúa:** comprensión real de propagación y causalidad.
- **Qué responder:** `raise` sin argumento relanza la excepción actual conservando su traceback; `raise Nueva(...)` la reemplaza; `raise Nueva(...) from e` la traduce preservando la causa (`__cause__`) para debugging entre capas.
- **Respuesta de alto impacto:** *"Traduzco la excepción cuando atraviesa una frontera arquitectónica, pero conservo la causalidad con `from`."*
- **Follow-up:** "¿Cuándo usarías `from None`?" → Cuando el detalle interno no aporta valor al consumidor y deliberadamente no quieres exponer la cadena de implementación, no como forma habitual de "limpiar" errores.

### ¿Para qué sirve `else` en `try/except` y qué problema tiene `finally`?

- **Nivel:** Intermedio
- **Qué evalúa:** precisión del área protegida por el `try` y uso correcto de cleanup.
- **Qué responder:** `else` ejecuta solo si el `try` terminó sin excepción, permitiendo mantener el bloque `try` pequeño y evitar capturar accidentalmente errores de operaciones que no pretendías proteger; `finally` debe reservarse para limpieza/garantías (cerrar conexiones, liberar recursos), no para lógica de negocio — un `return` dentro de `finally` puede reemplazar el resultado o suprimir la propagación de una excepción, y Python 3.14 emite `SyntaxWarning` para ciertos casos problemáticos.
- **Respuesta de alto impacto:** *"Cuanto más pequeño es el `try`, más precisa es la semántica del manejo de errores."*

### ¿Cómo diseñarías una jerarquía de excepciones para una aplicación AI?

- **Nivel:** Senior/Staff
- **Qué evalúa:** capacidad de traducir requisitos de negocio en un modelo de errores accionable.
- **Qué NO responder:** enumerar 15 clases de excepción sin explicar su política.
- **Qué responder:** partir de las políticas de manejo (rate limit → backoff/retry; auth → no retry; timeout → retry limitado/fallback; invalid request → no retry) y solo entonces derivar la jerarquía (ver §17).
- **Respuesta de alto impacto:** *"El valor de la jerarquía no está en tener muchos nombres, sino en permitir que el sistema tome decisiones distintas de forma explícita."*
- **Follow-up Staff:** "¿Cómo evitas que un cambio de SDK rompa tu aplicación?" → con un adapter que traduzca las excepciones del SDK a tu propio contrato (§14).

### ¿Cómo manejarías un error de RAG o de una herramienta de un agente?

- **Nivel:** Senior/Staff (AI Engineering)
- **Qué evalúa:** si reduces "AI Engineering" a "llamar una API" o si diseñas el sistema completo.
- **Qué NO responder:** `except Exception: return "No se encontró información."` — oculta si falló la vector DB, el embedding, la autenticación o si simplemente no había resultados relevantes.
- **Qué responder:** distinguir la etapa que falló (`EmbeddingError`, `VectorStoreError`, `RetrievalError`, `LLMProviderError`) y, para tools de agentes, distinguir herramientas de solo lectura (retry razonablemente seguro) de herramientas con side effects (retry solo con idempotencia verificada, para evitar p. ej. duplicar un `send_email()` o un `crear_pedido()`).
- **Respuesta de alto impacto:** *"Un agente robusto no es un agente que nunca falla; es un agente cuyo sistema de fallos es predecible, observable y controlado."*

---

## 24. Banco general de entrevistas

### 3 preguntas básicas

**B1. ¿Qué hace `raise` exactamente?**
Evalúa: comprensión literal del mecanismo. NO responder: "detiene el programa". Responder: interrumpe el flujo normal y comienza la propagación hasta un `except` compatible; si lo hay, la ejecución continúa después del bloque. Respuesta de alto impacto: *"`raise` no termina el programa, inicia la búsqueda de un manejador."* Error común: creer que toda excepción es fatal.

**B2. ¿Cuál es la diferencia entre `Exception` y `BaseException`?**
Evalúa: conocimiento de la jerarquía estándar. NO responder: "son lo mismo". Responder: `BaseException` es la raíz e incluye `SystemExit`, `KeyboardInterrupt` y `GeneratorExit`; las excepciones de aplicación deben heredar de `Exception`. Respuesta de alto impacto: *"`except Exception` no captura `KeyboardInterrupt` porque no hereda de `Exception`."* Error común: usar `except:` desnudo pensando que es equivalente a `except Exception:`.

**B3. ¿Por qué es mala práctica un `except:` desnudo?**
Evalúa: criterio básico de manejo de errores. NO responder: "porque sí, se ve feo". Responder: oculta indistintamente bugs, errores de configuración, de red, de validación e interrupciones del usuario, impidiendo diagnosticar el problema real. Respuesta de alto impacto: *"Un `except:` desnudo convierte cualquier fallo en el mismo mensaje genérico."* Error común: usarlo "temporalmente" y olvidarlo en producción.

### 3 preguntas intermedias

**I1. ¿Cuándo usarías una excepción personalizada en vez de `ValueError` o `TypeError`?**
Evalúa: aplicación y criterio de diseño de API. NO responder: "siempre" ni "nunca". Responder: cuando la condición representa una regla de dominio que otros componentes necesitan manejar explícitamente por tipo. Respuesta de alto impacto: ver §23. Trade-offs: más tipos = más superficie de API y de tests. Follow-up: "¿Cómo evitas explosión de tipos?" → agrupar por política de recuperación, no por causa textual. Red flags: crear una excepción por cada `if`.

**I2. ¿Qué problema tiene capturar una excepción y devolver `None` o una lista vacía?**
Evalúa: comprensión de pérdida semántica del error (§9.1). NO responder: "ninguno, así no se rompe el programa". Responder: puede convertir una caída de infraestructura en un resultado de negocio válido, corrompiendo la interpretación de las capas superiores. Respuesta de alto impacto: *"Un `None` que en realidad significa 'la base de datos está caída' es un bug disfrazado de dato."* Trade-offs: a veces `None` sí es un resultado de dominio legítimo (p. ej. "no encontrado" en una búsqueda); hay que distinguir resultado esperado de violación de contrato. Follow-up: "¿Cómo lo verificarías en code review?" → revisando si la capa que captura tiene realmente estrategia de recuperación. Red flags: `except Exception: return None` sin logging.

**I3. ¿Por qué reintentar (`retry`) no es sinónimo de resiliencia?**
Evalúa: madurez en sistemas distribuidos. NO responder: "porque puede fallar de nuevo". Responder: un retry sin clasificación, timeout, backoff/jitter y análisis de idempotencia puede producir un retry storm que amplifica el incidente y, en operaciones no idempotentes, duplica efectos. Respuesta de alto impacto: ver §11. Trade-offs: más resiliencia normalmente implica más latencia percibida en el peor caso. Follow-up: "¿Cómo limitarías el impacto?" → retry budget + circuit breaker. Red flags: `except Exception: retry()` sin límite.

### 3 preguntas avanzadas

**A1. ¿Cómo diseñarías el manejo de errores de un pipeline RAG en producción?**
Evalúa: arquitectura end-to-end, no solo sintaxis. NO responder: una única captura genérica al final del pipeline. Responder: distinguir cada etapa (`EmbeddingError`, `VectorStoreError`, `RetrievalError`, `LLMProviderError`) y decidir en cada una si se recupera, se degrada (p. ej. responder sin contexto adicional) o se propaga como fallo del sistema. Respuesta de alto impacto: ver §23 (RAG). Trade-offs: mayor granularidad = mayor complejidad de código y de tests. Follow-up Senior/Staff: "¿Cómo diferenciarías 'no hay documentos relevantes' de 'la vector DB está caída'?" → la primera es un resultado válido del dominio; la segunda debe propagarse como `RetrievalError` y no camuflarse como lista vacía. Red flags: `except Exception: return "No se encontró información."`.

**A2. ¿Cómo evitarías que un agente duplique una acción con efectos secundarios tras un timeout?**
Evalúa: comprensión de idempotencia + retry + estado del agente. NO responder: "reintento la tool call automáticamente". Responder: distinguir tools de solo lectura de tools con side effects; para estas últimas, usar idempotency keys o verificación de estado antes de reintentar, y considerar que un timeout no implica que la operación no se ejecutó del lado del servidor. Respuesta de alto impacto: ver §12. Trade-offs: verificar idempotencia añade latencia y complejidad al diseño de cada tool. Follow-up Staff: "¿Cómo lo probarías?" → tests que simulan pérdida de respuesta tras ejecución exitosa del lado servidor. Red flags: asumir que timeout siempre significa "no se ejecutó".

**A3. ¿Cómo diseñarías observabilidad para distinguir un incidente real de ruido normal?**
Evalúa: pensamiento de Staff/SRE aplicado a excepciones. NO responder: "reviso los logs cuando algo falla". Responder: logging estructurado con `error_type`, `provider`, `retryable`, `attempt`, `request_id`/`trace_id`; comparar tasa de error actual contra baseline histórico (p. ej. 5% vs. 0.2% habitual) para decidir si es tendencia o incidente aislado. Respuesta de alto impacto: ver §20. Trade-offs: más instrumentación = más coste de almacenamiento y mantenimiento de dashboards. Follow-up: "¿Qué harías si el error rate sube pero no hay excepciones nuevas en el código?" → sospechar de una dependencia externa (proveedor LLM, DB) y correlacionar por `trace_id` a través de servicios. Red flags: depender solo de `print(e)` como mecanismo de diagnóstico en producción.

---

## 25. Preguntas Senior / Staff — diferenciación de respuestas

**Pregunta:** *"Este código: `except Exception: return None`. ¿Qué opinas?"*

```text
Junior → "Está bien, así no se rompe el programa."

Mid    → "Debería capturar excepciones más específicas."

Senior → "Depende de la capa: si es un repository, probablemente está
          ocultando una caída de infraestructura como si fuera 'no
          encontrado'. Necesito saber si None es un resultado de
          dominio válido o una pérdida de información."

Staff  → "El problema no es solo esta línea: es si el sistema tiene
          una taxonomía de errores que permita distinguir 'resultado
          esperado' de 'infraestructura caída', y si existe un error
          boundary superior donde SÍ es apropiado un except Exception
          amplio con logging y respuesta segura."
```

Este patrón de diferenciación (Junior → Mid → Senior → Staff) es la forma más rápida de entrenar criterio de entrevista: cada nivel no da una respuesta "más larga", da una respuesta que evalúa un nivel de contexto distinto.

---

## 26. System Design Interview — cuando corresponda

Ejemplo de guion para *"Diseña el manejo de errores de una API de agentes con RAG y múltiples proveedores LLM"*:

```text
Clarificar requisitos
    ↓ ¿qué SLA de disponibilidad? ¿hay proveedor de fallback?
Estimar escala
    ↓ ¿cuántas requests/seg? ¿cuántas tools por agente?
Definir restricciones
    ↓ presupuesto de latencia total, coste por token
Arquitectura
    ↓ Domain / Application / Infrastructure / API (§16)
Bottlenecks
    ↓ rate limit del proveedor LLM, timeout de vector DB
Fallos
    ↓ taxonomía de errores (§17), partial failure (§18)
Seguridad
    ↓ no filtrar secretos, no exponer SDK interno (§19)
Observabilidad
    ↓ logging estructurado + trace ID (§20)
Coste
    ↓ impacto de retries en facturación de tokens (§21)
Trade-offs
    ↓ granularidad de errores vs. complejidad de mantenimiento
Evolución
    ↓ ¿qué cambia si agregamos un tercer proveedor LLM?
```

---

## 27. Tech Lead / Leadership Interview

Preguntas de escenario (hipotético, sin inventar experiencia personal del usuario):

- ¿Cómo resolverías un desacuerdo con otro Senior sobre si un error debe capturarse en el repository o en el service?
- ¿Cómo defenderías, frente a Producto, que un retry automático en pagos requiere trabajo adicional (idempotency keys) antes de lanzarse?
- ¿Cómo priorizarías arreglar un `except Exception: pass` heredado frente a una nueva feature?
- ¿Cómo manejarías un incidente donde un retry storm tumbó un proveedor LLM compartido por varios equipos?
- ¿Cómo convencerías a otro equipo de adoptar tu taxonomía de errores en vez de la suya?
- ¿Cómo mentorearías a un Senior Engineer que abusa de `except Exception` "por seguridad"?

Guía de respuesta de alto impacto para todas: ancla la respuesta en **contexto + trade-off explícito + condición de cambio**, no en autoridad ("porque yo lo digo") ni en absolutos ("nunca se debe usar except Exception").

---

## 28. Ejercicio práctico

**Enunciado:** el repository de usuarios actual hace:

```python
def get_user(self, user_id: str):
    try:
        return self._db.query_user(user_id)
    except Exception:
        return None
```

**Tarea:** rediseña este método aplicando lo aprendido en la clase.

**Razonamiento esperado (antes del código):**
1. `except Exception: return None` mezcla "usuario no existe" (resultado de dominio) con "la base de datos falló" (infraestructura) — pérdida semántica (§9.1).
2. Se debe capturar solo la excepción específica del driver de base de datos, no `Exception` genérica.
3. Un error de infraestructura debe traducirse y propagarse (`raise ... from`), no silenciarse.

**Implementación:**

```python
class UserRepositoryError(Exception):
    """Fallo de infraestructura al acceder a usuarios."""


class UserRepository:
    def get_user(self, user_id: str) -> User | None:
        try:
            return self._db.query_user(user_id)
        except DatabaseTimeoutError as exc:
            raise UserRepositoryError(
                "No se pudo consultar el usuario"
            ) from exc
```

Nota: si `query_user` ya devuelve `None` de forma nativa cuando el usuario no existe (sin lanzar excepción), esa parte del contrato se conserva sin cambios — el fix es exclusivamente sobre el `except Exception` que enmascaraba fallos de infraestructura.

**Testing:** un test debe verificar que un `DatabaseTimeoutError` real se traduce a `UserRepositoryError` (no se silencia), y otro que un usuario inexistente sigue devolviendo `None` sin lanzar excepción.

**Pregunta de entrevista asociada:** "¿Por qué separaste estos dos casos?" → Respuesta de alto impacto: *"Porque 'no encontrado' y 'no se pudo consultar' son señales completamente distintas para el consumidor, y colapsarlas en un mismo `None` le quita al sistema la capacidad de reaccionar correctamente a una caída de infraestructura."*

---

## 29. Checklist de dominio final

```markdown
- [ ] Puedo explicar el concepto con mis propias palabras.
- [ ] Puedo explicar cómo funciona la propagación de excepciones.
- [ ] Puedo implementar una jerarquía de excepciones con propósito.
- [ ] Puedo identificar cuándo usar una excepción y cuándo no.
- [ ] Puedo identificar cuándo NO capturar una excepción.
- [ ] Puedo comparar alternativas de diseño con trade-offs explícitos.
- [ ] Puedo diagnosticar un except Exception mal usado en code review.
- [ ] Puedo relacionar el manejo de errores con retries, timeouts e idempotencia.
- [ ] Puedo relacionarlo con AI Engineering (LLM providers, RAG, agentes).
- [ ] Puedo defender una decisión técnica ante otro Senior/Staff.
- [ ] Puedo responder preguntas de entrevista Senior/Staff sobre el tema.
```

---

## 30. Recursos oficiales

| Prioridad | Recurso                                                | Qué aporta |
| --------- | ------------------------------------------------------- | ---------- |
| Crítica   | Python — *Errors and Exceptions* (docs.python.org, 3.14) [1] | `try/except/else/finally`, `raise`, propagación |
| Crítica   | Python — *Built-in Exceptions* (docs.python.org, 3.14) [2]   | Jerarquía `BaseException`/`Exception`, `add_note()`, `ExceptionGroup` |
| Alta      | PEP 3134 — *Exception Chaining and Embedded Tracebacks*      | Base conceptual de `raise ... from`, `__cause__`, `__context__` |
| Alta      | PEP 654 — *Exception Groups and `except*`*                   | Manejo de múltiples excepciones concurrentes |
| Crítica (AI backend) | FastAPI — *Handling Errors*                        | `HTTPException`, exception handlers, traducción a HTTP |
| Alta      | Python — módulo `traceback` (docs.python.org, 3.14) [3]      | Inspección programática de tracebacks |
| Crítica   | Python — módulo `logging`                                    | Niveles de log, logging estructurado |
| Alta      | Python — `asyncio`                                           | `TaskGroup`, cancelación, concurrencia y `ExceptionGroup` |

No se citan URLs adicionales más allá de las ya verificadas en este documento para evitar referencias no confirmadas.

---

## 31. Glosario técnico de alto ROI

**Exception propagation** — proceso por el cual una excepción sube por la cadena de llamadas hasta encontrar un handler compatible.

**Error boundary** — punto arquitectónico donde una aplicación decide cómo transformar, registrar o presentar un error (API, worker, agente).

**Exception translation** — transformar una excepción de una capa/dependencia en una excepción del contrato de otra capa.

**Exception chaining** — preservar la relación causal entre una excepción original y una nueva (`raise ... from e`, `__cause__`, `__context__`).

**Retryable / non-retryable error** — clasificación de si repetir la operación puede razonablemente resolver el fallo.

**Backoff / jitter** — espaciado progresivo entre reintentos, con variación aleatoria para evitar sincronización de clientes.

**Retry storm** — sobrecarga generada cuando muchos clientes reintentan simultáneamente sobre un servicio ya degradado.

**Retry budget** — límite de tiempo/intentos/carga que una operación puede gastar intentando recuperarse.

**Idempotencia** — propiedad por la cual ejecutar una operación varias veces produce el mismo efecto que ejecutarla una vez.

**Side effect** — cambio producido fuera del cálculo local (crear pedido, enviar email, cobrar tarjeta).

**Circuit breaker** — patrón que detiene temporalmente las llamadas a una dependencia que falla repetidamente (CLOSED → OPEN → HALF-OPEN).

**Partial failure** — parte de un sistema distribuido falla mientras otras partes funcionan correctamente.

**Dead Letter Queue (DLQ)** — cola donde se aíslan mensajes que no pudieron procesarse tras agotar la política de retries.

**Poison message** — mensaje que falla repetidamente por una condición que el retry no resolverá.

**Error taxonomy** — clasificación estructurada de errores que representa políticas de manejo, no solo nombres.

**Error budget** — cantidad de fallo/indisponibilidad aceptable dentro de un objetivo de confiabilidad (conecta con SRE).

**Correlation ID / Trace ID** — identificadores que permiten relacionar eventos de una misma operación distribuida a través de servicios.

---

## 32. Principio final de la clase

> **No se trata de capturar errores. Se trata de diseñar el flujo de fallos.**

> **Detecta el problema donde tienes conocimiento suficiente para identificarlo. Propágalo hasta donde exista contexto suficiente para decidir. Captúralo únicamente cuando sepas qué hacer con él.**

> **Una excepción bien diseñada no es un mensaje de error: es parte del contrato entre componentes.**

### Las cinco preguntas que confirman dominio real

1. ¿Dónde capturo una excepción y por qué exactamente ahí?
2. ¿Cuándo debo propagarla y cuándo traducirla?
3. ¿Cómo sé si un error debe reintentarse?
4. ¿Qué cambia cuando la operación tiene side effects?
5. ¿Cómo diseño el sistema para que un fallo de LLM, RAG, herramienta o infraestructura no se convierta silenciosamente en un resultado incorrecto?

Si puedes responderlas con **trade-offs, causalidad, observabilidad, idempotencia y límites arquitectónicos**, ya no estás razonando como alguien que conoce `try/except`. Estás razonando como un **Senior AI Engineer con criterio de Tech Lead**.

---

**Referencias:**
[1] docs.python.org — *Errors and Exceptions*, Python 3.14.7 (vigente, serie estable actual a septiembre de 2026; 3.15 en pre-lanzamiento con fecha planeada para octubre de 2026).
[2] docs.python.org — *Built-in Exceptions*, Python 3.14.7.
[3] docs.python.org — módulo `traceback`, Python 3.14.7.
[4] FastAPI — *Handling Errors* (fastapi.tiangolo.com).