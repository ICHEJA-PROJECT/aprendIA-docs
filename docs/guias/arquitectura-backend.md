# Documento Maestro: Guía del Equipo Backend

Este documento establece la arquitectura base y las convenciones para el desarrollo de microservicios en nuestro ecosistema. El objetivo es proporcionar una guía limpia, pragmática y agnóstica al lenguaje que garantice coherencia, mantenibilidad y escalabilidad en todos nuestros servicios. Nos enfocamos en una arquitectura RESTful inicial, con una visión clara hacia la futura integración de sistemas basados en eventos.

Adoptamos principios clave de Domain-Driven Design (DDD) y Screaming Architecture, organizando el código de manera intuitiva por funcionalidades (features) y capas.

---

## 1. Principios Arquitectónicos Clave

- **Clean Architecture & DDD:** La lógica de negocio (dominio) es el corazón y es independiente de tecnologías externas (frameworks, bases de datos, sistemas de mensajería). El dominio "grita" sus capacidades.
- **Screaming Architecture:** La estructura del código debe reflejar el dominio del negocio, no las tecnologías o frameworks. Una mirada rápida a la estructura de carpetas debe indicar qué hace el sistema.
- **Separación de Preocupaciones (SoC):** Cada capa y componente tiene una única responsabilidad bien definida.
- **Independencia del Lenguaje:** Las ideas y patrones aquí descritos son aplicables en Java, TypeScript, Python y otros lenguajes orientados a objetos.
- **Mínimo Over-engineering:** Buscamos soluciones directas y efectivas, evitando complejidades innecesarias que no aporten valor inmediato.
- **Orientación a la Testeabilidad:** La separación de capas facilita las pruebas unitarias y de integración de cada componente de forma aislada.

---

## 2. Estructura General del Microservicio

Todo microservicio seguirá una estructura de carpetas que refleje sus features (funcionalidades de negocio) y capas internas.

```bash
.
├── src/                            # Carpeta raíz del código fuente
│   ├── app/                        # Código principal de la aplicación (Java/TypeScript/Python)
│   │   ├── <nombre-servicio>/      # Namespace o paquete raíz del microservicio
│   │   │   ├── application/        # Orquestación de dependencias (DI)
│   │   │   ├── core/               # Componentes fundamentales y transversales
│   │   │   ├── features/           # Módulos de funcionalidad (Screaming Architecture por feature)
│   │   │   │   ├── <feature-name>/ # Ej: users, products, orders
│   │   │   │   │   ├── controllers/    # Entrada HTTP (REST)
│   │   │   │   │   ├── services/       # Aplicación de casos de uso (orquesta dominio y data)
│   │   │   │   │   ├── domain/         # Lógica de negocio, entidades, CASOS DE USO, interfaces de repositorio
│   │   │   │   │   ├── data/           # Implementaciones de repositorios, DTOs, adaptadores
│   │   │   │   │   └── transports/     # Futura migración a eventos/colas
│   │   │   │   └── common/             # Componentes comunes entre features (si aplica)
│   │   │   └── utils/              # Funciones auxiliares/comunes
│   │   └── MainApplication.java    # Punto de entrada (o main.ts, main.py)
│   ├── resources/                  # Archivos de configuración, plantillas, etc. (Java)
│   │   └── application.properties.example / .env.example
│   └── test/                       # Código de pruebas
├── Dockerfile.dev                  # Dockerfile para entorno de desarrollo
├── Dockerfile.prod                 # Dockerfile para entorno de producción
├── README.md                       # Documentación del servicio
├── swagger.yaml / openapi.yaml     # Documentación Swagger/OpenAPI
└── pom.xml / package.json / requirements.txt # Dependencias y configuración del proyecto
```

---

## 3. Descripción de Capas Mínimas Necesarias

### 3.1. Capa `core`

Componentes fundamentales y transversales, independientes de la lógica de negocio. Es la base de la aplicación.

- **Propósito:** Proporcionar infraestructura básica y genérica que otras capas necesitan.

- **Contiene:**

    - **Configuraciones Globales:** Parámetros de aplicación, variables de entorno.
    - **Middlewares/Interceptors:** Lógica transversal para solicitudes/respuestas (ej. autenticación JWT, correlación de logs).
    - **Manejo de Excepciones:** Clases base de excepciones personalizadas para errores genéricos (`AppException`, `ValidationException`, `NotFoundException`).
    - **Logging:** Configuración centralizada del sistema de logging.

