# Python Orientado a Objetos — Clases, Objetos y Modelado de Sistemas

## 1. Objetivo de la clase

Comprender cómo Python utiliza la Programación Orientada a Objetos (OOP) para modelar entidades de un dominio mediante clases y objetos, y desarrollar la capacidad de transformar requisitos del mundo real en estructuras de software mantenibles.

En esta clase se construirá progresivamente un pequeño sistema de biblioteca utilizando una clase `Libro`.

El objetivo no es únicamente aprender la sintaxis de `class` o `__init__`, sino entender la responsabilidad que tiene cada elemento dentro del modelo:

- `class` define la estructura y el comportamiento.
- Una instancia representa una entidad concreta.
- Los atributos representan el estado del objeto.
- Los métodos representan comportamientos.
- `self` permite acceder al estado perteneciente a una instancia específica.
- El constructor inicializa el estado inicial del objeto.

Este conocimiento constituye una base fundamental para trabajar posteriormente con APIs, FastAPI, ORM, sistemas empresariales, agentes de IA, RAG y arquitecturas de software más complejas.

---

## 2. Contenido del curso

El contenido base de esta clase introduce:

- Programación Orientada a Objetos.
- Clases.
- Objetos.
- Atributos.
- Métodos.
- Constructor `__init__`.
- Referencia `self`.
- Instanciación de objetos.
- Modelado de una entidad real mediante una clase.
- Uso de `uv` para crear el proyecto.
- Uso de entornos virtuales.
- Almacenamiento de objetos en listas.
- Iteración sobre objetos mediante `for`.
- Uso de atributos booleanos.

---

# 3. Programación Orientada a Objetos

La Programación Orientada a Objetos permite representar entidades de un dominio mediante objetos que encapsulan:

- estado;
- comportamiento;
- reglas relacionadas con ese estado.

En lugar de trabajar únicamente con variables y funciones independientes, OOP permite agrupar datos y operaciones relacionadas dentro de una estructura coherente.

Por ejemplo, en un sistema de biblioteca existen entidades como:

- libros;
- usuarios;
- préstamos;
- autores;
- categorías.

Un libro puede representarse mediante un objeto que contenga información como:

- título;
- autor;
- ISBN;
- disponibilidad.

Posteriormente, la clase puede incorporar comportamientos relacionados con el libro.

---

# 4. ¿Qué es una clase?

Una clase es una definición que describe la estructura y el comportamiento que tendrán determinadas instancias.

Puede considerarse como el modelo a partir del cual se crean objetos.

Por ejemplo:

    class Libro:
        pass

La palabra reservada `class` indica que estamos declarando una clase.

`Libro` es el nombre de la clase.

`pass` indica que actualmente no existe ninguna implementación dentro del cuerpo de la clase.

La clase todavía no representa un libro concreto.

Representa el modelo que permitirá crear libros.

---

# 5. Clase vs. objeto

Esta distinción es fundamental.

La clase define cómo será una entidad.

El objeto es una instancia concreta de esa clase.

Por ejemplo:

    class Libro:
        pass

Aquí `Libro` es una clase.

Cuando hacemos:

    libro_1 = Libro()
    libro_2 = Libro()

se crean dos objetos diferentes a partir de la misma clase.

Conceptualmente:

    Clase
       |
       +----> libro_1
       |
       +----> libro_2
       |
       +----> libro_3

Los objetos comparten la estructura definida por la clase, pero cada instancia puede mantener valores diferentes.

---

# 6. Modelado del dominio

Una de las capacidades más importantes de un ingeniero de software no es memorizar sintaxis, sino identificar entidades relevantes dentro de un problema.

Supongamos que necesitamos desarrollar un sistema de biblioteca.

Una primera identificación del dominio podría ser:

    Biblioteca
        |
        +-- Libro
        |     +-- título
        |     +-- autor
        |     +-- ISBN
        |     +-- disponible
        |
        +-- Usuario
        |
        +-- Préstamo

En esta clase comenzaremos únicamente con `Libro`.

La decisión de modelar `Libro` como una clase permite representar múltiples libros utilizando una estructura común.

---

