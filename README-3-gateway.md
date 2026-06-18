# Detección de inquilino por subdominio con NGINX Gateway Fabric

## Descripción

La [**Kubernetes Gateway API**](https://gateway-api.sigs.k8s.io/docs/introduction/) (`gateway.networking.k8s.io/v1`) es el sucesor de Ingress: separa las responsabilidades de la infraestructura (`Gateway`, gestionado por el equipo de plataforma) de las del enrutamiento de aplicaciones (`HTTPRoute`, gestionado por cada equipo de producto/desarrollo). [**NGINX Gateway Fabric (NGF)**](https://docs.nginx.com/nginx-gateway-fabric/) es la implementación oficial de NGINX.

En este ejemplo, un único `Gateway` escucha en un *listener* con **hostname comodín** (`*.app.ejemplo.com`). Se crea un `HTTPRoute` por cada inquilino, que coincide con su subdominio específico, y reenvía el tráfico a un servicio y deployment comunes, considerando que todos los inquilinos utilicen la misma lógica de negocio.

Opcionalmente, cada inquilino puede tener su propio `Service` (y, por detrás, a su propio `Deployment`). Así, `acme.app.ejemplo.com` llegaría al Deployment de acme y `globex.app.ejemplo.com` al de globex.

## Instalar NGINX Gateway Fabric

Primero se instalan los CRDs de Gateway API

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.6.5" | kubectl apply -f -
```

y posteriormente el NGINX Gateway Fabric utilizando Helm

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --namespace nginx-gateway \
  --create-namespace \
  --wait
```

Para verificar que todo este correcto, utilizamos

```bash
kubectl get gatewayclass     
```

que debe mostrar un resultado como este

```txt
NAME    CONTROLLER                                   ACCEPTED   AGE
nginx   gateway.nginx.org/nginx-gateway-controller   True       2d
```

y al listar los `pods`

```bash
kubectl get pods -n nginx-gateway
```

debemos recibir

```txt
NAME                                        READY   STATUS    RESTARTS        AGE
ngf-nginx-gateway-fabric-7cd5775c76-dg5fb   1/1     Running   2 (6h28m ago)   2d
```

## Definir el Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-multitenant
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      hostname: "*.app.ejemplo.com"
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: wildcard-app-ejemplo-tls
      allowedRoutes:
        namespaces:
          from: All
```

`allowedRoutes.namespaces.from: All` permite que los `HTTPRoute` de cualquier *namespace* (uno por inquilino) se asocien a este Gateway compartido.

### Crear registro de DNS comodín para el Gateway

NGF se expone a internet mediante un `Service` de tipo `LoadBalancer`. En LKE, ese servicio provisiona automáticamente un [**NodeBalancer**](https://techdocs.akamai.com/cloud-computing/docs/nodebalancer) de Akamai. Para obteber la  IP pública usamos:

```bash
kubectl get svc -n nginx-gateway
```

En Cloud Manager, dentro de la [sección de dominios](https://techdocs.akamai.com/cloud-computing/docs/dns-manager), cree un registro comodín que apunte al NodeBalancer:

```txt
*.app.ejemplo.com   A   <IP_DEL_NODEBALANCER>
```

Con esto, cualquier subdominio nuevo de inquilino resuelve sin tener que tocar el DNS otra vez.

> DNS Manager está disponible **sin costo** en todas las regiones.

## Crear el Secret de un inquilino

Cada inquilino se conecta a PostgreSQL con su propio rol y contraseña. Esas credenciales nunca deben vivir en la imagen ni en el repositorio: se almacenan en un **Secret de Kubernetes** por inquilino, y el Deployment las consume como variable de entorno. Esto mantiene la cadena de conexión fuera del código y permite rotar credenciales sin reconstruir imágenes.

La forma más segura de hacerlo, es evitando dejar la contraseña en el historial de comandos, es desde un archivo o con `--from-literal` en un entorno controlado:

```bash
kubectl create secret generic db-tenant-acme \
  --namespace tenant-acme \
  --from-literal=DATABASE_URL="postgresql://tenant_acme:CONTRASENA_ACME@HOST-pgsql.akamai.com:PUERTO/defaultdb?sslmode=require"
```

Equivalente declarativo (**no lo guarde en git en texto plano**):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-tenant-acme
  namespace: tenant-acme
type: Opaque
stringData:
  DATABASE_URL: "postgresql://tenant_acme:CONTRASENA_ACME@HOST-pgsql.akamai.com:PUERTO/defaultdb?sslmode=require"
```

> El parámetro `sslmode=require` es importante: Managed Database de Akamai **fuerza el cifrado SSL**, por lo que la conexión debe negociarlo.

### Buenas prácticas y endurecimiento

- **No versionar Secrets en texto plano.** Use **Sealed Secrets** o el **External Secrets Operator** con un gestor externo, de modo que en git solo viva material cifrado o una referencia.
- **Privilegio mínimo.** El rol del inquilino solo debe tener los `GRANT` necesarios sobre las tablas de la aplicación, nunca permisos de administración.
- **Rotación de credenciales.** Cambie la contraseña del rol en PostgreSQL, actualice el Secret y reinicie el Deployment (`kubectl rollout restart`). Con External Secrets, la rotación puede ser automática.
- **RBAC sobre los Secrets.** Restrinja con RBAC quién puede leer Secrets en cada namespace de inquilino.

## Desplegar la aplicación

Cada componente expuesto de la aplicación consiste en un *deployment* y su *service* correspondiente. Por ejemplo, esto puede significar  uno o varios *front-end"*, sumados a diferentes microservicios, APIs, etc.

Cuando se consume la base de datos, se integra a la configuración la referencia de la variable de entorno.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-tenant-acme
  namespace: tenant-acme
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-tenant-acme
  template:
    metadata:
      labels:
        app: app-tenant-acme
    spec:
      containers:
        - name: app
          image: registro.ejemplo.com/mi-saas:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL          # ← inyectado desde el Secret
              valueFrom:
                secretKeyRef:
                  name: db-component
                  key: DATABASE_URL
---
apiVersion: v1
kind: Service
metadata:
  name: svc-tenant-acme
  namespace: tenant-acme
spec:
  selector:
    app: app-tenant-acme
  ports:
    - port: 80
      targetPort: 8080
```

### Crear el HTTPRoute por inquilino

Para procesar el subdominio correspondiente al inquilino, creamos un *HTTPRoute* para cada uno.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: route-tenant-acme
  namespace: tenant-acme
spec:
  parentRefs:
    - name: gateway-multitenant
      namespace: nginx-gateway
  hostnames:
    - "acme.app.ejemplo.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: svc-tenant-acme
          port: 80
    - filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            set:
              - name: X-Tenant-ID
                value: tenant-acme
      backendRefs:
        - name: svc-app-compartida
          port: 80
```

El campo `hostnames` es el que realiza la **detección del inquilino por subdominio**: NGF solo dirige a este servicio las peticiones cuyo encabezado `Host` sea `acme.app.ejemplo.com`. Como utilizamos **un solo *deployment* compartido**, se puede enrutar todos los subdominios al mismo servicio y dejar que NGF inyecte el inquilino como encabezado mediante un fil tro `RequestHeaderModifier` para que  sepa qué inquilino la invoca.

La aplicación lee `X-Tenant-ID` y selecciona el Secret/rol correspondiente. Esta variante reduce la cantidad de Deployments, a costa de concentrar más responsabilidad de aislamiento en el código de la aplicación, por eso RLS sigue siendo imprescindible; cada Deployment se conecta con el rol de su inquilino, y RLS hace el resto.

Para incorporar otro inquilino se repiten esta configuración cambiando el subdominio y el (opcionalmente) namespace. Y, finalmente, para verificar el estado de cada ruta usamos:

```bash
kubectl describe httproute route-tenant-acme -n tenant-acme
```

Que debe mostrarnos un resultado como este, donde las condiciones Accepted y ResolvedRefs están como True

``` txt
Name:         route-tenant-acme
Namespace:    tenant-acme
[...]
Status:
  Parents:
    Conditions:
      Message:               The Route is accepted
      Reason:                Accepted
      Status:                True
      Message:               All references are resolved
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
    Controller Name:         gateway.nginx.org/nginx-gateway-controller
```
