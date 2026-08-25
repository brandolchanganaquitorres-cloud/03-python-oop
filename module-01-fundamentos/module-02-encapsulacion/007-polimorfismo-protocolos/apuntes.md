# Clase 7: Python Protocol for Flexible Polymorphism
### Entrenamiento Senior AI Engineer / Tech Lead

---

# PARTE 1 — FUNDAMENTOS Y EXPLICACIÓN

## 1. Objetivo de la clase

Al finalizar esta clase, el ingeniero debe ser capaz de:

- Explicar la diferencia entre **tipado nominal** (herencia) y **tipado estructural** (`Protocol`), y justificar técnicamente cuándo usar cada uno.
- Diseñar contratos de interfaz en Python sin acoplar el código a una jerarquía de clases.
- Entender qué ocurre internamente cuando `mypy` valida un `Protocol` y qué ocurre en runtime cuando no se cumple.
- Aplicar `Protocol` como herramienta de **Dependency Inversion** en sistemas reales (clientes de proveedores de IA, repositorios, servicios).
- Defender, en una entrevista Senior/Staff, la decisión de usar `Protocol` frente a `ABC` o frente a no tipar en absoluto.

Este no es un repaso de sintaxis. Es la base de cómo se diseñan **interfaces desacopladas y testeables** en sistemas Python de producción.

## 2. Contenido del material fuente

El material entregado ("Python Protocol for Flexible Polymorphism") cubre:

- Polimorfismo como pilar de OOP, con el caso previo `solicitar_libro` en `Estudiante` y `Profesor`.
- Duck typing como fundamento nativo de Python.
- Introducción a `typing.Protocol`: sintaxis, herencia de `Protocol`, cuerpo `...`.
- Aplicación de un `Protocol` para tipar una colección heterogénea (`list[SolicitanteProtocol]`).
- Comportamiento ante un objeto que no cumple el contrato: error estático (editor/mypy) + `AttributeError` en runtime.
- Un reto propuesto (`LibroProtocol`, `prestar`, `calcular_duracion`), que se resuelve en la sección 28 de esta misma clase.

El material es correcto en su núcleo conceptual. No se detectaron errores graves; sí omisiones relevantes para nivel Senior, cubiertas como ampliación profesional.

## 3. Conceptos fundamentales

| Concepto | Prioridad (Pareto) |
|---|---|
| Polimorfismo (repaso) | Complementario |
| Duck typing | Importante |
| `typing.Protocol` | **Crítico** |
| Tipado estructural vs. nominal | **Crítico** |
| Validación estática (mypy) vs. dinámica (runtime) | **Crítico** |
| `@runtime_checkable` | Importante *(ampliación profesional)* |
| `Protocol` vs `ABC` | **Crítico** *(ampliación profesional)* |

## 4. Explicación técnica

### 4.1 Polimorfismo — el porqué, no solo el qué

El polimorfismo evita que el código consumidor necesite conocer el tipo concreto de un objeto. Sin él, cada llamador haría `if isinstance(x, Estudiante): ... elif isinstance(x, Profesor): ...` — antipatrón que rompe Open/Closed cada vez que aparece un nuevo tipo.

### 4.2 Duck typing — el mecanismo que lo habilita

Python resuelve `usuario.solicitar_libro(...)` mediante **atributo dinámico**: en runtime busca el método en la instancia, luego en la clase, luego en la cadena MRO. Si lo encuentra, lo ejecuta; si no, lanza `AttributeError`. Esto permite que cualquier objeto con la forma correcta funcione, sin relación de herencia declarada.

### 4.3 `typing.Protocol` — formalizando el duck typing

```python
from typing import Protocol


class SolicitanteProtocol(Protocol):
    def solicitar_libro(self, titulo: str) -> str:
        """Solicita el préstamo de un libro por su título."""
        ...
```

`Protocol` no es una clase base. Activa un modo de verificación distinto en los type checkers (`mypy`, `pyright`): en lugar de "¿esta clase hereda de X?", preguntan "¿esta clase tiene todos los miembros que X declara, con firmas compatibles?".

### 4.4 Tipado estructural vs. tipado nominal

| | Nominal (`ABC` / herencia) | Estructural (`Protocol`) |
|---|---|---|
| Relación de tipos | Declarada explícitamente | Inferida por la forma del objeto |
| Acoplamiento | El implementador depende de la base | El implementador no conoce el Protocol |
| Validación | Definición de clase (+ runtime con ABC) | Análisis estático (mypy/pyright) |
| Clases de terceros | Difícil de adaptar | Se adapta sin modificarlas |

> **Ampliación profesional:** por esto `Protocol` es el mecanismo idiomático para tipar dependencias externas (SDKs, clientes de LLMs, librerías de terceros) sin wrappers forzados por herencia.

## 5. Funcionamiento interno / Deep Dive

### 5.1 Cómo valida `mypy` un Protocol

1. Extrae los miembros declarados en el Protocol.
2. Para cada objeto, verifica que su clase tenga un atributo con ese nombre.
3. Compara la firma encontrada contra la declarada (contravariancia en parámetros, covariancia en retorno).
4. Si falta un miembro o la firma es incompatible, reporta error — sin ejecutar código.

Este proceso es *structural subtyping check*, análogo a interfaces en TypeScript o Go.