# 7. Constructor `__init__`

Una clase puede necesitar recibir información cuando se crea una nueva instancia.

Para esto se utiliza normalmente el método especial:

    __init__

Ejemplo:

    class Libro:

        def __init__(self, titulo, autor):
            self.titulo = titulo
            self.autor = autor

Cuando se crea un objeto:

    libro = Libro("Cien años de soledad", "Gabriel García Márquez")

Python ejecuta automáticamente `__init__` para inicializar el objeto.

El constructor recibe los valores necesarios y los almacena como atributos de la instancia.

---

# 8. ¿Qué significa `self`?

`self` representa la instancia actual.

Este concepto es esencial para comprender OOP en Python.

En:

    self.titulo = titulo

existen dos elementos diferentes:

- `titulo` es el parámetro recibido por el método.
- `self.titulo` es el atributo almacenado dentro de la instancia.

Por ejemplo:

    libro_1 = Libro("Cien años de soledad", "Gabriel García Márquez")
    libro_2 = Libro("El principito", "Antoine de Saint-Exupéry")

Internamente podemos conceptualizar el estado como:

    libro_1
        titulo = "Cien años de soledad"
        autor = "Gabriel García Márquez"

    libro_2
        titulo = "El principito"
        autor = "Antoine de Saint-Exupéry"

Ambos objetos utilizan la misma clase, pero mantienen estados diferentes.

---

# 9. ¿Por qué `self` no se escribe al instanciar?

Cuando hacemos:

    libro = Libro("Cien años de soledad", "Gabriel García Márquez")

no escribimos:

    Libro(self, "Cien años de soledad", "Gabriel García Márquez")

Python determina la instancia y pasa la referencia correspondiente al método.

Por eso `self` aparece en la definición del método, pero no se proporciona explícitamente en una llamada normal al método.

Conceptualmente:

    libro = Libro(...)
              |
              v
        __init__(self, ...)
              |
              v
        self = libro

Esto permite que:

    self.titulo

haga referencia al atributo perteneciente específicamente a ese objeto.

---

# 10. Atributos de instancia

Los atributos definidos mediante `self` pertenecen a una instancia concreta.

Ejemplo:

    class Libro:

        def __init__(self, titulo, autor):
            self.titulo = titulo
            self.autor = autor

Cada objeto puede tener valores diferentes:

    libro_1 = Libro(
        "Cien años de soledad",
        "Gabriel García Márquez"
    )

    libro_2 = Libro(
        "El principito",
        "Antoine de Saint-Exupéry"
    )

Por lo tanto:

    libro_1.titulo
    # Cien años de soledad

    libro_2.titulo
    # El principito

El atributo `titulo` existe en ambas instancias, pero cada instancia posee su propio valor.

---

# 11. Crear objetos

Una vez definida la clase, podemos crear múltiples instancias.

    class Libro:

        def __init__(self, titulo, autor):
            self.titulo = titulo
            self.autor = autor


    mi_libro = Libro(
        "Cien años de soledad",
        "Gabriel García Márquez"
    )

    otro_libro = Libro(
        "El principito",
        "Antoine de Saint-Exupéry"
    )

Aquí:

- `Libro` es la clase.
- `mi_libro` es una instancia.
- `otro_libro` es otra instancia.
- ambos objetos utilizan la misma estructura.
- cada uno mantiene sus propios valores.

---

# 12. Acceder a los atributos

Los atributos de una instancia se pueden consultar utilizando la notación:

    objeto.atributo

Por ejemplo:

    print(mi_libro.titulo)
    print(mi_libro.autor)

    print(otro_libro.titulo)
    print(otro_libro.autor)

Esto permite acceder al estado de cada objeto.

---

# 13. Métodos

Las funciones definidas dentro de una clase reciben el nombre de métodos.

Aunque esta clase se centra principalmente en clases, objetos y constructores, es importante establecer la relación:

    Clase
      |
      +-- atributos -> estado
      |
      +-- métodos   -> comportamiento

Por ejemplo, un `Libro` podría posteriormente incorporar comportamientos como:

    prestar()
    devolver()
    mostrar_informacion()

