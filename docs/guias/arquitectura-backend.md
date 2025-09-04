# Guía de Arquitectura Base para Microservicios

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

### 3.1. Capa core

Componentes fundamentales y transversales, independientes de la lógica de negocio. Es la base de la aplicación.

**Propósito:** Proporcionar infraestructura básica y genérica que otras capas necesitan.

**Contiene:**

- Configuraciones Globales
- Middlewares/Interceptors
- Manejo de Excepciones
- Logging

**Dependencias:** No debe depender de ninguna otra capa específica de negocio (features).

---

### 3.2. Capa utils

Funciones auxiliares o comunes que no encajan en ninguna otra capa específica.

**Propósito:** Contener utilidades genéricas y reutilizables.

**Contiene:** Funciones de ayuda para fechas, cadenas, criptografía, etc.

**Dependencias:** Debe ser lo más independiente posible.

---

### 3.3. Capa application

Punto central para la inyección de dependencias y la orquestación inicial de los servicios.

**Propósito:** Configurar y proporcionar las instancias concretas de los servicios y sus dependencias a la capa de entrada (controllers).

**Contiene:** Módulos de configuración de inyección de dependencias.

**Dependencias:** Depende de las interfaces definidas en domain y de las implementaciones de los servicios.

---

### 3.4. Capa features/<feature>

Cada feature es un módulo de negocio independiente, encapsulando una funcionalidad completa.

#### 3.4.1. controllers

Punto de entrada de las solicitudes HTTP (REST).

**Propósito:** Recibir solicitudes externas, delegar la ejecución de casos de uso y formatear la respuesta.

**Dependencias:** Depende de services y dtos. No debe contener lógica de negocio directa.

#### 3.4.2. services

Aplicación de casos de uso de negocio.

**Propósito:** Orquestar el flujo de un caso de uso.

**Dependencias:** Depende de domain y data.

#### 3.4.3. domain

El corazón de la lógica de negocio. Es completamente independiente de la tecnología.

**Propósito:** Contener las reglas de negocio, entidades, objetos de valor y Casos de Uso puros, así como las interfaces para interactuar con datos.

**Contiene:**

- Entidades
- Objetos de Valor
- Casos de Uso
- Interfaces de Repositorio
- Excepciones de Dominio

**Dependencias:** No debe depender de ninguna otra capa de la feature.

#### 3.4.4. data

Implementaciones de acceso a la base de datos y otros sistemas externos.

**Propósito:** Implementar las interfaces de repositorio definidas en domain.

**Contiene:**

- Implementaciones de Repositorios
- DTOs
- Adaptadores/Mapeadores

**Dependencias:** Depende de domain y puede usar utilidades de core.

#### 3.4.5. transports

Mecanismos de comunicación futuros para eventos o colas.

**Propósito:** Abstraer los detalles de la comunicación asíncrona.

**Dependencias:** Debería depender solo de domain y de core.

---

## 4. Flujo de Comunicación Típico (RESTful)

1. **Solicitud HTTP Externa:** Llega al Controller.
2. **Controller:** Valida el DTO de entrada. Si falla, responde con 400. Invoca el método apropiado en Services.
3. **Service:** Invoca el Caso de Uso de Domain. Maneja excepciones y transforma el resultado en un DTO de respuesta.
4. **Caso de Uso (Domain):** Ejecuta la lógica de negocio pura.
5. **Repository (Data):** Traduce la solicitud a operaciones de bajo nivel y mapea la respuesta.
6. **Dominio:** Aplica transformaciones o validaciones internas.
7. **Retorno de Datos:** El resultado es devuelto al Caso de Uso, luego al Service, y finalmente al Controller.
8. **Controller:** Envía el DTO de respuesta como respuesta HTTP.

---

## 5. Componentes Obligatorios del Microservicio

### 5.1. Dockerfile Funcional

- **Dockerfile.dev:** Optimizado para desarrollo.
- **Dockerfile.prod:** Optimizado para producción.

### 5.2. Archivo de Configuración de Entorno

- **.env.example** (TypeScript/Python)
- **application.properties.example** (Java)

### 5.3. Documentación Swagger/OpenAPI

- Archivo `swagger.yaml` o `openapi.yaml`.
- Accesible vía `/docs` en desarrollo.

### 5.4. README.md Completo

- Descripción del microservicio y su dominio.
- Prerrequisitos.
- Pasos para ejecutar el servicio.
- Comandos de pruebas.
- Instrucciones para documentación Swagger/OpenAPI.

### 5.5. Punto de Entrada Bien Definido

- **MainApplication.java** (Java)
- **main.ts** (TypeScript)
- **main.py** (Python)

### 5.6. Control Centralizado de Errores y Status HTTP

- Uso de interceptores, filtros o aspectos globales.
- Mapeo de excepciones a códigos HTTP semánticos.
- Cuerpo de respuesta de error estandarizado.
- Logging adecuado de errores.

### 5.7. Sistema de Logging Estructurado

- Preferir formatos JSON.
- Usar niveles apropiados (DEBUG, INFO, WARN, ERROR).
- Incluir contexto relevante.

---

## 6. Directrices Adicionales

- **Inmutabilidad:** Preferir la inmutabilidad para Entidades y DTOs.
- **Validación de Entrada:** Validación robusta de todos los DTOs de entrada.
- **Nomenclatura Consistente:** Seguir convenciones claras.
- **Pruebas Unitarias e Integración:** Fomentar cobertura en todas las capas, especialmente en domain y services.
