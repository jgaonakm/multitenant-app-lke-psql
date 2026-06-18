# Aislamiento de datos con Row-Level Security en PostgreSQL

## Descripción

La funcionalidad de [Row-Level Security de PostgreSQL](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) permite asociar *políticas* a una tabla que se evalúan fila por fila. Cada vez que un inquilino consulta o modifica datos, PostgreSQL aplica automáticamente la política y descarta cualquier fila que no le pertenezca, sin que la aplicación tenga que añadir cláusulas `WHERE tenant_id = ...` en cada consulta.

La estrategia de este ejemplo es asignar **un rol de base de datos por inquilino** y vincular la propiedad de cada fila a ese rol mediante la función `current_user`. Así, la pertenencia de los datos queda atada a la identidad con la que se conecta cada inquilino, que se proporciona mendiante un *Secret* de Kubernetes a la aplicación desplegada.

## Crear la tabla compartida con la columna discriminadora

Conéctese como el usuario administrador de la base gestionada (el cliente exige una base existente, normalmente `defaultdb`):

```bash
psql "postgresql://akmadmin:CONTRASENA_ADMIN@HOST-pgsql.akamai.com:PUERTO/defaultdb?sslmode=require"
```

> Para trabajar en entorno gráfico, se puede utilizar pgAdmin

Cree la tabla. La columna `tenant_id` toma por defecto el rol conectado, de modo que cada inserción queda "estampada" automáticamente con el inquilino correcto:

```sql
CREATE TABLE IF NOT EXISTS public."Shipments"
(
    "Tenant" text COLLATE pg_catalog."default" NOT NULL,
    "ShipmentId" uuid NOT NULL,
    "CustomerId" uuid NOT NULL,
    "Origin" text COLLATE pg_catalog."default" NOT NULL,
    "Destination" text COLLATE pg_catalog."default" NOT NULL,
    "PackageDetails" text COLLATE pg_catalog."default" NOT NULL,
    "Priority" text COLLATE pg_catalog."default" NOT NULL,
    "DeliveryWindow" text COLLATE pg_catalog."default" NOT NULL,
    "CurrentBusinessStatus" text COLLATE pg_catalog."default" NOT NULL,
    CONSTRAINT "PK_Shipments" PRIMARY KEY ("ShipmentId")
)
```

## Activar y forzar RLS

```sql
ALTER TABLE public."Shipments" ENABLE ROW LEVEL SECURITY;
ALTER TABLE public."Shipments" FORCE ROW LEVEL SECURITY; -- Recomendado
```

- `ENABLE ROW LEVEL SECURITY` activa la verificación de políticas.
- `FORCE ROW LEVEL SECURITY` hace que las políticas se apliquen **incluso al propietario de la tabla**. Sin esto, el dueño de la tabla ignoraría RLS, lo que sería un punto ciego de seguridad.

## Definir la política basada en `current_user`

```sql
CREATE POLICY tenant_policy ON public."Shipments"
    USING ("Tenant" = current_user)
    WITH CHECK  ("Tenant" = current_user);
```

- La cláusula `USING` filtra las filas **visibles** en `SELECT`, `UPDATE` y `DELETE`: solo se ven las filas cuyo `tenant_id` coincide con el rol conectado.
- La cláusula `WITH CHECK` valida las filas que se **insertan o modifican**: impide que un inquilino escriba una fila con el `tenant_id` de otro.

## Crear un rol por inquilino y otorgar permisos

```sql
CREATE ROLE "tenant-a" LOGIN PASSWORD 'CONTRASENA-TN-A';
CREATE ROLE "tenant-b" LOGIN PASSWORD 'CONTRASENA-TN-B';

GRANT USAGE ON SCHEMA public TO "tenant-a", "tenant-b";
GRANT SELECT, INSERT, UPDATE, DELETE ON public."Shipments" TO "tenant-a", "tenant-b";
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO "tenant-a", "tenant-b";
```

### Verificar el aislamiento

```sql
-- Conectado como tenant-a:
SET ROLE tenant-a;
INSERT INTO public."Shipments" VALUES
('tenant-a', '11111111-1111-1111-1111-111111111111', 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa', 'Monterrey', 'Mexico City', 'Small box', 'Normal', 'Morning', 'Created');

SELECT * FROM Shipments;   -- solo verá filas de tenant-a

-- Conectado como tenant-b:
SET ROLE tenant-b;
SELECT * FROM Shipments;   -- 0 filas: no ve los datos de tenant-a

INSERT INTO public."Shipments" -- falla por WITH CHECK
VALUES
('tenant-a', '22222222-2222-2222-2222-222222222222', 'bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb', 'Guadalajara', 'Queretaro', 'Envelope', 'High', 'Afternoon', 'In transit'); 
```

## Variante recomendada cuando no se puede crear un rol por inquilino

Si el número de inquilinos crece mucho o la base gestionada limita la creación de roles, en lugar de un rol por inquilino se usa **un único rol de aplicación** y una **variable de sesión** que la aplicación establece tras autenticar al inquilino:

```sql
-- Política alternativa basada en variable de sesión
CREATE POLICY aislamiento_por_sesion ON Shipments
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

La aplicación ejecuta al inicio de cada transacción:

```sql
SET LOCAL app.tenant_id = 'tenant-a';
```

`SET LOCAL` limita el valor a la transacción actual, lo que es importante cuando se usa un *pool* de conexiones compartido. Esta variante escala mejor a miles de inquilinos; la del `current_user` ofrece un aislamiento más fuerte a nivel de motor. Elija según el volumen esperado de inquilinos.
