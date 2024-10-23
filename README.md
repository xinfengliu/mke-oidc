Integrating MKE with OIDC

# 1. (Optional) Keycloack

## 1.1 Installation

```yaml
ingress:
  enabled: true
  ingressClassName: nginx
  tls: true
  hostname: keycloak.k8s.pac.dockerps.io
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
production: true
proxy: edge
```

```sh
$ helm -n keycloak upgrade --install --create-namespace keycloak oci://registry-1.docker.io/bitnamicharts/keycloak --values keycloak-values-tmp.yaml
```

Connect to Keycloack admin console using the user `user`. The password can be retrieved using

```sh
$ kubectl -n keycloak get secrets keycloak -o jsonpath='{.data.admin-password}' | base64 --decode ; echo
```

## 2.2 Keycloak configuration

**Create Realm**
- Name --> pac.dockerps.io

**Create Client**
- Client Type : OpenID Connect
- Client ID : kubernetes
- Client authentication: off
- Authorization : off
- Direct access grants : disabled
- Valid redirect URIs:
  - http://localhost:18000
  - http://localhost:8000
  - https://mke.pac.dockerps.io/login
- Web origins: +

**Client scopes --> kubernetes-dedicated --> Configure a new mapper --> User Client Role**
- Name: iam_roles (used by MKE)
- Client ID: kubernetes
- Multivalued: on
- Claim JSON Type: String
- Add to ID token : on
- Add to access token : off
- Add to userinfo : off
- Add to token introspection : off

**Add Roles - These roles will be used to map MKE Teams**
> For Instance, we're going to create a role "myorg:view" that will map in MKE with Org:myorg,Team:view

Clients --> Kubernetes --> Roles --> Create role
- kube-access (will be used to give access to MKE)
- kube-admin (users with this role will be granted MKE admin)
- myorg:admin
- myorg:edit
- myorg:view
- myotherorg:admin
- myotherorg:edit
- myotherorg:view

**Create Groups that will map with these roles, and the Role mapping**
1. Groups --> Create group
2. Groups --> <existing group> --> Role mapping --> Assign role --> <existing role>

Create the following groups :  

- kube-access (role mapping: kube-access))
- kube-admin (role mapping: kube-admin)
- myorg
  - admin (role mapping: myorg:admin)
  - edit (role mapping: myorg:edit)
  - view (role mapping: myorg:view)
- myotherorg
  - admin (role mapping: myotherorg:admin)
  - edit (role mapping: myotherorg:edit)
  - view (role mapping: myotherorg:view)

**Create users and assign them to some Groups**

- admin-user --> kube-access + kube-admin
- dev-myorg-1 --> kube-access + myorg/edit
- ops-myotherorg-1 --> kube-access + myotherorg/admin
- random-user --> no group assigned

# 2. Configure MKE

See https://docs.mirantis.com/mke/3.7/ops/administer-cluster/configure-oidc-identity-provider.html

Add the following to `mke-config.toml`

```yaml
[auth]
  [auth.external_identity_provider]
    issuer = ""
    userServiceId = ""
    clientId = "kubernetes"
    wellKnownConfigUrl = "https://keycloak.k8s.pac.dockerps.io/realms/pac.dockerps.io/.well-known/openid-configuration"
    caBundle = ""
    usernameClaim = "sub"
    httpProxy = ""
    httpsProxy = ""

    [[auth.external_identity_provider.signInCriteria]]
    term = "iam_roles"
    value = "kube-access"
    matchType = "contains"

    [[auth.external_identity_provider.adminRoleCriteria]]
    term = "iam_roles"
    value = "kube-admin"
    matchType = "contains"
```

The important part here is that we're only accepting users that have the claim `iam_roles=kube-access` and we give MKE admin access to users that have the claim `iam_roles=kube-admin`.  

NOTE : `iam_roles` claim is used by default by MKE and doesn't seem to be configurable.

You should now be able to log into MKE UI using the external provider.  
- admin-user --> has MKE admin privileges
- dev-myorg-1 --> access to MKE, but no permission
- ops-myotherorg-1 --> access to MKE, but no permission yet
- random-user --> no access to MKE

# 3. Configure kubectl

## 3.1 Install kubelogin