### 5.2 Qué pasa en runtime

`SolicitanteProtocol` **no participa** en la resolución del método en runtime por defecto. Python simplemente busca el atributo (ver 4.2). Si no existe, `AttributeError`.

> **Punto clave Tech Lead:** `Protocol`, por defecto, es una herramienta de análisis **estático**, no de validación en runtime. Si el código corre sin pasar por `mypy` en CI, el Protocol no protege nada por sí mismo.

### 5.3 `@runtime_checkable`

> **Ampliación profesional** (no presente en el material fuente)

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class SolicitanteProtocol(Protocol):
    def solicitar_libro(self, titulo: str) -> str:
        ...
```

Habilita `isinstance(libro, SolicitanteProtocol)`.

**Limitación crítica para entrevista:** solo verifica que los **nombres** de los métodos existan, no valida firmas (tipos de parámetros ni retorno). Es una verificación de "forma superficial", no de tipos real. Confundir esto es un error típico de nivel Mid.

### 5.4 Internals: `_ProtocolMeta`

`Protocol` se implementa mediante una metaclase especial (`_ProtocolMeta`, heredada de `ABCMeta`). Esto explica:

- Por qué puede usarse con `isinstance()` cuando es `runtime_checkable` (reutiliza `ABCMeta.__instancecheck__`).
- Por qué **no se puede instanciar** un Protocol directamente (`SolicitanteProtocol()` lanza `TypeError`) — describe forma, no una entidad concreta.

## 6. Código y ejemplos

```python
from typing import Protocol


class SolicitanteProtocol(Protocol):
    """Contrato estructural para cualquier entidad que pueda solicitar un libro."""

    def solicitar_libro(self, titulo: str) -> str:
        """Solicita el préstamo de un libro por su título."""
        ...


def procesar_solicitudes(usuarios: list[SolicitanteProtocol], titulo: str) -> None:
    """Itera una colección de solicitantes y procesa su solicitud."""
    for usuario in usuarios:
        print(usuario.solicitar_libro(titulo))
```

El código del material ya respeta PEP 8. Se agregó `procesar_solicitudes` para mostrar el patrón dentro de una unidad reutilizable, sin sobreingeniería.

## 7. Correcciones técnicas detectadas

No se detectaron errores conceptuales. Las afirmaciones sobre duck typing y el doble nivel de error (estático + `AttributeError`) son correctas y consistentes con [PEP 544 – Structural subtyping](https://peps.python.org/pep-0544/).

**Matiz de precisión:** el material afirma que "el editor marca el error inmediatamente". Esto depende de tener un type checker activo (`mypy`/`pyright`) configurado en el editor. Sin esa herramienta, no hay señal alguna hasta el runtime. `Protocol` aporta valor **cuando está integrado en un pipeline de verificación** (editor + CI), no por sí solo.

---

# PARTE 2 — PROFESIONALIZACIÓN

## 8. Curso vs. práctica profesional actual

| Material del curso | Práctica profesional (2026) |
|---|---|
| Un solo Protocol con un método | Protocols como "puertos" en arquitectura hexagonal, con múltiples métodos cohesivos |
| No menciona `@runtime_checkable` | Se usa selectivamente en boundaries donde no hay garantía de `mypy` previo (ej. deserialización dinámica) |
| No compara con `ABC` | En producción, la elección entre `Protocol` y `ABC` es una decisión arquitectónica explícita (ver sección 16) |
| No menciona herramientas de CI | `mypy --strict` (o `pyright`) en pipeline de CI es el estándar para que Protocol tenga efecto real |

Estado: el contenido del curso está **vigente**, pero incompleto para producción — es la base correcta, no la práctica completa.

## 9. Buenas prácticas

- **Un Protocol, una responsabilidad.** Evitar Protocols con muchos métodos no cohesivos — viola Interface Segregation (SOLID).
- **Nombrar el Protocol por el rol, no por la implementación.** `SolicitanteProtocol` es mejor que `TieneMetodoSolicitar`.
- **Docstrings en cada método del contrato.** El Protocol es documentación viva del sistema; su claridad importa más que en una clase concreta.
- **`@runtime_checkable` solo cuando se necesite `isinstance` real.** Usarlo por defecto agrega falsa sensación de seguridad de tipos.
- **`mypy --strict` en CI**, no solo en el editor local — de lo contrario el Protocol no se aplica de forma consistente en el equipo.

## 10. Problemas reales en producción

```text
Problema
↓
Un Protocol se extiende agregando un método nuevo.
↓
Causa
↓
Cambio de contrato sin coordinar con los equipos que implementan clases que lo satisfacen.
↓
Síntomas
↓
mypy empieza a fallar en múltiples módulos no relacionados directamente con el cambio.
↓
Diagnóstico
↓
El Protocol es un contrato compartido; extenderlo es un cambio breaking para todos los implementadores.
↓
Solución
↓
Versionar el Protocol (ej. SolicitanteProtocolV2) o hacer el nuevo método opcional mediante un Protocol adicional más pequeño (composición de Protocols).
↓
Prevención
↓
Tratar los Protocols como contratos públicos de API interna: cualquier cambio requiere el mismo cuidado que cambiar un endpoint REST.
```

## 11. Debugging

```text
Síntoma
↓
"AttributeError: 'Libro' object has no attribute 'solicitar_libro'" en producción, pese a que mypy no reportó error.
↓
Hipótesis
↓
El objeto fue construido dinámicamente (deserializado, mockeado, o generado por una librería) fuera del alcance del análisis estático.
↓
Experimento
↓
Reproducir el flujo de construcción del objeto y verificar su tipo real en runtime con type(obj) y dir(obj).
↓
Evidencia
↓
El objeto no es una instancia de la clase esperada por mypy (ej. viene de una fábrica genérica que retorna Any).
↓
Diagnóstico
↓
El type checker fue "engañado" por una anotación imprecisa (Any) en un punto anterior del flujo de datos.
↓
Corrección
↓
Tipar correctamente el punto de construcción del objeto; si es imposible, agregar una validación runtime explícita con @runtime_checkable + isinstance.
↓
Prevención
↓
Evitar Any en las fronteras del sistema; usar mypy --disallow-any-explicit en CI para detectar fugas de tipo.
```

## 12. Testing

Este es el punto de mayor impacto profesional de `Protocol`: **testabilidad sin herencia**.

```python
class SolicitanteFalso:
    """Test double que cumple SolicitanteProtocol sin heredar de nada."""

    def solicitar_libro(self, titulo: str) -> str:
        return f"Solicitud simulada: {titulo}"


