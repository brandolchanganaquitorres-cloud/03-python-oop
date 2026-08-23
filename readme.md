# Python Orientado a Objetos

Curso de Programación Orientada a Objetos (OOP) con Python, organizado en 17 clases y 5 módulos.

## Objetivo

Desarrollar fundamentos sólidos de diseño y programación orientada a objetos en Python, aplicables a proyectos reales de software y como base para avanzar posteriormente hacia AI Engineering.

## Estructura del curso

### Módulo 01 — Fundamentos de Clases y Objetos

#### Clase 01 — OOP in Python: Modeling Real-World Systems

- Programación Orientada a Objetos
- Modelado de sistemas
- Entidades
- Clases
- Objetos
- Estado
- Comportamiento
- Modelado de un sistema de biblioteca

#### Clase 02 — Classes and Objects in Python Explained

- Definición de clases
- Constructor `__init__`
- Atributos de instancia
- Creación de objetos
- Múltiples instancias
- Estado independiente

#### Clase 03 — Instance Methods That Make Python Objects Act

- Métodos de instancia
- `self`
- `__str__`
- Cambio de estado
- Métodos de dominio
- `return`
- `print`
- Préstamo y devolución de libros

#### Clase 04 — Private Attributes and Data Integrity in Python

- Encapsulación
- Atributos privados
- Protección del estado
- Acceso controlado
- Validación
- Integridad de datos
- Invariantes

### Módulo 02 — Encapsulación y Comportamiento de Objetos

#### Clase 05 — Herencia en Python para Evitar Duplicación de Código

- Herencia
- Clase base
- Clase derivada
- Reutilización
- Especialización
- Sobrescritura
- `super()`

#### Clase 06 — Simple Inheritance with Python Classes

- Clase padre
- Clase hija
- Herencia de atributos
- Herencia de métodos
- Extensión de comportamiento
- Sobrescritura
- `super()`

#### Clase 07 — Python Protocol for Flexible Polymorphism

- Polimorfismo
- Protocolos
- Duck typing
- Interfaces implícitas
- Contratos de comportamiento
- Flexibilidad

### Módulo 03 — Implementar Protocolos con Métodos Especiales

#### Clase 08 — Composition Over Inheritance in Python

- Composición
- Relaciones entre objetos
- Delegación
- Dependencias
- Composición frente a herencia
- Diseño flexible

#### Clase 09 — Organizing Python Projects into Modules

- Módulos
- Paquetes
- `__init__.py`
- Imports absolutos
- Imports relativos
- Separación de responsabilidades
- Organización del proyecto

#### Clase 10 — Custom Exceptions That Make Python Errors Clear

- Excepciones personalizadas
- `Exception`
- `raise`
- `try`
- `except`
- Errores de dominio
- Manejo de errores

#### Clase 11 — Sistema de Préstamos con Identificación de Usuarios en Python

- Identificación de usuarios
- Usuarios y libros
- Reglas de préstamo
- Relaciones entre objetos
- Estado del préstamo
- Flujo de negocio

### Módulo 04 — Relaciones entre Clases y Polimorfismo

#### Clase 12 — Book Search and Lending Flow in Python OOP

- Búsqueda de libros
- Catálogo
- Flujo de préstamo
- Integración entre métodos
- Identificación de libros
- Disponibilidad
- Reglas del dominio

#### Clase 13 — Clases Abstractas en Python con `abc` y `abstractmethod`

- Clases abstractas
- `ABC`
- `abstractmethod`
- Contratos
- Abstracción
- Polimorfismo

#### Clase 14 — Decorador `property` en Python para Atributos con Validación

- `property`
- Getter
- Setter
- Validación
- Control del estado
- Encapsulación
- Invariantes

#### Clase 15 — Decoradores `staticmethod` y `classmethod` en Python

- Métodos de instancia
- `staticmethod`
- `classmethod`
- Responsabilidades
- Métodos asociados a la clase
- Constructores alternativos

#### Clase 16 — Serialización de Objetos Python a JSON para Persistencia de Datos

- Serialización
- JSON
- Conversión de objetos
- Diccionarios
- Persistencia
- Deserialización
- Estructuras de datos

### Módulo 05 — Diseño Avanzado y Persistencia

#### Clase 17 — Loading JSON Data to Restore App State

- Lectura de JSON
- Deserialización
- Reconstrucción de objetos
- Restauración del estado
- Persistencia
- Continuidad de ejecución

## Flujo de aprendizaje

```text
Problema
    ↓
Modelado
    ↓
Clases
    ↓
Objetos
    ↓
Estado y comportamiento
    ↓
Encapsulación
    ↓
Herencia y polimorfismo
    ↓
Composición
    ↓
Módulos
    ↓
Excepciones
    ↓
Abstracción
    ↓
Validación
    ↓
Serialización
    ↓
Persistencia
    ↓
Restauración del estado
```

## Proyecto conductor

El curso utiliza un sistema de biblioteca para aplicar progresivamente los conceptos de OOP.

```text
Library
├── Book
├── User
├── Loan
└── Catalog
```

## Arquitectura conceptual

```text
Usuario
   ↓
Library
   ↓
Catalog
   ↓
Book
   ↓
Loan
   ↓
Persistencia JSON
```