- **Dependencias:** No debe depender de ninguna otra capa específica de negocio (`features`). Puede ser utilizada por cualquiera.

---

### 3.2. Capa `utils`

Funciones auxiliares o comunes que no encajan en ninguna otra capa específica.

- **Propósito:** Contener utilidades genéricas y reutilizables.

- **Contiene:** Funciones de ayuda para fechas, cadenas, criptografía, etc.

- **Dependencias:** Debe ser lo más independiente posible.

---

### 3.3. Capa `application`

Punto central para la inyección de dependencias y la orquestación inicial de los servicios.

- **Propósito:** Configurar y proporcionar las instancias concretas de los servicios (`features/<feature>/services`) y sus dependencias (interfaces de repositorio de `features/<feature>/domain`) a la capa de entrada (`controllers`). Cumple con el Principio de Inversión de Dependencias (DIP) de SOLID.

- **Contiene:** Módulos de configuración de inyección de dependencias (ej. módulos de Spring, contenedores de inyección en TypeScript/Python).

**Dependencias:** Depende de las interfaces definidas en `features/<feature>/domain` y de las implementaciones de los servicios en `features/<feature>/services`.

---

### 3.4. Capa `features/<feature>`

Cada `feature` es un módulo de negocio independiente, encapsulando una funcionalidad completa.

#### 3.4.1. `features/<feature>/controllers`

Punto de entrada de las solicitudes HTTP (REST).

- **Propósito:** Recibir solicitudes externas, delegar la ejecución de casos de uso (a services) y formatear la respuesta.

- **Contiene:** Endpoints REST

- **Dependencias:** Depende de services y dtos. No debe contener lógica de negocio directa.

#### 3.4.2. `features/<feature>/services`

Aplicación de casos de uso de negocio

- **Propósito**: Orquestar el flujo de un caso de uso. Recibe la petición del Controller, invoca el Caso de Uso correspondiente de la capa de `domain`, y maneja la respuesta o las excepciones que este propague. Si es necesario, transformará el resultado del dominio a un DTO de respuesta.

- **Contiene**: Métodos que representan la orquestación de casos de uso de negocio (ej. `createUser`, `processOrder`).

- **Dependencias**: Depende de `features/<feature>/domain` (casos de uso, entidades, excepciones) y `features/<feature>/data` (para DTOs de respuesta).

#### 3.4.3. Capa `features/<feature>/domain`

El corazón de la lógica de negocio. Es completamente independiente de la tecnología.

- **Propósito:** Contener las reglas de negocio, entidades, objetos de valor y Casos de Uso puros, así como las interfaces para interactuar con datos.

- **Contiene:**

    - **Entidades**: Representan los conceptos clave del negocio con su comportamiento y estado (ej. `User`, `Product`).

    - **Objetos de Valor (Value Objects)**: Objetos inmutables que representan un valor descriptivo (ej. `EmailAddress`).

    - **Casos de Uso (Use Cases)**: Clases o funciones que encapsulan una operación de negocio específica y atómica (ej. `CreateUserUseCase`, `UpdateProductUseCase`). Operan sobre entidades y utilizan las interfaces de repositorio definidas en esta misma capa.

    - **Interfaces de Repositorio**: Contratos (interfaces) que definen cómo se accede y se persiste la información del dominio (ej. `UserRepository`).

    - **Excepciones de Dominio**: Clases para representar errores de negocio específicos (ej. `UserAlreadyExistsException`, `ProductNotFoundException`).

- **Dependencias:** No debe depender de ninguna otra capa de la `feature` (`data`, `services`, `controllers`). Puede depender de `core/errors` para fallos genéricos.

#### 3.4.4. Capa `features/<feature>/data`

Implementaciones de acceso a la base de datos y otros sistemas externos.

- **Propósito**: Implementar las interfaces de repositorio definidas en `features/<feature>/domain`. Se encarga de la interacción con la base de datos, caché, o llamadas a otros servicios externos, y del mapeo entre DTOs y entidades de dominio.

- **Contiene**:

    - **Implementaciones de Repositorios**: Clases concretas que implementan las interfaces de repositorio (ej. `PostgreUserRepository`, `RedisProductRepository`).

    - **DTOs: Data Transfer Objects (DTOs)** para la transferencia de datos con fuentes externas (APIs, BD). Se crearán manualmente con métodos de serialización/deserialización y mapeo.

    - **Adaptadores/Mapeadores**: Lógica para mapear entre los DTOs de esta capa y las Entidades de Dominio.