def test_procesar_solicitudes(capsys) -> None:
    usuarios: list[SolicitanteProtocol] = [SolicitanteFalso()]
    procesar_solicitudes(usuarios, "Cien años de soledad")

    salida = capsys.readouterr().out
    assert "Solicitud simulada" in salida
```

`SolicitanteFalso` no hereda de nada — no necesita conocer `SolicitanteProtocol` para ser un test double válido. Esto reduce fricción frente a jerarquías `ABC`, donde el mock necesitaría heredar explícitamente o usar `unittest.mock.Mock(spec=...)`.

## 13. Aplicaciones reales

- **Repositorios de datos**: `RepositorioProtocol` con `guardar`, `obtener_por_id` — implementado por un repositorio en memoria (tests), uno en PostgreSQL (producción) y uno en un fake para desarrollo local.
- **Clientes de proveedores de LLM**: `LLMClientProtocol` con `generar(prompt: str) -> str` — implementado por un wrapper de Anthropic, uno de OpenAI y uno simulado para tests, sin que el código de negocio dependa del SDK concreto.
- **Notificadores**: `NotificadorProtocol` con `enviar(mensaje: str) -> None` — implementaciones por email, Slack, o consola, intercambiables por configuración.

## 14. Relación con el ecosistema AI

En sistemas de AI Engineering, `Protocol` es la base idiomática para desacoplar la lógica de negocio del proveedor de modelo concreto:

```python
class LLMClientProtocol(Protocol):
    def generar(self, prompt: str, *, max_tokens: int = 1024) -> str:
        ...
```

Cualquier wrapper de Anthropic, OpenAI o un modelo local puede satisfacer este contrato sin heredar de una clase base común del framework. Esto es lo que permite implementar **model routing** y **fallbacks** (cambiar de proveedor sin tocar la lógica de orquestación) — un requisito real de sistemas LLMOps en producción.

---

# PARTE 3 — ARQUITECTURA Y TECH LEAD

## 15. System Design

```text
Requisitos
↓
Sistema de préstamos con múltiples tipos de usuarios y tipos de material (libros físicos, electrónicos, futuros: audiolibros).
↓
Restricciones
↓
Debe soportar nuevos tipos de usuario/material sin modificar el código que los procesa.
↓
Escala
↓
Decenas de tipos de entidad a mediano plazo; no es un problema de volumen de datos, es un problema de extensibilidad de tipos.
↓
Arquitectura
↓
Capa de dominio expone Protocols (SolicitanteProtocol, PrestableProtocol); capa de infraestructura provee implementaciones concretas.
↓
Componentes
↓
Protocols (contratos) + implementaciones concretas + servicio de orquestación que solo conoce los Protocols.
↓
Datos
↓
No aplica directamente — Protocol es un mecanismo de tipado, no de persistencia.
↓
Comunicación
↓
Llamadas directas en memoria (no es un patrón de comunicación distribuida).
↓
Fallos
↓
Objeto que no cumple el contrato → detectado en CI (mypy) antes de deploy; si escapa, AttributeError en runtime, mitigado con logging.
↓
Seguridad
↓
No aplica de forma directa a Protocol (ver sección 19).
↓
Observabilidad
↓
Logging del tipo concreto que implementa el Protocol en cada operación, útil para auditar qué implementación se ejecutó.
↓
Coste
↓
Costo de desarrollo marginal (definir el contrato) vs. ahorro significativo en mantenibilidad a mediano plazo.
↓
Trade-offs
↓
Mayor indirección conceptual para desarrolladores Junior vs. mucho menor acoplamiento para el sistema completo.
↓
Evolución
↓
Nuevas implementaciones (ej. AudiolibroProtocol) se agregan sin modificar el servicio de orquestación existente (Open/Closed).
```

## 16. Decisiones arquitectónicas: `Protocol` vs `ABC`

```text
Opción A: typing.Protocol
Ventajas
  - No requiere herencia; funciona con clases de terceros.
  - Verificación en tiempo de análisis estático, sin coste en runtime.
  - Favorece bajo acoplamiento entre módulos.
