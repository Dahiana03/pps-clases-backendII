# ETAPA 10 - Migración a Spring Boot + Spring Data JPA

## Objetivo
Convertir la base actual de `etapa09` en una aplicación **Spring real**, reemplazando la persistencia en memoria por **Spring Boot + Spring Data JPA**, manteniendo:

- entidades JPA con relaciones reales
- DTOs y mappers existentes
- services como capa de negocio
- flujo principal de carrito y checkout

En esta etapa **todavía no se exponen endpoints**. Eso queda para `etapa11`.

---

## Cambios Realizados

### 1) Bootstrap Spring Boot real
Se agregó la clase de arranque:

- `src/main/java/co/edu/cesde/pps/PpsApplication.java`

Se agregó configuración centralizada:

- `src/main/resources/application.yml`

Con esto el proyecto ya puede levantar como aplicación Spring Boot y gestionar beans automáticamente.

---

### 2) `pom.xml` migrado a Spring Boot estándar
Se reorganizó `pom.xml` para usar:

- `spring-boot-starter-parent`
- `spring-boot-starter-data-jpa`
- `spring-boot-starter-web`
- `spring-boot-starter-validation`
- `spring-boot-starter-test`
- `mysql-connector-j`
- `lombok`
- `h2` en pruebas
- `spring-boot-maven-plugin`

**Resultado:** se dejó atrás la gestión manual de dependencias JPA/Hibernate para pasar al modelo estándar de Spring Boot.

---

### 3) Repositorios Spring Data JPA creados
Se creó el package:

- `src/main/java/co/edu/cesde/pps/repository`

Repositorios agregados:

1. `UserRepository`
2. `RoleRepository`
3. `AddressRepository`
4. `CategoryRepository`
5. `ProductRepository`
6. `CartRepository`
7. `OrderRepository`
8. `OrderStatusRepository`
9. `UserSessionRepository`

### Queries derivadas principales
- `findByEmailIgnoreCase(...)`
- `existsByEmailIgnoreCase(...)`
- `findBySlugIgnoreCase(...)`
- `findBySkuIgnoreCase(...)`
- `findByUser_UserIdAndStatus(...)`
- `findByOrderNumberIgnoreCase(...)`
- `findByOrderStatus_OrderStatusId(...)`
- `findByCreatedAtBetween(...)`

---

### 4) Services migrados de memoria a JPA real
Se refactorizaron los services para usar:

- `@Service`
- `@Transactional`
- inyección por constructor
- repositorios JPA en lugar de listas `inMemory`
- IDs autogenerados por la base de datos

#### Archivos migrados
- `src/main/java/co/edu/cesde/pps/service/UserService.java`
- `src/main/java/co/edu/cesde/pps/service/AddressService.java`
- `src/main/java/co/edu/cesde/pps/service/CategoryService.java`
- `src/main/java/co/edu/cesde/pps/service/ProductService.java`
- `src/main/java/co/edu/cesde/pps/service/CartService.java`
- `src/main/java/co/edu/cesde/pps/service/OrderService.java`

#### Cambios clave por servicio

##### `UserService`
- ya no crea `Role` temporal en memoria
- ahora carga el rol `CUSTOMER` desde `RoleRepository`
- registro y consultas operan sobre `UserRepository`

##### `AddressService`
- validación de máximo de direcciones ahora consulta BD real
- manejo de dirección default ya no depende de lista en memoria
- altas, cambios y borrados persisten sobre `AddressRepository`

##### `CategoryService`
- CRUD y jerarquía migrados a `CategoryRepository`
- `slug` único ahora se valida sobre persistencia real
- se mantiene la consistencia bidireccional `parent/subcategories`

##### `ProductService`
- CRUD y búsquedas migrados a `ProductRepository`
- control de stock ya actualiza entidades persistidas
- cambio de categoría conserva consistencia bidireccional

