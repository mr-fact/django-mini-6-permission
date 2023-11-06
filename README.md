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
