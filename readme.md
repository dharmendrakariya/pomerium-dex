### Pomerium-Dex-Freeipa Exercise

**This exercise depicts the authentication flow for the services which don't have authentication flow**

*Flow with the diagram*

![alt text](https://github.com/dharmendrakariya/Learning/blob/master/pomerium/image.jpg?raw=true)


1. User makes an unauthenticated request to the service

2. Pomerium proxy receives the request and recognizes it as anonymous

3. It redirects the user to the auth provider for authentication

4. Upon successful login, Pomerium provides an auth cookie to the user. In the picture you can see the dex approval page before that.

5. Based on the cookie, Pomerium identifies the user and checks policy to determine whether to permit access. Authorization is based on identity factors like id, email,      group, role, or email domain.

6. When the cookie expires, the login flow gets triggered all over again.


*Here is our flow for accessing nextcloud service*

1. User access https://hello.YOURDOMAIN.dev

2. It will be redirected to the https://authenticate.YOURDOMAIN.dev (which is pomerium's authenticate service url)

3. Pomerium's authenticate service will redirect this to check at oidc provider( in our case DEX).

4. Dex(which is backed by FreeIpa in our case, freeipa's LDAP as backend) will check if the user is valid or not and after that flow gets redirected to pomerium back if user is valid.

5. User is finally redirected to the nextcloud sevrice if all goes well.


Now to implement this flow we have configured static dex client ```pom``` with pomerium's authenticate service redirectURL

```
connectors:
      - config:
          bindDN: uid=dex,cn=sysaccounts,cn=etc,dc=YOURDOMAIN,dc=dev
          bindPW: mN****tG****
          host: ipa.YOURDOMAIN.dev:636
          insecureNoSSL: false
          insecureSkipVerify: true
          groupSearch:
            baseDN: cn=groups,cn=accounts,dc=YOURDOMAIN,dc=dev
            filter: "(|(objectClass=posixGroup)(objectClass=group))"
            userAttr: DN # Use "DN" here not "uid"
            groupAttr: member
            nameAttr: cn
          userSearch:
            baseDN: cn=users,cn=accounts,dc=YOURDOMAIN,dc=dev
            emailAttr: mail
            filter: ""
            idAttr: uidNumber
            nameAttr: displayName
            preferredUsernameAttr: uid
            username: mail
          usernamePrompt: Email
        id: ldap
        name: FreeIPA/LDAP
        type: ldap
      issuer: http://dex.YOURDOMAIN.dev
      logger:
        level: debug
      oauth2:
        responseTypes:
        - code
        skipApprovalScreen: false
      staticClients:
      - id: pom
        name: pom
        redirectURIs:
        - https://authenticate.YOURDOMAIN.dev/oauth2/callback
        secret: pomerium

```
Below is configuration which supposed to be done in Pomerium

```Note: I am using Pomerium helm chart```

```
config:
  # routes under this wildcard domain are handled by pomerium
  rootDomain: YOURDOMAIN.dev
    # YOURDOMAIN.dev
    # localhost.pomerium.io

  policy:
    - from: https://hello.YOURDOMAIN.dev #(give any name instead of hello, this will be the proxy url to access the particular service)
      to: http://nextcloud.nextcloud.svc.cluster.local:8080 #(give fqdn of the actual service which is being authenticated, here i=I am giving nextcloud service endpoint)
      # allowed_domains:
      #   - YOURDOMAIN.dev #(in general give here your domain)
      #   # - YOURDOMAIN.com
      allowed_groups: #(If you want to give access to particular group members, I have tested this by creating devops group and members in that group, in freeipa)
        - devops
      allowed_idp_claims: #(If you want to give access to particular group members, I have tested this by creating devops group and members in that group, in freeipa)
        groups:
        - devops

  insecure: true # (I didn't specify the root level CAs so)

extraEnv:
  POMERIUM_DEBUG: true #(This will give you details if user is not able to authenticate, ideally this should be turned off)
  LOG_LEVEL: "error"
  IDP_SCOPES: "openid,profile,email,groups,offline_access"

authenticate:
  redirectUrl: "https://authenticate.YOURDOMAIN.dev/oauth2/callback" #(This we have set in dex's static client also remember! should be same)
  idp:
    provider: oidc
    clientID: pom
    clientSecret: pomerium
    url: http://dex.YOURDOMAIN.dev #(your dex url)
    scopes: "openid profile email groups offline_access"
    serviceAccount: "pomerium-authenticate" #(for group based access policy)

ingress:
  enabled: true
  authenticate:
    name: ""
  secretName: ""
  secret:
    name: ""
    cert: ""
    key: ""
  tls:
    hosts: []
  hosts: []
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.allow-http: "true"
resources:
  limits:
    cpu: 150m
    memory: 100Mi
  requests:
    cpu: 100m
    memory: 100Mi

```


For more details and example you can visit [banzaicloud-example](https://banzaicloud.com/blog/pomerium-authentication/)

Thanks