Kubelogin is a kubectl plugin that allows to interact with OIDC

- Install Krew https://krew.sigs.k8s.io/docs/user-guide/quickstart/

- Install kubelogin plugin

```sh
$ kubectl krew install oidc-login
```

- Test kubelogin

```sh
# Get a Token
# Kubelogin will open a browser that will redirect to Keycloak. Log in with admin-user 
$ kubectl oidc-login get-token --oidc-issuer-url=https://keycloak.k8s.pac.dockerps.io/realms/pac.dockerps.io --oidc-client-id=kubernetes --oidc-use-pkce | jq -r .status.token | jwt decode -

Token header
------------
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "wiE6GVet_yj4G4ikYvO3U3qtlcUgIUB9eJhf3AVRwzg"
}

Token claims
------------
{
  "acr": "1",
  "at_hash": "1yJA4rQY32Un5Ao1tZS2cQ",
  "aud": "kubernetes",
  "auth_time": 1729684694,
  "azp": "kubernetes",
  "email": "admin-user@noreply.org",
  "email_verified": true,
  "exp": 1729684994,
  "family_name": "User",
  "given_name": "Admin",
  "iam_roles": [
    "kube-access",
    "kube-admin"
  ],
  "iat": 1729684694,
  "iss": "https://keycloak.k8s.pac.dockerps.io/realms/pac.dockerps.io",
  "jti": "de1b63d3-186e-4cd5-8ebd-2d69282e8b8a",
  "name": "Admin User",
  "nonce": "2XE5tZxdyLI-iJ4V8Q5vGrAdl-0O9IDtCFd3nZf8C44",
  "preferred_username": "admin-user",
  "sid": "5773bc91-aac6-4cb4-bcc5-9b30e0158a9d",
  "sub": "24c5eeec-920a-4b94-982a-c3efc52b02e3",
  "typ": "ID"
}
```

We can see that Kubelogin is able to get a Token, and gets the claim `iam_roles`

## 3.2 Kubeconfig

Create a kubeconfig file `kube-oidc.yml` that will uses `kubelogin` to get OIDC token. 

```yaml
apiVersion: v1
kind: Config
current-context: <mke url>.io_oidc
preferences: {}
contexts:
- context:
    cluster: <mke url>
    user: oidc
  name: <mke url>.io_oidc
clusters:
- cluster:
    certificate-authority-data:  < get content from $(curl https://<mke url>/ca | base64) >
  name: <mke url>
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://<keycloack url>/realms/<realm>
      - --oidc-client-id=<client id>
      - --oidc-use-pkce
      command: kubectl
      env: null
      interactiveMode: IfAvailable
      provideClusterInfo: false
```

Example

