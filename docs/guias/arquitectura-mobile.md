# Guía de Arquitectura para Frontend Mobile (Flutter)

Este documento es una guía técnica esencial para desarrolladores que se unen o colaboran en nuestro proyecto frontend en Flutter. Detalla la arquitectura, las convenciones de código y las buenas prácticas que seguimos para asegurar una integración fluida y un desarrollo coherente, mantenible y escalable.

Nuestra arquitectura se basa en una combinación estricta de Clean Architecture y Domain-Driven Design (DDD), con una organización modular que sigue los principios de Screaming Architecture dentro de las funcionalidades (features).

---

## 1. Estructura del Proyecto y Capas

El proyecto está organizado en capas de alto nivel y, dentro de estas, en módulos de funcionalidad. Las dependencias siempre fluyen hacia el centro, manteniendo el desacoplamiento y cumpliendo con el Principio de Inversión de Dependencias (DIP) de SOLID.

```bash
.
├── lib
│   ├── core                    # Componentes fundamentales y transversales
│   ├── shared                  # Widgets y utilidades reutilizables
│   ├── features                # Módulos de funcionalidad (Screaming Architecture por feature)
│   │   ├── auth                # Ejemplo de Feature: Autenticación
│   │   │   ├── application     # Inyección de dependencias para esta feature
│   │   │   ├── domain          # Lógica de negocio, Casos de Uso, Entidades, Repositorios (Interfaces)
│   │   │   ├── data            # Implementaciones de repositorios (fuentes de datos), DTOs, Adaptadores
│   │   │   └── presentation    # Vistas, Widgets y View Models
│   │   └── ... otros features
│   └── main.dart               # Punto de entrada de la aplicación
├── pubspec.yaml
├── analysis_options.yaml
└── ...
```

---

### 1.1. Capa core (`/lib/core`)

Contiene componentes fundamentales y transversales, independientes de la lógica de negocio y de la UI.

- **Propósito:** Proporcionar la infraestructura básica y genérica que otras capas necesitan.
- **Contiene:**
    - `/config`: Configuraciones de entorno, constantes globales.
    - `/errors`: Clases base para `Failures` y excepciones a nivel de aplicación (ej. `ServerFailure`, `NetworkFailure`).
    - `/network`: Configuración global de Dio  (`dio_config.dart`), interceptores genéricos.
    - `/utils`: Utilidades genéricas (ej. `logger.dart`).

- **Convenciones:**
    - No debe depender de ninguna otra capa de negocio (`features`, `shared`). Puede ser utilizada por cualquiera.

---

### 1.2. Capa shared (`/lib/shared`)

Componentes reutilizables comunes a través de múltiples `features`, más cercanos a la presentación o utilidades de UI.

- **Propósito:** Centralizar código común para evitar duplicación y promover la reutilización de componentes UI.

- **Contiene:**
    - `/widgets`: Widgets personalizados reutilizables (ej. `CustomButton`, `LoadingIndicator`).
    - `/styles`: Definiciones de temas, colores, fuentes.
    - `/helpers`: Funciones de ayuda generales (ej. `ValidationHelper`).
    - `/constants`: Constantes relacionadas con la UI o valores compartidos.

- **Convenciones:**
    - Puede depender de `core`. No debe depender de ninguna `feature`.

---

### 1.3. Capa features (`/lib/features`)

Esta es la capa principal donde reside la lógica de negocio modularizada. Cada `feature` (ej. `/lib/features/auth`) es una funcionalidad independiente que sigue la Clean Architecture internamente.

- **Propósito:** Encapsular funcionalidades completas del negocio, permitiendo un desarrollo, prueba y mantenimiento aislados.

