# django-mini-6-permission

منبع: https://www.django-rest-framework.org/api-guide/permissions/

## Permissions
in view: ([authentication](https://github.com/mr-fact/django-mini-5-authentication) -> **permission** -> throttling -> other codes)

simplest style of permission -> `IsAuthenticated`, `IsAuthenticatedOrReadOnly`

## How permissions are determined
- run a list of permission classes
- if permission ❌ & authentication ✅ -> `HTTP 403 Forbidden response`
- if permission ❌ & authentication ❌ & WWW-Authenticate ❌ -> `HTTP 403 Forbidden`
- if permission ❌ & authentication ❌ & WWW-Authenticate ✅ -> `HTTP 401 Unauthorized`

## Object level permissions
in view: (.get_object() -> .check_object_permissions(request, obj))
``` python
def get_object(self):
    obj = get_object_or_404(self.get_queryset(), pk=self.kwargs["pk"])
    self.check_object_permissions(self.request, obj)
    return obj
```

### Limitations of object level permissions
one object:
- object permission

a list of objects:
- [filter](https://www.django-rest-framework.org/api-guide/filtering/) the queryset
- filter in `serializer`
- override the `perform_create()` method of your `ViewSet` class.

## Setting the permission policy
set in `settings`
``` python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
#         'rest_framework.permissions.AllowAny',
        'rest_framework.permissions.IsAuthenticated',
    ]
}
```
set in `APIView`
``` python
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```
set in `@api_vew`
``` python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

@api_view(['GET'])
@permission_classes([IsAuthenticated])
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```
set in `View` -> ignore the default list set in the `settings`

Provided they inherit from `rest_framework.permissions.BasePermission`, permissions can be composed using `standard Python bitwise operators`:
- & (and)
- | (or)
- ~ (not)
``` python
from rest_framework.permissions import BasePermission, IsAuthenticated, SAFE_METHODS
from rest_framework.response import Response
from rest_framework.views import APIView

class ReadOnly(BasePermission):
    def has_permission(self, request, view):
        return request.method in SAFE_METHODS

class ExampleView(APIView):
    permission_classes = [IsAuthenticated|ReadOnly]

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```

## API Reference
- AllowAny
- IsAuthenticated
- IsAdminUser (`user.is_staff` == True)
- IsAuthenticatedOrReadOnly (safe methods = `GET`, `HEAD`, `OPTIONS`)
- DjangoModelPermissions
- DjangoModelPermissionsOrAnonReadOnly
- DjangoObjectPermissions

### DjangoModelPermissions & DjangoObjectPermissions
- DjangoModelPermissions ties into Django's standard `django.contrib.auth` [model permissions](https://docs.djangoproject.com/en/stable/topics/auth/customizing/#custom-permissions)
- DjangoObjectPermissions ties into Django's standard [object permissions framework](https://docs.djangoproject.com/en/stable/topics/auth/customizing/#handling-object-permissions). In order to use this permission class, you'll also need to add a permission backend that supports object-level permissions, such as [django-guardian](https://github.com/lukaszb/django-guardian).

These permissions must only be applied to views that have a `.queryset` property or `get_queryset()` method

The appropriate model is determined by checking `get_queryset().model` or `queryset.model`.

- `POST` Method -> check `add` permission on the model
- `PUT`, `PATCH` Method -> check `change` permission on the model
- `DELETE` Method -> check `delete` permission on the model

you might want to include a `view model permission` for `GET` requests.

To use custom model permissions, override `DjangoModelPermissions` or `DjangoObjectPermissions`, and set the `.perms_map` property.

**Note**: If you need object level view permissions for `GET`, `HEAD` and `OPTIONS` requests and are using `django-guardian` for your object-level permissions backend, you'll want to consider using the `DjangoObjectPermissionsFilter` class provided by the [djangorestframework-guardian package](https://github.com/rpkilby/django-rest-framework-guardian). It ensures that list endpoints only return results including objects for which the user has appropriate view permissions.

## Custom permissions
- override `BasePermission` class
- override `.has_permission(self, request, view)` method
- override `.has_object_permission(self, request, view, obj)` method
- return `True` if the request should be granted access
- return `False` otherwise
- rase `PermissionDenied` exception if return `False`

`safe methods`
``` python
if request.method in permissions.SAFE_METHODS:
    # Check permissions for read-only request
else:
    # Check permissions for write request
```

for instance-level in view: (`.check_object_permissions()` -> `.has_permission()` -> `.has_object_permission()`)

To change the error message associated with the exception, implement a `message` attribute directly on your `custom permission`
``` python
from rest_framework import permissions

class CustomerAccessPermission(permissions.BasePermission):
    message = 'Adding customers not allowed.'

    def has_permission(self, request, view):
         ...
```

examples:
``` python
from rest_framework import permissions

class BlocklistPermission(permissions.BasePermission):
    """
    Global permission check for blocked IPs.
    """

    def has_permission(self, request, view):
        ip_addr = request.META['REMOTE_ADDR']
        blocked = Blocklist.objects.filter(ip_addr=ip_addr).exists()
        return not blocked
```
``` python
class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Object-level permission to only allow owners of an object to edit it.
    Assumes the model instance has an `owner` attribute.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Instance must have an attribute named `owner`.
        return obj.owner == request.user
```

in `generic views` will check the appropriate the object level permissions

if you're writing your own `custom views`, you'll need to make sure you check the object level permission checks yourself (by calling `self.check_object_permissions(request, obj)`)

view rase `APIException` if permission checks fail

## Overview of access restriction methods
- `queryset`/`get_queryset()`
- `permission_classes`/`get_permissions()`
- `serializer_class`/`get_serializer()`

![image](https://github.com/mr-fact/django-mini-6-permission/assets/110711776/aaa6663b-81d5-4b9a-9a0b-5168786826d6)

- A `Serializer` should not raise `PermissionDenied` in a list action, or the entire list would not be returned
- The `get_*()` methods have access to the current view and can return different `Serializer` or `QuerySet` instances based on the `request` or `action`

## Third party packages
- [Django REST - Access Policy](https://github.com/rsinger86/drf-access-policy)
- [Composed Permissions](https://github.com/niwibe/djangorestframework-composed-permissions)
- [REST Condition](https://github.com/caxap/rest_condition)
- [DRY Rest Permissions](https://github.com/FJNR-inc/dry-rest-permissions)
- [Django Rest Framework Roles](https://github.com/computer-lab/django-rest-framework-roles)
- [Django REST Framework API Key](https://florimondmanca.github.io/djangorestframework-api-key/)
- [Django Rest Framework Role Filters](https://github.com/allisson/django-rest-framework-role-filters)
- [Django Rest Framework PSQ](https://github.com/drf-psq/drf-psq)