Desventajas
  - No garantiza nada en runtime salvo que se use @runtime_checkable (y con validación superficial).
  - Requiere disciplina de equipo con mypy en CI; sin eso, pierde su valor.

Opción B: abc.ABC / abc.abstractmethod
Ventajas
  - Garantiza en tiempo de instanciación que los métodos abstractos están implementados (falla con TypeError si falta alguno).
  - Permite compartir implementación común (métodos concretos) además del contrato.
Desventajas
  - Fuerza herencia explícita; no aplica a clases de terceros sin modificarlas.
  - Acopla las implementaciones a una jerarquía específica del dominio.

Decisión
  Usar Protocol cuando el contrato debe ser satisfecho por clases que no controlas o que no deben acoplarse a una jerarquía común
  (SDKs externos, tests doubles, múltiples implementaciones independientes).
  Usar ABC cuando además del contrato necesitas compartir comportamiento común entre las implementaciones, y todas viven bajo tu control.

Trade-offs
  Protocol sacrifica la garantía en runtime a cambio de flexibilidad y desacoplamiento.
  ABC sacrifica flexibilidad a cambio de una garantía runtime más fuerte.

Condiciones de cambio
  Si el equipo detecta que objetos mal formados llegan sistemáticamente a producción pese a mypy en CI
  (por ejemplo, por deserialización dinámica), reconsiderar y migrar el contrato crítico a ABC o agregar
  validación runtime explícita con @runtime_checkable.
