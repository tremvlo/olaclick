# Prueba DBA OlaClick

**Autor:** Kevin Valverde Bejarano  
**Fecha:** 20/07/2025

---

## 🔍 Parte 1: Modelado de Datos (Relacional)

### Requerimiento:

Diseña un modelo de datos relacional en PostgreSQL para soportar:

- Restaurantes con múltiples sucursales
- Clientes que realizan pedidos
- Pedidos que contienen uno o más productos
- Estado de cada pedido (`initiated`, `sent`, `delivered`)
- Registro de cambios de estado (con timestamp)

## Entregables

### **Entregable 1**  

***A continuación, se comparte el Script SQL para la Creación de Tablas e Índices***

```sql
-- ENUMs (tipo de datos)
CREATE TYPE order_status AS ENUM ('initiated', 'sent', 'delivered');
CREATE TYPE user_status AS ENUM ('A', 'I', 'B'); -- A: Activo, I: Inactivo, B: Bloqueado

-- Tabla: restaurants
CREATE TABLE restaurants (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla: users
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    restaurant_id INTEGER NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    username VARCHAR(100) NOT NULL UNIQUE,
    full_name VARCHAR(150),
    role VARCHAR(50),
    email VARCHAR(100) UNIQUE,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status user_status NOT NULL
);

-- Agregar FK circular tras creación de users
ALTER TABLE restaurants
ADD COLUMN created_by_user_id INTEGER REFERENCES users(id);

-- Tabla: branches
CREATE TABLE branches (
    id SERIAL PRIMARY KEY,
    restaurant_id INTEGER NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    address TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_user_id INTEGER REFERENCES users(id)
);

-- Tabla: clients
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    restaurant_id INTEGER NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (restaurant_id, email),
    created_by_user_id INTEGER REFERENCES users(id)
);

-- Tabla: products
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    restaurant_id INTEGER NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_user_id INTEGER REFERENCES users(id)
);

-- Tabla: orders
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    restaurant_id INTEGER NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    client_id INTEGER NOT NULL REFERENCES clients(id),
    branch_id INTEGER NOT NULL REFERENCES branches(id),
    total NUMERIC(10,2) NOT NULL DEFAULT 0,
    status order_status NOT NULL DEFAULT 'initiated',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by_user_id INTEGER REFERENCES users(id)
);

-- Tabla: order_details
CREATE TABLE order_details (
    order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    created_by_user_id INTEGER REFERENCES users(id),
    PRIMARY KEY (order_id, product_id)
);

-- Tabla: order_status_history
CREATE TABLE order_status_history (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    status order_status NOT NULL,
    changed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by_user_id INTEGER REFERENCES users(id)
);

-- Índices
-- Índices en tabla users
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_restaurant ON users(restaurant_id, status);

--Índices en tabla clients
CREATE INDEX idx_clients_name ON clients(restaurant_id, name);
CREATE INDEX idx_clients_phone ON clients(restaurant_id, phone);
CREATE INDEX idx_clients_restaurant_email ON clients(restaurant_id, email);

-- Índices en tabla products
CREATE INDEX idx_products_name ON products(restaurant_id, name);

-- Índices en tabla branches
CREATE INDEX idx_branches_name ON branches(restaurant_id, name);

-- Índices en tabla orders
CREATE INDEX idx_orders_restaurant_status_created ON orders(restaurant_id, status, created_at);
CREATE INDEX idx_orders_created_at ON orders(restaurant_id, created_at);
CREATE INDEX idx_orders_client_date ON orders(client_id, created_at);
CREATE INDEX idx_orders_restaurant_client ON orders(restaurant_id, client_id);
CREATE INDEX idx_orders_restaurant_branch ON orders(restaurant_id, branch_id);
CREATE INDEX idx_orders_branch ON orders(branch_id);

-- Índices en order_status_history
CREATE INDEX idx_order_status_history_order ON order_status_history(order_id);
CREATE INDEX idx_order_status_changed_at ON order_status_history(changed_at);

-- Índices en order_details
CREATE INDEX idx_order_details_product ON order_details(product_id);
```
### **Entregable 2**

***justificación de modelado***

El modelado realizado se justifica en los siguientes 9 enunciados:

1. Se aplicó un diseño normalizado 3NF lo cual nos permite:
   - Eliminar redudancia
   - Facilita la mantenibilidad
   - Reduce errores de inconsistencia

2. Multitenant para soporte a multiples restaurantes  

   Todas las entidades contienen el campo *restaurant_id* para:
   - Garantizar que los datos estén aislados por restaurante, fundamental en una arquitectura SaaS multicliente.
   - Permitir segmentación lógica sin necesidad de múltiples bases de datos.  

   Se diseñaron índices compuestos con *restaurant_id* para que las consultas filtradas por empresa sean eficientes.

3. Trazabilidad de usuarios  

   Cada tabla relevante incluye el campo *created_by_user_id*, que nos permite saber:
   - Quién creó cada registro, para auditoría y análisis.
   - Este campo se relaciona con la tabla *users*.  

   Esto fortalece el control y la rendición de cuentas en entornos regulados o multiusuario.

4. Tipo de datos adecuados  

   Elegí los tipos según el uso y buenas prácticas de PostgreSQL (VARCHAR(n), TEXT, NUMERIC(10,2), TIMESTAMP, ENUM).

