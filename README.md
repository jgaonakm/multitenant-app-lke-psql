# Guía de implementación: aplicaciones multitenant con Kubernetes Gateway API y Row-Level Security de PostgreSQL en Akamai Cloud (Linode)

## Introducción

Esta guía describe cómo construir una aplicación **multitenant** (multiples inquilinos) sobre **Akamai Cloud (Linode)** combinando tres mecanismos complementarios:

1. **Aislamiento de datos en la base de datos** mediante *Row-Level Security* (RLS) de PostgreSQL, apoyándose en `current_user`.
2. **Enrutamiento por inquilino** detectando el subdominio de cada petición con la **Kubernetes Gateway API** y la implementación **NGINX Gateway Fabric**, dirigiendo el tráfico al `Service` y `Deployment` correspondiente.
3. **Gestión de credenciales por inquilino** usando **Secrets de Kubernetes** para almacenar la cadena de conexión de cada base de datos o rol.

La idea central es que la *seguridad no dependa de una sola capa* ;la última línea de defensa siempre es la base de datos: con RLS activo, ningún inquilino puede leer ni escribir filas de otro, incluso si una capa superior fallara.

Apoyándose en los servicios gestionados de Linode (LKE, Managed PostgreSQL, NodeBalancer, DNS Manager, VPC y Cloud Firewall), el equipo puede concentrarse en la lógica del producto en lugar de operar infraestructura.

### Arquitectura de referencia

```txt
        acme.app.ejemplo.com
        globex.app.ejemplo.com              ┌─────────────────────────────┐
   Usuario ───────────────► NodeBalancer ─► │   NGINX Gateway Fabric      │
                                            │   (Gateway + HTTPRoutes)    │
                                            └──────────────┬──────────────┘
                                       detección por subdominio (hostname)
                          ┌──────────────────────┼──────────────────────┐
                          ▼                      ▼                      ▼
                  svc-tenant-acme         svc-tenant-globex        svc-tenant-...
                  Deployment acme         Deployment globex        Deployment ...
                          │  (Secret db-tenant-acme: DATABASE_URL)      │
                          └──────────────────────┼──────────────────────┘
                                                 ▼
                              Managed PostgreSQL (Akamai Cloud)
                           tabla compartida + RLS por current_user
```

Componentes de Akamai Cloud (Linode) utilizados:

- **[Linode Kubernetes Engine (LKE)](https://www.akamai.com/products/kubernetes):** clúster de Kubernetes gestionado. El plano de control estándar es gratuito; existe la opción de plano de control de alta disponibilidad y la variante **LKE Enterprise** para cargas más grandes.
- **[Managed Database para PostgreSQL](https://www.akamai.com/products/databases):** base de datos gestionada con respaldos automáticos diarios, *failover* multinodo y cifrado SSL forzado. Se puede desplegar en configuración de 1 nodo (standalone) o 3 nodos (alta disponibilidad).
- **[NodeBalancer](https://www.akamai.com/products/nodebalancers):** balanceador de carga gestionado que expone el Gateway al exterior.
- **[DNS Manager](https://www.akamai.com/products/dns-manager):** gestión de DNS sin costo adicional para apuntar los subdominios al NodeBalancer.
- **[Cloud Firewall](https://www.akamai.com/products/cloud-firewall) / [VPC](https://www.akamai.com/products/private-networking):** aislamiento de red entre el clúster y la base de datos.

### Modelo de multitenancy elegido

Existen varios modelos para aislar inquilinos en una base de datos relacional:

| Modelo | Descripción | Aislamiento | Costo operativo |
| --- | --- | --- | --- |
| Base de datos por inquilino | Una BD por cliente | Muy alto | Alto (no escala a miles) |
| Esquema por inquilino | Un `schema` por cliente | Alto | Medio |
| **Tabla compartida + RLS** | Tablas comunes con columna `tenant_id` y políticas RLS | Lógico, fuerte si se aplica bien | Bajo, escala muy bien |

Esta guía usa el modelo **tabla compartida con RLS**, también llamado *pool model*. Es el más eficiente en consumo de recursos y el que mejor escala cuando hay muchos inquilinos pequeños o medianos, que es el caso típico de un SaaS. El riesgo (que una consulta mal escrita filtre datos entre inquilinos) se neutraliza delegando el filtrado a PostgreSQL mediante RLS, no a la aplicación.

## Pre-requisitos

Antes de implementar los ejemplos, asegúrese de contar con:

1. [Crear una cuenta en Akamai Cloud](https://login.linode.com/signup)
2. Un **clúster LKE** activo y el archivo `kubeconfig` descargado desde Cloud Manager.
3. Una instancia de **Managed PostgreSQL** en la misma región que el clúster (para minimizar latencia).
4. Las herramientas de línea de comandos `kubectl` y `helm` para trabajar con Kubernetes.
5. El cliente `psql` o `pgAdmin` para administrar la base de datos.
6. Un dominio administrado en **DNS Manager** (por ejemplo `app.ejemplo.com`).

Verifique el acceso al clúster:

```bash
export KUBECONFIG=./mi-clutser-kubeconfig.yaml
kubectl get nodes
```

> **Nota de seguridad:** Managed Database de Akamai mantiene una lista de control de acceso (ACL). Para clústeres LKE [existe un proceso en segundo plano que **sincroniza automáticamente las IP de los nodos del clúster en la ACL de la base de datos**](https://techdocs.akamai.com/cloud-computing/docs/aiven-manage-database#lke-and-database-clusters-connectivity), de modo que sólo el clúster pueda conectarse. Para mayor aislamiento, es posible utilizar LKE Enterprise y configurar la base de datos dentro de la **VPC** correspondiente.

---
Siguientes pasos:

- **[Aislamiento de datos con Row-Level Security](README-2-aislamiento.md)**
- [Gateway API y rutas para exponer la aplicación](README-3-gateway.md)
- [Pruebas](README-4-pruebas.md)
- [Recomendaciones adicionales](README-5-siguientes-pasos.md)

---

## Lista de pasos para dar de alta un nuevo inquilino

1. **Base de datos:** crear el rol `tenant_<nombre>` con su contraseña y otorgar los `GRANT` sobre las tablas de la aplicación.
2. **Namespace:** `kubectl create namespace tenant-<nombre>` (opcional).
3. **Secret:** crear `db-tenant-<nombre>` con la `DATABASE_URL` del nuevo rol.
4. **Carga de trabajo:** desplegar `Deployment` y `Service` del inquilino, inyectando el Secret.
5. **Enrutamiento:** crear el `HTTPRoute` con el `hostname` `<nombre>.app.ejemplo.com`.
6. **DNS:** ya cubierto por el registro comodín; sin pasos adicionales.
7. **Verificación:** confirmar que `Accepted=True` y `ResolvedRefs=True` en el HTTPRoute, y que el inquilino solo ve sus propias filas.