```

## 17. Trade-offs (resumen ejecutivo)

| Dimensión | Protocol | ABC |
|---|---|---|
| Acoplamiento | Bajo | Medio-alto |
| Garantía en runtime | Ninguna (salvo `@runtime_checkable`, superficial) | Fuerte (`TypeError` al instanciar) |
| Compartir código común | No | Sí |
| Aplica a clases de terceros | Sí | No, sin modificarlas |
| Costo de adopción en equipo | Requiere disciplina de tipado estático (mypy en CI) | Menor dependencia de herramientas externas |

## 18. Escalabilidad

`Protocol` no afecta la escalabilidad en el sentido de throughput o carga — es un mecanismo de tipado, sin coste en runtime salvo el uso puntual de `@runtime_checkable`. Su impacto en "escalabilidad" es organizacional: permite que **múltiples equipos implementen el mismo contrato de forma independiente** sin coordinación estrecha, lo cual sí escala en términos de velocidad de desarrollo en equipos grandes.

## 19. Seguridad

No hay implicaciones de seguridad directas en el uso de `Protocol` en sí. La única consideración relevante: si se usa `@runtime_checkable` como único mecanismo de validación de un objeto que proviene de una fuente no confiable (ej. deserializado de una API externa), recordar que **solo valida presencia de métodos, no su comportamiento** — no debe tratarse como una barrera de seguridad.

## 20. Observabilidad

Cuando existen múltiples implementaciones de un mismo Protocol en producción (ej. distintos proveedores de LLM tras un `LLMClientProtocol`), es una buena práctica registrar en logs/traces **qué implementación concreta se ejecutó** (`type(cliente).__name__`) para poder correlacionar comportamiento, latencia y costo por proveedor durante la operación del sistema.

## 21. Coste

El costo de introducir `Protocol` es principalmente de **diseño inicial** (definir el contrato correcto) y de **tooling** (mantener `mypy`/`pyright` en CI). No introduce costo de runtime relevante salvo el caso puntual de `@runtime_checkable`. El retorno es mantenibilidad y velocidad de desarrollo a mediano plazo, no una ganancia de performance.

## 22. Tech Lead / Staff Thinking

| Nivel | Pregunta que se hace |
|---|---|
| Developer | ¿Cómo implemento `SolicitanteProtocol` en mi clase? |
| Senior | ¿Por qué usar Protocol en vez de hacer que `Libro` herede de una clase base? |
| Tech Lead | ¿Qué pasa si otro equipo necesita agregar un nuevo tipo de solicitante? ¿El contrato actual lo soporta sin romper nada? |
| Staff | ¿Este patrón de contratos estructurales debería estandarizarse en todos los servicios que interactúan con proveedores externos (LLMs, pagos, notificaciones), para reducir el costo de cambiar de proveedor en el futuro? |

---

# PARTE 4 — ENTREVISTA Y CONSOLIDACIÓN

## 23. Preguntas por concepto aprendido

### Concepto: `typing.Protocol`

**Pregunta:** ¿Qué es `typing.Protocol` y qué problema resuelve que el duck typing tradicional no resuelve?
**Nivel:** Senior
**Qué evalúa:** Comprensión de la diferencia entre comportamiento dinámico de Python y verificación estática de tipos.
**Qué NO responder:** "Es una forma de hacer interfaces en Python."
**Por qué NO responder eso:** Es vago, no demuestra comprensión del mecanismo ni de cuándo aporta valor.
**Qué responder:** Explicar tipado estructural, verificación en tiempo de análisis estático, y que el valor real aparece cuando está integrado en un pipeline de CI con mypy/pyright.
**Respuesta de alto impacto:** "Protocol formaliza el duck typing: permite detectar en tiempo de análisis estático que un objeto no cumple el contrato esperado, antes de que falle en runtime con AttributeError."
**Follow-up probable:** "¿Qué pasa si ese código nunca pasa por mypy?"
**Respuesta al follow-up:** "Entonces Protocol no aporta ninguna protección — el comportamiento es idéntico al duck typing normal, con el mismo riesgo de AttributeError en runtime."
**Red flags:** Confundir Protocol con una clase abstracta convencional, o afirmar que Protocol valida algo en runtime por defecto.

### Concepto: `Protocol` vs `ABC`

**Pregunta:** ¿Cuándo elegirías `Protocol` sobre `ABC`, y viceversa?
**Nivel:** Senior/Staff
**Qué evalúa:** Criterio arquitectónico, no memorización de sintaxis.
**Qué NO responder:** "Protocol es más moderno, siempre lo uso."
**Por qué NO responder eso:** Ignora que ambos resuelven problemas distintos; sugiere seguir tendencias sin criterio.
**Qué responder:** Usar Protocol cuando el contrato debe ser satisfecho por clases externas o no relacionadas; usar ABC cuando se necesita compartir implementación concreta y todas las clases están bajo control del equipo.
**Respuesta de alto impacto:** "Depende de si necesito compartir comportamiento común. Si solo necesito un contrato de forma, Protocol reduce acoplamiento. Si necesito lógica compartida entre implementaciones, ABC es más apropiado."
**Follow-up probable:** "¿Puedes combinar ambos?"
**Respuesta al follow-up:** "Sí — se puede tener una ABC que además satisface un Protocol externo; no son mutuamente excluyentes."
**Red flags:** Presentar una de las dos opciones como universalmente superior.

### Concepto: `@runtime_checkable`

**Pregunta:** ¿Qué garantiza exactamente `isinstance()` sobre un Protocol decorado con `@runtime_checkable`?
**Nivel:** Senior
**Qué evalúa:** Precisión técnica; distinguir verificación superficial de verificación de tipos completa.
**Qué NO responder:** "Garantiza que el objeto es del tipo correcto."
**Por qué NO responder eso:** Es impreciso — no valida firmas, solo presencia de nombres de métodos.
**Qué responder:** Que solo verifica la existencia de los atributos/métodos declarados, no sus firmas (tipos de parámetros ni retorno).
**Respuesta de alto impacto:** "Solo valida que los métodos existan por nombre, no que las firmas sean compatibles. Es una verificación superficial, útil como guard, pero no reemplaza el chequeo estático de mypy."
**Follow-up probable:** "¿Cuándo usarías esto en producción?"
**Respuesta al follow-up:** "En fronteras donde no hay garantía de que el objeto pasó por mypy, como al recibir objetos de una librería externa o de deserialización dinámica."
**Red flags:** Afirmar que reemplaza la validación estática.

## 24. Banco general de entrevistas

### 3 Básicas

**1. ¿Qué es el duck typing en Python?**
- Qué evalúa: fundamento del lenguaje.
- Qué NO responder: "Es cuando usas tipos dinámicos."
- Qué responder: la validez de un objeto se determina por los métodos que expone, no por su clase.
- Respuesta de alto impacto: "Si un objeto tiene los métodos que necesito, Python lo trata como válido, sin importar su clase concreta."
- Error común: confundirlo con tipado dinámico en general (son conceptos relacionados, no idénticos).

**2. ¿Qué diferencia hay entre polimorfismo y sobrecarga de métodos?**
- Qué evalúa: fundamentos de OOP.
- Qué NO responder: "Son lo mismo."
- Qué responder: el polimorfismo es que el mismo método se comporta distinto según el objeto que lo ejecuta; Python no soporta sobrecarga de métodos por firma como Java/C++.
- Respuesta de alto impacto: "El polimorfismo es sobre comportamiento según el tipo del objeto en runtime; la sobrecarga (que Python no tiene nativamente) sería múltiples firmas del mismo nombre."
- Error común: asumir que Python tiene sobrecarga de métodos tradicional.

**3. ¿Por qué el cuerpo de un método en un Protocol es `...`?**
- Qué evalúa: comprensión del propósito de Protocol.
- Qué NO responder: "Es solo un placeholder porque no me acordé de la implementación."
- Qué responder: un Protocol describe forma, no comportamiento; el `...` indica explícitamente que no hay lógica real ahí.
- Respuesta de alto impacto: "Protocol define un contrato estructural, no una implementación; el `...` refuerza que ese método nunca se ejecuta desde ahí."
- Error común: intentar poner lógica real dentro del Protocol.

### 3 Intermedias

**1. ¿Qué ocurre si agrego un objeto que no cumple un Protocol a una lista tipada como `list[MiProtocol]`?**
- Qué evalúa: comprensión del doble nivel de verificación (estático + runtime).
- Qué NO responder: "Python lo bloquea automáticamente."
- Qué responder: mypy lo marca como error estático si se analiza; en runtime, Python no lo bloquea — solo falla cuando se invoca el método faltante, con AttributeError.
- Respuesta de alto impacto: "No hay bloqueo automático en runtime. mypy lo detecta si corre en el pipeline; si no, el fallo aparece como AttributeError al invocar el método ausente."
- Trade-offs: velocidad de desarrollo (no hay overhead runtime) vs. dependencia de disciplina de CI.
- Follow-up: "¿Cómo asegurarías que esto se detecte siempre, incluso sin mypy en el editor de cada developer?"
- Red flags: asumir que Python valida tipos en runtime por defecto.

**2. ¿Cómo testearías una función que depende de un Protocol, sin usar mocks de una librería?**
- Qué evalúa: aplicación práctica de tipado estructural en testing.
- Qué NO responder: "Uso `unittest.mock.Mock()` siempre."
- Qué responder: crear una clase simple ("test double") que implemente el método del Protocol directamente, sin herencia ni librerías de mocking.
- Respuesta de alto impacto: "Creo una clase mínima que implemente el método requerido — no necesita heredar del Protocol ni de nada, solo cumplir la forma."
- Trade-offs: más código explícito vs. mayor claridad y menor "magia" que un Mock genérico.
- Follow-up: "¿Cuándo preferirías `unittest.mock.Mock(spec=Protocol)` en su lugar?"
- Red flags: no saber que Mock también puede usarse con `spec` para Protocols.

**3. ¿Qué problema aparece si extiendes un Protocol agregando un método nuevo en un sistema con múltiples implementaciones?**
- Qué evalúa: criterio de mantenibilidad de contratos compartidos.
- Qué NO responder: "Ninguno, solo agrego el método."
- Qué responder: es un cambio breaking para todas las clases que deben satisfacer el Protocol; mypy fallará en todos los módulos que lo implementan sin el nuevo método.
- Respuesta de alto impacto: "Un Protocol es un contrato compartido; extenderlo rompe a todos los implementadores existentes, igual que cambiar un endpoint de API."
- Trade-offs: extender el Protocol existente (rompe compatibilidad) vs. crear un Protocol nuevo/versión (mantiene compatibilidad, más contratos que gestionar).
- Follow-up: "¿Cómo lo versionarías?"
- Red flags: no reconocer el impacto en otros equipos/módulos.

### 3 Avanzadas

**1. ¿Cómo verifica `mypy` internamente si una clase satisface un Protocol?**
- Qué evalúa realmente: comprensión de internals de análisis estático, no solo uso superficial.
- Qué NO responder: "Revisa si la clase hereda del Protocol."
- Qué responder: mypy compara estructuralmente los miembros declarados en el Protocol contra los de la clase candidata, verificando compatibilidad de firmas (contravariancia en parámetros, covariancia en retorno), sin requerir relación de herencia.
- Respuesta de alto impacto: "Es structural subtyping: mypy compara la forma y las firmas de los métodos, no la jerarquía de clases."
- Trade-offs: mayor flexibilidad vs. mayor complejidad conceptual para el equipo.
- Follow-up de nivel Senior/Staff: "¿Cómo se relaciona esto con la varianza de tipos (co/contravariancia) en Python?"
- Red flags: no poder explicar el concepto de compatibilidad de firmas más allá de "coinciden los nombres".

**2. Diseña la interfaz de un sistema que debe soportar múltiples proveedores de LLM intercambiables (Anthropic, OpenAI, modelo local) sin acoplar la lógica de negocio a ningún SDK concreto. ¿Usarías Protocol o ABC? Justifica.**
- Qué evalúa realmente: aplicación de tipado estructural a un problema real de AI Engineering (Dependency Inversion + model routing).
- Qué NO responder: "Uso directamente el SDK de OpenAI en el código de negocio."
- Qué responder: definir un `LLMClientProtocol` con la forma mínima necesaria (`generar(prompt) -> str`), implementado por wrappers independientes por proveedor; Protocol es preferible porque los SDKs son código de terceros que no se puede forzar a heredar de una base común.
- Respuesta de alto impacto: "Uso Protocol porque los SDKs de terceros no pueden adaptarse a una jerarquía propia; el contrato estructural permite intercambiar proveedores (incluyendo fallback y routing) sin tocar la lógica de negocio."
- Trade-offs: sin garantía runtime fuerte (mitigable con `@runtime_checkable` en el punto de carga de configuración) vs. desacoplamiento total del SDK.
- Follow-up de nivel Senior/Staff: "¿Cómo manejarías diferencias de comportamiento entre proveedores que técnicamente cumplen el mismo Protocol pero tienen semántica distinta (ej. límites de tokens, formato de errores)?"
- Red flags: acoplar la lógica de negocio directamente a un SDK concreto, o ignorar el problema de semántica divergente entre implementaciones.

**3. ¿Qué limitaciones tiene el tipado estructural de Python frente al de lenguajes como TypeScript o Go, y por qué importa en un sistema grande?**
- Qué evalúa realmente: profundidad comparativa y conciencia de los límites de la herramienta.
- Qué NO responder: "Ninguna, Protocol funciona igual."
- Qué responder: en Python, el tipado estructural es **opcional y no enforced en runtime** (a diferencia de Go, donde las interfaces son parte del sistema de tipos del compilador); esto significa que la garantía depende completamente de la disciplina del equipo con herramientas externas (mypy/pyright) en CI, no del propio lenguaje.
- Respuesta de alto impacto: "El tipado estructural en Python es un análisis externo y opcional; en Go es parte del compilador. Esto implica que en Python la garantía depende de la disciplina de CI del equipo, no del lenguaje en sí."
- Trade-offs: flexibilidad de un lenguaje dinámico vs. menor garantía de correctitud comparado con lenguajes con tipado estructural nativo.
- Follow-up de nivel Senior/Staff: "¿Cómo comunicarías a un equipo que ‘pasar mypy localmente’ no es suficiente garantía?"
- Red flags: no reconocer que el análisis de tipos en Python es opcional y externo al intérprete.

## 25. Preguntas Senior / Staff (diferenciación de respuestas)

**Pregunta:** "¿Por qué elegiste Protocol para este contrato?"

```text
Respuesta Junior:
  "Porque no quería usar herencia."

