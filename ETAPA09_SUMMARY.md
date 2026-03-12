# ETAPA 09 - Relaciones JPA Reales + Prevención de Ciclos de Serialización

## Objetivo
Convertir los modelos anotados en Etapa 08 (JPA básico) a **relaciones JPA reales** (FKs navegables) usando:

- `@ManyToOne` + `@JoinColumn`
- `@OneToMany(mappedBy = "...")`
- `@Table(uniqueConstraints = ...)` para constraints compuestos

Además, preparar el dominio para la futura exposición por JSON evitando ciclos con:

- `@JsonManagedReference` / `@JsonBackReference`

> Nota: En esta etapa aún **no hay endpoints**. Las referencias Jackson se agregan de forma preventiva para la etapa de controllers.

---

## Cambios Realizados

### 1) Relaciones agregadas (JPA real)

#### Jerarquía de categorías
- ✅ `Category.parent` → `@ManyToOne` (FK: `parent_id`, nullable)
- ✅ `Category.subcategories` → `@OneToMany(mappedBy = "parent")`
- ✅ `Product.category` → `@ManyToOne` (FK: `category_id`, NOT NULL)
- ✅ `Category.products` → `@OneToMany(mappedBy = "category")`

#### Usuarios y direcciones
- ✅ `User.role` → `@ManyToOne` (FK: `role_id`, NOT NULL)
- ✅ `Address.user` → `@ManyToOne` (FK: `user_id`, NOT NULL)
- ✅ `User.addresses` → `@OneToMany(mappedBy = "user")`

#### Sesiones, carrito e items
- ✅ `UserSession.user` → `@ManyToOne` (FK: `user_id`, nullable para guest)
- ✅ `Cart.user` → `@ManyToOne` (FK: `user_id`, nullable)
- ✅ `Cart.session` → `@ManyToOne` (FK: `session_id`, nullable según schema actual)
- ✅ `Cart.items` → `@OneToMany(mappedBy = "cart")`
- ✅ `CartItem.cart` → `@ManyToOne` (FK: `cart_id`, NOT NULL)
- ✅ `CartItem.product` → `@ManyToOne` (FK: `product_id`, NOT NULL)
- ✅ `CartItem` constraint compuesto:
  - `UNIQUE(cart_id, product_id)`

#### Órdenes e items (refactor clave)
- ✅ `Order` pasó de **IDs Long** a **relaciones**:
  - `Order.user` (FK: `user_id`, NOT NULL)
  - `Order.orderStatus` (FK: `order_status_id`, NOT NULL)
  - `Order.shippingAddress` (FK: `shipping_address_id`, NOT NULL)
  - `Order.billingAddress` (FK: `billing_address_id`, NOT NULL)
- ✅ `Order.items` → `@OneToMany(mappedBy = "order")`
- ✅ `OrderItem.order` → `@ManyToOne` (FK: `order_id`, NOT NULL)
- ✅ `OrderItem.product` → `@ManyToOne` (FK: `product_id`, NOT NULL)
- ✅ `OrderItem` constraint compuesto:
  - `UNIQUE(order_id, product_id)`

#### Pagos
- ✅ `Payment.order` → `@ManyToOne` (FK: `order_id`, NOT NULL)
- ✅ `Payment.paymentMethod` → `@ManyToOne` (FK: `payment_method_id`, NOT NULL)
- ✅ `Payment.paymentStatus` → `@ManyToOne` (FK: `payment_status_id`, NOT NULL)

---

### 2) Prevención de ciclos de serialización (Jackson)

Se agregaron referencias para evitar bucles infinitos cuando una entidad tiene relaciones bidireccionales:

- ✅ `Category.subcategories` → `@JsonManagedReference("category-parent")`
- ✅ `Category.parent` → `@JsonBackReference("category-parent")`

- ✅ `User.addresses` → `@JsonManagedReference("user-addresses")`
- ✅ `Address.user` → `@JsonBackReference("user-addresses")`

- ✅ `Cart.items` → `@JsonManagedReference("cart-items")`
- ✅ `CartItem.cart` → `@JsonBackReference("cart-items")`

- ✅ `Order.items` → `@JsonManagedReference("order-items")`
- ✅ `OrderItem.order` → `@JsonBackReference("order-items")`

---

### 3) Cambios que impactan Services / Mappers (rompen flujo)

#### OrderService
Archivo: `src/main/java/co/edu/cesde/pps/service/OrderService.java`

- ✅ `checkout(...)` ahora construye `Order` asignando entidades:
  - `user`, `shippingAddress`, `billingAddress`, `orderStatus`
- ✅ `findOrdersByUser(...)` ahora filtra por `o.getUser().getUserId()`
- ✅ `findOrdersByStatus(...)` ahora filtra por `o.getOrderStatus().getOrderStatusId()`

> Nota didáctica: como aún no hay repositorios JPA, el `OrderStatus` PENDING se crea como entidad mínima (in-memory).

#### OrderMapper
Archivo: `src/main/java/co/edu/cesde/pps/mapper/OrderMapper.java`

- ✅ `toDTO(Order)` ahora toma `userId/userEmail/userFullName` de `Order.user`
- ✅ `toDTO(Order)` toma `orderStatusName` de `Order.orderStatus`
- ✅ No se mapean direcciones completas a `AddressDTO` en este mapper (se deja para servicio/mapper dedicado en etapas posteriores).

---

## Commits sugeridos (granulares)

> En la guía paso a paso se detalla la secuencia con mensajes sugeridos.

Ejemplo de desglose ideal:
1. `feat(model): add JPA relations to Category and Product`
2. `feat(model): add Jackson managed/back refs to Category`
3. `feat(model): add JPA relations to User and Address`
4. `feat(model): add Jackson managed/back refs to User/Address`
5. `feat(model): add JPA relations to Cart/CartItem with unique constraint`
6. `feat(model): add Jackson managed/back refs to Cart/CartItem`
7. `refactor(model): migrate Order ids to entity relations`
8. `feat(model): add JPA relations to OrderItem with unique constraint`
9. `feat(model): add Jackson managed/back refs to Order/OrderItem`
10. `feat(model): add JPA relations to Payment`
11. `fix(service): update OrderService for Order entity relations`
12. `fix(mapper): update OrderMapper for Order entity relations`
13. `docs: add ETAPA09_SUMMARY and JPA relationships step-by-step`

---

## Validación

### Build
- ✅ `mvn clean test` → **BUILD SUCCESS**

---

## Próximos pasos

- Etapa 10 (futura): endpoints/controllers y manejo de Lazy Loading (DTOs, proyecciones, `@Transactional`, etc.).

---
**Fecha:** 24 de Febrero de 2026  
**Rama:** `etapa09`  
**Estado:** ✅ Implementada y compilando

