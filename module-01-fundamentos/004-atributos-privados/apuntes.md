# Clase 04 — Encapsulación en Python: proteger estado, invariantes y comportamiento

> **Objetivo profesional:** comprender la encapsulación no como una simple técnica para "ocultar variables", sino como un principio de diseño que permite **proteger el estado de un objeto, preservar sus invariantes, controlar sus transiciones, ocultar detalles de implementación y reducir el acoplamiento entre componentes**.

Esta clase consolida la progresión natural de la clase anterior sobre métodos de instancia: primero aprendiste que un objeto puede contener **estado + comportamiento**; ahora aprenderás por qué ese estado no debería quedar expuesto a modificaciones arbitrarias.

La base de la clase anterior es precisamente que un objeto puede consultar su propio estado, modificarlo, validar reglas, ejecutar comportamientos y devolver resultados.

La encapsulación será especialmente importante cuando posteriormente trabajes con **SDKs, APIs, RAG, agentes, tools y MCP**, porque todos estos ecosistemas utilizan objetos y componentes que mantienen estado interno y exponen interfaces públicas.

---

# 1. Antes de encapsular: ¿qué problema estamos intentando resolver?

En la clase anterior construimos objetos capaces de comportarse:

```python
class Libro:

    def __init__(self, titulo, isbn, disponible=True):
        self.titulo = titulo
        self.isbn = isbn
        self.disponible = disponible

    def prestar(self):
        if self.disponible:
            self.disponible = False
            return f"{self.titulo} prestado exitosamente"

        return f"{self.titulo} no está disponible"

    def devolver(self):
        self.disponible = True
        return f"{self.titulo} ha sido devuelto y disponible nuevamente"

    def __str__(self):
        return f"{self.titulo} - {self.isbn}"
```

El objeto ya no es solamente:

```text
objeto = datos
```

Ahora tenemos:

```text
objeto = estado + comportamiento + reglas
```

Los métodos `prestar()` y `devolver()` representan acciones del dominio y producen transiciones de estado. La clase anterior utiliza precisamente este enfoque para pasar de simples atributos a objetos capaces de ejecutar comportamiento.

Pero todavía existe un problema.

Cualquier código externo puede hacer:

```python
libro.disponible = False
```

o incluso:

```python
libro.disponible = True
```

sin utilizar:

```python
libro.prestar()
```

Eso significa que estamos permitiendo que cualquier parte del programa manipule directamente el estado que debería estar sujeto a reglas.

Aquí aparece la necesidad de encapsular.

---

# 2. ¿Qué es la encapsulación?

La **encapsulación** es un principio de diseño mediante el cual un objeto controla cómo el código externo accede y modifica su estado interno.

La idea fundamental es:

> **El código externo no debería poder modificar arbitrariamente el estado interno de un objeto cuando existen reglas que deben mantenerse.**

Por ejemplo, imaginemos un libro:

```text
                    ┌──────────────────────┐
                    │        Libro         │
                    │                      │
                    │  título              │
                    │  autor               │
                    │  disponibilidad      │
                    │  préstamos           │
                    └──────────┬───────────┘
                               │
                         interfaz pública
                               │
                    ┌──────────▼───────────┐
                    │       Métodos        │
                    │                      │
                    │  prestar()           │
                    │  devolver()          │
                    │  es_popular()        │
                    │  consultar_estado()  │
                    └──────────────────────┘
```

El consumidor interactúa con el objeto mediante una **interfaz pública**, en lugar de manipular directamente todos sus detalles internos.

La encapsulación no significa simplemente "poner `__` delante de una variable". Es un principio mucho más amplio:

```text
Encapsulación
      │
      ├── controlar acceso al estado
      ├── controlar modificaciones
      ├── proteger invariantes
      ├── ocultar detalles de implementación
      ├── definir una interfaz pública
      └── reducir acoplamiento
```

Por tanto:

```text
Encapsulación ≠ __atributo
```

Más correctamente:

```text
Encapsulación
      ↓
Principio de diseño
      ↓
_, __, métodos, @property, etc.
      ↓
mecanismos que pueden ayudar a implementarlo
```

Esta distinción es fundamental para evitar una comprensión superficial del concepto.

---

# 3. La analogía del banco

Imagina que vas a un banco para retirar dinero.

No puedes entrar directamente a la bóveda y modificar el saldo que quieras.

Interactúas con una interfaz:

```text
Cliente
   │
   ▼
Cajero
   │
   ▼
Validaciones
   │
   ▼
Bóveda / sistema interno
```

El cajero no es el dinero.

El cajero es el **punto de control** entre el consumidor y el estado interno.

En un objeto ocurre algo similar:

```text
Código externo
      │
      ▼
Método / property
      │
      ▼
Validación
      │
      ▼
Estado interno
```

El objetivo no es impedir que el objeto sea utilizado.

El objetivo es impedir que sea utilizado **de cualquier manera**.

---

# 4. El problema real: integridad del estado

Consideremos:

```python
class Libro:

    def __init__(self, titulo, veces_prestado):
        self.titulo = titulo
        self.veces_prestado = veces_prestado

    def prestar(self):
        self.veces_prestado += 1

    def es_popular(self):
        return self.veces_prestado > 5
```

Creamos:

```python
mi_libro = Libro("Clean Code", 1)
```

Prestamos el libro:

```python
mi_libro.prestar()
```

Ahora:

```python
print(mi_libro.veces_prestado)
```

produce:

```text
2
```

Hasta aquí todo funciona.

Pero cualquier código externo puede hacer:

```python
mi_libro.veces_prestado = 1000
```

Ahora el objeto afirma:

```text
veces_prestado = 1000
```

aunque realmente solamente haya ocurrido un préstamo.

El problema profesional no es:

> "Alguien modificó una variable."

El problema real es:

> **El objeto ya no puede garantizar que su estado represente correctamente las reglas del dominio.**

El resultado puede propagarse:

```text
veces_prestado = 1000
          │
          ▼
   Estado inconsistente
          │
          ├── es_popular() incorrecto
          ├── estadísticas incorrectas
          ├── reportes incorrectos
          └── otros componentes reciben datos inválidos
```

El mismo patrón aparece en:

* inventarios;
* saldos;
* pagos;
* permisos;
* estados de procesos;
* contadores;
* disponibilidad;
* límites;
* configuraciones;
* APIs;
* datos persistidos;
* componentes de sistemas de IA.

---

# 5. El origen del problema: estado expuesto

Tenemos:

```python
libro.disponible = False
libro.veces_prestado += 1
```

El consumidor está modificando dos piezas del estado.

Pero realmente quería expresar una sola operación:

```python
libro.prestar()
```

Estas dos formas son conceptualmente diferentes.

### Manipulación directa

```python
libro.disponible = False
libro.veces_prestado += 1
```

El consumidor necesita conocer:

* qué atributos existen;
* qué significa cada uno;
* qué atributos debe modificar;
* en qué orden;
* qué validaciones son necesarias.

### Operación de dominio

```python
libro.prestar()
```

El consumidor solamente necesita conocer:

> "El libro sabe cómo prestarse."

La clase se encarga de:

```text
1. verificar disponibilidad
2. cambiar disponibilidad
3. incrementar contador
4. registrar información si corresponde
5. mantener las invariantes
6. devolver un resultado
```

La clase anterior ya introducía esta idea: `prestar()` es superior conceptualmente a `cambiar_disponibilidad()` porque expresa la intención del dominio y no un detalle técnico.

---

# 6. Estado + comportamiento

Una clase orientada a objetos combina normalmente:

```text
Estado
   +
Comportamiento
```

Nuestro `Libro` puede tener:

```text
Libro
│
├── Estado
│   ├── título
│   ├── disponibilidad
│   └── veces_prestado
│
└── Comportamiento
    ├── prestar()
    ├── devolver()
    └── es_popular()
```

La encapsulación controla la relación entre ambos.

Por ejemplo:

```python
libro.prestar()
```

es diferente de:

```python
libro.disponible = False
libro.veces_prestado += 1
```

La primera representa una **operación del dominio**.

La segunda expone directamente los detalles internos.

---

# 7. La verdadera pregunta de diseño

No empieces preguntando:

> "¿Qué variables debo hacer privadas?"

Pregunta:

> **"¿Qué estado debe protegerse y qué operaciones están autorizadas a modificarlo?"**

Este cambio de perspectiva es uno de los aprendizajes más importantes de la clase.

Por ejemplo:

```text
veces_prestado
      │
      ├── prestar() → incrementar
      │
      └── importación histórica → validar
```

Mientras que:

```text
titulo
autor
```

podrían no necesitar el mismo nivel de protección.

No todos los atributos necesitan encapsulación.

---

# 8. Atributos públicos

Un atributo público puede utilizarse directamente:

```python
class Libro:

    def __init__(self, titulo):
        self.titulo = titulo
```

Podemos hacer:

```python
libro = Libro("Clean Code")

print(libro.titulo)
```

y:

```python
libro.titulo = "Effective Python"
```

Si el dominio permite esta modificación, no existe necesariamente un problema.

La encapsulación no exige ocultarlo todo.

La regla profesional es:

> **No protejas un atributo simplemente porque puedes hacerlo. Protégelo cuando exista una razón de diseño.**

---

# 9. Single underscore: `_atributo`

Python utiliza una convención muy importante:

```python
self._veces_prestado
```

El `_` comunica:

> **"Este atributo es interno y no debería utilizarse directamente desde código externo."**

Por ejemplo:

```python
class Libro:

    def __init__(self, veces_prestado):
        self._veces_prestado = veces_prestado
```

Podemos hacer:

```python
print(libro._veces_prestado)
```

y Python lo permite.

Por tanto:

```text
_veces_prestado
       │
       └── convención de API interna
```

No existe una barrera técnica absoluta.

Python confía mucho en la responsabilidad del programador:

```text
"Puedes hacerlo,
pero estás accediendo a un detalle interno."
```

Esto es especialmente importante al trabajar con bibliotecas y frameworks.

---

# 10. Double underscore: `__atributo`

Python también permite:

```python
class Libro:

    def __init__(self, veces_prestado):
        self.__veces_prestado = veces_prestado
```

Aquí ocurre algo diferente.

Python aplica **name mangling**.

Dentro de una clase llamada:

```python
Libro
```

el atributo:

```python
__veces_prestado
```

se transforma conceptualmente en:

```text
_Libro__veces_prestado
```

Por eso:

```python
libro.__veces_prestado
```

produce normalmente:

```text
AttributeError
```

porque ese nombre literal no corresponde al nombre interno generado por Python.

---

# 11. ¿Qué es exactamente name mangling?

El **name mangling** es una transformación de nombres aplicada a identificadores que comienzan con doble underscore.

Por ejemplo:

```python
class Cuenta:

    def __init__(self):
        self.__saldo = 1000
```

Python transforma conceptualmente:

```text
__saldo
```

en:

```text
_Cuenta__saldo
```

Por tanto:

```python
cuenta.__saldo
```

no encuentra ese nombre literal.

Pero esto **no es cifrado**.

Tampoco es un mecanismo de seguridad.

Su objetivo principal es:

* evitar accesos accidentales;
* evitar colisiones de nombres;
* proteger detalles internos frente al uso casual;
* ayudar especialmente en escenarios de herencia.

---

# 12. Name mangling no significa privacidad absoluta

Este punto debes dominarlo.

No digas:

> "`__atributo` es completamente privado."

Eso es técnicamente incorrecto.

Si conocemos el nombre transformado:

```python
_Cuenta__saldo
```

podemos acceder:

```python
cuenta._Cuenta__saldo
```

Por tanto:

```text
__atributo
     ↓
name mangling
     ↓
no privacidad absoluta
```

El objetivo principal es evitar acceso accidental y colisiones, no proporcionar seguridad.

Por eso:

```text
Encapsulación
→ integridad y diseño

Seguridad
→ autenticación, autorización, control de acceso, protección de secretos, etc.
```

`__` no proporciona:

* cifrado;
* autorización;
* autenticación;
* control de acceso de seguridad;
* protección frente a código malicioso.

---

# 13. `_` vs `__` vs público

| Sintaxis     | Significado            | ¿Bloquea acceso directo? | Propósito                             |
| ------------ | ---------------------- | -----------------------: | ------------------------------------- |
| `atributo`   | Público                |                       No | API pública                           |
| `_atributo`  | Interno por convención |                       No | Indicar implementación interna        |
| `__atributo` | Name mangling          |     No de forma absoluta | Evitar acceso accidental y colisiones |

Memoriza:

```text
_foo
 ↓
convención

__foo
 ↓
name mangling
```

No memorices:

```text
_foo = privado
__foo = súper privado
```

Esa simplificación te puede hacer fallar una entrevista técnica.

---

# 14. ¿Cuándo utilizar `__`?

No debes utilizar doble underscore automáticamente.

Tiene especial sentido cuando quieres evitar:

* acceso accidental;
* colisiones de nombres;
* sobrescrituras involuntarias;
* ciertos problemas en jerarquías de herencia.

Por ejemplo:

```python
class Padre:

    def __procesar(self):
        pass
```

El name mangling ayuda a evitar que una subclase utilice accidentalmente exactamente el mismo nombre interno.

Para muchos casos cotidianos, `_atributo` es suficiente como convención.

---

# 15. Getter: controlar la lectura

Si queremos permitir consultar un atributo interno, podemos utilizar un getter:

```python
class Libro:

    def __init__(self, veces_prestado):
        self.__veces_prestado = veces_prestado

    def get_veces_prestado(self):
        return self.__veces_prestado
```

Uso:

```python
libro = Libro(3)

print(libro.get_veces_prestado())
```

El flujo es:

```text
Código externo
      │
      ▼
get_veces_prestado()
      │
      ▼
__veces_prestado
```

El consumidor no necesita conocer cómo se almacena internamente el dato.

---

# 16. ¿Por qué un getter puede reducir acoplamiento?

Inicialmente podemos tener:

```python
def get_veces_prestado(self):
    return self.__veces_prestado
```

Más adelante podríamos cambiar completamente la implementación:

```python
def get_veces_prestado(self):
    return self.historial_prestamos.total()
```

El consumidor continúa utilizando:

```python
libro.get_veces_prestado()
```

No necesita conocer el cambio interno.

La idea es:

```text
Consumidor
    │
    ▼
Interfaz pública
    │
    ▼
Implementación interna
```

en lugar de:

```text
Consumidor
    │
    ▼
Detalle interno
```

Esto reduce el acoplamiento.

---

# 17. Setter: controlar la escritura

Un setter permite controlar modificaciones externas:

```python
class Libro:

    def __init__(self, veces_prestado):
        self.__veces_prestado = veces_prestado

    def get_veces_prestado(self):
        return self.__veces_prestado

    def set_veces_prestado(self, veces_prestado):
        self.__veces_prestado = veces_prestado
```

Ahora:

```python
libro.set_veces_prestado(10)
```

modifica el estado mediante una operación definida por la clase.

Pero aparece una pregunta mucho más importante:

> **¿Realmente queremos permitir que cualquier consumidor establezca arbitrariamente el número de préstamos?**

Muchas veces la respuesta es **no**.

---

# 18. El problema de un setter sin reglas

Este setter:

```python
def set_veces_prestado(self, valor):
    self.__veces_prestado = valor
```

solamente cambia:

```text
asignación directa
```

por:

```text
asignación mediante método
```

Pero no necesariamente aporta encapsulación útil.

Todavía podemos hacer:

```python
libro.set_veces_prestado(-100)
```

o:

```python
libro.set_veces_prestado(999999999)
```

si no existe validación.

Por eso:

> **Encapsular no significa simplemente ocultar la asignación; significa controlar las condiciones bajo las cuales el estado puede cambiar.**

Esta es una de las diferencias más importantes entre **ocultamiento** y **encapsulación real**.

---

# 19. Setter con validación

Podemos establecer una regla:

```text
veces_prestado >= 0
```

Entonces:

```python
class Libro:

    def __init__(self, veces_prestado):
        self.__veces_prestado = 0
        self.set_veces_prestado(veces_prestado)

    def get_veces_prestado(self):
        return self.__veces_prestado

    def set_veces_prestado(self, valor):
        if valor < 0:
            raise ValueError(
                "El número de préstamos no puede ser negativo"
            )

        self.__veces_prestado = valor
```

Ahora:

```python
libro.set_veces_prestado(-10)
```

produce:

```text
ValueError
```

El objeto evita entrar en un estado inválido.

---

# 20. ¿Qué es una invariante?

Una **invariante** es una condición que debe mantenerse válida durante la vida útil de un objeto.

Por ejemplo:

```text
veces_prestado >= 0
```

Otra:

```text
un libro no disponible no puede prestarse nuevamente
```

Otra:

```text
un libro disponible no debería devolverse nuevamente
```

Y en otro dominio:

```text
un saldo debe respetar las reglas financieras de la cuenta
```

La encapsulación permite centralizar las operaciones que podrían romper esas invariantes.

```text
                 Objeto
                   │
          ┌────────┴────────┐
          │                 │
       Estado          Operaciones
          │                 │
          └────────┬────────┘
                   │
                   ▼
              Validaciones
                   │
                   ▼
            Estado consistente
```

Esta es una de las razones más importantes para aprender encapsulación correctamente.

---

# 21. El salto conceptual: proteger invariantes

Ahora podemos formular el problema de manera profesional:

```text
Estado
  ↓
Reglas que deben mantenerse
  ↓
Invariantes
  ↓
Operaciones autorizadas
  ↓
Validación
  ↓
Estado consistente
```

Por eso la pregunta:

> "¿Cómo hago privada esta variable?"

es secundaria.

La pregunta importante es:

> **"¿Cómo evito que el objeto entre en un estado inválido?"**

Esa es la mentalidad de ingeniería.

---

# 22. Python moderno: `@property`

En Python existe una alternativa idiomática al patrón tradicional:

```python
get_x()
set_x()
```

Python proporciona:

```python
@property
```

Por ejemplo:

```python
class Libro:

    def __init__(self, veces_prestado):
        self.__veces_prestado = 0
        self.veces_prestado = veces_prestado

    @property
    def veces_prestado(self):
        return self.__veces_prestado

    @veces_prestado.setter
    def veces_prestado(self, valor):
        if valor < 0:
            raise ValueError(
                "El número de préstamos no puede ser negativo"
            )

        self.__veces_prestado = valor
```

Ahora podemos escribir:

```python
libro.veces_prestado
```

para leer.

Y:

```python
libro.veces_prestado = 10
```

para modificar.

Pero detrás de esta sintaxis existe lógica controlada.

---

# 23. ¿Qué ocurre internamente con `@property`?

Cuando hacemos:

```python
libro.veces_prestado
```

no debemos pensar simplemente:

> "Python devuelve una variable."

Existe un objeto `property` que controla el acceso.

Conceptualmente:

```text
libro.veces_prestado
        │
        ▼
     property
        │
        ▼
      getter
        │
        ▼
__veces_prestado
```

Cuando hacemos:

```python
libro.veces_prestado = 10
```

la operación puede ser redirigida al setter:

```text
libro.veces_prestado = 10
        │
        ▼
 property setter
        │
        ▼
   validación
        │
        ▼
__veces_prestado = 10
```

Esto conecta una sintaxis aparentemente sencilla con un mecanismo interno más profundo de Python.

---

# 24. Propiedad de solo lectura

Una de las aplicaciones más útiles de `@property` es permitir lectura sin proporcionar escritura directa.

```python
class Libro:

    def __init__(self, titulo):
        self.titulo = titulo
        self.__veces_prestado = 0

    @property
    def veces_prestado(self):
        return self.__veces_prestado

    def prestar(self):
        self.__veces_prestado += 1
```

Podemos hacer:

```python
libro.prestar()

print(libro.veces_prestado)
```

Pero no proporcionamos:

```python
@veces_prestado.setter
```

Por tanto:

```python
libro.veces_prestado = 100
```

no forma parte de la interfaz pública permitida.

Esto expresa mejor la regla:

> **El contador cambia como consecuencia de un préstamo, no mediante una asignación arbitraria.**

---

# 25. ¿Cuándo NO crear un setter?

Este es un criterio profesional importante.

Si un valor debe cambiar solamente como consecuencia de una acción del dominio, no necesariamente debemos proporcionar un setter.

Por ejemplo:

```text
veces_prestado
```

puede modificarse mediante:

```python
libro.prestar()
```

pero no necesariamente mediante:

```python
libro.veces_prestado = 50
```

La regla es:

> **No expongas una operación de escritura simplemente porque técnicamente puedes hacerlo.**

Primero pregunta:

> **¿El dominio realmente permite esta modificación?**

Los métodos de dominio pueden ser superiores a los setters cuando representan transiciones válidas del estado.

---

# 26. Getter/setter tradicional vs `@property`

## Patrón tradicional

```python
def get_veces_prestado(self):
    return self.__veces_prestado

def set_veces_prestado(self, valor):
    self.__veces_prestado = valor
```

Ventajas:

* explícito;
* fácil de comprender inicialmente.

Desventajas:

* puede producir una API verbosa;
* puede generar getters y setters innecesarios;
* no garantiza que exista una regla de negocio.

## Python idiomático

```python
@property
def veces_prestado(self):
    return self.__veces_prestado

@veces_prestado.setter
def veces_prestado(self, valor):
    ...
```

Uso:

```python
libro.veces_prestado
```

Ventajas:

* API natural;
* permite incorporar validación;
* permite cambiar la implementación interna;
* permite pasar de un atributo simple a lógica calculada sin cambiar necesariamente la interfaz.

Pero recuerda:

> `@property` es una herramienta, no el objetivo.

El objetivo sigue siendo diseñar una interfaz correcta.

---

# 27. Encapsulación mediante operaciones del dominio

Ahora podemos construir una clase mucho más coherente:

```python
class Libro:

    def __init__(self, titulo):
        self.titulo = titulo
        self.__disponible = True
        self.__veces_prestado = 0

    @property
    def disponible(self):
        return self.__disponible

    @property
    def veces_prestado(self):
        return self.__veces_prestado

    def prestar(self):
        if not self.__disponible:
            return "El libro no está disponible"

        self.__disponible = False
        self.__veces_prestado += 1

        return "Préstamo realizado"

    def devolver(self):
        if self.__disponible:
            return "El libro ya está disponible"

        self.__disponible = True

        return "Libro devuelto"

    def es_popular(self):
        return self.__veces_prestado > 5
```

Aquí existe una arquitectura clara:

```text
                    Libro
                      │
           ┌──────────┴──────────┐
           │                     │
         Estado             Operaciones
           │                     │
           │                 prestar()
           │                 devolver()
           │                 es_popular()
           │
           ├── disponible
           └── veces_prestado
```

El código externo puede consultar:

```python
libro.disponible
```

y:

```python
libro.veces_prestado
```

pero las modificaciones importantes ocurren mediante:

```python
libro.prestar()
libro.devolver()
```

Esto es una aplicación mucho más madura de encapsulación.

---

# 28. ¿Por qué `prestar()` es mejor que modificar atributos?

Compara:

```python
libro.disponible = False
libro.veces_prestado += 1
```

con:

```python
libro.prestar()
```

La primera opción obliga al consumidor a conocer la implementación.

La segunda expresa una operación del dominio.

Además, `prestar()` puede garantizar:

```text
1. Verificar disponibilidad
2. Cambiar disponibilidad
3. Incrementar contador
4. Registrar el préstamo
5. Ejecutar otras reglas necesarias
6. Devolver un resultado
```

El consumidor solamente necesita conocer:

```python
libro.prestar()
```

Esta es una forma importante de encapsulación y abstracción.

---

# 29. Encapsulación vs abstracción

Estos conceptos están relacionados, pero no son iguales.

## Encapsulación

Se enfoca en:

> **Controlar el acceso y modificación del estado y comportamiento interno.**

## Abstracción

Se enfoca en:

> **Exponer lo relevante mientras se ocultan detalles innecesarios de implementación.**

Por ejemplo:

```python
libro.prestar()
```

puede representar ambas ideas.

### Encapsulación

La clase controla cómo se modifica:

```text
disponibilidad
veces_prestado
```

### Abstracción

El consumidor no necesita conocer:

```text
cómo se valida
cómo se actualiza
cómo se registra
cómo se persiste
```

Por tanto:

```text
Encapsulación
      ↓
control de internals

Abstracción
      ↓
interfaz simplificada
```

Son conceptos distintos, aunque frecuentemente trabajan juntos.

---

# 30. Encapsulación vs privacidad

Tampoco son exactamente lo mismo.

```text
Encapsulación
    ↓
principio de diseño

Privacidad
    ↓
restricción sobre acceso

Name mangling
    ↓
mecanismo de Python
```

En Python:

```python
__saldo
```

no proporciona seguridad absoluta.

Por tanto:

```text
Encapsulación ≠ privacidad absoluta
```

y:

```text
Name mangling ≠ seguridad
```

Esta distinción es importante tanto para diseño como para entrevistas técnicas.

---

# 31. ¿Todos los atributos deben ser privados?

No.

Aplicar encapsulación indiscriminadamente puede generar código innecesariamente complejo.

Esto:

```python
class Libro:

    def __init__(self, titulo, autor):
        self.__titulo = titulo
        self.__autor = autor
        self.__disponible = True
        self.__veces_prestado = 0
```

no es automáticamente mejor que:

```python
class Libro:

    def __init__(self, titulo, autor):
        self.titulo = titulo
        self.autor = autor
        self.__disponible = True
        self.__veces_prestado = 0
```

La decisión depende del dominio.

Pregunta:

```text
¿Este estado tiene reglas?
¿Puede entrar en un estado inválido?
¿Quién debería poder modificarlo?
¿Necesita validación?
¿Es un detalle de implementación?
```

Si no existe una razón para ocultarlo, convertirlo en privado puede añadir complejidad innecesaria.

---

# 32. Candidatos habituales para encapsulación

Buenos candidatos:

* contadores;
* saldos;
* estados;
* identificadores con reglas;
* permisos;
* configuraciones;
* límites;
* información derivada;
* datos cuyo cambio requiere validación.

Por ejemplo:

```text
saldo
estado_pago
disponibilidad
veces_prestado
limite_credito
```

suelen requerir reglas.

En cambio:

```text
titulo
autor
descripcion
```

podrían ser públicos dependiendo del diseño.

No existe una regla universal.

---

# 33. Encapsulación y cohesión

Una buena clase debería ser responsable de mantener las reglas relacionadas con su propio estado.

Esto favorece la **cohesión**.

Ejemplo:

```text
Libro
│
├── prestar()
├── devolver()
├── es_popular()
├── disponibilidad
└── veces_prestado
```

Tiene sentido que `Libro` controle esas reglas.

Sería menos cohesivo tener:

```text
ServicioA
    ↓
modifica disponibilidad

ServicioB
    ↓
modifica veces_prestado

ServicioC
    ↓
decide si es popular
```

porque la lógica relacionada con el libro queda distribuida.

La encapsulación ayuda a mantener juntas las reglas que pertenecen al mismo concepto.

---

# 34. Encapsulación y acoplamiento

La encapsulación ayuda a reducir el acoplamiento entre componentes.

Un consumidor debería depender de:

```text
interfaz pública
```

y no de:

```text
detalle interno
```

Por ejemplo:

```text
Aplicación
    │
    ▼
libro.prestar()
    │
    ▼
Libro
    │
    ├── estado
    ├── validaciones
    └── implementación
```

Si mañana cambiamos cómo se almacena el contador, el consumidor no debería necesitar conocer ese cambio.

La idea es:

```text
Consumidor
    │
    ▼
Contrato / interfaz
    │
    ▼
Implementación
```

y no:

```text
Consumidor
    │
    ▼
detalle interno
```

---

# 35. Problema real en producción: estado inconsistente

## Síntoma

Una aplicación empieza a mostrar datos contradictorios.

Por ejemplo:

```text
disponible = True
```

pero existe un préstamo activo.

O:

```text
veces_prestado = -5
```

## Causa

El estado puede modificarse desde múltiples lugares sin una autoridad clara.

Ejemplo:

```python
libro.disponible = False
libro.veces_prestado += 1
```

## Diagnóstico

Busca:

* asignaciones directas;
* múltiples lugares que modifican el mismo atributo;
* setters sin validación;
* atributos públicos que representan estado crítico;
* lógica de negocio distribuida.

## Solución

Centralizar las transiciones:

```text
Código externo
      ↓
Método / property
      ↓
Validación
      ↓
Cambio de estado
```

## Prevención

* definir invariantes;
* limitar mutaciones;
* centralizar reglas;
* utilizar propiedades cuando corresponda;
* utilizar métodos de dominio;
* escribir pruebas sobre las reglas.

---

# 36. Ejemplo profesional con invariantes

Podemos diseñar:

```python
class Libro:

    def __init__(self, titulo):
        self.titulo = titulo
        self.__disponible = True
        self.__veces_prestado = 0

    @property
    def disponible(self):
        return self.__disponible

    @property
    def veces_prestado(self):
        return self.__veces_prestado

    def prestar(self):
        if not self.__disponible:
            raise ValueError("El libro no está disponible")

        self.__disponible = False
        self.__veces_prestado += 1

    def devolver(self):
        if self.__disponible:
            raise ValueError("El libro ya está disponible")

        self.__disponible = True

    def es_popular(self):
        return self.__veces_prestado > 5
```

Ahora la clase protege varias reglas:

```text
Invariante 1:
veces_prestado >= 0

Invariante 2:
un libro no disponible no puede prestarse

Invariante 3:
un libro disponible no debería devolverse nuevamente
```

Las operaciones válidas están concentradas dentro de la clase.

---

# 37. La máquina de estados del `Libro`

El comportamiento puede visualizarse como una pequeña máquina de estados:

```text
        +----------------+
        |   Disponible   |
        | available=True |
        +-------+--------+
                |
             prestar()
                |
                v
        +-------------------+
        |  No disponible    |
        | available=False   |
        +---------+---------+
                  |
               devolver()
                  |
                  v
        +----------------+
        |   Disponible   |
        +----------------+
```

Esto es mucho más poderoso que pensar simplemente en:

```python
available = True
```

Porque estamos pensando en:

```text
estado
  +
transición
  +
regla
```

El mismo modelo aparece en sistemas reales:

```text
Pedido
  ↓
creado
  ↓
pagado
  ↓
preparado
  ↓
enviado
  ↓
entregado
```

Y también en workflows y sistemas de IA.

La clase anterior ya introducía esta idea al representar `prestar()` y `devolver()` como transiciones explícitas.

---

# 38. El ejercicio `es_popular()`

En la clase anterior se introduce el reto de implementar:

```python
es_popular()
```

que debe devolver:

```text
True
```

si el libro ha sido prestado más de cinco veces.

Por tanto:

```python
def es_popular(self):
    return self.__veces_prestado > 5
```

El reto revela algo importante.

`es_popular()` depende del estado:

```text
veces_prestado
```

Por tanto, si cualquier código externo puede hacer:

```python
libro.veces_prestado = 1000
```