Esto permite que el objeto no sea solamente un contenedor de datos, sino una representación de una entidad con comportamiento.

---

# 14. Flujo completo

El flujo conceptual de nuestro programa es:

    Definición de clase
            |
            v
    class Libro
            |
            v
    Definición de __init__
            |
            v
    Definición de atributos
            |
            v
    Libro(...)
            |
            v
    Creación de instancia
            |
            v
    Objeto Libro
            |
            v
    libro.titulo
    libro.autor

La clase define la estructura.

La instancia representa una entidad concreta.

Los atributos almacenan el estado.

---

# 15. Proyecto con `uv`

El contenido de la clase utiliza `uv` para preparar el proyecto Python.

Antes de comenzar es conveniente comprobar que `uv` está disponible:

    uv --version

Si el comando devuelve una versión, `uv` está disponible en el entorno.

Para inicializar un proyecto:

    uv init

El proyecto generado proporciona una estructura inicial para trabajar de forma organizada.

La estructura exacta puede variar según la versión de `uv` y las operaciones posteriores, pero conceptualmente trabajaremos con:

    proyecto-biblioteca/
    ├── main.py
    ├── pyproject.toml
    ├── README.md
    └── .python-version

Después de resolver dependencias o sincronizar el entorno, también puede aparecer:

    uv.lock

---

# 16. ¿Por qué utilizar `uv`?

`uv` permite gestionar diferentes aspectos del entorno Python de forma rápida:

- creación de proyectos;
- gestión de versiones de Python;
- creación de entornos virtuales;
- instalación de dependencias;
- resolución de dependencias;
- ejecución de comandos dentro del entorno.

Para un AI Engineer, el valor real no está en memorizar comandos aislados, sino en comprender la reproducibilidad del entorno.

Un proyecto profesional debe poder pasar de:

    Developer A
          |
          v
    Git repository
          |
          v
    Developer B / CI
          |
          v
    mismo entorno esperado

La gestión correcta de versiones y dependencias reduce problemas de compatibilidad.

---

# 17. Entornos virtuales

Un entorno virtual permite aislar las dependencias de un proyecto.

Sin aislamiento, diferentes proyectos pueden requerir versiones incompatibles de una misma biblioteca.

Conceptualmente:

    Sistema operativo
          |
          +-------------------+
          |                   |
          v                   v
      Proyecto A          Proyecto B
      Python/deps         Python/deps
      aisladas             aisladas

Con `uv` podemos crear un entorno virtual mediante:

    uv venv

Después se puede activar según el shell utilizado.

En Windows PowerShell:

    .venv\Scripts\Activate.ps1

En Linux/macOS:

    source .venv/bin/activate

Otra alternativa es utilizar directamente:

    uv run python main.py

Esto permite ejecutar el programa utilizando el entorno administrado por `uv` sin depender de una activación manual del entorno.

---

# 18. Implementación inicial

Una implementación completa de la clase sería:

    class Libro:

        def __init__(self, titulo, autor):
            self.titulo = titulo
            self.autor = autor


    mi_libro = Libro(
        "Cien años de soledad",
        "Gabriel García Márquez"
    )

    otro_libro = Libro(
        "El principito",
        "Antoine de Saint-Exupéry"
    )


    print(mi_libro.titulo)
    print(mi_libro.autor)

    print(otro_libro.titulo)
    print(otro_libro.autor)

El programa crea dos objetos independientes a partir de una misma clase.

---

# 19. Comprensión interna

Es importante diferenciar lo que ocurre conceptualmente durante la creación del objeto.

Cuando ejecutamos:

    libro = Libro("Cien años de soledad", "Gabriel García Márquez")

Python debe:

    1. Crear una nueva instancia de Libro.
    2. Ejecutar la inicialización correspondiente.
    3. Pasar la referencia de la instancia a __init__ mediante self.
    4. Asignar el título.
    5. Asignar el autor.
    6. Entregar la instancia resultante a la variable libro.

Conceptualmente:

    Libro(...)
       |
       v
    instancia
       |
       v
    __init__(self, titulo, autor)
       |
       +--> self.titulo = titulo
       |
       +--> self.autor = autor
       |
       v
    objeto inicializado

