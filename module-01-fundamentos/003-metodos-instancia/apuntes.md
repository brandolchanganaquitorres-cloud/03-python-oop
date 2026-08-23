# Clase 03 — Métodos de Instancia en Python: De Datos a Objetos con Comportamiento

## 1. Objetivo de la clase

Comprender cómo los **métodos de instancia** permiten que los objetos Python no sean únicamente contenedores de datos, sino entidades capaces de:

- consultar su propio estado;
- modificarlo;
- validar reglas;
- ejecutar comportamientos;
- devolver resultados al código que los utiliza.

El objetivo profesional es pasar de:

    objeto = datos

a:

    objeto = estado + comportamiento + reglas

Este concepto es fundamental para construir modelos de dominio que posteriormente puedan integrarse en APIs, servicios backend, sistemas empresariales y aplicaciones de AI Engineering.

---

# 2. Idea central

Una clase con atributos puede almacenar información:

    class Book:

        def __init__(self, title, isbn, available):
            self.title = title
            self.isbn = isbn
            self.available = available

Pero el objeto todavía funciona principalmente como un contenedor de datos.

Los métodos de instancia agregan comportamiento:

    class Book:

        def prestar(self):
            ...

        def devolver(self):
            ...

Ahora el objeto puede actuar sobre su propio estado.

Conceptualmente:

    Book
      |
      +-- Estado
      |     +-- title
      |     +-- isbn
      |     +-- available
      |
      +-- Comportamiento
            +-- prestar()
            +-- devolver()
            +-- __str__()

---

# 3. ¿Qué es un método de instancia?

Un método de instancia es una función definida dentro de una clase que opera sobre una instancia específica.

Ejemplo:

    class Book:

        def prestar(self):
            self.available = False

Cuando hacemos:

    my_book.prestar()

el método opera sobre `my_book`.

El parámetro `self` permite acceder al estado de esa instancia.

---

# 4. ¿Qué ocurre con `self`?

Cuando ejecutamos:

    my_book.prestar()

Python vincula el método con la instancia.

Conceptualmente:

    my_book.prestar()

se comporta como:

    Book.prestar(my_book)

Por eso:

    self

representa la instancia sobre la que se está ejecutando el método.

Entonces:

    self.available

significa:

> el atributo `available` perteneciente a esta instancia.

Esto permite que diferentes libros mantengan estados independientes.

---

# 5. Modificar el estado mediante un método

Supongamos que tenemos:

    class Book:

        def __init__(self, title, available):
            self.title = title
            self.available = available

Podemos implementar:

    def cambiar_disponibilidad(self):
        if self.available:
            self.available = False

La operación consulta primero el estado actual:

    self.available

y posteriormente modifica ese estado:

    self.available = False

---

# 6. El problema del método `cambiar_disponibilidad`

Aunque funciona técnicamente, existe un problema de diseño.

El nombre:

    cambiar_disponibilidad()

no expresa claramente la intención del dominio.

En una biblioteca no queremos simplemente "cambiar un booleano".

Queremos representar una acción:

    prestar()

o:

    devolver()

El nombre del método debe comunicar la intención de negocio.

---

# 7. Refactorización hacia comportamiento de dominio

Una implementación más expresiva es:

    class Book:

        def prestar(self):
            if self.available:
                self.available = False
                return f"{self.title} prestado exitosamente"

El método ahora representa una operación del dominio.

El objeto sabe:

- cuál es su título;
- cuál es su estado;
- si está disponible;
- cómo realizar un préstamo;
- qué resultado comunicar.

Esto es más cercano a un modelo de dominio real.

---

# 8. Método `devolver`

Podemos implementar la operación inversa:

    def devolver(self):
        self.available = True
        return f"{self.title} ha sido devuelto y disponible nuevamente"

La clase ahora conoce dos operaciones relacionadas:

    prestar()
    devolver()

El estado puede evolucionar:

    available = True
          |
          | prestar()
          v
    available = False
          |
          | devolver()
          v
    available = True

---

# 9. Estado y comportamiento

Este es uno de los conceptos más importantes de OOP.

El objeto mantiene:

    estado

y proporciona:

    comportamiento

Por ejemplo:

    Book
      |
      +-- Estado
      |    +-- title
      |    +-- isbn
      |    +-- available
      |
      +-- Comportamiento
           +-- prestar()
           +-- devolver()