entonces también puede manipular artificialmente el resultado de:

```python
libro.es_popular()
```

El problema de encapsulación afecta directamente al comportamiento del objeto.

---

# 39. Contador vs historial de préstamos

Para registrar préstamos existen al menos dos estrategias.

## Estrategia A — contador

```python
self.__veces_prestado = 0
```

Cada préstamo exitoso:

```python
self.__veces_prestado += 1
```

Ventaja:

```text
menos memoria
```

Si solamente necesitamos saber:

```text
¿Cuántas veces?
```

un contador puede ser suficiente.

## Estrategia B — historial

```python
self.historial_prestamos = []
```

Cada préstamo exitoso registra un evento.

Esto permite conservar información como:

```text
usuario
fecha
hora
libro
duración
```

Es más apropiado cuando necesitamos:

* auditoría;
* trazabilidad;
* análisis histórico;
* timestamps;
* usuarios;
* eventos completos.

La decisión debe basarse en requisitos funcionales y de auditoría, no simplemente en "qué estructura es más sofisticada".

---

# 40. Estado actual vs historial de eventos

Esta distinción será muy importante en sistemas reales.

```text
Estado actual
    ↓
available = False
```

dice:

> ¿Cuál es el estado ahora?

Mientras:

```text
Historial
    ↓
préstamo 1
préstamo 2
préstamo 3
...
```

dice:

> ¿Qué acontecimientos ocurrieron?

Y:

```text
veces_prestado = 27
```

representa:

> ¿Cuántos acontecimientos acumulados conocemos?

En sistemas de IA ocurre exactamente la misma decisión:

```text
estado actual
vs.
historial / eventos / trazas
```

---

# 41. `return` vs `print`: conexión con encapsulación

La clase anterior también introduce una regla fundamental.

Esto:

```python
def prestar(self):
    print("Libro prestado")
```

acopla el método a una forma concreta de salida.

En cambio:

```python
def prestar(self):
    return "Libro prestado"
```

permite que el consumidor decida qué hacer:

```python
mensaje = libro.prestar()
```

o:

```python
print(libro.prestar())
```

o:

```python
logger.info(libro.prestar())
```

o utilizar el resultado dentro de una API.

Por eso, cuando el resultado tiene valor para el consumidor, normalmente es mejor:

```python
return resultado
```

que:

```python
print(resultado)
```

La diferencia es una aplicación de separación de responsabilidades: la clase ejecuta comportamiento; otra capa decide cómo presentar el resultado.

---

# 42. ¿Qué ocurre si no existe `return`?

Si un método no tiene un `return` explícito:

```python
def prestar(self):
    self.__disponible = False
```

Python devuelve:

```python
None
```

Por tanto:

```python
resultado = libro.prestar()
```

produce conceptualmente:

```python
resultado is None
```

Esto es importante porque un método puede:

```text
modificar estado
```

sin:

```text
devolver resultado
```

No debemos confundir:

```text
efecto secundario
```

con:

```text
valor retornado
```

---

# 43. `__str__` y encapsulación

La clase anterior también introduce:

```python
def __str__(self):
    return f"{self.titulo} - {self.isbn}"
```

Esto permite:

```python
print(libro)
```

y proporciona una representación textual útil.

`__str__` debe devolver un objeto `str`.

Correcto:

```python
def __str__(self):
    return f"{self.titulo} - {self.isbn}"
```

Incorrecto:

```python
def __str__(self):
    print(self.titulo)
```

`print()` muestra información, pero no devuelve la cadena que `__str__` necesita.

Esto pertenece a la misma filosofía de diseño:

```text
método
  ↓
produce un resultado
  ↓
consumidor decide qué hacer
```

---

# 44. f-strings

Para construir mensajes utilizando atributos:

```python
return f"{self.titulo} prestado exitosamente"
```

La `f` permite interpolar expresiones.

Sin `f`:

```python
return "{self.titulo} prestado exitosamente"
```

el texto se trataría literalmente.

Es un detalle sencillo, pero importante para producir resultados correctos.

---

# 45. Una clase no debe hacer todo

La encapsulación no significa meter absolutamente toda la lógica dentro de una clase.

Una clase `Libro` que además:

* envía correos;
* realiza consultas SQL;
* llama APIs externas;
* genera embeddings;
* registra logs;
* autentica usuarios;
* procesa pagos;

tendría demasiadas responsabilidades.

Una arquitectura más profesional puede separar:

```text
Book
  │
  ▼
BookService
  │
  ├── Repository
  ├── NotificationService
  └── External APIs
```

La clase de dominio debe concentrarse en las reglas que realmente pertenecen a la entidad.

Esto conecta encapsulación con **cohesión** y separación de responsabilidades.

---

# 46. Encapsulación y testing

La encapsulación también afecta cómo diseñamos las pruebas.

Una buena prueba debería centrarse principalmente en el comportamiento observable:

```python
libro.prestar()

assert libro.veces_prestado == 1
assert libro.disponible is False
```

No necesariamente en detalles internos como:

```python
assert libro._Libro__veces_prestado == 1
```

¿Por qué?

Porque los tests deberían depender preferentemente de la interfaz pública.

Si mañana cambiamos:

```text
cómo se almacena el contador
```

pero mantenemos:

```text
veces_prestado
```

los tests de comportamiento pueden continuar funcionando.

La encapsulación, por tanto, también protege a los consumidores de cambios internos.

---

# 47. Error conceptual: "`__` hace que el código sea seguro"

Incorrecto.

```python
self.__password = password
```

no convierte ese dato en un secreto seguro.

`__` activa name mangling.

No proporciona:

* cifrado;
* autorización;
* autenticación;
* control de acceso;
* protección frente a código malicioso.

La seguridad requiere mecanismos específicos.

```text
Encapsulación
→ integridad y diseño

Seguridad
→ amenazas y control de acceso
```

---

# 48. Error conceptual: "todos los atributos deberían ser privados"

Incorrecto.

La encapsulación debe aplicarse donde existen razones de diseño.

Pregúntate:

```text
¿Este estado tiene reglas?

¿Puede entrar en un estado inválido?

¿Quién debería poder modificarlo?

¿Necesita validación?

¿Es un detalle de implementación?
```

Si ninguna respuesta justifica ocultarlo, hacerlo privado puede añadir complejidad innecesaria.

---

# 49. Error conceptual: "un setter siempre mejora la encapsulación"

Incorrecto.

Un setter puede incluso debilitar el modelo de dominio.

Por ejemplo:

```python
libro.veces_prestado = 100
```

puede ser conceptualmente inválido aunque Python permita la operación.

La encapsulación correcta no consiste en:

```text
"permitir todo mediante métodos"
```

sino en:

```text
"permitir solamente las operaciones válidas"
```

---

# 50. Error conceptual: "todos los atributos privados necesitan getter y setter"

Incorrecto.

Puede existir:

```python
@property
def veces_prestado(self):
    return self.__veces_prestado
```

sin setter.

O puede no ser necesario exponer el atributo en absoluto.

La interfaz debe diseñarse según las necesidades del dominio.

---

# 51. Error conceptual: demasiados getters y setters

Un diseño como:

```python
get_nombre()
set_nombre()

get_email()
set_email()

get_edad()
set_edad()

get_estado()
set_estado()

get_saldo()
set_saldo()
```

para absolutamente todo puede ser una señal de que estamos creando una clase que simplemente expone datos.

La pregunta correcta es:

> **¿Dónde deberían vivir las reglas del dominio?**

No:

> "¿Cuántos getters y setters puedo crear?"

Una clase debe ofrecer comportamiento útil, no simplemente una colección de métodos de acceso.

---

# 52. Un ejemplo externo: una cuenta bancaria

La analogía del banco puede convertirse en código:

```python
class Cuenta:

    def __init__(self, saldo):
        self.__saldo = saldo

    @property
    def saldo(self):
        return self.__saldo

    def retirar(self, cantidad):
        if cantidad <= 0:
            raise ValueError("La cantidad debe ser positiva")

        if cantidad > self.__saldo:
            raise ValueError("Saldo insuficiente")

        self.__saldo -= cantidad
```

