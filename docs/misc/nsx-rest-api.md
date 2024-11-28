# NSX REST API

## Postman Collection

- [NSX-T Manager API Collection](https://www.postman.com/dadoolittle/nsx-t-workspace/documentation/n7z451k/nsx-t-manager-api)

## Delete NSX Protected Objects

When deleting resources you might get an error similar to

!!! failure "Error"
    "Principal 'admin' with role '[enterprise_admin]'attempts to delete or modify an object of type nsx$InternalLogicalPort it doesn't own.

This can happen if objects have been created by another system using a Principal Identity. E.g. Tanzu Application Service (TAS) or Tanzu Kubernetes Grid Integrated Edition (TKGi) can integrate with NSX-T via the [NSX Container Plugin (NCP)](https://github.com/vmware/nsx-container-plugin-operator) which creates NSX objects at runtime.

If you still want to delete this object, use the header `X-Allow-Overwrite: true`.