Esto es mucho más importante para un ingeniero que memorizar simplemente que `__init__` es "el constructor".

---

# 20. Precisión técnica sobre `__init__`

En Python, `__init__` se utiliza para inicializar una instancia después de que esta ha sido creada.

Técnicamente, `__init__` no es el mecanismo que crea físicamente la instancia.

La creación de la instancia está relacionada con `__new__`.

El flujo conceptual simplificado es:

    Clase(...)
       |
       v
    __new__()
       |
       v
    nueva instancia
       |
       v
    __init__()
       |
       v
    instancia inicializada

Para esta clase no es necesario implementar `__new__`, pero conocer esta distinción evita una simplificación conceptual incorrecta.

---

# 21. Challenge — Extender `Libro`

El ejercicio propuesto consiste en ampliar el modelo.

La clase debe incorporar:

- `titulo`;
- `autor`;
- `isbn`;
- `disponible`.

`disponible` debe utilizar un valor booleano:

    True

o:

    False

Una implementación posible:

    class Libro:

        def __init__(self, titulo, autor, isbn, disponible):
            self.titulo = titulo
            self.autor = autor
            self.isbn = isbn
            self.disponible = disponible

---

# 22. Crear un catálogo

Después podemos crear una colección de libros:

    catalogo = [
        Libro(
            "Cien años de soledad",
            "Gabriel García Márquez",
            "978-0307474728",
            True
        ),
        Libro(
            "El principito",
            "Antoine de Saint-Exupéry",
            "978-0156012195",
            False
        )
    ]

Ahora `catalogo` contiene objetos `Libro`.

La estructura conceptual es:

    catalogo
       |
       +--> Libro
       |
       +--> Libro
       |
       +--> Libro
       |
       +--> ...

Esto representa un patrón extremadamente frecuente en software:

    colección de entidades
            |
            v
        objetos
            |
            v
       procesamiento

---

# 23. Iterar sobre los objetos

Podemos utilizar un ciclo `for` para recorrer el catálogo:

    for libro in catalogo:
        print(
            libro.titulo,
            libro.autor,
            libro.disponible
        )

Cada iteración asigna una instancia diferente a la variable `libro`.

Conceptualmente:

    catalogo
       |
       v
    for libro in catalogo
       |
       +--> libro 1
       |
       +--> libro 2
       |
       +--> libro 3
       |
       v
    procesar cada objeto

---

# 24. Booleanos y reglas de negocio

El atributo:

    disponible

no es solamente un dato.

En un sistema real puede representar una regla de negocio.

Por ejemplo:

    if libro.disponible:
        print("Libro disponible")
    else:
        print("Libro no disponible")

Más adelante, una arquitectura más madura podría evitar modificar directamente el atributo desde cualquier parte del programa y encapsular la operación:

    libro.prestar()

    libro.devolver()

Esto permite centralizar reglas como:

- un libro no puede prestarse si ya está prestado;
- al prestar un libro debe cambiar su estado;
- al devolverlo debe volver a estar disponible.

Este cambio representa la evolución desde un simple modelo de datos hacia un objeto con comportamiento.

---

# 25. Expansión técnica — criterio de ingeniería

El ejemplo del curso es correcto como introducción, pero un AI Engineer debe aprender a identificar sus límites.

Esta clase:

    class Libro:

        def __init__(self, titulo, autor, isbn, disponible):
            self.titulo = titulo
            self.autor = autor
            self.isbn = isbn
            self.disponible = disponible

funciona.

Sin embargo, en un sistema de producción aparecen preguntas adicionales:

- ¿Puede existir un libro sin ISBN?
- ¿Puede haber dos libros con el mismo ISBN?
- ¿Quién controla la disponibilidad?
- ¿Puede cambiarse `disponible` directamente?
- ¿Qué ocurre si se intenta prestar un libro no disponible?
- ¿Cómo se valida el ISBN?
- ¿Dónde se persiste el libro?
- ¿Cómo se serializa a JSON?
- ¿Cómo se expone mediante una API?
- ¿Cómo se prueba?
- ¿Cómo se registra un error?
- ¿Cómo se integra con una base de datos?