```yaml
apiVersion: v1
kind: Config
current-context: mke.pac.dockerps.io_oidc
preferences: {}
contexts:
- context:
    cluster: mke.pac.dockerps.io
    user: oidc
  name: mke.pac.dockerps.io_oidc
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURnVENDQXdpZ0F3SUJBZ0lTQk5ra2ROWWpvdTFjSWV1K1RlTVJIdU1CTUFvR0NDcUdTTTQ5QkFNRE1ESXgKQ3pBSkJnTlZCQVlUQWxWVE1SWXdGQVlEVlFRS0V3MU1aWFFuY3lCRmJtTnllWEIwTVFzd0NRWURWUVFERXdKRgpOVEFlRncweU5EQTRNRFV3T1RJMU5EZGFGdzB5TkRFeE1ETXdPVEkxTkRaYU1Cd3hHakFZQmdOVkJBTU1FU291CmNHRmpMbVJ2WTJ0bGNuQnpMbWx2TUZrd0V3WUhLb1pJemowQ0FRWUlLb1pJemowREFRY0RRZ0FFUkNCWk5GTTIKYW05OXA4VWdpUm83WGdmNUpIZytFM1VXRURTSHdjSG82Y0FRWk1YSXZRQ1YzeVIwYXlhZTkwQ3RqNkl1RVJESwo1WTY4QnNyWUdIdEFxS09DQWhJd2dnSU9NQTRHQTFVZER3RUIvd1FFQXdJSGdEQWRCZ05WSFNVRUZqQVVCZ2dyCkJnRUZCUWNEQVFZSUt3WUJCUVVIQXdJd0RBWURWUjBUQVFIL0JBSXdBREFkQmdOVkhRNEVGZ1FVNkxuUUI3SWEKRmZ4ZDJtOVM1SzNZSEFTUHJzQXdId1lEVlIwakJCZ3dGb0FVbnl0Znp6d2hUNTBFdCswckxNVEdjSXZTMXcwdwpWUVlJS3dZQkJRVUhBUUVFU1RCSE1DRUdDQ3NHQVFVRkJ6QUJoaFZvZEhSd09pOHZaVFV1Ynk1c1pXNWpjaTV2CmNtY3dJZ1lJS3dZQkJRVUhNQUtHRm1oMGRIQTZMeTlsTlM1cExteGxibU55TG05eVp5OHdIQVlEVlIwUkJCVXcKRTRJUktpNXdZV011Wkc5amEyVnljSE11YVc4d0V3WURWUjBnQkF3d0NqQUlCZ1puZ1F3QkFnRXdnZ0VEQmdvcgpCZ0VFQWRaNUFnUUNCSUgwQklIeEFPOEFkUUFabUJCeENmRFdVaTR3Z05LZVAyUzdnMjRvelBrUFVvN3UzODVLClB4YTB5Z0FBQVpFaUVYL1hBQUFFQXdCR01FUUNJQzdEUUN5cFU5dE5vR2NZTWd5WVNsVlBnV3plbmhVTm5xMnEKNWdmc1VCdzNBaUFnWStMWnM3U1RuYUMxZzdpVDBvMkNsVHhFckVSTzJ0WG5yK1lNdm1lVmZBQjJBSGIvaUQ4Swp0dnVWVWNKaHpQV0h1alMwcE0yN0tkeG9RZ3FmNW1kTVdqcDBBQUFCa1NJUmdBQUFBQVFEQUVjd1JRSWdGZG5oCnNIaEE0MFhiWDd6SkNseTlONVlGSGg4WHpoWlJ5NXJFWmRNTU5Ib0NJUUNBc0ZYU0p0STlrSzc2eFM3eUZkNDMKZWhWZ2RNMm9NU2QwcUp1dzVqTFRIekFLQmdncWhrak9QUVFEQXdObkFEQmtBakF6LzU3a1V6Wm1mT05ZdW45TgpnNnhYZ3doWUdCTWhVUXlmYThZOGkvZzFuSnZmWG4zd0wyZERaK0I2R3FzM3R2WUNNRkc5VDU2eW0rdW1JRlNyCnVHUmZKWFdOWHpSRXlYemdWL0s1TnRZTFpNMEx3enVncUZFNEdXaTFRR09WdUpiM0hRPT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQotLS0tLUJFR0lOIENFUlRJRklDQVRFLS0tLS0KTUlJRVZ6Q0NBaitnQXdJQkFnSVJBSU9QYkdQT3NUbU1ZZ1ppZ3hYSi9kNHdEUVlKS29aSWh2Y05BUUVMQlFBdwpUekVMTUFrR0ExVUVCaE1DVlZNeEtUQW5CZ05WQkFvVElFbHVkR1Z5Ym1WMElGTmxZM1Z5YVhSNUlGSmxjMlZoCmNtTm9JRWR5YjNWd01SVXdFd1lEVlFRREV3eEpVMUpISUZKdmIzUWdXREV3SGhjTk1qUXdNekV6TURBd01EQXcKV2hjTk1qY3dNekV5TWpNMU9UVTVXakF5TVFzd0NRWURWUVFHRXdKVlV6RVdNQlFHQTFVRUNoTU5UR1YwSjNNZwpSVzVqY25sd2RERUxNQWtHQTFVRUF4TUNSVFV3ZGpBUUJnY3Foa2pPUFFJQkJnVXJnUVFBSWdOaUFBUU5DenFLCmEyR090dS9jWDFqbnhrSkZWS3RqOW1aaFNBb3VXWFcwZ1FJM1VMYy9Gbm5jbU95aEtKZHlJQndzejlWOFVpQk8KVkhoYmhCUnJ3SkN1aGV6QVVVRThXb2QvQmszVS9tRFIrbXd0NFgyVkVJaWlDRlFQbVJwTTV1b0tyTmlqZ2ZndwpnZlV3RGdZRFZSMFBBUUgvQkFRREFnR0dNQjBHQTFVZEpRUVdNQlFHQ0NzR0FRVUZCd01DQmdnckJnRUZCUWNECkFUQVNCZ05WSFJNQkFmOEVDREFHQVFIL0FnRUFNQjBHQTFVZERnUVdCQlNmSzEvUFBDRlBuUVMzN1Nzc3hNWncKaTlMWERUQWZCZ05WSFNNRUdEQVdnQlI1dEZubWU3Ymw1QUZ6Z0FpSXlCcFk5dW1iYmpBeUJnZ3JCZ0VGQlFjQgpBUVFtTUNRd0lnWUlLd1lCQlFVSE1BS0dGbWgwZEhBNkx5OTRNUzVwTG14bGJtTnlMbTl5Wnk4d0V3WURWUjBnCkJBd3dDakFJQmdabmdRd0JBZ0V3SndZRFZSMGZCQ0F3SGpBY29CcWdHSVlXYUhSMGNEb3ZMM2d4TG1NdWJHVnUKWTNJdWIzSm5MekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBZ0VBSDNLZE5FVkNRZHFrMExLeXVOSW1US2RSSlkxQwoydXcyU0phanVocWt5R1BZOEMrenpzdWZaK21nbmhucTFBMktWUU9TeWtPRW5VYngxY3k2MzdyQkFpaHg5N3IrCmJjd2JaTTZzVERJYUVyaVIvUExrNkxLczlCZTB1b1Z4Z09LRGNwRzlzdkQzM0orRzlMY2Z2MUs5bHVEbVNUZ0cKNlhORklONXZmSTVncy9sTVB5b2pFTWRJeks5YmxjbDIvMXZLeE84V0dDY2p2c1ExbkovUHd0OExRWkJmT0Z5VgpYUDh1YkFwL2F1M2RjNEVLV0c5TU81emN4MXFUOStOWFJHZFZXeEd2bUJGUkFhamNpTWZYTUUxWnVHbWszL0dPCmtvQU03WmtqWm1sZXlva1AxTEd6bWZKY1VkOXM3ZWV1MS85L2VnNVhsWGQvNTVHdFlqQU0rQzRERzVpN2VhTnEKY20yRit5eFlJUHQ2Y2JidFlWTkpDR2ZIV3FIRVE0RllTdFV5Rm52OHNqeXFVOHlwZ1phTko5YVZjV1NJQ0xPSQpFMS9Rdi83b0tzblpDV0o5MjZ3VTZScUcxT1lQR09pMXp1QUJoTHc2MWN1UFZEVDI4blFTL2U2ejk1Y0pYcTBlCksxQmNhSjZmSlpzbWJqUmdENXAzbXZFZjV2ZFFNN01DRXZVMHRIYnN4Mkk1bUhISm9BQkhiOEtWQmdXcC9sY1gKR1dpV2FlT3lCN1JQK09mRHR2aTJPc2FweFhpVjd2TlZzN2ZNbHJSalkxam9LYXFtbXljbkJ2QXExNEFFYnR5TApzVmZPUzY2QjhhcGtlRlgyTlk0WFBFWVY0WlNDZThWSFByZHJFUmsyd0lMRzNUL0VHbVNJa0NZVlVNU25qbUpkClZRRDlGNk5hLyt6bVhDYz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQotLS0tLUJFR0lOIENFUlRJRklDQVRFLS0tLS0KTUlJQmF6Q0NBUkNnQXdJQkFnSVVXSjVMZGo4UEg5Z1pyckRucWhoa2tWS2s1RGt3Q2dZSUtvWkl6ajBFQXdJdwpFekVSTUE4R0ExVUVBeE1JYzNkaGNtMHRZMkV3SGhjTk1qUXdPVE13TVRNeU56QXdXaGNOTkRRd09USTFNVE15Ck56QXdXakFUTVJFd0R3WURWUVFERXdoemQyRnliUzFqWVRCWk1CTUdCeXFHU000OUFnRUdDQ3FHU000OUF3RUgKQTBJQUJLMmFKM2wrS3NPUHFFVEMvTWwxRzMzUDF1d0JPOStKYVVnVFhaa0xtWEVvZzRWSlV0eHBZTStuTGJUWQpxdWtmRWdrN01EQkpmZU16VTFrZUd2cDl2Z1dqUWpCQU1BNEdBMVVkRHdFQi93UUVBd0lCQmpBUEJnTlZIUk1CCkFmOEVCVEFEQVFIL01CMEdBMVVkRGdRV0JCUUE1N1RhMU1KTjZHUXBNbWhtczhCMkcrVWVGakFLQmdncWhrak8KUFFRREFnTkpBREJHQWlFQTdiTTRadFRCbmRpalNpQUI1QWV1Y0JvZjhNeTNmejRvSlVONEVia1FIMk1DSVFENgp5OWVRLzBtQnFvNENjb0t0UDA1S0crVWF0TUlWQWVpSFFZeFg4V0gvMkE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCi0tLS0tQkVHSU4gQ0VSVElGSUNBVEUtLS0tLQpNSUlCZmpDQ0FTU2dBd0lCQWdJVUR0R1hUOGFJZ2tVdlUrZGZmYlR0dHFEMEkvZ3dDZ1lJS29aSXpqMEVBd0l3CkhURWJNQmtHQTFVRUF4TVNWVU5RSUVOc2FXVnVkQ0JTYjI5MElFTkJNQjRYRFRJME1Ea3pNREV6TWpjd01Gb1gKRFRRME1Ea3lOVEV6TWpjd01Gb3dIVEViTUJrR0ExVUVBeE1TVlVOUUlFTnNhV1Z1ZENCU2IyOTBJRU5CTUZrdwpFd1lIS29aSXpqMENBUVlJS29aSXpqMERBUWNEUWdBRUl5Z1dQMExhYlJXcS80ZDFtcjhFbGVZTC9QcHM0bGN6CnFJSzYxMlptdFEzcjJoKzRvVXJiQklraTZEN0swRUM0dlBYK2JLVGdRUys2ckdwZzFub2lNcU5DTUVBd0RnWUQKVlIwUEFRSC9CQVFEQWdFR01BOEdBMVVkRXdFQi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZCaFRmT1Jnc09hWApBK1VTKzRJTzdtNVdCQXRiTUFvR0NDcUdTTTQ5QkFNQ0EwZ0FNRVVDSVFER2c3WG80Z3FscXJQQ0pRKzBWUGt4ClpNVXc4Rm1JNURBMjVsSWJFRVZIT1FJZ0xiRHozZFdlVWtYUHdYWGZoeFFEakJLNGdaL1A2cStsRVp4cWNOMG0KcUhZPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCi0tLS0tQkVHSU4gQ0VSVElGSUNBVEUtLS0tLQpNSUlCZWpDQ0FTQ2dBd0lCQWdJVWVwL2JHNHN5K0NnTmNoL0ZZd1o0UTJybk1pUXdDZ1lJS29aSXpqMEVBd0l3Ckd6RVpNQmNHQTFVRUF4TVFUVXRGSUdWMFkyUWdVbTl2ZENCRFFUQWVGdzB5TkRBNU16QXhNekkzTURCYUZ3MDAKTkRBNU1qVXhNekkzTURCYU1Cc3hHVEFYQmdOVkJBTVRFRTFMUlNCbGRHTmtJRkp2YjNRZ1EwRXdXVEFUQmdjcQpoa2pPUFFJQkJnZ3Foa2pPUFFNQkJ3TkNBQVNnMmhPa3lrY0NtOEhEb1dMN0t6VGpMeDFMemNxL2hkNnRlVjdLCnQvRmhDc2xUV3ZKZUlHK2hONnVWZDh3aGZpNEt6b2VEc3dqWk40SEdOdWdFMFlVcG8wSXdRREFPQmdOVkhROEIKQWY4RUJBTUNBUVl3RHdZRFZSMFRBUUgvQkFVd0F3RUIvekFkQmdOVkhRNEVGZ1FVSXY4VXlaSHJuOWVZUy9kRQpnaTVqUk9vekFtZ3dDZ1lJS29aSXpqMEVBd0lEU0FBd1JRSWhBS2Jqd1NkQTY5YlR2TnZKUlVybFZJU05taHhRCkhxTUd3aGhyeGdqdno5WllBaUJ3ODdTQ0JUR1JyUkJpUUcrVjBRL1o0RnBzWmNydnFjMHVKTktkdm1MR1BRPT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://mke.pac.dockerps.io:6443
  name: mke.pac.dockerps.io
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://keycloak.k8s.pac.dockerps.io/realms/pac.dockerps.io
      - --oidc-client-id=kubernetes
      - --oidc-use-pkce
      command: kubectl
      env: null
      interactiveMode: IfAvailable
      provideClusterInfo: false
```