Respuesta Mid:
  "Porque Protocol permite que distintas clases cumplan el contrato sin heredar de una base común."

Respuesta Senior:
  "Porque el contrato debe ser satisfecho por SDKs de terceros que no controlo, y Protocol permite
   validación estática sin forzar una jerarquía. El trade-off es que no hay garantía en runtime salvo
   que agregue @runtime_checkable en los puntos de entrada críticos."

Respuesta Staff:
  "Además de lo anterior, esto establece un patrón reutilizable en la organización: cualquier integración
   futura con un proveedor externo (pagos, notificaciones, otros LLMs) puede seguir el mismo enfoque,
   reduciendo el costo de cambiar de proveedor y estandarizando cómo el equipo diseña boundaries externos."
```

## 26. System Design Interview (cuando corresponda)

```text
Clarificar requisitos
↓
¿Cuántos tipos de "solicitante" y "material" existen hoy? ¿Se espera que crezcan?
↓
Estimar escala
↓
No es un problema de volumen de datos; es un problema de variedad de tipos y equipos independientes implementándolos.
↓
Definir restricciones
↓
El servicio de orquestación no debe modificarse al agregar un nuevo tipo de solicitante o material.
↓
Arquitectura
↓
Protocols en la capa de dominio; implementaciones concretas en la capa de infraestructura; inyección de
implementaciones concretas en el servicio de orquestación.
↓
Bottlenecks
↓
No hay bottleneck de performance; el "bottleneck" es organizacional (coordinación al cambiar un contrato compartido).
↓
Fallos
↓
Objeto mal formado → mypy en CI (preferido) o AttributeError en runtime (fallback, mitigado con logging/alertas).
↓
Seguridad
↓
No aplica directamente a Protocol.
↓
Observabilidad
↓
Loggear qué implementación concreta se ejecutó en cada operación.
↓
Coste
↓
Bajo costo de runtime; costo principal es de diseño y de mantener mypy en CI.
↓
Trade-offs
↓
Indirección conceptual adicional vs. extensibilidad sin modificar código existente.
↓
Evolución
↓
Nuevos tipos se agregan implementando el Protocol existente, sin tocar el servicio de orquestación.
```

## 27. Tech Lead / Leadership Interview (cuando corresponda)

**¿Cómo defenderías la decisión de introducir Protocol en un equipo que nunca lo ha usado y lo considera "complejidad innecesaria"?**

Respuesta orientada a Tech Lead: mostrar el costo real que se evita (acoplamiento a SDKs de terceros, dificultad de testear con mocks basados en herencia), proponer una introducción incremental (empezar por un solo boundary externo, como el cliente de LLM), y medir el impacto en la facilidad de agregar un nuevo proveedor o de escribir tests, en lugar de imponerlo como estándar de golpe en todo el código base.

**¿Cómo priorizarías corregir un Protocol mal diseñado (con demasiados métodos no cohesivos) frente a otras tareas de backlog?**

Respuesta orientada a Tech Lead: evaluar cuántos equipos ya implementan ese Protocol (costo de cambio), si el problema está bloqueando features activamente, y proponer una migración gradual (Protocols más pequeños compuestos) en lugar de un refactor masivo de una sola vez.

## 28. Ejercicio práctico — Resolución del reto propuesto

Siguiendo la regla de la clase, primero el razonamiento y luego la implementación.

### Análisis del problema

Se necesita un contrato para objetos "prestables": libros físicos y electrónicos, cada uno con su propia lógica de duración de préstamo. La forma común, sin importar el tipo concreto, es: poder prestarse (`prestar`) y poder calcular su duración de préstamo (`calcular_duracion`).

### Diseño

```python
from typing import Protocol


