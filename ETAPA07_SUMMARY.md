# ETAPA 07 - Refactorización de Modelos con Lombok

## Objetivo
Refactorizar todos los modelos de la capa `model` para usar anotaciones de Lombok, eliminando código boilerplate y mejorando la mantenibilidad del código.

## Cambios Realizados

### 1. Migración de Modelos a Lombok

Se refactorizaron **14 clases** del paquete `co.edu.cesde.pps.model` aplicando las siguientes anotaciones de Lombok:

- `@Getter` - Genera getters para todos los campos
- `@Setter` - Genera setters para todos los campos
- `@NoArgsConstructor` - Genera constructor sin argumentos
- `@AllArgsConstructor` - Genera constructor con todos los argumentos
- `@Builder` - Genera patrón Builder para construcción fluida de objetos

#### Clases Refactorizadas:

1. **Address.java** - Modelo de dirección
2. **Cart.java** - Modelo de carrito de compras
3. **CartItem.java** - Modelo de ítem del carrito
4. **Category.java** - Modelo de categoría de productos
5. **Order.java** - Modelo de orden de compra
6. **OrderItem.java** - Modelo de ítem de orden
7. **OrderStatus.java** - Modelo catálogo de estados de orden
8. **Payment.java** - Modelo de pago
9. **PaymentMethod.java** - Modelo catálogo de métodos de pago
10. **PaymentStatus.java** - Modelo catálogo de estados de pago
11. **Product.java** - Modelo de producto
12. **Role.java** - Modelo de rol de usuario
13. **User.java** - Modelo de usuario
14. **UserSession.java** - Modelo de sesión de usuario

### 2. Estrategias Aplicadas

#### 2.1 Setters Personalizados con Validación
Se mantuvieron los setters personalizados que contienen lógica de validación, sobrescribiendo los generados por Lombok:

```java
// Ejemplo en CartItem.java
public void setQuantity(Integer quantity) {
    ValidationUtils.validatePositive(quantity, "quantity");
    this.quantity = quantity;
}

public void setUnitPrice(BigDecimal unitPrice) {
    ValidationUtils.validateNonNegative(unitPrice, "unitPrice");
    this.unitPrice = unitPrice;
}
```

**Clases afectadas:**
- `CartItem` - validación de quantity y unitPrice
- `Product` - validación de price y stockQty
- `Order` - validación de subtotal, tax, shippingCost y total
- `OrderItem` - validación de quantity, unitPrice y lineTotal
- `Payment` - validación de amount

#### 2.2 ToString Personalizados
Se mantuvieron implementaciones personalizadas de `toString()` para evitar referencias circulares y mejorar el debugging:

```java
// Ejemplo en Cart.java
@Override
public String toString() {
    return "Cart{" +
            "cartId=" + cartId +
            ", userId=" + (user != null ? user.getUserId() : null) +
            ", sessionId=" + (session != null ? session.getSessionId() : null) +
            ", status=" + status +
            ", createdAt=" + createdAt +
            ", updatedAt=" + updatedAt +
            ", itemsCount=" + (items != null ? items.size() : 0) +
            ", total=" + calculateTotal() +
            '}';
}
```

**Estrategia:**
- Solo se muestran IDs de objetos relacionados, no los objetos completos
- Se incluyen conteos de colecciones en lugar de su contenido completo
- Se calculan valores útiles como totales

#### 2.3 Valores Por Defecto con @Builder.Default

Se aplicó `@Builder.Default` para mantener valores por defecto consistentes:

```java
// Ejemplo en Product.java
@Builder.Default
private Boolean isActive = true;

@Builder.Default
private LocalDateTime createdAt = LocalDateTime.now();

// Ejemplo en Order.java
@Builder.Default
private BigDecimal subtotal = BigDecimal.ZERO;

@Builder.Default
private List<OrderItem> items = new ArrayList<>();
```

**Campos con valores por defecto:**
- `Address.isDefault` → false
- `Cart.status` → CartStatus.OPEN
- `Cart.createdAt`, `Cart.updatedAt` → LocalDateTime.now()
- `Cart.items` → new ArrayList<>()
- `Category.subcategories`, `Category.products` → new ArrayList<>()
- `Order.subtotal`, `tax`, `shippingCost`, `total` → BigDecimal.ZERO
- `Order.createdAt` → LocalDateTime.now()
- `Order.items` → new ArrayList<>()
- `Product.isActive` → true
- `Product.createdAt` → LocalDateTime.now()
- `User.status` → UserStatus.ACTIVE
- `User.createdAt` → LocalDateTime.now()
- `User.addresses` → new ArrayList<>()
- `UserSession.createdAt` → LocalDateTime.now()

### 3. Actualización de Servicios

Se actualizaron los servicios para usar el patrón Builder en lugar de constructores tradicionales:

#### 3.1 UserService
```java
// Antes
User user = new User(defaultRole, email.toLowerCase().trim(), passwordHash,
                    firstName.trim(), lastName.trim());
user.setUserId(generateNextId());
user.setPhone(phone != null ? phone.trim() : null);
user.setStatus(UserStatus.ACTIVE);
user.setCreatedAt(LocalDateTime.now());

// Después
User user = User.builder()
        .userId(generateNextId())
        .role(defaultRole)
        .email(email.toLowerCase().trim())
        .passwordHash(passwordHash)
        .firstName(firstName.trim())
        .lastName(lastName.trim())
        .phone(phone != null ? phone.trim() : null)
        .status(UserStatus.ACTIVE)
        .createdAt(LocalDateTime.now())
        .build();
```