Esto permite encapsular reglas relacionadas con una entidad.

---

# 10. `__str__`: representación legible del objeto

Si ejecutamos:

    print(my_book)

sin definir una representación personalizada, Python no proporciona normalmente una descripción de negocio útil del objeto.

Podemos implementar:

    def __str__(self):
        return f"{self.title} - {self.isbn}"

Entonces:

    print(my_book)

utilizará automáticamente `__str__`.

La ventaja es que la representación textual queda centralizada dentro de la propia clase.

---

# 11. Regla importante de `__str__`

`__str__` debe devolver un objeto de tipo `str`.

Correcto:

    def __str__(self):
        return f"{self.title} - {self.isbn}"

Incorrecto:

    def __str__(self):
        print(self.title)

`print()` muestra información, pero no devuelve la cadena requerida por `__str__`.

El método debe utilizar:

    return

para proporcionar la representación textual.

---

# 12. `return` vs. `print`

Esta diferencia es fundamental en software profesional.

## `print`

`print()` envía información a una salida.

Ejemplo:

    def prestar(self):
        print("Libro prestado")

El método está acoplado a una forma concreta de mostrar información.

## `return`

`return` entrega un valor al código que llamó al método.

Ejemplo:

    def prestar(self):
        return "Libro prestado"

El consumidor puede decidir qué hacer con ese resultado:

    mensaje = my_book.prestar()

o:

    print(my_book.prestar())

o:

    logger.info(my_book.prestar())

o incluso utilizarlo dentro de otra operación.

---

# 13. ¿Qué ocurre si no existe `return`?

Si una función o método no tiene un `return` explícito, Python devuelve:

    None

Por ejemplo:

    def prestar(self):
        self.available = False

Entonces:

    resultado = my_book.prestar()

produce conceptualmente:

    resultado == None

Esto es importante porque un método puede modificar correctamente el estado y, aun así, no entregar ningún resultado al llamador.

---

# 14. Interpolación con f-strings

Cuando queremos incorporar atributos dentro de una cadena podemos utilizar f-strings:

    return f"{self.title} prestado exitosamente"

La `f` antes de la cadena permite interpolar expresiones.

Sin ella:

    return "{self.title} prestado exitosamente"

el texto sería tratado literalmente y no se sustituiría `self.title`.

Este error es pequeño sintácticamente, pero puede producir mensajes incorrectos en una aplicación.

---

# 15. Implementación integrada

Una versión coherente de la clase sería:

    class Book:

        def __init__(self, title, isbn, available=True):
            self.title = title
            self.isbn = isbn
            self.available = available

        def prestar(self):
            if self.available:
                self.available = False
                return f"{self.title} prestado exitosamente"

            return f"{self.title} no está disponible"

        def devolver(self):
            self.available = True
            return f"{self.title} ha sido devuelto y disponible nuevamente"

        def __str__(self):
            return f"{self.title} - {self.isbn}"

---

# 16. Flujo de ejecución

Cuando ejecutamos:

    my_book.prestar()

el flujo conceptual es:

    my_book
       |
       v
    prestar()
       |
       v
    consultar self.available
       |
       +---- False ---> no disponible
       |
       +---- True ----> cambiar estado
                          |
                          v
                    available = False
                          |
                          v
                       return
                          |
                          v
                    mensaje al caller

El método no solo ejecuta instrucciones.

Está implementando una transición de estado.

---

# 17. Máquina de estados simplificada

El comportamiento de `Book` puede visualizarse como una pequeña máquina de estados:

    +----------------+
    |   Disponible   |
    | available=True |
    +-------+--------+
            |
         prestar()
            |
            v
    +----------------+
    | No disponible  |
    | available=False|
    +-------+--------+
            |
         devolver()
            |
            v
    +----------------+
    |   Disponible   |
    +----------------+

Esta forma de pensar es especialmente útil en sistemas reales.

Muchas entidades empresariales tienen estados y transiciones:

    Pedido
       |
       +--> creado
       |
       +--> pagado
       |
       +--> preparado
       |
       +--> enviado
       |
       +--> entregado

El mismo principio puede modelarse mediante objetos y métodos.

---

# 18. Expansión técnica — encapsulación

El ejemplo introduce una idea importante: el objeto puede ser responsable de sus propias reglas.

En lugar de permitir que cualquier parte del sistema haga:

    book.available = False