Estas preguntas convierten un ejercicio de sintaxis en un problema real de ingeniería.

---

# 26. De modelo simple a sistema real

La evolución natural podría ser:

    Clase Python
         |
         v
    Objetos
         |
         v
    Colección
         |
         v
    Reglas de negocio
         |
         v
    Servicios
         |
         v
    Persistencia
         |
         v
    API
         |
         v
    Aplicación

En una arquitectura backend:

    Cliente
       |
       v
    API / FastAPI
       |
       v
    Service Layer
       |
       v
    Domain Model
       |
       v
    Repository
       |
       v
    Base de datos

La clase `Libro` puede evolucionar desde un ejercicio educativo hasta formar parte de un modelo de dominio.

---

# 27. Software Engineering

El aprendizaje de OOP no debe reducirse a crear muchas clases.

Un diseño orientado a objetos debe mejorar propiedades importantes del software:

- separación de responsabilidades;
- modularidad;
- legibilidad;
- mantenibilidad;
- testabilidad;
- reutilización;
- manejo de errores;
- seguridad;
- escalabilidad.

Una clase debe tener una responsabilidad clara.

Un error frecuente es crear clases enormes que concentran:

    datos
    validaciones
    acceso a base de datos
    llamadas HTTP
    lógica de negocio
    logging
    autenticación

Una mejor arquitectura separa responsabilidades.

---

# 28. OOP y AI Engineering

La Programación Orientada a Objetos aparece de forma directa e indirecta en el ecosistema de IA.

Por ejemplo:

    Aplicación AI
          |
          v
    API
          |
          v
    Servicio
          |
          v
    Modelo de dominio
          |
          +--> Cliente LLM
          |
          +--> Retriever
          |
          +--> Vector Store
          |
          +--> Embedding Service
          |
          +--> Agent
          |
          +--> Tool

Frameworks y SDKs modernos utilizan clases y objetos para representar componentes configurables.

Por eso comprender:

    clases
    objetos
    atributos
    métodos
    composición
    encapsulación
    herencia
    interfaces/protocolos

facilita posteriormente la lectura y extensión de código profesional relacionado con:

    OpenAI SDK
    FastAPI
    LangChain
    LangGraph
    SDKs de cloud
    clientes de bases de datos
    herramientas de observabilidad

No significa que todo código de IA deba diseñarse mediante jerarquías complejas de clases.

En sistemas modernos, la composición y las funciones pequeñas suelen ser preferibles a una herencia excesiva.

---

# 29. Contenido del curso vs. expansión técnica

## Contenido del curso

El núcleo de esta clase establece que:

- una clase funciona como un modelo;
- los objetos son instancias de una clase;
- una clase puede definir atributos;
- `__init__` inicializa una instancia;
- `self` referencia la instancia actual;
- pueden crearse múltiples objetos de una misma clase;
- cada objeto puede mantener valores diferentes;
- `uv` permite preparar el entorno del proyecto;
- los entornos virtuales aíslan dependencias;
- una lista puede almacenar múltiples objetos;
- un ciclo `for` puede recorrer dichos objetos;
- un atributo booleano puede representar disponibilidad.

## Expansión técnica

Desde una perspectiva de ingeniería profesional:

- `__init__` inicializa una instancia, pero la creación está relacionada con `__new__`;
- los atributos de instancia representan estado asociado al objeto;
- las clases deben tener responsabilidades claras;
- el estado y comportamiento pueden encapsular reglas de negocio;
- una colección de objetos puede representar entidades de dominio;
- OOP debe utilizarse cuando mejora el diseño, no por obligación;
- composición suele ser una alternativa importante frente a jerarquías profundas de herencia;
- la arquitectura debe separar dominio, servicios, infraestructura y transporte cuando el sistema lo requiera.

## Actualización moderna

En proyectos actuales con Python:

- `uv` puede utilizarse para gestionar proyectos y dependencias;
- `pyproject.toml` es un elemento central de configuración del proyecto Python;
- `uv.lock` permite fijar versiones resueltas de dependencias cuando el proyecto utiliza el flujo correspondiente;
- `uv run` facilita ejecutar comandos dentro del entorno gestionado;
- la reproducibilidad del entorno debe formar parte del diseño del proyecto;
- en sistemas backend y AI Engineering, las clases deben integrarse dentro de una arquitectura modular y testeable.

---

# 30. Errores frecuentes

## Error 1 — Confundir clase con objeto

Incorrecto conceptualmente:

    Libro = "Cien años de soledad"

Esto no representa una clase.

Una clase debe definir una estructura:

    class Libro:
        ...

Y una instancia representa una entidad concreta:

    libro = Libro(...)

---

## Error 2 — Olvidar `self`

Incorrecto:

    class Libro:

        def __init__(titulo, autor):
            self.titulo = titulo
            self.autor = autor

Correcto:

    class Libro:

        def __init__(self, titulo, autor):
            self.titulo = titulo
            self.autor = autor

---

## Error 3 — Confundir parámetro con atributo

En:

    def __init__(self, titulo):
        self.titulo = titulo

`titulo` y `self.titulo` cumplen funciones diferentes.

    titulo
       |
       v
    parámetro recibido

    self.titulo
       |
       v
    estado almacenado en la instancia

---

## Error 4 — Compartir accidentalmente estado entre instancias

Los atributos de instancia deben utilizar `self`.

Por ejemplo:

    self.titulo = titulo

es diferente de definir un atributo de clase:

    class Libro:
        titulo = "desconocido"

Los atributos de clase y de instancia tienen semánticas diferentes y deben utilizarse deliberadamente.

---

## Error 5 — Utilizar una clase cuando una función sería suficiente

No todo problema necesita OOP.

Si una operación es simple, independiente y no necesita mantener estado, una función puede ser una solución más clara.

La decisión profesional no es:

    "¿Puedo crear una clase?"

sino:

    "¿Una clase mejora el diseño del sistema?"

---

# 31. Problemas reales en producción

## Problema: estado inconsistente

### Causa

Permitir que cualquier parte de la aplicación modifique directamente atributos críticos.

Ejemplo:

    libro.disponible = True

sin comprobar reglas de negocio.

### Síntoma

El sistema puede terminar representando estados imposibles.

### Diagnóstico

Buscar dónde se modifica el estado y qué reglas se ejecutan antes de la modificación.

### Solución

Centralizar las operaciones relevantes:

    libro.prestar()
    libro.devolver()

### Prevención

Definir claramente quién tiene responsabilidad sobre cada transición de estado y cubrir las reglas mediante pruebas automatizadas.

---

# 32. Problemas reales en producción

## Problema: entorno no reproducible

### Causa

Diferencias entre versiones de Python o dependencias.

### Síntomas

- funciona en una máquina;
- falla en otra;
- falla en CI/CD;
- aparecen errores de compatibilidad.

### Diagnóstico

Revisar:

    Python version
    pyproject.toml
    lock file
    dependencias
    entorno virtual

### Solución

Utilizar gestión reproducible de Python y dependencias.

### Prevención

Versionar la configuración necesaria del proyecto y utilizar CI para verificar que el proyecto pueda instalarse y ejecutarse desde cero.

---

# 33. Buenas prácticas

Para esta etapa:

1. Utilizar nombres de clases claros.
2. Utilizar nombres de atributos representativos.
3. Mantener las responsabilidades de las clases limitadas.
4. No introducir herencia sin una relación real.
5. Preferir composición cuando represente mejor el dominio.
6. Validar los datos en los límites apropiados.
7. Evitar estado global innecesario.
8. Escribir pruebas para reglas de negocio.
9. Mantener las dependencias explícitas.
10. Diseñar pensando en mantenimiento y evolución.

---

# 34. Preguntas de entrevista técnica

## Pregunta 1

### ¿Cuál es la diferencia entre una clase y una instancia en Python?

**Qué evalúa el entrevistador:**

Comprueba si el candidato entiende el modelo fundamental de OOP y no solamente la sintaxis.

**Error común:**

Responder únicamente que "la clase es el molde y el objeto es lo creado", sin explicar el estado independiente de las instancias.