Podemos consultar:

```python
cuenta.saldo
```

pero las modificaciones pasan por:

```python
cuenta.retirar(100)
```

y son sometidas a validaciones.

La diferencia conceptual es:

```text
Antes:

saldo = estado público
       ↓
cualquier código puede modificarlo


Después:

saldo = estado interno
       ↓
operaciones controladas
       ↓
validaciones
       ↓
estado consistente
```

Este patrón aparece continuamente en sistemas reales.

---

# 53. Encapsulación y APIs

La misma idea aparece en una API.

Una API define:

```text
¿Qué puedo hacer?
```

y oculta:

```text
¿Cómo lo implementa internamente?
```

Por ejemplo:

```text
POST /loans
```

puede representar:

```text
crear préstamo
```

sin exponer al cliente:

```text
cómo se actualiza disponibilidad
cómo se incrementa el contador
cómo se registra auditoría
cómo se persiste información
```

Esto es la misma separación entre:

```text
interfaz pública
```

y:

```text
implementación interna
```

que estamos aprendiendo en Python.

---

# 54. Encapsulación y arquitectura backend

Una arquitectura habitual puede ser:

```text
HTTP
  │
  ▼
FastAPI
  │
  ▼
Application Service
  │
  ▼
Domain Object
  │
  ▼
Repository
  │
  ▼
Database
```

Por ejemplo:

```text
POST /books/{id}/lend
            │
            ▼
       BookService
            │
            ▼
       book.prestar()
            │
            ▼
       Repository.save(book)
            │
            ▼
         Database
```

Aquí:

```text
FastAPI
```

se ocupa del transporte HTTP.

```text
Application Service
```

coordina el caso de uso.

```text
Book
```

protege las reglas del dominio.

```text
Repository
```

se ocupa de persistencia.

Esta separación evita mezclar lógica de negocio con infraestructura.

---

# 55. Encapsulación y AI Engineering

Ahora conecta esta clase con tu ruta profesional.

Un sistema de IA moderno puede tener:

```text
                    AI SYSTEM
                        │
          ┌─────────────┼─────────────┐
          │             │             │
         LLM         Retriever       Tools
          │             │             │
          │             │             │
       estado       configuración   ejecución
          │             │             │
          └─────────────┼─────────────┘
                        │
                      Agent
```

Cada componente puede mantener:

* configuración;
* estado;
* dependencias;
* validaciones;
* conexiones;
* comportamiento.

La encapsulación ayuda a establecer límites:

```text
qué puede hacer
qué puede consultar
qué puede modificar
qué debe permanecer interno
```

---

# 56. Encapsulación en SDKs

Cuando utilizas un SDK, normalmente interactúas con una interfaz pública.

Conceptualmente:

```text
Tu aplicación
      ↓
SDK
      ↓
Cliente
      ↓
validación
      ↓
serialización
      ↓
HTTP
      ↓
API
```

No necesitas manipular directamente todos los detalles internos.

Por eso comprender POO y encapsulación será importante cuando trabajes con SDKs de IA.

Podrás encontrarte con objetos conceptualmente similares a:

```python
client
agent
retriever
vector_store
tool
mcp_client
```

Cada uno puede encapsular:

```text
configuración
estado
dependencias
validaciones
conexiones
comportamiento
```

---

# 57. Encapsulación y agentes

Un agente puede verse conceptualmente como:

```text
Agent
 │
 ├── recibe contexto
 │
 ├── decide
 │
 ├── selecciona herramienta
 │
 ├── ejecuta
 │
 └── produce resultado
```

No sería una buena arquitectura que cualquier componente externo pudiera modificar arbitrariamente:

```text
estado interno del agente
herramientas internas
configuración interna
memoria interna
```

sin respetar las reglas del sistema.

La encapsulación contribuye a establecer fronteras entre componentes.

---

# 58. Encapsulación y RAG

Un componente de RAG puede tener conceptualmente:

```text
Retriever
│
├── configuración
├── embedding model
├── vector store
├── filtros
└── comportamiento
    └── search()
```

El consumidor podría utilizar:

```python
retriever.search(query)
```

sin necesitar conocer:

```text
cómo se construye el embedding
cómo se consulta el vector store
cómo se aplican filtros
cómo se normalizan resultados
```

Esto es exactamente la separación:

```text
interfaz pública
      ↓
implementación interna
```

que estamos aprendiendo.

---

# 59. Encapsulación y tools

Una tool puede conceptualizarse como:

```text
Tool
  │
  ▼
input
  │
  ▼
validación
  │
  ▼
ejecución
  │
  ▼
resultado
```

El consumidor no debería necesitar manipular internamente todos los detalles de ejecución.

Una interfaz puede expresar:

```python
tool.run(input)
```

mientras internamente existen:

```text
validación
autorización
serialización
ejecución
manejo de errores
observabilidad
```

La encapsulación proporciona una forma de pensar en esos límites.

---

# 60. Encapsulación y MCP

Cuando llegues a MCP encontrarás una arquitectura basada en:

```text
Host
  ↓
MCP Client
  ↓
MCP Server
  ↓
Tools / Resources / Prompts
```

MCP es un protocolo y no simplemente una característica de POO, pero la idea de **interfaces y límites de responsabilidad** continúa siendo fundamental.

Un consumidor interactúa con una interfaz definida:

```text
Tool
  ↓
input
  ↓
validación
  ↓
ejecución
  ↓
resultado
```

en lugar de manipular directamente todos los detalles internos del servidor.

Por eso esta clase no es un conocimiento aislado: prepara el modelo mental necesario para comprender abstracciones posteriores de AI Engineering.

---

# 61. La conexión completa con tu formación

La progresión conceptual es:

```text
Python
  │
  ▼
POO
  │
  ├── clases
  ├── objetos
  ├── self
  ├── métodos
  ├── encapsulación
  ├── abstracción
  ├── herencia
  └── polimorfismo
          │
          ▼
      SDKs de IA
          │
          ▼
         APIs
          │
          ▼
         LLMs
          │
          ▼
         RAG
          │
          ▼
       Agents
          │
          ▼
        Tools
          │
          ▼
         MCP
          │
          ▼
      Producción
```

El objetivo de esta clase no es convertirte en experto únicamente en `__atributo`.

El objetivo es que puedas reconocer **cómo las abstracciones complejas se construyen sobre límites claros entre estado, comportamiento e interfaz**.

---

# 62. Principio fundamental para AI Engineering

Una forma profesional de pensar es:

> **Cada componente debe ser responsable de proteger las reglas de su propio estado y exponer únicamente la interfaz necesaria para que otros componentes interactúen con él.**

La progresión es:

```text
Encapsulación
      ↓
Cohesión
      ↓
Menor acoplamiento
      ↓
Componentes reutilizables
      ↓
Sistemas mantenibles
```

Esto es mucho más importante que memorizar únicamente la diferencia entre `_` y `__`.

---

# 63. Problemas reales en producción

## Problema 1 — Estado inconsistente

### Síntoma

Datos contradictorios.

### Causa

Mutaciones directas desde múltiples lugares.

### Solución

Centralizar transiciones.

```text
Código externo
      ↓
Método
      ↓
Validación
      ↓
Estado
```

---

## Problema 2 — Setter sin validación

### Síntoma

Valores inválidos.

```python
libro.set_veces_prestado(-100)
```

### Causa

El setter únicamente reasigna.

### Solución

Definir invariantes y validarlas.

---

## Problema 3 — Setter innecesario

### Síntoma

El consumidor puede realizar operaciones que el dominio realmente no debería permitir.

```python
libro.veces_prestado = 100
```

### Solución

Eliminar la escritura pública y modificar el estado únicamente mediante operaciones válidas:

```python
libro.prestar()
```