podemos centralizar la operación:

    book.prestar()

Esto tiene una ventaja arquitectónica.

El código consumidor expresa intención:

    book.prestar()

en lugar de manipular directamente una representación interna:

    book.available = False

Esto reduce el conocimiento que otras partes del sistema necesitan sobre cómo funciona internamente el objeto.

---

# 19. Pero cuidado con la encapsulación en Python

Python no aplica encapsulación privada de la misma forma que algunos lenguajes como Java o C#.

No debemos afirmar que:

    self.available

queda completamente inaccesible desde fuera.

En Python existe una filosofía más flexible.

Podemos utilizar convenciones como:

    self._available

para comunicar que un atributo es de uso interno.

También existen mecanismos como:

    @property

cuando necesitamos controlar cómo se accede o modifica un atributo.

La encapsulación en Python depende tanto del diseño de la API de la clase como de las convenciones del lenguaje.

---

# 20. Challenge — `es_popular`

El ejercicio de esta clase consiste en agregar:

    es_popular()

El método debe devolver:

    True

si el libro ha sido prestado más de cinco veces.

Para implementar esto necesitamos registrar cada préstamo.

Podemos agregar un atributo:

    self.historial_prestamos = []

en `__init__`.

Posteriormente, cada préstamo exitoso puede registrar un evento.

Por ejemplo:

    self.historial_prestamos.append(...)

Y:

    def es_popular(self):
        return len(self.historial_prestamos) > 5

---

# 21. Diseño del contador de préstamos

Una solución posible es almacenar cada evento:

    self.historial_prestamos = []

Cada vez que `prestar()` tenga éxito:

    self.historial_prestamos.append(...)

Posteriormente:

    len(self.historial_prestamos)

permite conocer cuántos préstamos se han registrado.

Sin embargo, desde una perspectiva de producción, debemos preguntarnos si necesitamos almacenar **todos los eventos** o solamente el contador.

Si únicamente necesitamos saber cuántas veces se prestó:

    self.total_prestamos = 0

puede ser suficiente.

Después:

    self.total_prestamos += 1

Esto utiliza menos memoria.

Si necesitamos auditoría, fechas, usuarios y trazabilidad, entonces almacenar eventos puede ser más apropiado.

---

# 22. Evolución hacia un diseño profesional

El ejercicio plantea una lista de préstamos.

En una aplicación real podría evolucionar hacia:

    Book
      |
      +-- estado actual
      |
      +-- comportamiento
      |
      +-- historial de eventos
               |
               v
          Loan records
               |
               v
          Database

Esto introduce una distinción importante:

    estado actual

vs.

    historial de eventos

Por ejemplo:

    available = False

indica el estado actual.

Pero:

    préstamos realizados = 27

representa información acumulada.

Y:

    historial de préstamos

representa eventos históricos.

Estas decisiones afectan la persistencia, auditoría y arquitectura del sistema.

---

# 23. Conexión con AI Engineering

Los métodos de instancia son relevantes en AI Engineering porque muchas entidades de sistemas de IA mantienen estado y comportamiento.

Por ejemplo:

    Agent
      |
      +-- estado
      |    +-- messages
      |    +-- tools
      |    +-- configuration
      |
      +-- comportamiento
           +-- invoke()
           +-- plan()
           +-- execute()
           +-- reset()

Otro ejemplo:

    Document
      |
      +-- metadata
      +-- content
      +-- embeddings
      |
      +-- comportamiento
           +-- chunk()
           +-- validate()
           +-- serialize()

Otro:

    Retriever
      |
      +-- configuración
      |
      +-- comportamiento
           +-- search()

La misma idea estudiada con `Book` aparece posteriormente en componentes de aplicaciones de IA.

---

# 24. Conexión con APIs

En una aplicación FastAPI podríamos tener:

    HTTP Request
         |
         v
      FastAPI
         |
         v
      Service
         |
         v
    Domain Object
         |
         v
      book.prestar()
         |
         v
    Estado actualizado
         |
         v
      Response

El endpoint no debería necesariamente conocer todos los detalles internos del objeto.

El servicio puede invocar:

    book.prestar()

y utilizar el resultado.

Esto ayuda a separar:

    transporte HTTP

de:

    lógica de dominio

---

# 25. Conexión con arquitectura de software