- **Dependencias**: Depende de `features/<feature>/domain` (entidades, interfaces de repositorio) y puede usar utilidades de `core` (ej. conexión a base de datos, configuración de Dio).

#### 3.4.5. transports

Mecanismos de comunicación futuros para eventos o colas.

- **Propósito:** Abstraer los detalles de la comunicación asíncrona. Esta capa se mantendrá vacía inicialmente para los microservicios RESTful puros, pero sirve como un placeholder para una futura migración a eventos/colas (ej. Kafka, RabbitMQ).

- **Contiene:** Interfaces o clases para publicar/suscribir eventos.

**Dependencias:** Debería depender solo de `features/<feature>/domain` (para publicar eventos de dominio) y de `core` (para la configuración del bus de mensajes).

---

## 4. Flujo de Comunicación Típico (RESTful)

1. **Solicitud HTTP Externa**: Llega al Controller (`features/<feature>/controllers`).

2. **Controller:**
    - Valida el **DTO de entrada** (`features/<feature>/data/dtos`).

    - Si la validación falla, responde inmediatamente con un código de estado ``400 Bad Request`` y un cuerpo de error detallado.

    - Invoca el método apropiado en la capa de **Services** (`features/<feature>/services`), pasándole DTOs de entrada o datos básicos.

3. **Service:**

    - Recibe la llamada del **Controller**.

    - Invoca el Caso de Uso de la capa de Dominio (`features/<feature>/domain/use_cases`).

    - Si un caso de uso propaga una Excepción de Dominio (ej. `UserNotFoundException`) o un `Failure` de `core` (ej. `ServerFailure`), el Service la maneja y la propaga adecuadamente (o la convierte si es necesario, aunque generalmente se relanza).

    - Transforma el resultado del caso de uso en un DTO de respuesta si es necesario para el **Controller**.

4. **Caso de Uso (Domain):**

    - Ejecuta la lógica de negocio pura.

    - Interactúa con **Entidades** del **Dominio** (`features/<feature>/domain/entities`) para aplicar reglas de negocio.

    - Utiliza las **interfaces de Repositorio** (`features/<feature>/domain/repositories`) para la persistencia o recuperación de datos.

5. **Repository (Implementación en Data):**

    - Traduce la solicitud de la interfaz de **Repositorio** (del Dominio) a operaciones de bajo nivel (ej. consultas SQL, llamadas a APIs externas).

    - Utiliza los **DTOs** de `features/<feature>/data/dtos` para el transporte y mapeo de la respuesta de la fuente de datos a **Entidades de Dominio** (usando adaptadores/mapeadores).

    - **Si ocurre un error técnico (ej. conexión a DB fallida, API externa responde 500), lanza un** `Failure` **de** `core`.

6. **Dominio**: Las Entidades y la lógica de negocio aplican transformaciones o validaciones internas.

7. **Retorno de Datos**: El resultado, a menudo una **Entidad de Dominio**, es devuelto al **Caso de Uso**, luego al **Service**, y finalmente al **Controller**.

8. **Controller**: Envía el DTO de respuesta como respuesta HTTP.

---

## 5. Componentes Obligatorios del Microservicio

Todo microservicio desplegado en nuestro ecosistema debe incluir los siguientes artefactos y configuraciones:

### 5.1. Dockerfile Funcional (`Dockerfile.dev, Dockerfile.prod`)

- **Propósito**: Estandarizar la construcción y el despliegue de los servicios.

- `Dockerfile.dev`: Optimizado para desarrollo (instalación de herramientas de depuración, hot-reloading, volúmenes montados).

- `Dockerfile.prod`: Optimizado para producción (imágenes ligeras, minimización de capas, ejecución de la aplicación compilada/bundle).

### 5.2. Archivo de Configuración de Entorno

- **Propósito**: Gestionar configuraciones sensibles o específicas del entorno (credenciales de DB, claves API, puertos).

- **Formato:**

    - `.env.example` (para TypeScript/Python).

    - `application.properties.example` (para Java).

- **Contenido**: Debe listar todas las variables de entorno esperadas con valores de ejemplo o predeterminados.

### 5.3. Documentación Swagger/OpenAPI

- **Propósito**: Describir las APIs REST del microservicio de manera estandarizada y legible por máquinas y humanos.

- **Herramienta**: Usar anotaciones en el código (Springdoc, Express-Swagger, FastAPI) o definir un archivo `swagger.yaml/openapi.yaml` manualmente si el lenguaje no ofrece buen soporte.

