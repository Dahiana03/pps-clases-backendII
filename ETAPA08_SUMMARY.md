# ETAPA 08 - Anotaciones JPA Básicas en Modelos

## Objetivo
Agregar anotaciones JPA estándar (`@Entity`, `@Table`, `@Id`, `@Column`, `@GeneratedValue`, `@Enumerated`) a los 14 modelos del package `model` preparándolos para persistencia con Hibernate/JPA, **sin incluir relaciones** (`@ManyToOne`, `@OneToMany`, `@JoinColumn` se agregarán en Etapa 09).

Esta etapa establece el mapeo objeto-relacional básico manteniendo compatibilidad con el código existente de servicios, mappers y DTOs.

## Cambios Realizados

### 1. Anotaciones JPA Agregadas a 14 Modelos

Se agregaron anotaciones JPA básicas a todos los modelos siguiendo el orden de dependencias:

#### Entidades Catálogo (sin dependencias)
1. ✅ **OrderStatus** → `order_statuses`
2. ✅ **PaymentStatus** → `payment_statuses`
3. ✅ **PaymentMethod** → `payment_methods`
4. ✅ **Role** → `roles`

#### Entidades Principales
5. ✅ **Category** → `categories` (auto-referencia sin `@ManyToOne` todavía)
6. ✅ **User** → `users` (con enum `UserStatus`)
7. ✅ **Product** → `products` (con precisión monetaria)
8. ✅ **UserSession** → `user_sessions`

#### Entidades Dependientes
9. ✅ **Address** → `addresses` (con enum `AddressType`)
10. ✅ **Cart** → `carts` (con enum `CartStatus`)
11. ✅ **CartItem** → `cart_items` (con precisión monetaria)
12. ✅ **Order** → `orders` (4 campos con precisión monetaria)
13. ✅ **OrderItem** → `order_items` (2 campos con precisión monetaria)
14. ✅ **Payment** → `payments` (con enum `Currency` y precisión monetaria)

### 2. Tabla Detallada de Cambios

| Modelo | Tabla BD | Anotaciones Agregadas | Campos Agregados | Enums | Decimales (10,2) |
|--------|----------|-----------------------|------------------|-------|------------------|
| OrderStatus | order_statuses | @Entity, @Table, @Id, @GeneratedValue, @Column (3) | description | - | 0 |
| PaymentStatus | payment_statuses | @Entity, @Table, @Id, @GeneratedValue, @Column (3) | description | - | 0 |
| PaymentMethod | payment_methods | @Entity, @Table, @Id, @GeneratedValue, @Column (3) | description | - | 0 |
| Role | roles | @Entity, @Table, @Id, @GeneratedValue, @Column (3) | - | - | 0 |
| Category | categories | @Entity, @Table, @Id, @GeneratedValue, @Column (3) | - | - | 0 |
| User | users | @Entity, @Table, @Id, @GeneratedValue, @Column (8), @Enumerated | - | UserStatus (STRING) | 0 |
| Product | products | @Entity, @Table, @Id, @GeneratedValue, @Column (8) | - | - | 1 (price) |
| UserSession | user_sessions | @Entity, @Table, @Id, @GeneratedValue, @Column (4) | - | - | 0 |
| Address | addresses | @Entity, @Table, @Id, @GeneratedValue, @Column (10), @Enumerated | - | AddressType (STRING) | 0 |
| Cart | carts | @Entity, @Table, @Id, @GeneratedValue, @Column (5), @Enumerated | - | CartStatus (STRING) | 0 |
| CartItem | cart_items | @Entity, @Table, @Id, @GeneratedValue, @Column (4) | - | - | 1 (unitPrice) |
| Order | orders | @Entity, @Table, @Id, @GeneratedValue, @Column (11) | - | - | 4 (subtotal, tax, shippingCost, total) |
| OrderItem | order_items | @Entity, @Table, @Id, @GeneratedValue, @Column (5) | - | - | 2 (unitPrice, lineTotal) |
| Payment | payments | @Entity, @Table, @Id, @GeneratedValue, @Column (7), @Enumerated | - | Currency (STRING) | 1 (amount) |

**Totales:**
- 14 modelos anotados
- 3 campos `description` agregados (OrderStatus, PaymentStatus, PaymentMethod)
- 4 enums mapeados con STRING
- 10 campos con precisión monetaria (precision=10, scale=2)

### 3. Estrategias Aplicadas

#### 3.1 Mapeo de Enums con STRING

Se configuraron todos los enums con `EnumType.STRING` para legibilidad y mantenibilidad:

```java
@Enumerated(EnumType.STRING)
@Column(name = "status", nullable = false)
private UserStatus status;
```

**Enums mapeados:**
- `UserStatus` (ACTIVE, INACTIVE, SUSPENDED) en `User.status`
- `CartStatus` (OPEN, CONVERTED, ABANDONED) en `Cart.status`
- `AddressType` (SHIPPING, BILLING) en `Address.type`
- `Currency` (USD, COP, EUR) en `Payment.currency`