- **Contiene**: Cada `feature` tiene sus propias subcarpetas:
    - `/application`:
        - **Propósito**: Módulo de inyección de dependencias y orquestación de alto nivel para la `feature`. Proporciona las instancias concretas de casos de uso y sus dependencias a la capa de `presentation`.
        - **Contiene**: Archivos de configuración de inyección (ej. `auth_injector.dart`).

    - `/domain`:
        - **Propósito**: El corazón de la `feature`. Contiene la lógica de negocio pura, incluyendo los Casos de Uso.
        - **Contiene**: `/entities`, `/value_objects`, `/repositories` (Interfaces), `/use_cases`, `/exceptions` (de negocio).
        - **Convenciones**: No debe depender de ninguna otra capa de la feature (data, presentation). Puede depender de core/errors.

    - `/data`:
        - **Propósito**: Implementa los contratos (interfaces) del Dominio, manejando la entrada y salida de datos (APIs, bases de datos locales).
        - **Contiene**: `/repositories` (Implementaciones concretas), `/data_sources` o `/api_clients` (usan Dio), `/dtos`, `/adapters` o `/mappers`.
        - **Convenciones**: Solo debe depender de la capa `domain` de su `feature` y de la capa `core`.

    - `/presentation`:
        - **Propósito**: Contiene la interfaz de usuario.
        - **Contiene**: `/pages`, `/widgets` (específicos de la feature), `/view_models` (`ChangeNotifier` de Provider).
        - **Convenciones**: Solo debe depender de la capa `application` de su `feature` y de la capa `shared`.

- **Convenciones Generales** de `features`:
    - Cada `feature` es un módulo autónomo. Las dependencias internas siguen: `Presentation` -> `Application` -> `Domain` <- `Data`.
    - Puede depender de `core` y `shared`. No debe depender directamente de otras `features`.

---

## 2. Flujo de Comunicación y Gestión de Dependencias

### 2.1. Flujo de Datos (Una Vía)

1. **Vista (Feature/Presentation):** Evento de UI.
2. **View Model (Feature/Presentation):** Invoca el Caso de Uso de `feature/domain/use_cases`.
3. **Caso de Uso (Feature/Domain):** Ejecuta lógica de negocio, utiliza interfaces de Repositorio de `feature/domain/repositories`.
4. **Repositorio (Feature/Domain - Interfaz):** Llama a un método definido en la interfaz del `Repository`.
5. **Implementación del Repositorio (Feature/Data):** Realiza la operación real (ej. llamada a API con Dio), maneja **DTOs** de `feature/data/dtos` y los mapea a las Entidades de Dominio.
6. **Dominio (Feature/Domain):** Entidades aplican reglas de negocio puras.
7. **Retorno:** El resultado se propaga al ViewModel.
8. **View Model:** Actualiza su estado interno y notifica a sus oyentes (`notifyListeners()`).
9. **Vista:** Se reconstruye para reflejar el nuevo estado.

### 2.2. inyección de Dependencias (Capa de Aplicación)

La capa de **Aplicación** es clave para la **Inversión de Dependencias (DIP)**, configurando y proporcionando las dependencias concretas.

- **Configuración centralizada en** `main.dart` **o** `app_router.dart`:

```dart
MultiProvider(
  providers: [
    // Dependencias Core/Globales
    Provider<Dio>(create: (_) => DioConfig.createDio()),

    // Feature: Auth Dependencies (Data -> Domain -> Presentation)
    Provider<AuthApiClient>(create: (context) => AuthApiClient(context.read<Dio>())),
    Provider<UserRepository>(create: (context) => UserRepositoryImpl(context.read<AuthApiClient>())),
    Provider<SignInUserUseCase>(create: (context) => SignInUserUseCase(context.read<UserRepository>())),
    ChangeNotifierProvider<AuthViewModel>(create: (context) => AuthViewModel(context.read<SignInUserUseCase>())),
  ],
  child: MyApp(),
);
```

- **Inyección en Constructores**: Todas las clases reciben sus dependencias a través de sus constructores.

