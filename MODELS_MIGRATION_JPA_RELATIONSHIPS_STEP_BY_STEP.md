# MODELS_MIGRATION_JPA_RELATIONSHIPS_STEP_BY_STEP

Guía **paso a paso** para convertir los modelos de `co.edu.cesde.pps.model` desde JPA básico (Etapa 08) a **relaciones JPA reales** (Etapa 09), incluyendo prevención de ciclos de serialización con Jackson.

> Objetivo: que el estudiante pueda aplicar relaciones JPA **sin romper el flujo** del proyecto (compilación + ejecución de servicios/mappers actuales).

---

## 0) Prerrequisitos

1. Debes estar en la rama base de Etapa 08.
2. Crear una rama nueva:

```bash
git checkout -b etapa09
```

3. Validar estado inicial:

```bash
mvn -q clean test
```

---

## 1) Reglas del ejercicio (importante)

- ✅ Mantén las anotaciones JPA básicas de Etapa 08 (`@Entity`, `@Table`, `@Id`, `@Column`, etc.).
- ✅ En Etapa 09 **sí** se agregan las relaciones:
  - `@ManyToOne`, `@OneToMany`, `@JoinColumn`, `mappedBy`
- ✅ `fetch = FetchType.LAZY` en todas las relaciones (evita cargas grandes por defecto).
- ✅ Agregar Jackson `@JsonManagedReference/@JsonBackReference` **solo en pares bidireccionales** donde hay riesgo de ciclo.
- ✅ El mapper puede quedar como está actualmente (patrón de `new Entity()` + setters). Solo cambia donde ya no existan los campos.
- ✅ El checklist de validación al final incluye **solo los puntos que rompen el flujo**.

---

## 2) Mapa de relaciones (qué FK representa cada relación)

> Referencia: `src/main/resources/sql/schema.sql`

| Modelo | Campo | Tipo relación | FK/Columna | Nullable | Lado dueño | Inverso (mappedBy) |
|---|---|---|---|---:|---|---|
| Category | parent | ManyToOne (self) | parent_id | Sí | Category.parent | Category.subcategories |
| Category | subcategories | OneToMany | - | - | - | mappedBy="parent" |
| Product | category | ManyToOne | category_id | No | Product.category | Category.products |
| Category | products | OneToMany | - | - | - | mappedBy="category" |
| User | role | ManyToOne | role_id | No | User.role | (opcional) Role.users |
| Address | user | ManyToOne | user_id | No | Address.user | User.addresses |
| User | addresses | OneToMany | - | - | - | mappedBy="user" |
| UserSession | user | ManyToOne | user_id | Sí | UserSession.user | (no requerido) |
| Cart | user | ManyToOne | user_id | Sí | Cart.user | (no requerido) |
| Cart | session | ManyToOne | session_id | Sí* | Cart.session | (no requerido) |
| Cart | items | OneToMany | - | - | - | mappedBy="cart" |
| CartItem | cart | ManyToOne | cart_id | No | CartItem.cart | Cart.items |
| CartItem | product | ManyToOne | product_id | No | CartItem.product | (no requerido) |
| Order | user | ManyToOne | user_id | No | Order.user | (no requerido) |
| Order | orderStatus | ManyToOne | order_status_id | No | Order.orderStatus | (no requerido) |
| Order | shippingAddress | ManyToOne | shipping_address_id | No | Order.shippingAddress | (no requerido) |
| Order | billingAddress | ManyToOne | billing_address_id | No | Order.billingAddress | (no requerido) |
| Order | items | OneToMany | - | - | - | mappedBy="order" |
| OrderItem | order | ManyToOne | order_id | No | OrderItem.order | Order.items |
| OrderItem | product | ManyToOne | product_id | No | OrderItem.product | (no requerido) |
| Payment | order | ManyToOne | order_id | No | Payment.order | (no requerido) |
| Payment | paymentMethod | ManyToOne | payment_method_id | No | Payment.paymentMethod | (no requerido) |
| Payment | paymentStatus | ManyToOne | payment_status_id | No | Payment.paymentStatus | (no requerido) |

\* En el `schema.sql` actual, `session_id` se define nullable. A nivel de negocio la sesión suele ser obligatoria, pero aquí seguimos el schema.

---

## 3) Serialización JSON (por qué hay ciclos y cómo se previenen)

### 3.1 Problema
En relaciones bidireccionales, Jackson puede entrar en un ciclo infinito:

- `Order` → `items` → `Order` → `items` ...