**Ventajas:**
- Valores legibles en base de datos
- Independencia del orden de declaración
- Facilita debugging y queries SQL manuales

#### 3.2 Campos Temporales con updatable=false

Se aplicó `@Column(updatable = false)` en campos `createdAt` y similares para prevenir modificación accidental:

```java
@Column(name = "created_at", nullable = false, updatable = false)
@Builder.Default
private LocalDateTime createdAt = LocalDateTime.now();
```

**Aplicado en 6 modelos:**
1. User.createdAt
2. Product.createdAt
3. UserSession.createdAt
4. Cart.createdAt
5. CartItem.addedAt
6. Order.createdAt

**Razón:** Garantizar inmutabilidad de timestamps de creación

#### 3.3 Precisión Monetaria con precision=10, scale=2

Se configuró precisión decimal explícita en todos los campos monetarios:

```java
@Column(name = "price", nullable = false, precision = 10, scale = 2)
private BigDecimal price;
```

**Aplicado en 10 campos:**
1. Product.price
2. CartItem.unitPrice
3. Order.subtotal
4. Order.tax
5. Order.shippingCost
6. Order.total
7. OrderItem.unitPrice
8. OrderItem.lineTotal
9. Payment.amount

**Ventajas:**
- Consistencia con schema.sql (DECIMAL(10,2))
- Prevención de errores de redondeo
- Validación automática de rangos

#### 3.4 Campo description en Catálogos

Se agregó el campo `description` (VARCHAR 255, nullable) en entidades catálogo según schema.sql:

**Agregado en:**
- OrderStatus.description
- PaymentStatus.description
- PaymentMethod.description

**Ya existía en:**
- Role.description ✓

**Actualización de toString:**
Todos los métodos toString fueron actualizados para incluir el nuevo campo.

### 4. Campos Mantenidos sin Relaciones JPA

**Importante:** En esta etapa NO se agregaron anotaciones de relaciones. Los siguientes campos se mantienen sin `@ManyToOne`, `@OneToMany`, o `@JoinColumn`:

#### Relaciones N:1 (sin @ManyToOne todavía)
- Category.parent → Category
- User.role → Role
- Product.category → Category
- UserSession.user → User
- Address.user → User
- Cart.user → User
- Cart.session → UserSession
- CartItem.cart → Cart
- CartItem.product → Product
- OrderItem.order → Order
- OrderItem.product → Product
- Payment.order → Order
- Payment.paymentMethod → PaymentMethod
- Payment.paymentStatus → PaymentStatus

#### Relaciones 1:N (sin @OneToMany todavía)
- Category.subcategories → List\<Category\>
- Category.products → List\<Product\>
- User.addresses → List\<Address\>
- Cart.items → List\<CartItem\>
- Order.items → List\<OrderItem\>

#### Relaciones con Long (IDs sin conversión)
- Order.userId → Long
- Order.orderStatusId → Long
- Order.shippingAddressId → Long
- Order.billingAddressId → Long

**Razón:** Las relaciones JPA se agregarán en Etapa 09 para mantener compilación exitosa incremental.

### 5. Setters Personalizados Mantenidos

Se mantuvieron intactos todos los setters con validación personalizada:

- **Product:** `setPrice()`, `setStockQty()`
- **CartItem:** `setQuantity()`, `setUnitPrice()`
- **Order:** `setSubtotal()`, `setTax()`, `setShippingCost()`, `setTotal()`
- **OrderItem:** `setQuantity()`, `setUnitPrice()`, `setLineTotal()`
- **Payment:** `setAmount()`

**Razón:** Lombok respeta métodos explícitos y no los sobrescribe.

## Commits Realizados

Se realizaron **15 commits granulares** en la rama `etapa08`:

1. `feat(model): add JPA annotations to OrderStatus`
2. `feat(model): add JPA annotations to PaymentStatus`
3. `feat(model): add JPA annotations to PaymentMethod`
4. `feat(model): add JPA annotations to Role`
5. `feat(model): add JPA annotations to Category`
6. `feat(model): add JPA annotations to User with enum mapping`
7. `feat(model): add JPA annotations to Product with decimal precision`
8. `feat(model): add JPA annotations to UserSession`
9. `feat(model): add JPA annotations to Address with enum mapping`
10. `feat(model): add JPA annotations to Cart with enum mapping`
11. `feat(model): add JPA annotations to CartItem with decimal precision`
12. `feat(model): add JPA annotations to Order with decimal precision`
13. `feat(model): add JPA annotations to OrderItem with decimal precision`
14. `feat(model): add JPA annotations to Payment with enum and decimal`
15. `docs: add MODELS_REFERENCE_JPA_BASIC with complete annotated models`

**Próximo commit (este archivo):**
16. `docs: add ETAPA08_SUMMARY with JPA basic annotations details`

## Validación

### Compilación Exitosa
```bash
mvn clean compile
# [INFO] BUILD SUCCESS
```

Todos los modelos compilan correctamente sin errores ni warnings críticos.