```dart
// Ejemplo: Caso de Uso recibiendo un Repositorio (Interfaz)
class SignInUserUseCase {
  final UserRepository _userRepository;
  SignInUserUseCase(this._userRepository);
  // ...
}
```

## 3. Manejo de Datos: DTOs y Adaptadores en la Capa `data`

### 3.1. Data Transfer Objects (DTOs)

Los **DTOs** son clases simples e inmutables, para transferir datos entre las capas. Se ubican en `/lib/features/[feature]/data/dtos.`

- **Implementación**: Se crearán **manualmente**, incluyendo métodos como `fromJson` y `toJson` para la serialización/deserialización, o métodos de mapeo directo si la comunicación no es JSON.

```dart
// Ejemplo: AuthRequestDto
class AuthRequestDto {
  final String email;
  final String password;
  AuthRequestDto({required this.email, required this.password});
  Map<String, dynamic> toJson() => {'email': email, 'password': password};
}

// Ejemplo: AuthResponseDto
class AuthResponseDto {
  final String token;
  final String userId;
  AuthResponseDto({required this.token, required this.userId});
  factory AuthResponseDto.fromJson(Map<String, dynamic> json) {
    return AuthResponseDto(token: json['token'], userId: json['user_id']);
  }
}
```

### 3.2. Adaptadores (Mapeadores) en la capa `data`

Transforman datos entre DTOs y entidades de dominio.

- **Propósito**: Mapear los DTOs de `data` (ej. `AuthResponseDto`) a las Entidades de Dominio (ej. `User`) y viceversa.

- **Ubicación**: Como métodos dentro de los DTOs, como extensiones, o en clases `Mapper` dedicadas en `/lib/features/[feature]/data/mappers`.

```dart
// Ejemplo: Método de mapeo en AuthResponseDto
class AuthResponseDto {
  // ... propiedades y fromJson
  User toDomainEntity() {
    return User(id: userId, token: token); // Mapeo a Entidad de Dominio
  }
}

// Uso en la Implementación del Repositorio:
class UserRepositoryImpl implements UserRepository {
  // ...
  Future<User> signIn(String email, String password) async {
    final requestDto = AuthRequestDto(email: email, password: password);
    final responseDto = await _authApiClient.signIn(requestDto);
    return responseDto.toDomainEntity(); // Usa el método de mapeo
  }
}
```

## 4. Comunicación con APIs: Dio

**Dio** es nuestro cliente HTTP principal, configurado globalmente en la capa `core` y utilizado por los `ApiClients` en la capa `data`.

### 4.1. Configuración Global de Dio

- La instancia de Dio se crea y configura en `lib/core/network/dio_config.dart`.

- Incluye `BaseOptions`, `LogInterceptor`, y `InterceptorsWrapper` para lógica transversal (autenticación, manejo de errores HTTP genéricos).

```dart
// lib/core/network/dio_config.dart
class DioConfig {
  static Dio createDio() {
    final dio = Dio(BaseOptions( /* ... */ ));
    dio.interceptors.add(LogInterceptor( /* ... */ ));
    dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) async { /* Añadir tokens */ return handler.next(options); },
      onError: (e, handler) async { /* Manejo 401, etc. */ return handler.next(e); },
    ));
    return dio;
  }
}
```

### 4.2. Clientes de API (en la capa `data`)

- Clases dedicadas en `/lib/features/[feature]/data/data_sources` que exponen métodos para cada endpoint de la API.

- Reciben la instancia de `Dio` global.

- Manejan la llamada HTTP y devuelven los DTOs de `data`.

```dart
// lib/features/auth/data/data_sources/auth_api_client.dart
class AuthApiClient {
  final Dio _dio;
  AuthApiClient(this._dio);
  Future<AuthResponseDto> signIn(AuthRequestDto requestDto) async {
    try {
      final response = await _dio.post('/auth/login', data: requestDto.toJson());
      return AuthResponseDto.fromJson(response.data);
    } on DioException {
      rethrow; // Relanzar para que el Repositorio la capture
    }
  }
}
```