---

## Problema 4 — Demasiados getters/setters

### Síntoma

La clase parece una estructura de datos con decenas de métodos de acceso.

### Causa

Crear getters y setters automáticamente.

### Solución

Diseñar primero las operaciones del dominio.

---

## Problema 5 — Clase con demasiadas responsabilidades

### Síntoma

Una clase conoce bases de datos, HTTP, emails, IA, pagos y reglas de negocio.

### Solución

Separar:

```text
Domain
Application Service
Repository
Infrastructure
External Services
```

---

# 64. Buenas prácticas profesionales

* Utilizar nombres que expresen intención.
* Preferir `prestar()` frente a `cambiar_disponibilidad()`.
* Mantener las reglas relacionadas con el estado cerca del objeto cuando corresponda.
* Preferir `return` frente a `print` para resultados reutilizables.
* Mantener `__str__` simple y orientado a representación humana.
* Evitar clases con responsabilidades no relacionadas.
* Validar las transiciones de estado.
* Escribir pruebas para comportamientos importantes.
* Diferenciar estado actual de historial de eventos.
* Diseñar pensando en cómo la clase será utilizada desde servicios y APIs.
* No utilizar `__` como sustituto de seguridad.
* No crear setters automáticamente.
* Usar `@property` cuando proporcione una interfaz natural.
* Proteger especialmente los atributos que representan invariantes.
* Preferir operaciones de dominio cuando una mutación representa una acción real.
* Diseñar la interfaz pública antes de decidir cómo almacenar internamente el estado.

---

# 65. Preguntas de entrevista técnica

## Pregunta 1 — ¿Qué es encapsulación en Python?

### Error común

> "Es poner `__` delante de una variable."

### Respuesta de alto impacto

> La encapsulación consiste en controlar cómo el código externo interactúa con el estado y comportamiento interno de un objeto. En Python se implementa mediante convenciones, name mangling, métodos y propiedades. Su objetivo es proteger invariantes, controlar modificaciones, reducir acoplamiento y ocultar detalles de implementación cuando corresponde.

---

## Pregunta 2 — ¿Cuál es la diferencia entre `_atributo` y `__atributo`?

### Respuesta de alto impacto

> Un atributo con un solo underscore es una convención que indica que forma parte de la implementación interna, pero Python no bloquea su acceso. Un atributo con doble underscore activa name mangling, mediante el cual Python transforma el nombre incorporando el nombre de la clase. Esto ayuda a evitar accesos accidentales y colisiones, especialmente en herencia, pero no constituye privacidad absoluta.

---

## Pregunta 3 — ¿Qué es name mangling?

### Respuesta de alto impacto

> Name mangling es la transformación que Python aplica a determinados identificadores que comienzan con doble underscore. Por ejemplo, `__saldo` dentro de una clase `Cuenta` se transforma conceptualmente en `_Cuenta__saldo`. Su objetivo principal es evitar colisiones y accesos accidentales, no proporcionar seguridad.

---

## Pregunta 4 — ¿Por qué no hacer públicos todos los atributos?

### Respuesta de alto impacto

> Porque algunos atributos representan estado sujeto a invariantes o reglas de negocio. Si cualquier componente puede modificarlos directamente, puede introducir estados inválidos. Encapsular permite centralizar las reglas de modificación y preservar la integridad del objeto.

---

## Pregunta 5 — ¿Por qué un setter puede ser una mala decisión?

### Respuesta de alto impacto

> Porque un setter expone una operación de escritura. Si el dominio establece que un valor solamente debe cambiar como consecuencia de una operación específica, como `prestar()`, proporcionar un setter permite saltarse esa regla. No debemos crear setters por defecto; debemos exponer únicamente las mutaciones que el dominio realmente permite.

---

## Pregunta 6 — ¿Qué diferencia existe entre un getter tradicional y `@property`?

### Respuesta de alto impacto

> Un getter tradicional utiliza un método explícito como `get_valor()`, mientras que `@property` permite exponer una interfaz con apariencia de atributo y ejecutar lógica internamente. `@property` suele ser más idiomático en Python cuando necesitamos encapsular lectura o escritura sin obligar al consumidor a utilizar una API basada en getters y setters.

---

## Pregunta 7 — ¿Por qué `__` no proporciona seguridad?

### Respuesta de alto impacto

> Porque `__` activa name mangling, no cifrado ni control de acceso. El nombre se transforma, por ejemplo, de `__saldo` a `_Cuenta__saldo`, pero sigue siendo accesible si alguien conoce el nombre transformado. Por tanto, name mangling ayuda principalmente a evitar colisiones y accesos accidentales.

---

## Pregunta 8 — ¿Cuándo no crearías un setter?

### Respuesta de alto impacto

> Cuando el estado solamente debería cambiar como consecuencia de una operación específica del dominio. Por ejemplo, el contador de préstamos de un libro debería incrementarse mediante `prestar()` y no necesariamente mediante `set_veces_prestado()`. El objetivo es exponer únicamente operaciones válidas.

---

## Pregunta 9 — ¿Qué es una invariante?

### Respuesta de alto impacto

> Es una condición que debe mantenerse válida durante la vida útil de un objeto. Por ejemplo, `veces_prestado >= 0` o que un libro no disponible no pueda prestarse nuevamente. La encapsulación permite centralizar las operaciones que podrían romper esas condiciones.

---

## Pregunta 10 — ¿Por qué prefieres `return` sobre `print`?

### Respuesta de alto impacto

> `print` acopla el método a una salida concreta. `return` entrega un valor al consumidor, que puede imprimirlo, registrarlo, convertirlo en una respuesta HTTP o procesarlo de otra manera. Esto favorece la reutilización y la separación de responsabilidades.

---

## Pregunta 11 — ¿Cuándo usarías un contador frente a un historial?

### Respuesta de alto impacto

> Si solamente necesito saber la cantidad de préstamos, un contador puede ser suficiente y más eficiente. Si necesito auditoría, usuarios, fechas, trazabilidad o análisis histórico, conservar eventos es más apropiado. La decisión depende de los requisitos.

---

## Pregunta 12 — ¿Cómo conectarías un objeto de dominio con una API?

### Respuesta de alto impacto

> Mantendría separadas las responsabilidades. FastAPI manejaría el transporte HTTP, un servicio de aplicación coordinaría el caso de uso y el objeto de dominio encapsularía las reglas correspondientes. Por ejemplo, el endpoint recibe la solicitud, el servicio obtiene el `Book`, ejecuta `book.prestar()` y posteriormente persiste el cambio mediante un repositorio.

---

# 66. Checklist de dominio

Al terminar esta clase debes poder explicar sin consultar documentación:

* [ ] Qué es encapsulación.
* [ ] Por qué encapsulación no significa simplemente "hacer variables privadas".
* [ ] Qué problema de integridad resuelve.
* [ ] Qué es un atributo público.
* [ ] Qué significa `_atributo`.
* [ ] Qué significa `__atributo`.
* [ ] Qué es name mangling.
* [ ] Por qué name mangling no es seguridad.
* [ ] Diferencia entre `_` y `__`.
* [ ] Qué es un getter.
* [ ] Qué es un setter.
* [ ] Por qué un setter sin validación puede aportar muy poco.
* [ ] Cuándo NO debería existir un setter.
* [ ] Qué es una invariante.
* [ ] Cómo proteger una invariante.
* [ ] Qué es `@property`.
* [ ] Cómo funciona conceptualmente un getter mediante `@property`.
* [ ] Cómo funciona conceptualmente un setter mediante `@property`.
* [ ] Cómo crear una propiedad de solo lectura.
* [ ] Por qué `prestar()` puede ser mejor que modificar atributos directamente.
* [ ] Diferencia entre encapsulación y abstracción.
* [ ] Diferencia entre encapsulación y privacidad.
* [ ] Cómo la encapsulación reduce acoplamiento.
* [ ] Cómo favorece la cohesión.
* [ ] Cómo afecta al testing.
* [ ] Cuándo utilizar contador frente a historial.
* [ ] Cómo modelar transiciones de estado.
* [ ] Cómo conectar un objeto de dominio con una API.
* [ ] Cómo este conocimiento aparece posteriormente en SDKs, RAG, agentes, tools y MCP.