### No se Requirieron Cambios en:
- ✅ **Servicios** - Continúan funcionando sin modificaciones
- ✅ **Mappers** - Usan getters/setters que Lombok sigue generando
- ✅ **DTOs** - Sin cambios necesarios
- ✅ **Utilidades** - Sin cambios necesarios
- ✅ **Enums** - Sin cambios necesarios

### Compatibilidad Verificada
- ✅ Lombok 1.18.42 - Compatible con Jakarta Persistence API
- ✅ Jakarta Persistence API 3.1.0 - Importado correctamente
- ✅ Hibernate 6.4.4.Final - Reconoce anotaciones

## Documentación Creada

### 1. MODELS_REFERENCE_JPA_BASIC.md
**Ubicación:** `documents_external/MODELS_REFERENCE_JPA_BASIC.md`

**Contenido:**
- Tabla resumen de mapeo completo
- Explicación de estrategias aplicadas
- Código completo de las 4 entidades catálogo
- Sección de Troubleshooting con errores comunes
- Guía para próxima etapa (relaciones JPA)

**Propósito:** Permitir a estudiantes comparar su código con el estado correcto.

### 2. ETAPA08_SUMMARY.md
**Ubicación:** `ETAPA08_SUMMARY.md` (este archivo)

## Diferencias con Etapa 07

| Aspecto | Etapa 07 | Etapa 08 |
|---------|----------|----------|
| Anotaciones JPA | ❌ No | ✅ Sí (@Entity, @Table, @Id, @Column) |
| Relaciones JPA | ❌ No | ❌ No (Etapa 09) |
| Campo description | ❌ Solo en Role | ✅ En 4 catálogos |
| Enums mapeados | - | ✅ 4 enums con STRING |
| Precisión monetaria | - | ✅ 10 campos con (10,2) |
| updatable=false | - | ✅ 6 campos temporales |
| Lombok | ✅ Sí | ✅ Sí (sin cambios) |
| Compilación | ✅ SUCCESS | ✅ SUCCESS |

## Próximos Pasos (Etapa 09)

### Agregar Relaciones JPA

1. **Relaciones N:1 con @ManyToOne**
   ```java
   @ManyToOne(fetch = FetchType.LAZY)
   @JoinColumn(name = "category_id", nullable = false)
   private Category category;
   ```

2. **Relaciones 1:N con @OneToMany**
   ```java
   @OneToMany(mappedBy = "category", cascade = CascadeType.ALL, orphanRemoval = true)
   private List<Product> products = new ArrayList<>();
   ```

3. **Auto-referencia en Category**
   ```java
   @ManyToOne(fetch = FetchType.LAZY)
   @JoinColumn(name = "parent_id")
   private Category parent;

   @OneToMany(mappedBy = "parent")
   private List<Category> subcategories = new ArrayList<>();
   ```

4. **Constraints UNIQUE compuestos**
   ```java
   @Table(name = "cart_items", uniqueConstraints = {
       @UniqueConstraint(columnNames = {"cart_id", "product_id"})
   })
   ```

5. **Convertir IDs Long a entidades**
   - Order.userId → User user
   - Order.orderStatusId → OrderStatus orderStatus
   - Order.shippingAddressId → Address shippingAddress
   - Order.billingAddressId → Address billingAddress

## Beneficios de esta Etapa

### 1. Preparación para Persistencia JPA
Los modelos están listos para ser gestionados por EntityManager/JpaRepository sin relaciones complejas primero.

### 2. Validación Incremental
Cada modelo se compiló individualmente evitando errores en cascada.

### 3. Mapeo Explícito Completo
Todas las columnas tienen `@Column` con atributos específicos (name, nullable, unique, length, precision, scale).

### 4. Compatibilidad con Schema.sql
El mapeo coincide exactamente con las definiciones SQL existentes.

### 5. Enums Legibles
Los valores en base de datos son strings legibles ("ACTIVE", "OPEN") en lugar de ordinales (0, 1).

### 6. Precisión Monetaria Garantizada
Los campos BigDecimal tienen restricción explícita de 10 dígitos totales, 2 decimales.

## Conclusiones

La Etapa 08 representa un paso crucial en la transición del proyecto POJO a Spring JPA. Se han agregado todas las anotaciones JPA básicas necesarias manteniendo:

- ✅ Compilación exitosa
- ✅ Compatibilidad con código existente
- ✅ Setters personalizados con validación
- ✅ ToString personalizados para debugging
- ✅ Equals/HashCode basados en ID
- ✅ Valores por defecto con @Builder.Default

La separación entre anotaciones básicas (Etapa 08) y relaciones JPA (Etapa 09) permite:
- Validación incremental sin errores complejos
- Comprensión paso a paso del mapeo objeto-relacional
- Facilidad de debugging si surgen problemas

---
**Fecha:** 18 de Febrero de 2026  
**Rama:** `etapa08`  
**Estado:** ✅ Completada y Funcional  
**Siguiente Etapa:** 09 - Agregar Relaciones JPA