This file has no credentials, and can be shared as-is with every users of this cluster.

- Test kubectl

```sh
$ export KUBECONFIG=$(pwd)/kube-oidc.yml
$ kubectl get ns
NAME                     STATUS   AGE
default                  Active   22d
ingress-nginx            Active   45h
keycloak                 Active   175m
kube-node-lease          Active   22d
kube-public              Active   22d
kube-system              Active   22d
node-feature-discovery   Active   22d
```

Because we used `admin-user`, it has MKE admin permissions and is able to see namespaces.

In order to force logout, clear the session in Keycloak and remove the cache token locally

```sh
$ rm -r ~/.kube/cache/oidc-login
```

Note : OIDC tokens are valid for 5 minutes, even if the Session is deleted in Keycloak

# 4. Create the RBAC in MKE and Kubernetes

### 4.1 MKE Organizations and Teams

First, we need to create Organizations and Teams and MKE, and map them to the correct claim `iam_roles`

- Get MKE token

```sh
MKE_URL=mke.pac.dockerps.io
MKE_USER=admin
MKE_PASSWORD=<redacted>
TOKEN=$(curl -sk -d '{"username":"'${MKE_USER}'","password":"'${MKE_PASSWORD}'"}' https://${MKE_URL}/auth/login | jq -r .auth_token)
```