**Respuesta de alto impacto:**

> Una clase define la estructura y comportamiento que tendrán sus instancias. Una instancia es un objeto concreto creado a partir de esa clase y mantiene su propio estado. Por ejemplo, `Libro` puede definir los atributos y comportamientos de un libro, mientras que `Libro("Cien años de soledad", "...")` representa una instancia específica. En diseño profesional, la decisión de utilizar una clase debe justificarse porque mejora encapsulación, mantenibilidad o modelado del dominio.

---

## Pregunta 2

### ¿Qué función cumple `self`?

**Qué evalúa el entrevistador:**

Comprueba si el candidato entiende cómo los métodos acceden al estado de una instancia.

**Error común:**

Decir simplemente que "`self` significa el objeto" sin explicar su relación con los atributos.

**Respuesta de alto impacto:**

> `self` es la referencia a la instancia sobre la que se está ejecutando el método. Permite acceder al estado de esa instancia mediante expresiones como `self.titulo`. Python pasa esa referencia cuando invocamos un método de instancia, por eso normalmente no la proporcionamos explícitamente al llamar al método.

---

## Pregunta 3

### ¿Qué ocurre cuando ejecutas `Libro("Cien años de soledad", "Gabriel García Márquez")`?

**Qué evalúa el entrevistador:**

Evalúa comprensión del ciclo de creación e inicialización de objetos.

**Error común:**

Responder solamente que "`__init__` crea el objeto".

**Respuesta de alto impacto:**

> Conceptualmente, Python crea la instancia y posteriormente ejecuta `__init__` para inicializarla. La creación está asociada a `__new__`, mientras que `__init__` configura el estado inicial. Por eso es más preciso decir que `__init__` inicializa la instancia y no que físicamente la crea.

---

## Pregunta 4

### ¿Cuándo utilizarías una clase en lugar de una función?

**Qué evalúa el entrevistador:**

Evalúa criterio de diseño.

**Error común:**

Decir que las clases son siempre mejores porque permiten reutilización.

**Respuesta de alto impacto:**

> Utilizaría una clase cuando necesito representar una entidad con estado y comportamiento relacionado, o cuando la encapsulación y el ciclo de vida de una entidad aportan claridad al diseño. Si la operación es stateless y puede expresarse claramente como una función, prefiero una función. La decisión debe basarse en cohesión, mantenibilidad y responsabilidades, no en utilizar OOP por obligación.

---

## Pregunta 5

### ¿Qué problemas pueden aparecer si una clase tiene demasiadas responsabilidades?

**Qué evalúa el entrevistador:**

Evalúa conocimiento de diseño y mantenibilidad.

**Error común:**

Responder solamente que "el código se vuelve largo".

**Respuesta de alto impacto:**

> Una clase con demasiadas responsabilidades aumenta el acoplamiento, dificulta las pruebas, hace más costosos los cambios y puede provocar que modificaciones aparentemente pequeñas tengan efectos secundarios. En producción prefiero separar responsabilidades mediante componentes cohesivos y utilizar composición o capas arquitectónicas cuando el dominio lo requiera.

---

# 35. Checklist de dominio

Al terminar esta clase debes poder explicar sin consultar documentación:

- [ ] Qué es una clase.
- [ ] Qué es una instancia.
- [ ] Qué diferencia existe entre clase y objeto.
- [ ] Qué es un atributo de instancia.
- [ ] Qué es un método.
- [ ] Qué función cumple `__init__`.
- [ ] Qué representa `self`.
- [ ] Cómo crear una instancia.
- [ ] Cómo acceder a un atributo.
- [ ] Cómo almacenar objetos en una lista.
- [ ] Cómo recorrer objetos con `for`.
- [ ] Cómo utilizar un atributo booleano.
- [ ] Por qué un entorno virtual es importante.
- [ ] Qué problema resuelve un gestor de dependencias.
- [ ] Por qué OOP no debe utilizarse indiscriminadamente.
- [ ] Cómo conectar un modelo de dominio con una arquitectura backend.

---

