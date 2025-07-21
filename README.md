# Prueba DBA OlaClick

**Autor:** Kevin Valverde Bejarano  
**Fecha:** 20/07/2025

---

## Parte 1: Modelado de Datos (Relacional)

**Requerimiento:**  
Diseñar un modelo de datos relacional en PostgreSQL para soportar:  
- Restaurantes con múltiples sucursales  
- Clientes que realizan pedidos  
- Pedidos que contienen uno o más productos  
- Estado de cada pedido (`initiated`, `sent`, `delivered`)  
- Registro de cambios de estado (con timestamp)

**Entregables de Parte 1: Modelado de Datos**  

### A continuación, se comparte el Script SQL para la Creación de Tablas e Índices

```sql
-- ENUMs
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

-- Índices adicionales
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_restaurant ON users(restaurant_id, status);

CREATE INDEX idx_clients_name ON clients(restaurant_id, name);
CREATE INDEX idx_clients_phone ON clients(restaurant_id, phone);
CREATE INDEX idx_clients_restaurant_email ON clients(restaurant_id, email);

CREATE INDEX idx_products_name ON products(restaurant_id, name);

CREATE INDEX idx_branches_name ON branches(restaurant_id, name);

CREATE INDEX idx_orders_restaurant_status_created ON orders(restaurant_id, status, created_at);
CREATE INDEX idx_orders_created_at ON orders(restaurant_id, created_at);
CREATE INDEX idx_orders_client_date ON orders(client_id, created_at);
CREATE INDEX idx_orders_restaurant_client ON orders(restaurant_id, client_id);
CREATE INDEX idx_orders_restaurant_branch ON orders(restaurant_id, branch_id);
CREATE INDEX idx_orders_branch ON orders(branch_id);

CREATE INDEX idx_order_status_history_order ON order_status_history(order_id);
CREATE INDEX idx_order_status_changed_at ON order_status_history(changed_at);

CREATE INDEX idx_order_details_product ON order_details(product_id);