- Create Organization myorg

```sh
curl -X POST "https://${MKE_URL}/accounts" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"isOrg\": true, \"name\": \"myorg\"}"
```

- Create Teams
  - myorg/admin
  - myorg/edit
  - myorg/view

```sh
curl -X POST "https://${MKE_URL}/accounts/myorg/teams" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"name\": \"admin\"}"
curl -X POST "https://${MKE_URL}/accounts/myorg/teams" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"name\": \"edit\"}"
curl -X POST "https://${MKE_URL}/accounts/myorg/teams" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"name\": \"view\"}"
```

- Map Teams and OIDC claim
  - myorg/admin --> iam_roles=myorg:admin
  - myorg/edit --> iam_roles=myorg:edit
  - myorg/view --> iam_roles=myorg:view

```sh
curl -X PUT "https://${MKE_URL}/accounts/myorg/teams/admin/kaasRoleConfig" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"enableIamRole\": true, \"roleName\": \"myorg:admin\"}"
curl -X PUT "https://${MKE_URL}/accounts/myorg/teams/edit/kaasRoleConfig" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"enableIamRole\": true, \"roleName\": \"myorg:edit\"}"
curl -X PUT "https://${MKE_URL}/accounts/myorg/teams/view/kaasRoleConfig" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"enableIamRole\": true, \"roleName\": \"myorg:view\"}"
```