#### 3.2 OrderService
```java
// Antes
Order order = new Order(orderNumber, userId, 1L,
    shippingAddressId, billingAddressId);
order.setOrderId(generateNextId());

OrderItem orderItem = new OrderItem(order, cartItem.getProduct(),
    cartItem.getQuantity(), cartItem.getUnitPrice());
orderItem.setOrderItemId(generateNextOrderItemId());
orderItem.setLineTotal(CalculationUtils.calculateOrderItemLineTotal(...));

// Después
Order order = Order.builder()
        .orderId(generateNextId())
        .orderNumber(orderNumber)
        .userId(userId)
        .orderStatusId(1L)
        .shippingAddressId(shippingAddressId)
        .billingAddressId(billingAddressId)
        .subtotal(BigDecimal.ZERO)
        .tax(BigDecimal.ZERO)
        .shippingCost(BigDecimal.ZERO)
        .total(BigDecimal.ZERO)
        .createdAt(LocalDateTime.now())
        .build();

OrderItem orderItem = OrderItem.builder()
        .orderItemId(generateNextOrderItemId())
        .order(order)
        .product(cartItem.getProduct())
        .quantity(cartItem.getQuantity())
        .unitPrice(cartItem.getUnitPrice())
        .lineTotal(lineTotal)
        .build();
```

#### 3.3 CartService
```java
// Antes
CartItem newItem = new CartItem(cart, product, quantity, product.getPrice());
newItem.setCartItemId(generateNextCartItemId());
newItem.setAddedAt(LocalDateTime.now());

// Después
CartItem newItem = CartItem.builder()
        .cartItemId(generateNextCartItemId())
        .cart(cart)
        .product(product)
        .quantity(quantity)
        .unitPrice(product.getPrice())
        .addedAt(LocalDateTime.now())
        .build();
```

## Beneficios de la Refactorización

### 1. Reducción de Código Boilerplate
- **Eliminados:** ~1,500 líneas de código repetitivo (getters, setters, constructores)
- **Reducción promedio:** ~60-80 líneas por clase
- **Mantenimiento:** Menor superficie de código para errores

### 2. Mejora en Legibilidad
- Código más limpio y enfocado en lógica de negocio
- Anotaciones descriptivas que documentan la intención
- Patrón Builder mejora la lectura de construcción de objetos

### 3. Consistencia
- Todos los modelos siguen el mismo patrón
- Comportamiento uniforme de equals/hashCode basado en ID
- ToString personalizados evitan problemas de serialización

### 4. Facilidad de Mantenimiento
- Cambios en estructura de modelos requieren menos modificaciones
- Lombok genera automáticamente código actualizado en tiempo de compilación
- Menor probabilidad de errores humanos

## Commits Realizados

Se realizaron commits granulares por cada clase refactorizada:

1. `refactor: migrate Address to Lombok annotations`
2. `refactor: migrate Cart to Lombok annotations`
3. `refactor: migrate CartItem to Lombok annotations`
4. `refactor: migrate Category to Lombok annotations`
5. `refactor: migrate Product to Lombok annotations`
6. `refactor: migrate User to Lombok annotations`
7. `refactor: migrate UserSession to Lombok annotations`
8. `refactor: migrate Order to Lombok annotations`
9. `refactor: migrate OrderItem to Lombok annotations`
10. `refactor: migrate Payment to Lombok annotations`
11. `refactor: migrate Role to Lombok annotations`
12. `refactor: migrate OrderStatus to Lombok annotations`
13. `refactor: migrate PaymentMethod to Lombok annotations`
14. `refactor: migrate PaymentStatus to Lombok annotations`
15. `refactor: update services to use Builder pattern with Lombok models`
16. `fix: correct missing closing brace in mergeGuestCartToUserCart method`
17. `docs: add ETAPA07_SUMMARY with Lombok refactoring details`

## Validación

### Compilación Exitosa
```bash
mvn clean compile
# BUILD SUCCESS
```

### No se Requieren Cambios en:
- Mappers (usan getters/setters que Lombok genera)
- DTOs (sin cambios necesarios)
- Utilidades (sin cambios necesarios)
- Tests (sin cambios necesarios)

## Configuración de Lombok

### Dependencia en pom.xml
```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.30</version>
    <scope>provided</scope>
</dependency>
```

### Plugin de IDE
Se requiere el plugin de Lombok en el IDE para reconocer las anotaciones y evitar errores de sintaxis falsos.

## Conclusiones

La refactorización de los modelos con Lombok en la Etapa 07 representa un paso importante hacia la modernización del código y mejora de la mantenibilidad del proyecto. El uso del patrón Builder en los servicios hace que la construcción de objetos sea más clara y menos propensa a errores.

### Próximos Pasos (Etapa 08)
- Migración a Spring Framework
- Implementación de JPA/Hibernate
- Configuración de Spring Data Repositories
- Integración de Spring Boot

---
**Fecha:** 18 de Febrero de 2026  
**Rama:** `etapa07`  
**Estado:** ✅ Completada