- [x] **Clase:** modelo que define estructura y comportamiento.
- [x] **Instancia:** objeto concreto creado a partir de una clase.
- [x] **Clase vs. objeto:** la clase define el modelo; el objeto representa una entidad concreta.
- [x] **Atributo de instancia:** dato asociado a un objeto mediante `self`.
- [x] **Método:** función definida dentro de una clase que representa comportamiento.
- [x] **`__init__`:** inicializa el estado de una instancia cuando se crea.
- [x] **`self`:** referencia a la instancia actual.
- [x] **Crear una instancia:** `libro = Libro("Título", "Autor")`.
- [x] **Acceder a un atributo:** `libro.titulo`.
- [x] **Almacenar objetos en una lista:** `catalogo = [libro1, libro2]`.
- [x] **Recorrer objetos:** `for libro in catalogo:`.
- [x] **Booleano:** representa un estado lógico mediante `True` o `False`.
- [x] **Entorno virtual:** aísla las dependencias de cada proyecto.
- [x] **Gestor de dependencias:** controla instalación, versiones y reproducibilidad de las librerías.
- [x] **OOP no debe usarse indiscriminadamente:** una clase debe aportar valor; para lógica simple y sin estado, una función puede ser mejor.
- [x] **Modelo de dominio + backend:** el modelo representa las entidades y reglas del dominio; servicios, repositorios y APIs lo integran con la infraestructura.

# 36. Ejercicio final

Implementa una versión ampliada de `Libro`.

Requisitos:

1. Crear la clase `Libro`.
2. Incorporar:
   - `titulo`;
   - `autor`;
   - `isbn`;
   - `disponible`.
3. Crear al menos cinco objetos.
4. Guardarlos en `catalogo`.
5. Recorrer `catalogo`.
6. Mostrar título, autor, ISBN y disponibilidad.
7. Utilizar correctamente valores booleanos.
8. Ejecutar el proyecto dentro del entorno gestionado por `uv`.
9. Mantener el código legible y correctamente estructurado.

Una solución mínima esperada sería:

    class Libro:

        def __init__(self, titulo, autor, isbn, disponible):
            self.titulo = titulo
            self.autor = autor
            self.isbn = isbn
            self.disponible = disponible


    catalogo = [
        Libro(
            "Cien años de soledad",
            "Gabriel García Márquez",
            "978-0307474728",
            True
        ),
        Libro(
            "El principito",
            "Antoine de Saint-Exupéry",
            "978-0156012195",
            False
        )
    ]


    for libro in catalogo:
        print(
            f"Título: {libro.titulo} | "
            f"Autor: {libro.autor} | "
            f"ISBN: {libro.isbn} | "
            f"Disponible: {libro.disponible}"
        )

La solución no debe considerarse definitiva para producción. El siguiente nivel consiste en encapsular las reglas de negocio y evitar que cualquier componente modifique libremente el estado del libro.

---

# 37. Conexión con las siguientes clases

Esta clase establece los fundamentos necesarios para avanzar hacia:

    Clases
       |
       v
    Objetos
       |
       v
    Atributos y métodos
       |
       v
    Encapsulación
       |
       v
    Herencia
       |
       v
    Polimorfismo
       |
       v
    Composición
       |
       v
    Protocolos
       |
       v
    Clases abstractas
       |
       v
    Diseño de software
       |
       v
    Aplicaciones reales
       |
       v
    AI Engineering

La prioridad es comprender el modelo mental antes de incorporar mecanismos más avanzados.

---

# 38. Resultado esperado

Al finalizar esta clase, no basta con poder escribir:

    class Libro:

        def __init__(self, titulo, autor):
            self.titulo = titulo
            self.autor = autor

El verdadero resultado esperado es poder razonar:

    Problema del dominio
           |
           v
    Identificar entidad
           |
           v
    Definir responsabilidad
           |
           v
    Diseñar clase
           |
           v
    Crear instancias
           |
           v
    Gestionar estado
           |
           v
    Definir comportamiento
           |
           v
    Integrar con el sistema

Ese cambio de perspectiva —de escribir sintaxis a diseñar software— es el objetivo principal de esta etapa de OOP.