### 3.2 Solución (obligatoria en esta etapa)
Usar:

- `@JsonManagedReference("...")` en el lado “padre/colección”
- `@JsonBackReference("...")` en el lado “hijo”

**Pares requeridos en Etapa 09:**

1. `Category.subcategories` (managed) ↔ `Category.parent` (back)
2. `User.addresses` (managed) ↔ `Address.user` (back)
3. `Cart.items` (managed) ↔ `CartItem.cart` (back)
4. `Order.items` (managed) ↔ `OrderItem.order` (back)

---

## 4) Paso a paso por clase (con referencias a archivos)

> Recomendación didáctica: aplica 1 o 2 entidades por commit y compila.

### 4.1 Category (auto-referencia + subcategories + products) + Jackson
- Archivo: `src/main/java/co/edu/cesde/pps/model/Category.java`

Cambios:
- `parent`:
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "parent_id")
@JsonBackReference("category-parent")
private Category parent;
```

- `subcategories`:
```java
@OneToMany(mappedBy = "parent", fetch = FetchType.LAZY)
@JsonManagedReference("category-parent")
@Builder.Default
private List<Category> subcategories = new ArrayList<>();
```

- `products`:
```java
@OneToMany(mappedBy = "category", fetch = FetchType.LAZY)
@Builder.Default
private List<Product> products = new ArrayList<>();
```

Validación:
```bash
mvn -q clean test
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/Category.java
git commit -m "feat(model): add JPA relations and jackson refs to Category"
```

---

### 4.2 Product (category)
- Archivo: `src/main/java/co/edu/cesde/pps/model/Product.java`

Cambios:
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "category_id", nullable = false)
private Category category;
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/Product.java
git commit -m "feat(model): add @ManyToOne relation Product.category"
```

---

### 4.3 User (role + addresses) + Jackson
- Archivo: `src/main/java/co/edu/cesde/pps/model/User.java`

Cambios:
- `role`:
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "role_id", nullable = false)
private Role role;
```

- `addresses`:
```java
@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
@JsonManagedReference("user-addresses")
@Builder.Default
private List<Address> addresses = new ArrayList<>();
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/User.java
git commit -m "feat(model): add User role relation and addresses collection"
```

---

### 4.4 Address (user) + Jackson
- Archivo: `src/main/java/co/edu/cesde/pps/model/Address.java`

Cambios:
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id", nullable = false)
@JsonBackReference("user-addresses")
private User user;
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/Address.java
git commit -m "feat(model): add Address.user relation and jackson back ref"
```

---

### 4.5 UserSession (user)
- Archivo: `src/main/java/co/edu/cesde/pps/model/UserSession.java`

Cambios:
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id")
private User user; // nullable para guest
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/UserSession.java
git commit -m "feat(model): add UserSession.user relation"
```

---

### 4.6 Cart (user + session + items) + Jackson
- Archivo: `src/main/java/co/edu/cesde/pps/model/Cart.java`

Cambios:
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id")
private User user;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "session_id")
private UserSession session;

@OneToMany(mappedBy = "cart", fetch = FetchType.LAZY)
@JsonManagedReference("cart-items")
@Builder.Default
private List<CartItem> items = new ArrayList<>();
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/Cart.java
git commit -m "feat(model): add Cart user/session relations and items collection"
```

---

### 4.7 CartItem (cart + product) + Jackson + UNIQUE(cart_id, product_id)
- Archivo: `src/main/java/co/edu/cesde/pps/model/CartItem.java`

Cambios:
- constraint compuesto:
```java
@Table(name = "cart_items", uniqueConstraints = {
  @UniqueConstraint(columnNames = {"cart_id", "product_id"})
})
```

- relaciones:
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "cart_id", nullable = false)
@JsonBackReference("cart-items")
private Cart cart;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "product_id", nullable = false)
private Product product;
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/CartItem.java
git commit -m "feat(model): add CartItem relations and unique constraint"
```

---

## 5) Refactor importante: Order pasa de IDs Long a entidades

### 5.1 Por qué se cambia
Con JPA real, las relaciones se modelan como objetos, no como IDs sueltos.

Esto habilita:
- Navegación (`order.getUser().getEmail()`)
- Joins y consistencia referencial
- Eliminación de errores típicos de “IDs duplicados / sin entidad real” en lógica

### 5.2 Qué cambia exactamente
- Archivo: `src/main/java/co/edu/cesde/pps/model/Order.java`

Antes (Etapa 08):
- `Long userId`
- `Long orderStatusId`
- `Long shippingAddressId`
- `Long billingAddressId`

Ahora (Etapa 09):
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id", nullable = false)
private User user;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "order_status_id", nullable = false)
private OrderStatus orderStatus;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "shipping_address_id", nullable = false)
private Address shippingAddress;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "billing_address_id", nullable = false)
private Address billingAddress;

@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
@JsonManagedReference("order-items")
@Builder.Default
private List<OrderItem> items = new ArrayList<>();
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/Order.java
git commit -m "refactor(model): migrate Order from id fields to entity relations"
```