- Repeat the same operations for myotherorg

```sh
curl -X POST "https://${MKE_URL}/accounts" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"isOrg\": true, \"name\": \"myotherorg\"}"
curl -X POST "https://${MKE_URL}/accounts/myotherorg/teams" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"name\": \"admin\"}"
curl -X POST "https://${MKE_URL}/accounts/myotherorg/teams" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"name\": \"edit\"}"
curl -X POST "https://${MKE_URL}/accounts/myotherorg/teams" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"name\": \"view\"}"
curl -X PUT "https://${MKE_URL}/accounts/myotherorg/teams/admin/kaasRoleConfig" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"enableIamRole\": true, \"roleName\": \"myotherorg:admin\"}"
curl -X PUT "https://${MKE_URL}/accounts/myotherorg/teams/edit/kaasRoleConfig" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"enableIamRole\": true, \"roleName\": \"myotherorg:edit\"}"
curl -X PUT "https://${MKE_URL}/accounts/myotherorg/teams/view/kaasRoleConfig" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{ \"enableIamRole\": true, \"roleName\": \"myotherorg:view\"}"
```

**Testing**

- Check there are no users in `Organization=myorg,Team=edit`

```sh
curl -X GET "https://${MKE_URL}/accounts/myorg/teams/edit/members" -H "accept: application/json" -H "Authorization: Bearer $TOKEN"
{
 "members": [],
 "nextPageStart": "",
 "resourceCount": 0
}
```