##### `CartService`
- deja de usar `cartsInMemory`
- ahora persiste `Cart` y `CartItem` con JPA
- `mergeGuestCartToUserCart(...)` ya opera sobre entidades persistidas
- `createCartForGuest(...)` resuelve `UserSession` real cuando llega `sessionId`

##### `OrderService`
- `checkout(...)` ya usa `OrderRepository`
- el estado `PENDING` se obtiene desde `OrderStatusRepository`
- se construye `Order` con entidades reales:
  - `user`
  - `shippingAddress`
  - `billingAddress`
  - `orderStatus`
- `Order` y `OrderItem` ya se guardan como agregado persistido
- el número de orden verifica unicidad real en BD

---

### 5) Ajustes de agregados JPA
Se reforzó la persistencia de hijos en:

- `src/main/java/co/edu/cesde/pps/model/Cart.java`
- `src/main/java/co/edu/cesde/pps/model/Order.java`

Cambio aplicado:
- `cascade = CascadeType.ALL`
- `orphanRemoval = true`

**Razón:**
`CartItem` y `OrderItem` dependen del ciclo de vida de su padre (`Cart` / `Order`).

---

### 6) Pruebas de integración Spring Boot
Se agregaron:

- `src/test/resources/application-test.yml`
- `src/test/java/co/edu/cesde/pps/Etapa10SpringBootIntegrationTest.java`

### Flujo validado en test
- crear catálogos mínimos (`Role`, `OrderStatus`)
- crear categoría
- crear producto
- registrar usuario
- agregar direcciones
- crear carrito
- agregar item
- ejecutar checkout
- validar:
  - orden persistida
  - estado `PENDING`
  - carrito `CONVERTED`
  - disminución real de stock

---

## Checklist de Validación

### Infraestructura
- [x] Existe clase `PpsApplication`
- [x] Existe `application.yml`
- [x] `pom.xml` usa Spring Boot parent
- [x] El proyecto compila con Maven

### Persistencia
- [x] Ya no existen listas `inMemory` en services
- [x] Ya no existen `generateNextId()` en services
- [x] Los roles y estados de orden ya no se crean fake en servicios
- [x] `OrderService.checkout(...)` asigna entidades, no IDs sueltos
- [x] `Cart` persiste `CartItem` por cascada
- [x] `Order` persiste `OrderItem` por cascada

### Validación funcional mínima
- [x] El contexto Spring Boot levanta en test
- [x] El flujo `register -> address -> cart -> checkout` pasa en test
- [x] El stock se reduce al hacer checkout
- [x] El carrito cambia a `CONVERTED`

---

## Commits sugeridos

1. `build: migrate pom to spring boot parent and plugin`
2. `feat(boot): add PpsApplication and application.yml`
3. `feat(repository): add spring data jpa repositories`
4. `refactor(service): migrate UserService and AddressService to repositories`
5. `refactor(service): migrate CategoryService and ProductService to repositories`
6. `refactor(model): add cascade and orphan removal to cart and order aggregates`
7. `refactor(service): migrate CartService to Spring Data JPA`
8. `refactor(service): migrate OrderService checkout to persisted entities`
9. `test: add Spring Boot integration test for checkout flow`
10. `docs: add ETAPA10_SUMMARY and migration step-by-step guide`

---

## Validación Ejecutada

### Compilación
```bash
mvn -q -DskipTests compile
```

### Pruebas
```bash
mvn -q test
```

Resultado validado en esta implementación:
- **compila correctamente**
- **prueba de integración OK**

---

## Próximo Paso Recomendado

### ETAPA 11
Exponer la aplicación con API REST real:

- `@RestController`
- endpoints por módulo
- `@RestControllerAdvice`
- manejo HTTP de excepciones
- request/response DTOs
- control explícito de serialización/lazy loading
- definición de contratos de API

---
**Fecha:** 11 de marzo de 2026  
**Rama:** `etapa10`  
**Estado:** ✅ Implementada y validada

