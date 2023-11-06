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