- Now log in to MKE UI, or use kubectl as user `dev-myorg-1`

```sh
$ kubectl get pods
Error from server (Forbidden): pods is forbidden: User "6ca8bea5-0993-48f2-b279-9b3cbe3da042" cannot list resource "pods" in API group "" in the namespace "default": access denied
````

This is normal because this user has no permissions in Kubernetes, but we want to verify that is being identified and correctly added to the MKE Team

- Check the JWT token

```sh
$ kubectl oidc-login get-token --oidc-issuer-url=https://keycloak.k8s.pac.dockerps.io/realms/pac.dockerps.io --oidc-client-id=kubernetes --oidc-use-pkce | jq -r .status.token | jwt decode -

Token header
------------
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "wiE6GVet_yj4G4ikYvO3U3qtlcUgIUB9eJhf3AVRwzg"
}

Token claims
------------
{
  "acr": "1",
  "at_hash": "rDMiQLUL_M5qW0by_8_UoQ",
  "aud": "kubernetes",
  "auth_time": 1729687353,
  "azp": "kubernetes",
  "email": "dev-myorg-1@noreply.org",
  "email_verified": true,
  "exp": 1729687653,
  "family_name": "Myorg1",
  "given_name": "Dev1",
  "iam_roles": [
    "kube-access",
    "myorg:edit"
  ],
  "iat": 1729687353,
  "iss": "https://keycloak.k8s.pac.dockerps.io/realms/pac.dockerps.io",
  "jti": "4682533f-db4d-430f-bce7-a637977ce5e9",
  "name": "Dev1 Myorg1",
  "nonce": "xWnwCKU74UMa-8YMfxbv3-SDhOF5kqUydqxvjWvJKXw",
  "preferred_username": "dev-myorg-1",
  "sid": "aad62867-6586-4369-8384-a3fc4ba6ded2",
  "sub": "b4b692c7-1db9-45b3-a42c-b876b4b063a6",
  "typ": "ID"
}
```

User `dev-myorg-1` has the claim `iam_roles=kube-access` which gives access to MKE, and `iam_roles=myorg:edit` which should add the user to the MKE team `Organization=myorg,Team=edit`

- Check the users in MKE team `Organization=myorg,Team=edit`

```sh
curl -X GET "https://${MKE_URL}/accounts/myorg/teams/edit/members" -H "accept: application/json" -H "Authorization: Bearer $TOKEN"

{
 "members": [
  {
   "member": {
    "name": "b4b692c7-1db9-45b3-a42c-b876b4b063a6",
    "id": "6ca8bea5-0993-48f2-b279-9b3cbe3da042",
    "fullName": "Dev1 Myorg1",
    "isOrg": false,
    "isAdmin": false,
    "isActive": true,
    "isImported": false,
    "onDemand": true,
    "otpEnabled": false,
    "locked": false
   },
   "isAdmin": false
  }
 ],
 "nextPageStart": "",
 "resourceCount": 1
}
```

`dev-myorg-1` has been added to the MKE team `Organization=myorg,Team=edit`


### 4.2 Kubernetes RBAC

Now that Organizations and Teams exist, we can create the Kubernetes RBAC

- Create Namespaces

```sh
kubectl create ns myorg-project-1
kubectl create ns myorg-project-2