class LibroProtocol(Protocol):
    """Contrato estructural para cualquier material prestable de tipo libro."""

    def prestar(self) -> str:
        """Registra el préstamo y retorna un mensaje de confirmación."""
        ...

    def calcular_duracion(self) -> int:
        """Retorna la duración del préstamo en días."""
        ...
```

### Implementación

```python
DURACION_LIBRO_FISICO_DIAS = 14
DURACION_LIBRO_ELECTRONICO_DIAS = 21


class LibroFisico:
    """Libro físico con duración de préstamo fija por política de biblioteca."""

    def __init__(self, titulo: str, autor: str) -> None:
        self.titulo = titulo
        self.autor = autor

    def prestar(self) -> str:
        return f"'{self.titulo}' prestado. Devolver en {self.calcular_duracion()} días."

    def calcular_duracion(self) -> int:
        return DURACION_LIBRO_FISICO_DIAS


class LibroElectronico:
    """Libro electrónico con mayor duración por no requerir devolución física."""

    def __init__(self, titulo: str, autor: str) -> None:
        self.titulo = titulo
        self.autor = autor

    def prestar(self) -> str:
        return f"'{self.titulo}' prestado en formato digital por {self.calcular_duracion()} días."

    def calcular_duracion(self) -> int:
        return DURACION_LIBRO_ELECTRONICO_DIAS
