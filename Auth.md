# Dalgo Auth system

Dalgo now supports role-based access with the following roles

Role/Permissions | User Mgmt | Ingest | Transform | Orchestrate | Usage Dashboard | Analytics
-- | -- | -- | -- | -- | -- | --
Super User(Dalgo Admin) |  Update | Update |  Update | Update |  View |  Update
Account Manager (Admin) |  Update | Update |  Update | Update |  View |  Update
Pipeline Manager |  Update | Update |  Update | Update  |  View |  View
Analyst | View  | View  | Update | View  |  View | View
Guest | View  | View  | View | View | View  |  View

Based on the table above; the database maintains the correct mapping of a role to its respective set of permissions. This is done by seeding the database and at the moment is not configurable once the seeded.

All api endpoint functions are now wrapped in a permission based decorator `has_permission`. Only if the requestor has this array of permissions, it can pass through the api else it will throw an `Unauthorized` error. 
```
@has_permission(["can_create_pipeline"])
def endpoint_handler(request):
  ...
```
The Ninja system allows us to group our handlers into collections based on the url prefix; these are defined in `urls.py`.

Any http requests to the Dalgo backend come through an auth middleware defined in the `@ninjaapi` decorator. Notice the `auth` parameter

```
@ninjaapi.get("/endpoint", auth=auth.CustomAuthMiddleware())
@has_permission(["can_create_pipeline"])
def endpoint_handler(request):
  ...
```

The middleware `CustomAuthMiddleware` is defined like this:

```
class CustomAuthMiddleware(HttpBearer):
    """new middleware that works based on permissions from db"""

    def authenticate(self, request, token):
        tokenrecord = Token.objects.filter(key=token).first()
        if tokenrecord and tokenrecord.user:
            request.user = tokenrecord.user
            q_orguser = OrgUser.objects.filter(user=request.user)
            if request.headers.get("x-dalgo-org"):
                orgslug = request.headers["x-dalgo-org"]
                q_orguser = q_orguser.filter(org__slug=orgslug)
            orguser = q_orguser.first()
            if orguser is not None:
                if orguser.org is None:
                    raise HttpError(400, "register an organization first")

                permission_slugs = RolePermission.objects.filter(
                    role=orguser.new_role
                ).values_list("permission__slug", flat=True)

                request.permissions = list(permission_slugs) or []
                request.orguser = orguser
                return request

        raise HttpError(400, UNAUTHORIZED)
```

It contains a single function `authenticate` which takes a Django `Request` object and a string `token_key`. The `token_key` is a lookup key into the `Token` table provided by the `django_rest_framework`.

## `authenticate`

This function does the following

1. Look up the `Token` from the `token_key`
2. If there is one (and if there is an associated `User`), store the `User` in the `request`
3. Find the first `OrgUser` mapped to this `User`. If the http headers contain an Org slug in the `x-dalgo-org` header, pick this Org
4. If the `OrgUser` is found, fetch the permissions for this `OrgUser` and attach in the `request` object. Also store the `OrgUser` in the `request`.
5. The request then goes through the `has_permission` decorator which then matches the set of permissions the `OrgUser` has against the permission needed to access the api.
6. If `request.permissions` is a superset of permissions needed to access the api, then request goes through otherwise an error `Unauthorized` is raised. 

If the `OrgUser` has no `Org` attached, then raise an error `register an organization first`. 
(This shouldn't happen, in the sense that we try not to have `OrgUser`s without `Org`s anymore...)

