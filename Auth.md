# Dalgo Auth system

All http requests to the Dalgo backend come through an endpoint handler which looks like this

```
@ninjaapi.get("/endpoint", auth=auth.CanManagePipelines())
def endpoint_handler(request):
  ...
```

The Ninja system allows us to group our handlers into collections based on the url prefix; these are defined in `urls.py`.

Notice the `auth` parameter to the Ninja decorator, this specifies which middleware to use. All auth middlewares are in `auth.py`.

The middleware `CanManagePipelines` is defined like this:

```
class CanManagePipelines(HttpBearer):
    """ninja middleware to allow only account-owners or pipeline-managers"""

    def authenticate(self, request, token_key):
        return authenticate_org_user(
            request,
            token_key,
            [
                OrgUserRole.PIPELINE_MANAGER,
                OrgUserRole.ACCOUNT_MANAGER,
            ],
            True,
        )
```

It contains a single function `authenticate` which takes a Django `Reuest` object and a string `token_key`. The `token_key` is a lookup key into the `Token` table provided by the `django_rest_framework`.

All auth middlewares are of this form, differing only in the 
- list of allowed roles
- a boolean flag called `require_org`

## `authenticate_org_user`

This function does the following

1. Look up the `Token` from the `token_key`
2. If there is one (and if there is an associated `User`), store the `User` in the `request`
3. Find the first `OrgUser` mapped to this `User`. If the http headers contain an Org slug in the `x-dalgo-org` header, pick this Org
4. If the `OrgUser` is found, check that its role is one of the allowed roles passed in
5. If it is, store the `OrgUser` in the `request` and return

If there is no `x-dalgo-org` header, and if the first `OrgUser` has no `Org` attached, then fail if `require_org` is `True`. 
(This shouldn't happen, in the sense that we try not to have `OrgUser`s without `Org`s anymore...)