5. Constraints y Reglas de Integridad
   
   - Se aplicaron llaves foráneas (FOREIGN KEY) para mantener relaciones válidas entre tablas.
   - Uso de ON DELETE CASCADE en entidades hijas, para evitar registros huérfanos.
   - Restricciones de unicidad como (restaurant_id, email) aseguran que un cliente no se repita dentro del mismo restaurante.

6. Índices diseñados para rendimiento  

   Se crearon índices estratégicos considerando los principales patrones de consulta:
   - orders(restaurant_id, status, created_at): para buscar pedidos recientes por estado.
   - clients(restaurant_id, email): para búsquedas de clientes por correo en una empresa.
   - products(restaurant_id, name): para autocompletado o búsqueda de productos.
   - Índices en claves foráneas (client_id, branch_id, etc.) para agilizar JOIN.  

   Esto mejora el rendimiento en lectura sin penalizar demasiado las escrituras.

7. Escalabilidad
   - El modelo está preparado para crecer horizontalmente, ya que todo está ligado a restaurant_id.
   - Puede adaptarse fácilmente a particionamiento por fecha o restaurante si el volumen lo exige.

8. Seguridad
   - Se modeló la tabla users con control de roles y estado (A, I, B) para habilitar/deshabilitar accesos.
   - Las contraseñas se almacenan como password_hash (no texto plano), siguiendo buenas prácticas.

9. Auditoría y Regulación
    - La combinación de campos como *created_by_user_id, status, changed_at, email* y trazabilidad por entidad permiten un diseño audit-ready.
    - Compatible con implementaciones de pgAudit para cumplimiento regulatorio si se requiere.

---

## ⚙️ Parte 2: Optimización de Consultas

Dada la siguiente consulta:

```sql
SELECT p.id, p.total, c.name, r.name
FROM orders p
JOIN clients c ON p.client_id = c.id
JOIN restaurants r ON p.restaurant_id = r.id
WHERE p.status = 'sent'
AND p.created_at >= NOW() - interval '7 days';
```
## Tarea 1

### Explica cómo identificarías cuellos de botella

   Realizaría un análisis de plan de ejecución con (EXPLAIN ANALYZE)  
     
   Un ejemplo real simplificado de un resultado seria:
   
   ```sql
Seq Scan on orders → 10,000 filas leídas  
Rows removed by filter → 8,815  
Hash Join con clients  
Hash Join con restaurants  
Tiempo total → 5.3 ms
   ```
   En base al ejemplo podemos identificar los siguientes problemas:
   - Se hace Seq Scan en orders, lo que significa que PostgreSQL lee toda la tabla completa para filtrar los pedidos enviados.
   - Se procesan 10,000 filas, pero se devuelven solo 1,185.
   - Las uniones (JOIN) usan Hash Join, lo que no es eficiente si hay índices disponibles.

## Tarea 2

### Propón al menos 2 formas de optimizar la consulta

### Solución 1:
   Crear un índice compuesto para los filtros

   ```sql
CREATE INDEX idx_orders_status_created ON orders (status, created_at);
   ```
   
   ***Ventajas***
   - Permite que PostgreSQL filtre rápidamente los pedidos sent creados en los últimos 7 días sin escanear toda la tabla.
   - Se adapta perfectamente al WHERE status = 'sent' AND created_at >= NOW() - interval '7 days'.

### Solución 2:
   Índices adicionales para mejorar los JOIN

   ```sql
CREATE INDEX idx_orders_client_id ON orders(client_id);
CREATE INDEX idx_orders_restaurant_id ON orders(restaurant_id);
CREATE INDEX idx_clients_id ON clients(id);
CREATE INDEX idx_restaurants_id ON restaurants(id);
   ```
   
   ***Ventajas***
   - Aceleran las uniones entre tablas.
   - Reducen la necesidad de escaneos completos en tablas grandes.
   - Fundamental en entornos multitenant y de alto volumen.

### Solución 3:
   Evaluar particionamiento de la tabla orders (si es muy grande)

   ```sql
CREATE TABLE orders_2025_07 PARTITION OF orders
FOR VALUES FROM ('2025-07-01') TO ('2025-08-01');
   ```
   
   ***Ventajas***
   - PostgreSQL accede directamente a la partición de los últimos 7 días.
   - Reduce dramáticamente el costo de escaneo y mejora la escalabilidad.

## Tarea 3

### Justifica cómo afectaría la solución al sistema en producción

   Con las soluciones anteriormente indicadas el impacto en producción seria:
   - Índices bien diseñados, mejora la reducción de tiempo de consulta de milisegundos a microsegundos.
   - Evitar Seq Scan, mejora la eficiencia, especialmente en tablas con alta concurrencia.
   - Mejora de JOIN con índices, reducción de CPU y RAM usada en operaciones complejas.
   - Particionamiento (si aplica), Escalabilidad horizontal, menor carga I/O.

**En conclusión**  
- Optimizar consultas en PostgreSQL no solo mejora la velocidad, sino también la estabilidad del sistema, reduce el consumo de recursos y mejora la experiencia del usuario.
- Un simple índice o una correcta estrategia de particionado puede ahorrar segundos por consulta, lo cual se traduce en horas ahorradas al día en sistemas con alta concurrencia.

---

### 🧪 Parte 3: Redis
1. ¿Cómo usarías Redis para mejorar el rendimiento del sistema en lectura de órdenes por estado?
2. ¿Qué estrategias usarías para evitar inconsistencias entre Redis y PostgreSQL?
3. ¿Cuándo preferirías evitar usar Redis en un sistema de alta concurrencia?
