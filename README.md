# Informe de Arquitectura y Optimización de Base de Datos en PostgreSQL y AWS

**Autor:** Kevin Valverde  
**Fecha:** [Completar]  
**Proyecto:** Modelo y estrategia para sistema de gestión de pedidos multirestaurante.

---

## ✅ Parte 1: Modelado de Datos (Relacional)

### Requerimientos

- Restaurantes con múltiples sucursales.
- Clientes que realizan pedidos.
- Pedidos que contienen uno o más productos.
- Estado de cada pedido: `initiated`, `sent`, `delivered`.
- Registro de cambios de estado con marca de tiempo.
- Control de trazabilidad de usuarios que registran los datos.

---

### 🔧 Script SQL (CREATE TABLE, relaciones, índices, constraints)

> Incluye definición de tipos, tablas principales, relaciones, claves foráneas, claves primarias, y trazabilidad de inserciones.

🔗 [Ver el script completo aquí](#) *(o colocar inline o como archivo `.sql` en el repo)*

---

### 📌 Justificación del Modelo

- **Normalización hasta 3FN**: Separación clara entre entidades, evitando redundancias.
- **Multitenencia**: Todas las entidades contienen `restaurant_id` para aislar datos por empresa.
- **Auditoría**: Cada tabla operativa incluye `created_by_user_id`.
- **Tipos de datos**: Uso adecuado de `ENUM`, `VARCHAR`, `NUMERIC`, `TIMESTAMP`.
- **Integridad referencial**: Claves foráneas con `ON DELETE CASCADE`.
- **Índices inteligentes**:
  - Por `restaurant_id` para operaciones multitenant.
  - Compuestos para filtros frecuentes: `status`, `created_at`, `email`, etc.
- **Seguridad**: Contraseñas almacenadas como `password_hash`.

---

## ⚙️ Parte 2: Optimización de Consultas

### 🎯 Consulta evaluada

```sql
SELECT p.id, p.total, c.name, r.name
FROM orders p
JOIN clients c ON p.client_id = c.id
JOIN restaurants r ON p.restaurant_id = r.id
WHERE p.status = 'sent'
AND p.created_at >= NOW() - interval '7 days';