---

# 67. Lo que NO debes memorizar

No memorices:

```text
_ = privado
__ = súper privado
```

Tampoco:

```text
Todo atributo debe tener getter y setter
```

Ni:

```text
__ = seguridad
```

Ni:

```text
Encapsulación = ocultar variables
```

Estas simplificaciones pueden hacer que apruebes un curso, pero no te convierten en un ingeniero sólido.

La comprensión correcta es:

```text
_foo
 ↓
convención de uso interno

__foo
 ↓
name mangling

@property
 ↓
interfaz controlada mediante apariencia de atributo

método de dominio
 ↓
operación válida sobre el estado

encapsulación
 ↓
protección del diseño y de las invariantes
```

---

# 68. Lo que SÍ debes poder explicar en una entrevista

Si un entrevistador pregunta:

> "¿Por qué encapsularías un atributo?"

No respondas:

> "Porque así es POO."

Una respuesta de alto nivel sería:

> "Lo encapsularía si representa estado cuya modificación debe respetar invariantes o reglas de negocio. De esa forma puedo controlar las transiciones mediante una interfaz pública, evitar modificaciones arbitrarias y mantener los detalles internos desacoplados de los consumidores. En Python utilizaría la herramienta apropiada según el caso: convención `_`, name mangling, métodos o `@property`. No asumiría que todos los atributos necesitan el mismo nivel de encapsulación."

Esta respuesta demuestra criterio de ingeniería.

---

# 69. Modelo mental definitivo

Cuando diseñes una clase, sigue esta secuencia:

```text
        ¿QUÉ ESTADO TIENE MI OBJETO?
                    │
                    ▼
       ¿QUÉ REGLAS DEBE PRESERVAR?
                    │
                    ▼
        ¿QUÉ INVARIANTES EXISTEN?
                    │
                    ▼
         ¿QUIÉN PUEDE MODIFICARLO?
                    │
          ┌─────────┴─────────┐
          │                   │
       consultar          modificar
          │                   │
          ▼                   ▼
      property          método / setter
                              │
                              ▼
                         validación
                              │
                              ▼
                       cambio de estado
                              │
                              ▼
                       estado consistente
```

Este es el modelo mental que debes llevarte de la clase.

---

# 70. Modelo mental definitivo aplicado a `Libro`

```text
                         LIBRO
                           │
              ┌────────────┴────────────┐
              │                         │
            ESTADO                 COMPORTAMIENTO
              │                         │
       ┌──────┴──────┐           ┌─────┴─────────┐
       │             │           │               │
  disponible   veces_prestado  prestar()     devolver()
       │             │           │               │
       └──────┬──────┘           └───────┬───────┘
              │                          │
              │                    valida reglas
              │                          │
              └──────────────┬───────────┘
                             ▼
                      ESTADO CONSISTENTE
```

El código externo debería pensar principalmente:

```python
libro.prestar()
libro.devolver()
libro.es_popular()
```

y no:

```python
libro.disponible = False
libro.veces_prestado += 1
```

---

# 71. La progresión completa desde la clase anterior

La evolución conceptual de estas clases es:

```text
Clase 02
    ↓
Atributos
    ↓
Clase 03
    ↓
Métodos de instancia
    ↓
Estado + comportamiento
    ↓
Clase 04
    ↓
Encapsulación
    ↓
Estado protegido
    ↓
Invariantes
    ↓
Transiciones controladas
    ↓
Interfaz pública
    ↓
Menor acoplamiento
    ↓
Modelos de dominio
    ↓
Servicios
    ↓
APIs
    ↓
Sistemas de IA
```

La clase anterior enseñó:

> **"El objeto puede actuar sobre su propio estado."**

Esta clase añade:

> **"El objeto debe controlar cómo puede cambiar ese estado."**

Ese es el salto conceptual.

---

# 72. Resultado profesional esperado

Al terminar esta clase, el objetivo no es simplemente poder escribir:

```python
class Libro:

    def __init__(self):
        self.__veces_prestado = 0
```

El objetivo es comprender que un objeto puede representar:

```text
Estado
   +
Comportamiento
   +
Reglas
   +
Invariantes
   +
Transiciones
   +
Interfaz pública
```

y que la encapsulación permite proteger esa relación.

La progresión profesional es:

```text
Datos
  ↓
Objetos
  ↓
Métodos
  ↓
Reglas de negocio
  ↓
Encapsulación
  ↓
Modelos de dominio
  ↓
Servicios
  ↓
APIs
  ↓
Sistemas distribuidos
  ↓
AI Applications
```

El salto importante es pasar de:

> **objetos que almacenan datos**

a:

> **objetos que controlan su estado, encapsulan comportamiento y preservan las reglas de su dominio.**

---

# 73. Principio de Pareto de esta clase

El 20 % que debes recordar para obtener el 80 % del impacto profesional es:

```text
1. Encapsulación protege el estado y controla su interacción.

2. _atributo es una convención de atributo interno.

3. __atributo activa name mangling.

4. Name mangling NO es seguridad absoluta.

5. No todos los atributos necesitan encapsulación.

6. No todos los atributos encapsulados necesitan setter.

7. @property es una herramienta idiomática de Python para
   controlar acceso mediante una interfaz similar a un atributo.

8. Las invariantes son reglas que el objeto debe preservar.

9. Los métodos de dominio pueden ser mejores que setters
   cuando representan transiciones válidas del estado.

10. La encapsulación reduce acoplamiento y protege detalles
    de implementación.

11. El objetivo no es ocultar datos por dogma.

12. El objetivo es diseñar componentes que mantengan
    correctamente su propio estado.
```

---

# 74. Regla de oro para todo tu futuro código Python

Antes de convertir un atributo en privado, antes de crear un getter, antes de crear un setter y antes de utilizar `@property`, pregunta:

```text
¿Qué estado representa?
        ↓
¿Qué puede salir mal?
        ↓
¿Qué invariantes existen?
        ↓
¿Quién debería poder modificarlo?
        ↓
¿Qué operación representa realmente ese cambio?
        ↓
¿Necesito un atributo público?
¿Necesito _?
¿Necesito __?
¿Necesito @property?
¿Necesito un método de dominio?
¿Necesito impedir completamente la escritura?
```

La herramienta viene **después** del diseño.

No diseñes alrededor de `__`.

Diseña alrededor de las **reglas del objeto**.

---

# 75. Conclusión

La encapsulación no consiste simplemente en esconder atributos.

Consiste en:

> **diseñar objetos que controlen correctamente su propio estado.**

Un buen diseño identifica:

```text
Estado
  ↓
Invariantes
  ↓
Operaciones válidas
  ↓
Interfaz pública
  ↓
Implementación interna
```

En Python:

```text
atributo público
    ↓
API pública directa

_atributo
    ↓
API interna por convención

__atributo
    ↓
name mangling

@property
    ↓
acceso controlado mediante interfaz de atributo

método de dominio
    ↓
transición válida del estado
```

La idea más importante de toda la clase es esta:

> **No encapsules porque "debes hacerlo". Encapsula cuando necesites proteger invariantes, controlar mutaciones, ocultar decisiones de implementación o reducir el acoplamiento entre componentes.**

Y la pregunta que debes empezar a hacerte como ingeniero es:

> **"¿Qué operaciones puede permitir mi objeto sin dejar de garantizar que su estado siga siendo válido?"**

Esa pregunta te servirá mucho más adelante, cuando los objetos ya no sean solamente `Libro` o `Cuenta`, sino clientes de APIs, retrievers, agentes, tools, componentes RAG, SDKs y sistemas de IA completos.
