# Recomendaciones adicionales

Estos pasos no forman parte del núcleo del ejemplo, pero pueden marcar diferencias entre un prototipo y un sistema listo para producción.

## Certificados TLS

Para el listener comodín del Gateway necesita un certificado para `*.app.ejemplo.com`, lo que requiere un certificado *wildcard*. Puede instalar **cert-manager** y resolver el desafío con DNS Manager mediante un *webhook solver* para Linode, o bien emitir certificados por subdominio con el desafío HTTP-01 si prefiere no usar comodín.

## Aislamiento de red

Coloque el clúster LKE y la Managed Database en la misma **VPC**, y use **Cloud Firewall** para limitar el tráfico entrante. Recuerde que la ACL de la base gestionada se sincroniza automáticamente con las IP de los nodos LKE.

## Observabilidad

Despliegue Prometheus y Grafana en el clúster. NGF expone métricas en un endpoint `/metrics` que puede recolectar con un `ServiceMonitor`. Monitoree por inquilino la latencia y los códigos de respuesta del Gateway.

## Escalado

Active el *autoscaling* de los *node pools* de LKE para absorber picos de tráfico. Por inquilino, puede usar `HorizontalPodAutoscaler` sobre cada Deployment.

## Automatizar el alta de inquilinos (onboarding)

A medida que crece el número de inquilinos, conviene plantillar el proceso. Un alta consiste siempre en los mismos pasos repetibles, ideales para una plantilla de Helm o un pipeline de CI/CD. Puede aprovechar las ventajas que ofrece [Akamai App Plaform para facilitar este proceso](https://techdocs.akamai.com/cloud-computing/docs/application-platform).