kubectl create ns myotherorg-project-1
kubectl create ns myotherorg-project-2
```

- Create RoleBindings with corresponding Teams

Example : Team myorg/admin will get admin access to namespaces myorg-project-*

```sh
# myorg projects
kubectl -n myorg-project-1 create rolebinding myorg:admin-admin --clusterrole admin --group team:myorg:admin
kubectl -n myorg-project-1 create rolebinding myorg:edit-edit --clusterrole edit --group team:myorg:edit
kubectl -n myorg-project-1 create rolebinding myorg:view-view --clusterrole view --group team:myorg:view

kubectl -n myorg-project-2 create rolebinding myorg:admin-admin --clusterrole admin --group team:myorg:admin
kubectl -n myorg-project-2 create rolebinding myorg:edit-edit --clusterrole edit --group team:myorg:edit
kubectl -n myorg-project-2 create rolebinding myorg:view-view --clusterrole view --group team:myorg:view

# myotherorg projects
kubectl -n myotherorg-project-1 create rolebinding myotherorg:admin-admin --clusterrole admin --group team:myotherorg:admin
kubectl -n myotherorg-project-1 create rolebinding myotherorg:edit-edit --clusterrole edit --group team:myotherorg:edit
kubectl -n myotherorg-project-1 create rolebinding myotherorg:view-view --clusterrole view --group team:myotherorg:view

kubectl -n myotherorg-project-2 create rolebinding myotherorg:admin-admin --clusterrole admin --group team:myotherorg:admin
kubectl -n myotherorg-project-2 create rolebinding myotherorg:edit-edit --clusterrole edit --group team:myotherorg:edit
kubectl -n myotherorg-project-2 create rolebinding myotherorg:view-view --clusterrole view --group team:myotherorg:view
```

## Tests

- admin-user
  - can use MKE
  - is MKE admin

- User random-user has no access
  - can't use MKE

- dev-myorg-1
  - can use MKE
  - has edit permissions on `myorg-project-1` and `myorg-project-2` namespaces

```sh
# myorg-project-1
$ kubectl -n myorg-project-1 create deploy itworks --image nginx:latest --replicas=1
deployment.apps/itworks created
k -n myorg-project-1 get pods
NAME                       READY   STATUS    RESTARTS   AGE
itworks-6d9cbcf989-r2dpg   1/1     Running   0          3m45s

# myotherorg-project-1
$ kubectl -n myotherorg-project-1 create deploy itworks --image nginx:latest --replicas=1
error: failed to create deployment: deployments.apps is forbidden: User "6ca8bea5-0993-48f2-b279-9b3cbe3da042" cannot create resource "deployments" in API group "apps" in the namespace "myotherorg-project-1": access denied
```

- dev-myorg-1
  - can use MKE
  - has admin permissions on `myotherorg-project-1` and `myotherorg-project-2` namespaces

# Debugging

- In kube apiserver debug logs, you can see the Groups

```sh
I1022 13:37:07.524087       1 queueset.go:953] QS(global-default) at t=2024-10-22 13:37:07.524056577 R=6.96011510ss: request &request.RequestInfo{IsResourceRequest:true, Path:"/api/v1/pods", Verb:"list", APIPrefix:"api", APIGroup:"", APIVersion:"v1", Namespa
ce:"", Resource:"pods", Subresource:"", Name:"", Parts:[]string{"pods"}} &user.DefaultInfo{Name:"https://keycloak.k8s.pac.dockerps.io/realms/pac.dockerps.io#toto", UID:"", Groups:[]string{"oidc:kube-access", "oidc:myorg:dev", "system:authenticated"}, Extra:map[string][]string(nil)} finished all use of 2 seats, adjusted queue 23 start R to 6.95972077ss due to service time 0.016294132s, queue will have 0 requests with queueset.queueSum{InitialSeatsSum:0, MaxSeatsSum:0, TotalWorkSum:0x0} waiting & 0 requests occupying 0 seats
```

> Groups:[]string{"oidc:kube-access", "oidc:myorg:dev", "system:authenticated"}

- Remove kubelogin cache

```sh
rm -r ~/.kube/cache/oidc-login
```

# TODO

- Get username instead of ID in MKE
- Scenarios with adding/removing users and groups
  - Wait 3 minutes for kubectl cache after a user is added to a MKE team
  - Wait 1 minute for kubectl cache after a user is removed from MKE team