```

### Uso polimórfico

```python
libros: list[LibroProtocol] = [
    LibroFisico("Cien años de soledad", "Gabriel García Márquez"),
    LibroElectronico("Fahrenheit 451", "Ray Bradbury"),
]

for libro in libros:
    print(libro.prestar())
```

### Explicación de las decisiones de diseño

- Las constantes `DURACION_*_DIAS` en `UPPER_CASE` evitan "números mágicos" dentro del método — mejora legibilidad y facilita cambios de política sin tocar la lógica.
- Ninguna clase hereda de `LibroProtocol` ni de una base común: ambas cumplen el contrato solo por su forma, demostrando el patrón central de la clase.
- No se introdujo una clase abstracta, factory, ni inyección de dependencias — la complejidad se mantuvo proporcional al problema (regla de la sección 14 del sistema maestro: PEP 8 no significa sobreingeniería).

### Testing del ejercicio

```python
def test_libro_fisico_calcula_duracion_correcta() -> None:
    libro = LibroFisico("Test", "Autor")
    assert libro.calcular_duracion() == 14


def test_libro_electronico_calcula_duracion_correcta() -> None:
    libro = LibroElectronico("Test", "Autor")
    assert libro.calcular_duracion() == 21


def test_lista_polimorfica_de_libros(capsys) -> None:
    libros: list[LibroProtocol] = [
        LibroFisico("A", "B"),
        LibroElectronico("C", "D"),
    ]
    for libro in libros:
        print(libro.prestar())

    salida = capsys.readouterr().out
    assert "A" in salida and "C" in salida
```

### Pregunta de entrevista sobre este ejercicio

**Pregunta:** ¿Por qué `LibroFisico` y `LibroElectronico` no heredan de `LibroProtocol`?
**Respuesta de alto impacto:** "Porque Protocol define un contrato estructural, no una relación de herencia. Cualquier clase que implemente `prestar` y `calcular_duracion` con las firmas correctas satisface el contrato automáticamente — eso es lo que permite usarlas de forma polimórfica en la misma lista sin acoplarlas entre sí."

## 29. Checklist de dominio

- [ ] Puedo explicar el concepto con mis propias palabras.
- [ ] Puedo explicar cómo funciona internamente (mypy structural check + resolución dinámica de atributos en runtime).
- [ ] Puedo implementarlo (definir un Protocol y clases que lo satisfagan sin herencia).
- [ ] Puedo identificar cuándo utilizarlo (contratos con clases de terceros, boundaries externos, testing sin mocks pesados).
- [ ] Puedo identificar cuándo NO utilizarlo (cuando necesito compartir implementación concreta entre clases relacionadas).
- [ ] Puedo comparar alternativas (`Protocol` vs `ABC`).
- [ ] Puedo explicar los trade-offs (desacoplamiento vs. garantía en runtime).
- [ ] Puedo diagnosticar problemas relacionados (AttributeError pese a mypy limpio, Any que rompe la verificación).
- [ ] Puedo relacionarlo con producción (CI con mypy, versionado de contratos compartidos).
- [ ] Puedo relacionarlo con AI Engineering (Protocol como base de model routing y fallbacks entre proveedores de LLM).
- [ ] Puedo defender una decisión técnica (por qué Protocol y no ABC en un caso concreto).
- [ ] Puedo responder preguntas de entrevista sobre el tema, incluyendo follow-ups de nivel Staff.

## 30. Recursos

- [PEP 544 – Protocols: Structural subtyping (static duck typing)](https://peps.python.org/pep-0544/)
- [Documentación oficial de `typing.Protocol`](https://docs.python.org/3/library/typing.html#typing.Protocol)
- [Documentación oficial de `typing.runtime_checkable`](https://docs.python.org/3/library/typing.html#typing.runtime_checkable)
- [mypy — Protocols and structural subtyping](https://mypy.readthedocs.io/en/stable/protocols.html)

## 31. Glosario

- **Tipado estructural (structural typing):** un objeto satisface un tipo si tiene la forma requerida, independientemente de su jerarquía de clases.
- **Tipado nominal (nominal typing):** un objeto satisface un tipo solo si está declarado explícitamente como parte de esa jerarquía (herencia).
- **Duck typing:** principio de Python donde la validez de un objeto se determina por sus métodos disponibles, no por su clase.
- **`runtime_checkable`:** decorador que habilita `isinstance()` sobre un Protocol, validando solo presencia de métodos por nombre, no firmas.
- **Structural subtyping check:** proceso mediante el cual un type checker (mypy/pyright) compara los miembros de una clase contra los declarados en un Protocol.

## 32. Principio final de la clase

> Un Protocol no es una alternativa moderna a la herencia — es una herramienta distinta, con un propósito distinto: describir la **forma** que necesita un consumidor, sin imponerle al proveedor cómo debe construirse. Un Senior no elige Protocol porque "es lo nuevo"; lo elige cuando el desacoplamiento que ofrece resuelve un problema real de mantenibilidad, testabilidad o integración con código que no controla — y sabe exactamente qué garantías pierde al hacerlo.