Un diseño más completo puede adoptar:

    API
     |
     v
    Application Service
     |
     v
    Domain Model
     |
     v
    Repository
     |
     v
    Database

Por ejemplo:

    POST /books/{id}/lend
              |
              v
       BookService
              |
              v
       book.prestar()
              |
              v
       Repository.save(book)
              |
              v
          Database

Aquí el método de instancia representa una regla de dominio, mientras que el repositorio se ocupa de persistencia.

Esta separación evita mezclar lógica de negocio con infraestructura.

---

# 26. Problemas reales en producción

## Problema: métodos que imprimen en lugar de retornar

### Causa

Diseñar métodos pensando solamente en interacción desde terminal.

### Síntoma

El método funciona en pruebas manuales, pero es difícil reutilizarlo desde:

- APIs;
- servicios;
- tests;
- workers;
- agentes;
- interfaces gráficas.

### Solución

Preferir:

    return resultado

sobre:

    print(resultado)

cuando el resultado tenga valor para el consumidor.

### Prevención

Diseñar las clases como componentes reutilizables y dejar la presentación en una capa superior.

---

# 27. Problemas reales en producción

## Problema: métodos que modifican estado sin validar reglas

Ejemplo:

    def prestar(self):
        self.available = False

Este método permite ejecutar el préstamo incluso si el libro ya estaba prestado.

Una implementación más segura valida:

    def prestar(self):
        if not self.available:
            return f"{self.title} no está disponible"

        self.available = False
        return f"{self.title} prestado exitosamente"

La validación debe ocurrir antes de la transición de estado.

---

# 28. Problemas reales en producción

## Problema: clase con demasiadas responsabilidades

Una clase `Book` que además:

- envía correos;
- realiza consultas SQL;
- llama APIs externas;
- genera embeddings;
- registra logs;
- autentica usuarios;
- procesa pagos;

tendría responsabilidades excesivas.

Una arquitectura profesional separaría esas responsabilidades.

Por ejemplo:

    Book
      |
      v
    BookService
      |
      +--> Repository
      |
      +--> NotificationService
      |
      +--> External APIs

La clase de dominio debe concentrarse en las reglas que realmente pertenecen a la entidad.

---

# 29. Buenas prácticas

- Utilizar nombres que expresen intención.
- Preferir `prestar()` frente a `cambiar_disponibilidad()`.
- Mantener las reglas relacionadas con el estado cerca del objeto cuando corresponda.
- Preferir `return` frente a `print` para resultados reutilizables.
- Mantener `__str__` simple y orientado a representación humana.
- Evitar que una clase acumule responsabilidades no relacionadas.
- Validar las transiciones de estado.
- Escribir pruebas para cada comportamiento importante.
- Diferenciar estado actual de historial de eventos.
- Diseñar pensando en cómo la clase será utilizada desde servicios y APIs.

---

# 30. Preguntas de entrevista técnica

## Pregunta 1 — ¿Qué es un método de instancia en Python?

### Qué evalúa

Comprueba si el candidato entiende cómo una instancia puede encapsular comportamiento además de datos.

### Error común

Decir únicamente que es "una función dentro de una clase".

### Respuesta de alto impacto del entrevistado

> Un método de instancia es un comportamiento asociado a una instancia concreta y recibe normalmente `self` como primer parámetro. Esto le permite leer y modificar el estado de esa instancia. Por ejemplo, `book.prestar()` no solo ejecuta una función, sino que representa una transición de estado del objeto según una regla de negocio.

---

## Pregunta 2 — ¿Qué ocurre internamente cuando ejecutas `book.prestar()`?

### Qué evalúa

Evalúa comprensión del binding de métodos.

### Error común

Decir simplemente que "Python llama al método".

### Respuesta de alto impacto del entrevistado

> Python resuelve el atributo `prestar` en la clase y lo enlaza con la instancia `book`, de modo que la instancia se proporciona como primer argumento, convencionalmente llamado `self`. Conceptualmente, `book.prestar()` equivale a invocar el método de clase pasando `book` como instancia, lo que permite acceder a `self.available`, `self.title` y al resto del estado.

---

## Pregunta 3 — ¿Por qué prefieres `return` sobre `print` dentro de una clase?

### Qué evalúa

Evalúa diseño reutilizable y separación de responsabilidades.

### Error común

Decir que `print` es incorrecto.

### Respuesta de alto impacto del entrevistado

