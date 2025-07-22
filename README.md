# Reto DBA OlaClick

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

**🎥 [Clic aquí para ver el video de Cración de BD](https://doem2yl3c8x6l.cloudfront.net/reto/Reto-Olaclick.mp4)**

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
***Adicional comparto el Script para insertar información en las tablas***

```sql
-- Insertar 50 restaurantes
INSERT INTO restaurants (name, created_at)
SELECT 'Restaurant ' || i,
       NOW() - (random() * interval '30 days')
FROM generate_series(1,50) i;

-- Insertar 50 usuarios por restaurante
INSERT INTO users (restaurant_id, username, full_name, role, email, password_hash, status, created_at)
SELECT r.id,
       'user_' || r.id || '_' || i,
       'User ' || i || ' R' || r.id,
       CASE WHEN i % 10 = 0 THEN 'admin' ELSE 'staff' END,
       'user' || i || '_r' || r.id || '@mail.com',
       md5(random()::text),
       (
         CASE 
           WHEN random() < 0.8 THEN 'A'
           WHEN random() < 0.9 THEN 'I'
           ELSE 'B'
         END
       )::user_status,
       NOW() - (random() * interval '30 days')
FROM restaurants r,
     generate_series(1,50) i;

-- Insertar 10 sucursales por restaurante
INSERT INTO branches (restaurant_id, name, address, created_at)
SELECT r.id,
       'Sucursal ' || i || ' R' || r.id,
       'Avenida Principal #' || (100 + i),
       NOW() - (random() * interval '30 days')
FROM restaurants r,
     generate_series(1,10) i;

-- Insertar 200 clientes por restaurante
INSERT INTO clients (restaurant_id, name, email, phone, created_at)
SELECT r.id,
       'Cliente ' || i || ' R' || r.id,
       'cliente' || i || '_r' || r.id || '@mail.com',
       '9' || trunc(random() * 100000000)::text,
       NOW() - (random() * interval '30 days')
FROM restaurants r,
     generate_series(1,200) i;

-- Insertar 100 productos por restaurante
INSERT INTO products (restaurant_id, name, description, price, created_at)
SELECT r.id,
       'Producto ' || i || ' R' || r.id,
       'Descripción del producto ' || i,
       ROUND((random()*90 + 10)::numeric, 2),   -- ✅ CAST A NUMERIC ANTES DE ROUND
       NOW() - (random() * interval '30 days')
FROM restaurants r,
     generate_series(1,100) i;

-- Insertar 10 000 pedidos con fecha aleatoria
WITH all_clients AS (
    SELECT c.id AS client_id, 
           c.restaurant_id, 
           (SELECT b.id FROM branches b WHERE b.restaurant_id = c.restaurant_id ORDER BY random() LIMIT 1) AS branch_id
    FROM clients c
)
INSERT INTO orders (restaurant_id, client_id, branch_id, total, status, created_at)
SELECT ac.restaurant_id,
       ac.client_id,
       ac.branch_id,
       0, -- se actualiza luego
       'initiated',
       -- 40% de pedidos estarán en los últimos 7 días, el resto en últimos 30
       CASE WHEN random() < 0.4 
            THEN NOW() - (random() * interval '7 days')
            ELSE NOW() - (random() * interval '30 days')
       END
FROM all_clients ac
ORDER BY random()
LIMIT 10000;

-- Insertar detalles a cada pedido (1 a 5 productos aleatorios)
DO $$
DECLARE
    o RECORD;
    prod RECORD;
    num_items INT;
    total_order NUMERIC(10,2);
BEGIN
    FOR o IN SELECT id, restaurant_id FROM orders LOOP
        total_order := 0;
        num_items := 1 + floor(random() * 5);
        FOR prod IN
            SELECT id, price FROM products p
            WHERE p.restaurant_id = o.restaurant_id
            ORDER BY random()
            LIMIT num_items
        LOOP
            INSERT INTO order_details (order_id, product_id, quantity, price, created_by_user_id)
            VALUES (
                o.id, 
                prod.id, 
                (1 + floor(random() * 3))::int, 
                prod.price,
                NULL
            );
            total_order := total_order + prod.price;
        END LOOP;
        UPDATE orders SET total = total_order WHERE id = o.id;
    END LOOP;
END $$;

-- Crear historial de estados para cada pedido con lógica temporal
DO $$
DECLARE
    o RECORD;
BEGIN
    FOR o IN SELECT id, created_at FROM orders LOOP
        -- siempre tiene initiated
        INSERT INTO order_status_history (order_id, status, changed_at)
        VALUES (o.id, 'initiated', o.created_at);

        -- 60% avanzan a sent en promedio 1-3 días después
        IF random() < 0.6 THEN
            INSERT INTO order_status_history (order_id, status, changed_at)
            VALUES (
                o.id, 
                'sent',
                o.created_at + (random() * interval '3 days')
            );
            UPDATE orders 
              SET status = 'sent' 
              WHERE id = o.id;

            -- 30% de esos se entregan después de 1-5 días
            IF random() < 0.5 THEN
                INSERT INTO order_status_history (order_id, status, changed_at)
                VALUES (
                    o.id, 
                    'delivered',
                    o.created_at + (3 + random() * 5) * interval '1 day'
                );
                UPDATE orders 
                  SET status = 'delivered' 
                  WHERE id = o.id;
            END IF;
        END IF;
    END LOOP;
END $$;
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

## 🧪 Parte 3: Redis

### 1. ¿Cómo usarías Redis para mejorar el rendimiento del sistema en lectura de órdenes por estado?
Redis es una base de datos en memoria extremadamente rápida, ideal para mejorar el rendimiento en consultas de lectura intensiva.

**Estrategia:**
Utilizar Redis como capa de **caché** para almacenar resultados frecuentes de consultas, por ejemplo:

- Pedidos por estado (`sent`, `delivered`, etc.)
- Conteo de pedidos por día o por estado
- Últimos pedidos realizados

**Ejemplo:**
```plaintext
Clave:  orders:status:sent
Valor:  JSON con los IDs o detalles resumidos de los pedidos
TTL:    60 segundos (Time To Live, para evitar desactualización)
   ```
Se puede establecer un `TTL` (Time To Live) de 30 a 60 segundos para asegurar la actualización automática de la caché, esto permite que, si un cliente o dashboard consulta constantemente los pedidos enviados, la información se entregue desde Redis en milisegundos, sin tocar PostgreSQL.

### 2. ¿Qué estrategias usarías para evitar inconsistencias entre Redis y PostgreSQL?

Redis no reemplaza a PostgreSQL como fuente de verdad, por eso se deben tomar medidas para evitar inconsistencias:

**Estrategias recomendadas:**

1. Cache-aside (lazy loading):
   - Consultar Redis primero.
   - Si no hay datos, leer desde PostgreSQL y almacenar en Redis.
   - Invalida o actualiza Redis manualmente después de cada escritura.

2. TTL corto:
   - Establecer expiración automática de los datos para garantizar frescura sin intervención manual.

3. Eventos de invalidación:
   - Emitir eventos (pub/sub, webhooks o triggers) que actualicen o eliminen caché cuando cambia el estado del pedido.

4. Marcas de tiempo en la caché:
   - Incluir `last_updated_at` para validar que la información aún sea válida antes de usarla.

### 3. ¿Cuándo preferirías evitar usar Redis en un sistema de alta concurrencia?
Aunque Redis es muy rápido, no siempre es la mejor solución en sistemas de alta concurrencia. Algunos escenarios en los que preferiría evitarlo o tener cuidado seria en:

❌ Sistemas con alta tasa de cambio: (Cuando los datos cambian constantemente)
- Si los pedidos cambian de estado constantemente (por ejemplo: cada segundo), Redis puede quedar obsoleto más rápido de lo que puede actualizarse y puede estar propenso a errores.
- Mantener la caché sincronizada entre PostgreSQL y Redis se vuelve costoso.

❌ Lógica crítica o financiera: (Cuando la lógica de negocio es crítica)
- No es recomendable confiar en Redis para operaciones que requieren consistencia fuerte.
- Como ejemplo: No usar Redis como fuente para pagos, saldos o stock, inventarios o auditoría. Puede mostrar información desactualizada y/o valores incorrectos.

❌ Casos donde la consistencia es más importante que el rendimiento: (Cuando la latencia de sincronización importa mucho)
- Redis es ideal para **rendimiento**, pero no para **consistencia estricta**.
- Si Redis no se actualiza al mismo tiempo que PostgreSQL, los usuarios podrían ver datos desfasados.
- En sistemas de auditoría, legal o financiero, esto no es aceptable.

**A continuación, comparto un pequeño cuadro resumen de cuando usar y no usar Redis:**

| Situación                          | ¿Redis es recomendable? | Comentario                                                 |
|-----------------------------------|--------------------------|-------------------------------------------------------------|
| Lecturas repetidas y costosas     | ✅ Sí                    | Ideal para acelerar consultas frecuentes                   |
| Datos que cambian constantemente  | ⚠️ Con precaución        | Podría quedar desactualizado                               |
| Datos críticos o financieros      | ❌ No                    | Usar solo PostgreSQL como fuente                           |

---

## 🧯 Parte 4: Respaldo y Recuperación
### 1. Describe tu estrategia de backup para una base de datos de 100 GB que no puede tener más de 5 minutos de pérdida de datos (RPO).
Si el objetivo es no perder más de 5 minutos de datos en caso de fallo (RPO: Recovery Point Objective), en base a mi experiencia usaría la siguiente estrategia:

WAL Archiving + Backups periódicos + Monitoreo
- Backups completos diarios
  - Realizar backups físicos diarios usando pg_basebackup o snapshots de disco en RDS/AWS EBS.
  - Programarlos en horarios de baja carga.
    
- Archivado de WALs (Write-Ahead Logs):
  - Configuraría PostgreSQL para guardar y archivar los WALs (con `archive_mode = on` y `archive_command`).
  - Esto me permite generar la recuperación punto a punto (PITR) hasta el último minuto, reproduciendo cambios desde el último backup completo.
    
- Incrementar frecuencia de backup para Reducir el tamaño del RPO:
  - En lugar de esperar 24h entre backups, se pueden hacer backups incrementales o diferenciales cada hora o 6 horas.
  - Validar que los WALs se archiven con frecuencia (cada 1–5 min).
    
- Si PostgreSQL lo tenemos en RDS/AWS:
  - Configuraría automated backups + retención de snapshots.
  - Habilitaría el point-in-time recovery (PITR) nativo.

### 2. ¿Cómo configurarías una réplica de solo lectura en PostgreSQL?
De manera on-premise realizaría lo siguiente:

1. En el **servidor primario**:
   - Configurar:
     ```conf
     wal_level = replica
     max_wal_senders = 5
     archive_mode = on
     ```
   - Crear usuario con permisos de `REPLICATION`.

2. En el **servidor secundario**:
   - Ejecutar `pg_basebackup`.
   - Crear archivo `standby.signal` en el directorio `data/`.
   - Configurar `primary_conninfo` con los datos del servidor primario.

Como Resultado obtendre que la réplica funcione en **modo hot standby** (solo lectura + sincronización en tiempo real).

En un SaaS por ejemplo RDS/AWS:
- Simplemente seleccionaría "Create read replica" desde la consola de RDS y elegiría la instancia primaria.
- AWS se encarga de la replicación, failover y sincronización automáticamente.

### 3. ¿Qué herramienta o enfoque usarías para automatizar y validar los backups?
En base a mi experiencia para automatizar recomiendo usar las siguientes herramientas:  

pgBackRest
  - Soporta backups incrementales, compresión, encriptación y PITR.
  - Verificación automática y restore por timestamp.

Barman
  - Especializado en recuperación ante desastres y múltiples servidores.

`pg_basebackup` + `cron` o `pg_dump` para entornos más simples.

En un SaaS por ejemplo RDS/AWS:
- Utiliza snapshots automáticos configurables por días/retención.
- Soporte de recuperación point-in-time (PITR).

Para validación de Backups, se debe automatizar scripts para:
  - Verificar el tamaño y éxito de cada backup.
  - Simulaciones de restauración con `pg_restore --list`.
  - Notificaciones vía correo o Slack si un backup falla.

En entornos productivos, también es buena práctica restaurar periódicamente en un entorno de pruebas.

**A continuación, comparto un cuadro resumen con la solución en base a las 3 preguntas:**

| Objetivo                        | Solución recomendada                                       |
|----------------------------------|-------------------------------------------------------------|
| RPO ≤ 5 minutos                  | Backups + WAL Archiving + PITR                             |
| Replica de solo lectura          | pg_basebackup + hot standby o AWS RDS Read Replica         |
| Automatización y validación      | pgBackRest / Barman / Cron + monitoreo y alertas           |

## 🔐 Parte 5: Seguridad y Acceso
### 1. ¿Qué prácticas aplicarías para proteger las credenciales de conexión a la base de datos?
Proteger las credenciales de acceso es fundamental para evitar accesos no autorizados o fuga de información, algunas buenas prácticas son:

1. Uso de gestores de secretos o variables de entorno
   - Nunca guardar credenciales en el código fuente.
   - En entornos on-premise, utilizar variables de entorno seguras (.env con protección de lectura).
   - Usar gestores como:
     - AWS Secrets Manager
     - HashiCorp Vault

2. Cambio de contraseñas periodicamente
   - Establecer políticas para que las contraseñas se renueven cada cierto tiempo (por ejemplo, cada 90 días).
   - Automatizar este proceso si es posible y más aun en sistemas cloud.

3. Conexiones cifradas (SSL/TLS)
   - Asegurar que todas las conexiones usen cifrado (`ssl = on` en PostgreSQL).
   - Especialmente crítico en entornos cloud o redes públicas.

4. Principio de menor privilegio
   - Cada aplicación o usuario solo debe tener los permisos estrictamente necesarios.
   - Separar usuarios para lectura, escritura o administración.
   
### 2. ¿Cómo controlarías el acceso a los datos entre entornos (producción, staging, desarrollo)?
Es clave y fundamental poder separar correctamente los entornos para evitar errores y fugas de información, en mi experiencia tomaría las siguientes acciones:

1. Bases de datos y roles separados por entorno
   - Instancias o bases independientes por entorno.
   - Roles distintos: `app_prod`, `app_staging`, `app_dev`.

2. Políticas de red restrictivas
   - Configurar firewalls o grupos de seguridad que controlen qué IPs pueden acceder a cada entorno.
   - Como ejemplo podria ser que para staging solo sea accesible desde ciertas IPs internas.

3. Datos ficticios en staging/desarrollo
   - Nunca usar datos reales de clientes en entornos de desarrollo.
   - Aplicar herramientas de nonimización o generación de datos ficticios.
   - Usar politicas de privacidad de datos.

4. Restricciones vía IAM (en la nube)
   - Por ejemplo, en AWS usar políticas de IAM para restricción por entorno, para evitar el acceso de usuarios comunes a producción.

### 3. ¿Cómo implementarías auditoría de acceso a datos sensibles?
Auditar quienes acceden a información crítica y/o sencible, es esencial para cumplir con normativas como (GDPR o HIPAA), yo aplicaría los siguientes metodos recomendados:

1. Configuración de registro de logs de auditoría
   - Configurar `log_statement = 'all'` o tambien podemos usar `log_statement = 'mod'` para registrar solo `INSERT`, `UPDATE`, `DELETE`.
   - Complementar con log_duration y log_connections.

2. Extensión `pgaudit`
   - Al instalar y configurar la extensión pgaudit para un control granular de auditoría, nos permite registrar SELECT, DDL y accesos a tablas específicas.

3. Triggers de auditoría personalizados
   - Para auditoría a nivel de tabla, se pueden crear AFTER INSERT/UPDATE/DELETE triggers que escriban en una tabla de auditoría (por ejemplo: audit_log).

5. Centralización y monitoreo
   - Al tener los logs, podemos enviarlos a un sistema de análisis centralizado como:
     - CloudWatch (AWS)
     - ELK Stack (Elasticsearch + Logstash + Kibana)
   - Tambien podemos generar alertas paraaccesos sospechosos.
