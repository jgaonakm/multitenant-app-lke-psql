
# Pruebas

Para validar el correcto funcionamiento, incluido el aislamiendo de datos, se incluye [un ejemplo](ejemplo/) compuesto de una API REST, que retorna información de envíos.

```bash
curl -X GET "http://tenant-a.example.com/orders/shipments"
```

Esta llamada debe mostrar solamente datos correspondientes al tenant definido en el subdominio.

```json
[{"tenant":"tenant-a",
"shipmentId":"9d1590a6-25c2-a53d-9348-c55e558f61d1","customerId":"7fcbaf04-69f8-7767-f2ba-44c0a2ec0320","origin":"tenant-a-origin-1","destination":"tenant-a-destination-1","packageDetails":"Box 1 containing assorted goods seeded on 2026-04-08","priority":"Standard","deliveryWindow":"2026-04-11 to 2026-04-13","currentBusinessStatus":"PendingPickup"}]
```

Para probar que no se puede modificar el valor de `x-tenant-id` se puede ejecutar:

```bash
curl -X GET "http://tenant-a.example.com/orders/shipments" \
  -H "x-tenant-id: tenant-x" # valor diferente al subdominio
```

El resultado **debe ser el mismo** de la llamada sin el header, confirmando que es imposible suplantar la identidad cambiando el valor de `x-tenant-id`.git