> `print` no es incorrecto, pero acopla el método a una salida concreta. Prefiero `return` cuando el resultado puede ser utilizado por otras capas, porque el consumidor puede decidir si desea imprimirlo, registrarlo, convertirlo en una respuesta HTTP o procesarlo de otra manera. En una arquitectura backend esta separación es especialmente importante.

---

## Pregunta 4 — ¿Qué diferencia existe entre modificar directamente un atributo y utilizar un método de dominio?

### Qué evalúa

Evalúa encapsulación y diseño orientado al dominio.

### Error común

Afirmar que acceder directamente a atributos siempre está mal.

### Respuesta de alto impacto del entrevistado

> La diferencia principal está en dónde reside la regla de negocio. Si hago `book.available = False`, cualquier consumidor puede modificar el estado sin necesariamente respetar las reglas del dominio. Con `book.prestar()`, la clase puede validar el estado, ejecutar la transición y devolver un resultado. No significa que todo atributo deba ocultarse, sino que las invariantes importantes deberían tener un punto de control claro.

---

## Pregunta 5 — ¿Cuándo almacenarías un historial de préstamos y cuándo solamente un contador?

### Qué evalúa

Evalúa capacidad de tomar decisiones de diseño según los requisitos.

### Error común

Decir que siempre es mejor almacenar todo.

### Respuesta de alto impacto del entrevistado

> Depende del requisito. Si únicamente necesito conocer cuántas veces se prestó el libro, un contador puede ser suficiente y más eficiente. Si necesito auditoría, fechas, usuarios, trazabilidad o análisis histórico, necesito conservar eventos o registros de préstamos. La decisión debe basarse en los requisitos funcionales, de auditoría y de análisis, no solamente en la estructura de la clase.

---

## Pregunta 6 — ¿Cómo conectarías un objeto de dominio con una API?

### Qué evalúa

Evalúa comprensión arquitectónica.

### Error común

Poner toda la lógica dentro del endpoint.

### Respuesta de alto impacto del entrevistado

> Mantendría separadas las responsabilidades. FastAPI manejaría el transporte HTTP, un servicio de aplicación coordinaría el caso de uso y el objeto de dominio encapsularía las reglas correspondientes. Por ejemplo, el endpoint recibe la solicitud, el servicio obtiene el `Book`, ejecuta `book.prestar()` y posteriormente persiste el cambio mediante un repositorio. Esto facilita testing, mantenimiento y evolución de la arquitectura.

---

# 31. Checklist de dominio

Al terminar esta clase debes poder explicar sin consultar documentación:

- [ ] Qué es un método de instancia.
- [ ] Cómo `self` permite acceder al estado de una instancia.
- [ ] Qué ocurre conceptualmente cuando ejecutas `book.prestar()`.
- [ ] Cómo un método puede modificar el estado de un objeto.
- [ ] Por qué los nombres de los métodos deben expresar intención de negocio.
- [ ] Qué función cumple `__str__`.
- [ ] Por qué `__str__` debe devolver un `str`.
- [ ] Diferencia entre `return` y `print`.
- [ ] Qué devuelve un método que no tiene `return`.
- [ ] Cómo funcionan las f-strings.
- [ ] Cómo diseñar `prestar()` y `devolver()` como transiciones de estado.
- [ ] Qué significa encapsular reglas de negocio dentro de un objeto.
- [ ] Cuándo conviene almacenar un historial y cuándo un contador.
- [ ] Cómo evitar clases con demasiadas responsabilidades.
- [ ] Cómo integrar un modelo de dominio con un servicio backend.
- [ ] Cómo estos conceptos aparecen en componentes de AI Engineering.

---

# 32. Respuestas del checklist — conexión con AI Engineering

## ¿Qué es un método de instancia?

Es un comportamiento asociado a una instancia concreta y capaz de operar sobre su estado mediante `self`. En AI Engineering aparece en objetos como agentes, documentos, retrievers y clientes de servicios que necesitan mantener configuración o estado.

## ¿Cómo `self` permite acceder al estado?

`self` referencia la instancia actual y permite acceder a atributos como `self.available`, `self.title` o `self.config`. En sistemas de IA esto permite que diferentes instancias de un componente mantengan configuraciones y estados independientes.

## ¿Qué ocurre al ejecutar `book.prestar()`?