---

## 6) OrderItem (order + product) + Jackson + UNIQUE(order_id, product_id)
- Archivo: `src/main/java/co/edu/cesde/pps/model/OrderItem.java`

Cambios:
- constraint compuesto:
```java
@Table(name = "order_items", uniqueConstraints = {
  @UniqueConstraint(columnNames = {"order_id", "product_id"})
})
```

- relaciones:
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "order_id", nullable = false)
@JsonBackReference("order-items")
private Order order;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "product_id", nullable = false)
private Product product;
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/OrderItem.java
git commit -m "feat(model): add OrderItem relations and unique constraint"
```

---

## 7) Payment (order + catálogos)
- Archivo: `src/main/java/co/edu/cesde/pps/model/Payment.java`

Cambios:
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "order_id", nullable = false)
private Order order;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "payment_method_id", nullable = false)
private PaymentMethod paymentMethod;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "payment_status_id", nullable = false)
private PaymentStatus paymentStatus;
```

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/model/Payment.java
git commit -m "feat(model): add Payment relations (order/method/status)"
```

---

## 8) Cambios rompe-flujo en Services / Mappers

### 8.1 OrderService (checkout + filtros)
- Archivo: `src/main/java/co/edu/cesde/pps/service/OrderService.java`

Cambios obligatorios:
- En `checkout(...)`, al construir `Order`, asignar entidades:
  - `user`, `shippingAddress`, `billingAddress`, `orderStatus`
- Actualizar filtros:
  - `findOrdersByUser`: usar `o.getUser().getUserId()`
  - `findOrdersByStatus`: usar `o.getOrderStatus().getOrderStatusId()`

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/service/OrderService.java
git commit -m "fix(service): update OrderService after Order relations refactor"
```

### 8.2 OrderMapper
- Archivo: `src/main/java/co/edu/cesde/pps/mapper/OrderMapper.java`

Cambios obligatorios:
- Ya no existe `order.getUserId()` ni `order.getOrderStatusId()`.
- Ahora se lee desde entidades:
  - `order.getUser().getUserId()`
  - `order.getUser().getEmail()`
  - `order.getOrderStatus().getName()`

Commit sugerido:
```bash
git add src/main/java/co/edu/cesde/pps/mapper/OrderMapper.java
git commit -m "fix(mapper): update OrderMapper after Order relations refactor"
```

---

## 9) Checklist de validación (solo rompe-flujo)

### 9.1 Relaciones clave
- [ ] `CartItem` tiene `UNIQUE(cart_id, product_id)`
- [ ] `OrderItem` tiene `UNIQUE(order_id, product_id)`
- [ ] `Order` ya no usa `userId/orderStatusId/shippingAddressId/billingAddressId` como `Long`

### 9.2 Serialización
- [ ] `Category` tiene `@JsonManagedReference/@JsonBackReference` en `subcategories/parent`
- [ ] `User/Address` tiene managed/back para `addresses/user`
- [ ] `Cart/CartItem` tiene managed/back para `items/cart`
- [ ] `Order/OrderItem` tiene managed/back para `items/order`

### 9.3 Services / Mappers (rompe compilación)
- [ ] `OrderService.checkout` construye `Order` con entidades
- [ ] `OrderService.findOrdersByUser/findOrdersByStatus` filtran por entidades
- [ ] `OrderMapper.toDTO` no usa getters eliminados (`getUserId`, `getOrderStatusId`, etc.)

### 9.4 Build
- [ ] `mvn -q clean test` → BUILD SUCCESS

---

## 10) Nota sobre commits granulares

Mantén commits pequeños para que el estudiante pueda detectar el punto exacto donde se rompe:

- 1 entidad = 1 commit (o 2 si son dependientes)
- Siempre compilar antes de hacer commit
- Mensajes consistentes: `feat(model): ...`, `refactor(model): ...`, `fix(service): ...`, `docs: ...`

---

**Fin de la guía**