- **Integración**: Debería ser accesible a través de una URL `/docs` o similar en el entorno de desarrollo.

### 5.4. README.md Completo

- **Propósito**: Proporcionar una guía rápida y completa para cualquier desarrollador que se una al proyecto.

- **Contenido Mínimo:**

    - Descripción del microservicio y su dominio de negocio.

    - Prerrequisitos (versiones de lenguaje, Docker, etc.).

    - **Pasos para ejecutar el servicio localmente (modo desarrollo y producción)**.

    - Comandos de pruebas.

    - Instrucciones para generar la documentación Swagger/OpenAPI.

### 5.5. Punto de Entrada Bien Definido

- **Propósito**: Clara demarcación del inicio de la aplicación.

- **Formato:**

    - `MainApplication.java` (Java)

    - `main.ts` (TypeScript)

    - `main.py` (Python)

- **Responsabilidad**: Inicializar el contexto de la aplicación, cargar configuraciones y arrancar el servidor.

### 5.6. Control Centralizado de Errores y Status HTTP

- **Propósito**: Gestionar y estandarizar la forma en que los errores son capturados, procesados y respondidos a los clientes HTTP.

- **Implementación**: Utilizar interceptores, filtros o aspectos globales (ej. `@ControllerAdvice` en Spring, `@Catch` en NestJS, `ExceptionHandlers` en FastAPI).

- **Mapeo de Excepciones a Status HTTP:**

    - **Errores de Negocio (Excepciones de Dominio)**: Deben mapearse a códigos de estado HTTP semánticamente correctos, reflejando el problema de negocio.

        - `400 Bad Request`: Para errores de validación de entrada o reglas de negocio que no pueden cumplirse debido a datos incorrectos (ej. `InvalidCredentialsException`, `UserAlreadyExistsException` al intentar registrar un usuario existente).

        - `401 Unauthorized`: Para fallos de autenticación (ej. token inválido o ausente).

        - `403 Forbidden`: Para fallos de autorización (ej. el usuario no tiene permisos para realizar la acción).

        - `404 Not Found`: Cuando un recurso solicitado no existe (ej. `ProductNotFoundException`).

        - `409 Conflict`: Cuando hay un conflicto con el estado actual del recurso (ej. intentar actualizar un recurso obsoleto).

        - `422 Unprocessable Entity`: Para errores de validación semántica o de reglas de negocio que impiden el procesamiento de la entidad (más específico que 400).

    - **Errores Técnicos** (`Failure` de `core`): Deben mapearse a códigos de estado HTTP que indiquen problemas del servidor o de la infraestructura.

        - `500 Internal Server Error`: Para errores no esperados en el servidor, fallos de lógica internos o excepciones no controladas.

        - `502 Bad Gateway`: Problemas de comunicación con un servicio ascendente.

        - `503 Service Unavailable`: El servicio no está disponible temporalmente.

        - `504 Gateway Timeout`: El servicio no recibió una respuesta a tiempo de un servicio ascendente.

    - **Cuerpo de la Respuesta de Error**: La respuesta debe incluir un cuerpo JSON estandarizado con al menos:

        - `code`: Un código de error interno (opcional, pero útil para depuración).

        - `message`: Una descripción legible para el cliente.

        - `details`: (Opcional) Un array o mapa con detalles adicionales sobre el error (ej. errores de validación de campos específicos).

    - **Logging**: Todo error manejado debe ser registrado adecuadamente

### 5.7. Sistema de Logging Estructurado

- **Propósito**: Facilitar la depuración, monitoreo y análisis de la operación del servicio.

- **Características**:

    - **Logging Estructurado (JSON)**: Preferir formatos JSON para facilitar el análisis por herramientas externas.

    - **Niveles de Log**: Usar niveles apropiados (DEBUG, INFO, WARN, ERROR).

    - **Contexto**: Incluir ID de usuario, feature involucrada..

---

## 6. Directrices Adicionales

- **Inmutabilidad:** Preferir la inmutabilidad para Entidades y DTOs.
- **Validación de Entrada:** Validación robusta de todos los DTOs de entrada.
- **Nomenclatura Consistente**: Seguir convenciones de nombres claras para clases, métodos y variables (ej. `camelCase` para métodos, `PascalCase` para clases).

- **Pruebas Unitarias e Integración**: Fomentar una cobertura de pruebas en todas las capas, especialmente en `domain` y `services`.