## 5. Manejo de errores

Un manejo de errores consistente a través de las capas es fundamental.

### 5.1. Clases de Fallo Base (`core/errors`)

- Define una jerarquía de clases de `Failure` en `/lib/core/errors` para fallos técnicos o del sistema (ej. `ServerFailure`, `NetworkFailure`).

```dart
// lib/core/errors/failures.dart
abstract class Failure { /* ... */ }
class ServerFailure extends Failure { /* ... */ }
class NetworkFailure extends Failure { /* ... */ }
```

### 5.2. Excepciones de Dominio (`features/[feature]/domain/exceptions`)

- Se definen por cada `feature` para representar errores de negocio puros (ej. `InvalidCredentialsException`, `UserNotFoundException`).

```dart
// lib/features/auth/domain/exceptions/auth_exceptions.dart
abstract class AuthException implements Exception { /* ... */ }
class InvalidCredentialsException extends AuthException { /* ... */ }
```

### 5.3. Mapeo y Propagación de Errores

El mapeo de errores es un proceso de dos pasos crucial:

- Capa `data` (`features/[feature]/data/repositories`):

    - Captura `DioException` y otras excepciones de bajo nivel.

    - Mapea estas excepciones a las Excepciones de Dominio o a los `Failure` genéricos de `core`.

    - Luego, la excepción/fallo se relanza.

```dart
// Dentro de UserRepositoryImpl.signIn
} on DioException catch (e) {
  if (e.response?.statusCode == 400) throw InvalidCredentialsException('Credenciales inválidas.');
  else if (e.type == DioExceptionType.connectionTimeout) throw NetworkFailure();
  else throw ServerFailure(); // u otro Failure más genérico
} catch (e) {
  throw Failure(message: 'Error desconocido en datos: ${e.toString()}');
}
```

- Capa `domain` (`features/[feature]/domain/use_cases`):

    - Los casos de uso **simplemente relanzan** las excepciones de Dominio o los `Failures` que les llegan.

```dart
// Dentro de SignInUserUseCase.execute
} on AuthException {
  rethrow;
} on Failure {
  rethrow;
} catch (e) {
  throw Failure(message: 'Error inesperado en caso de uso: ${e.toString()}');
}
```

- Capa `presentation` (`features/[feature]/presentation/view_models`):
    - Los ViewModel capturan las excepciones de Dominio y Failures.

    - Traducen estos fallos en mensajes amigables para el usuario (_errorMessage) y actualizan el estado de la UI.

```dart
// Dentro de AuthViewModel.signIn
} on InvalidCredentialsException catch (e) { _errorMessage = e.message; }
on NetworkFailure catch (e) { _errorMessage = e.message; }
on ServerFailure catch (e) { _errorMessage = e.message; }
// ... otros catches
finally { _isLoading = false; notifyListeners(); }
```

### 5.4. Presentación de Errores en la UI
- La Vista observa el `_errorMessage` del **ViewModel** y lo muestra usando componentes de `shared/widgets` (ej. `ErrorDialog`, `SnackBar`).

- El **ViewModel** debe limpiar el error (`clearError()`) después de mostrarlo.

## 6. Convenciones generales de código
- **Nomenclatura**:

    - Archivos y directorios: `snake_case`.

    - Clases/Enums/Typedefs: `PascalCase`.

    - Métodos/Variables: `camelCase`.

    - Constantes: `SCREAMING_SNAKE_CASE` (globales estáticas), `camelCase` con const.

- **Comentarios y Documentación**: Documentar clases, métodos y propiedades públicas con comentarios DART (`///.`)

- **Inmutabilidad**: Preferir objetos inmutables, especialmente en las capas de Dominio y en los DTOs. Considerar **freezed** para clases inmutables y **copyWith**.