## Estructura del repositorio

```text
03-python-oop/
├── README.md
├── pyproject.toml
├── uv.lock
├── .gitignore
│
├── module-01-fundamentos/
│   ├── 01-modelado-oop/
│   ├── 02-clases-objetos/
│   ├── 03-metodos-instancia/
│   └── 04-atributos-privados/
│
├── module-02-encapsulacion/
│   ├── 05-herencia/
│   ├── 06-simple-inheritance/
│   └── 07-polimorfismo-protocolos/
│
├── module-03-protocolos/
│   ├── 08-composicion/
│   ├── 09-modulos/
│   ├── 10-excepciones/
│   └── 11-sistema-prestamos/
│
├── module-04-relaciones-polimorfismo/
│   ├── 12-busqueda-prestamos/
│   ├── 13-clases-abstractas/
│   ├── 14-property/
│   ├── 15-staticmethod-classmethod/
│   └── 16-serializacion-json/
│
└── module-05-diseno-avanzado/
    └── 17-restauracion-json/
```

## Estrategia Git

Se recomienda utilizar `main` como rama estable y una rama de trabajo por módulo.

```text
main
├── module/01-fundamentos
├── module/02-encapsulacion
├── module/03-protocolos
├── module/04-relaciones-polimorfismo
└── module/05-diseno-avanzado
```

## Principios de ingeniería

Durante el curso se desarrollan criterios de:

- Separación de responsabilidades
- Encapsulación
- Bajo acoplamiento
- Alta cohesión
- Composición
- Polimorfismo
- Modularidad
- Manejo explícito de errores
- Código mantenible
- Código reutilizable
- Código testeable

## Relación con AI Engineering

Los fundamentos de OOP proporcionan una base para trabajar posteriormente con arquitecturas de software más complejas.

```text
Python
   ↓
OOP
   ↓
Software Engineering
   ↓
APIs
   ↓
FastAPI
   ↓
SDKs
   ↓
LLM APIs
   ↓
RAG
   ↓
AI Agents
   ↓
LangChain / LangGraph
   ↓
MCP
   ↓
MLOps / LLMOps
```

En aplicaciones de IA, componentes como clientes API, modelos, herramientas, agentes, memoria, retrievers y servicios pueden organizarse mediante clases, composición, interfaces y módulos.

## Buenas prácticas

- Mantener cada clase con responsabilidades claras.
- Evitar clases excesivamente grandes.
- Evitar jerarquías de herencia innecesariamente profundas.
- Preferir composición cuando el dominio represente una relación `has-a`.
- Utilizar herencia cuando exista una relación `is-a` clara.
- Crear excepciones específicas para errores de dominio.
- Evitar capturar `Exception` indiscriminadamente.
- Separar lógica de negocio de persistencia.
- Mantener los módulos organizados por responsabilidad.
- Utilizar nombres claros y consistentes.
- Mantener el código fácil de probar y modificar.

## Persistencia

El flujo de persistencia estudiado es:

```text
Objeto Python
    ↓
Diccionario
    ↓
JSON
    ↓
Archivo
```

El proceso inverso permite restaurar el estado:

```text
Archivo JSON
    ↓
Diccionario
    ↓
Objeto Python
    ↓
Estado restaurado
```

## Resultado esperado

Al finalizar las 17 clases, el estudiante debe poder:

- Modelar entidades mediante clases.
- Crear y utilizar objetos.
- Implementar métodos de instancia.
- Controlar el estado interno.
- Aplicar encapsulación.
- Utilizar herencia correctamente.
- Implementar polimorfismo.
- Utilizar protocolos.
- Aplicar composición.
- Organizar proyectos mediante módulos y paquetes.
- Crear excepciones personalizadas.
- Utilizar clases abstractas.
- Implementar `property`.
- Diferenciar `staticmethod` y `classmethod`.
- Serializar objetos.
- Deserializar información.
- Persistir y restaurar el estado de una aplicación.

## Objetivo profesional

El objetivo no es solamente aprender a escribir clases.

El objetivo es desarrollar criterio para diseñar software.

```text
¿Qué responsabilidad pertenece a esta clase?
        ↓
¿Qué objeto debe controlar este estado?
        ↓
¿Necesito herencia o composición?
        ↓
¿Qué contrato debe cumplir este componente?
        ↓
¿Cómo manejo los errores?
        ↓
¿Cómo separo dominio e infraestructura?
        ↓
¿Cómo puedo probar este componente?
        ↓
¿Cómo puedo modificar una implementación sin romper el sistema?
        ↓
¿Cómo puedo persistir y restaurar el estado?
```

## Proyección hacia AI Engineering

```text
Python
   ↓
Python OOP
   ↓
Software Engineering
   ↓
Git / GitHub
   ↓
APIs
   ↓
FastAPI
   ↓
Docker
   ↓
Cloud
   ↓
LLM APIs
   ↓
RAG
   ↓
Embeddings
   ↓
Vector Databases
   ↓
AI Agents
   ↓
LangChain
   ↓
LangGraph
   ↓
MCP
   ↓
MLOps
   ↓
LLMOps
   ↓
CI/CD
```

## Regla principal

> No estudiar OOP para escribir clases. Estudiar OOP para aprender a diseñar software.