Python enlaza el método con `book` y proporciona esa instancia como `self`. El método puede entonces validar y modificar su estado. Es el mismo mecanismo fundamental utilizado por objetos más complejos dentro de aplicaciones Python.

## ¿Cómo puede un método modificar el estado?

Mediante atributos de instancia:

    self.available = False

Esto representa una transición de estado controlada por comportamiento. En sistemas reales, las transiciones de estado son importantes para workflows, agentes, tareas y procesos empresariales.

## ¿Por qué utilizar nombres de intención de negocio?

Porque `prestar()` comunica qué operación representa, mientras `cambiar_disponibilidad()` expone un detalle técnico. Un código profesional debe expresar intención y reducir el conocimiento que los consumidores necesitan sobre la implementación.

## ¿Qué función cumple `__str__`?

Define una representación textual legible del objeto. Es útil para debugging, inspección y herramientas interactivas, aunque no debe confundirse con una serialización formal para APIs.

## ¿Por qué `__str__` debe retornar `str`?

Porque Python espera que la representación producida por `str(obj)` sea una cadena. Si el método no devuelve un `str`, puede producir errores de ejecución.

## ¿Por qué `return` es preferible a `print`?

Porque separa lógica de negocio de presentación. Un resultado retornado puede ser consumido por una API, un servicio, un test, un logger o cualquier otra capa.

## ¿Qué ocurre sin `return`?

El método devuelve `None` implícitamente. Esto es importante al diseñar APIs internas porque el consumidor debe saber si una operación produce un resultado o solamente genera efectos secundarios.

## ¿Cómo funcionan las f-strings?

Permiten interpolar expresiones dentro de cadenas:

    f"{self.title} prestado"

Son útiles para mensajes, logging y representación textual, aunque en aplicaciones profesionales debe evitarse utilizar texto generado manualmente cuando existe un contrato estructurado como JSON.

## ¿Cómo funcionan `prestar()` y `devolver()`?

Representan transiciones explícitas:

    disponible
        |
      prestar
        v
    no disponible

y:

    no disponible
        |
      devolver
        v
    disponible

Este modelo mental es aplicable a workflows y máquinas de estados utilizadas en sistemas backend y AI.

## ¿Qué significa encapsular reglas de negocio?

Significa colocar una regla cerca del estado al que pertenece cuando el diseño lo justifica. Por ejemplo, el propio `Book` puede impedir un préstamo cuando ya no está disponible.

## ¿Cuándo utilizar historial frente a contador?

Un contador es adecuado cuando solo interesa la cantidad. Un historial es necesario cuando necesitamos eventos, auditoría, timestamps, usuarios o análisis. En sistemas AI, esta misma decisión aparece al elegir entre guardar estado agregado o conservar trazas/eventos completos.

## ¿Cómo evitar clases demasiado grandes?

Manteniendo responsabilidades cohesivas y separando dominio, servicios, persistencia, integraciones y transporte. Esto permite que una clase no termine dependiendo de bases de datos, APIs externas y lógica de infraestructura simultáneamente.

## ¿Cómo conectar el modelo de dominio con backend?

Una arquitectura habitual es:

    HTTP
      |
      v
    FastAPI
      |
      v
    Application Service
      |
      v
    Domain Object
      |
      v
    Repository
      |
      v
    Database

El mismo patrón puede utilizarse en sistemas AI:

    API
      |
      v
    AI Service
      |
      v
    Domain / Agent
      |
      +--> LLM
      +--> Tools
      +--> Retriever
      +--> Vector Store
      |
      v
    Persistence / Observability

---

# 33. Resultado profesional esperado

Al terminar esta clase, el objetivo no es simplemente poder escribir:

    class Book:
        def prestar(self):
            ...

El objetivo es comprender que un objeto puede representar:

    Estado
      +
    Comportamiento
      +
    Reglas
      +
    Transiciones

y que este modelo puede convertirse posteriormente en componentes de sistemas reales.

La progresión profesional es:

    Datos
      |
      v
    Objetos
      |
      v
    Métodos
      |
      v
    Reglas de negocio
      |
      v
    Modelos de dominio
      |
      v
    Servicios
      |
      v
    APIs
      |
      v
    Sistemas distribuidos
      |
      v
    AI Applications

Este es el salto conceptual de la clase: pasar de **objetos que almacenan datos** a **objetos que encapsulan comportamiento y reglas de negocio**.