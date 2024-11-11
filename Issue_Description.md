## Expected Behavior

I use the oauth2-proxy as a sidecar to a K8s container-application which has to be secured using SSO (using OIDC) and relying on internal roles for access control. Since my application is accessed both from an UI as well as an API (MLFlow server), there are two different access methods in use:
1. Access Application using the webbrowser (K8s nginx-ingress forwards the request to the oauth2-proxy which opens the SSO Sign-In page and handles the authentication of the user)
2. Access API from code (e.g. Python-App, JupyterHub cells)

In other words, I expect that the user should be able to either access the application from the browser or via API-requests from some code as long as the user has the nessecary role/group access. For the second option, it is expected that the JWT Bearer access-token which is added to the API-request is validated by the proxy using the Identity-Provider endpoints given in the proxy-configuration or retrieved during OIDC discovery. This validation/authentication must include the check if the email-address-provider is valid as well as whether the user whose access-token was sent with the request has the nessecary roles/groups as defined in --allowed-group (just as it is done when accessing the application from the browser).

## Current Behavior

The first access method works as expected (role-based access control works fine). However, the second method doesn't work so far. The API request (using postman for this test) which is forwarded to the oauth2-proxy by the ingress consists of an JWT Bearer token (i.e. access token and NOT the ID-token):

```
15:08:41.567 GET https://<internal-url>/api/2.0/mlflow/runs/get?run_uuid=<run-uuid>&run_id=<run-id>: {
  "Warning": "Self signed certificate in certificate chain",
  "Network": {
    <network-stuff>
  },
  "Request Headers": {
    "authorization": "Bearer <valid-access-token>",
    "user-agent": "PostmanRuntime/7.32.2",
    "accept": "*/*",
    "postman-token": "<postman-token>",
    "host": "<application-url>",
    "accept-encoding": "gzip, deflate, br",
    "connection": "keep-alive",
    "cookie": "_oauth2_proxy_csrf_AHRBWiF_=<cookie-hash>=; _oauth2_proxy_csrf_dHOCq3RH=<cookie-hash>=; _oauth2_proxy_csrf_ue6lr93A=<cookie-hash>="
  },
  "Response Headers": {
    "date": "Wed, 21 Jun 2023 13:08:41 GMT",
    "content-type": "text/html; charset=utf-8",
    "transfer-encoding": "chunked",
    "connection": "keep-alive",
    "strict-transport-security": "max-age=15724800; includeSubDomains",
    "content-security-policy": "frame-ancestors *",
    "x-content-type-options": "nosniff",
    "x-frame-options": "sameorigin",
    "x-xss-protection": "1; mode=block"
  },
  "Response Body": "\n<!DOCTYPE html>\n<html lang=\"en\" charset=\"utf-8\">\n<head>\n<meta charset=\"utf-8\">\n<meta name=\"viewport\" content=\"width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no\">\n  <title>403 Forbidden</title>\n ... and more \n</html>\n"
}
```

The expected behaviour is that the oauth2-proxy fetches the access-token (which it does), creates a new session where the access-token is validated (works fine too) and then the last authorization step should consist of checking if the --allowed-group are met by the provided access-token (this check is also done). However, currently, the proxy blocks the request and returns the following error message, even if the user has the nessecary roles (checked via the /userinfo endpoint):

```
27.0.0.1:54000 - <some-hash> - <hidden-email> [2023/06/21 12:40:29] [2023-06-21T14:40:29.923814424+02:00 AuthFailure] Invalid authorization via session: removing session Session{email:<hidden-email> user:<some-user-id-string> PreferredUsername: token:true id_token:true expires:2023-06-21 12:42:27 +0000 UTC}
127.0.0.1:54000 - <some-hash> - 2023-06-21T14:40:29.924159375+02:00 <hidden-email> [2023/06/21 12:40:29] <application-url> GET - "/api/2.0/mlflow/runs/get?run_uuid=<run-hash>&run_id=<run-id>" 2023-06-21T14:40:29.924197710+02:00 HTTP/1.1 "mlflow-python-client/2.3.2" 403 2023-06-21T14:40:29.924230267+02:00 2889 0.263
```

The reason seems to be that the access-token does not contain any groups/roles and hence, the proxy simply builds a session for a user without any groups. This means, that the proxy never calls the /userinfo endpoint to retrieve the roles of the user associated with the given access token.

## Possible Solution

I expect the proxy to call the profileURL (/userinfo endpoint of the Identity Provider) with the provided access-token to retrieve the roles/groups of the user. I think that this is how the gitlab-flow is implemented as well. From my perspective, this is a bug since we provide --allowed-group and hence the proxy is supposed to also check the groups of a provided access-token (each access-token is uniquely assigned to an user which has specific roles/groups which are supposed to be checked for each call).

I have started to implement a solution, which calls the /userinfo endpoint and extracts missing information (i.e. groups) from there. If I didn't miss some setting that thakes care of this solution, I can create the PR where my suggested changes are implemented and tested (with the changes, the roles are retrieved from the /userinfo endpoint for each session which does not have roles yet).

## Steps to Reproduce (for bugs)

The following options are provided through the config-file when building our custom oauth2-proxy sidecar (since they are the same for several applications, we define them in the base-sidecar):

```
provider="oidc"
provider_ca_files="<path-to-certificates>"
http_address="0.0.0.0:<oauth2-proxy-sidecar-port>"
upstream_timeout="10m"
insecure_oidc_allow_unverified_email="true"
session_cookie_minimal="true"
cookie_secure="true"
ping_path="/api/health"
cookie_domains="<our-cookie-domains>"
whitelist_domains="<our-cookie-domains>"
cookie_csrf_per_request="true"
cookie_samesite="none"
```

The following options are provided through the kubernetes configuration (since they are application specific):

```
--upstream=http://127.0.0.1:<container-application-port>
--skip-jwt-bearer-tokens=true
--code-challenge-method=S256
--oidc-issuer-url=https://<our-idP-url>/oauth/token
--email-domain=<company-email-domain>
--profile-url=https://<our-idP-url>/userinfo
--extra-jwt-issuers=https://<our-idP-url>/oauth/token=<client-id-2>
--oidc-groups-claim=roles
--scope=openid roles profile user_attributes
--allowed-group=<some-group>
```

The client-id, client-secret as well as cookie-secret are stored in kubernetes secrets and added as environment variables and HTTPS_PROXY, HTTP_PROXY and NO_PROXY env-variables are deployed from a config-map.

## Context

I'm trying to use the oauth2-proxy to provide access to an API to both users without valid authentication and those who already have a valid access token. The users which have no valid Bearer access token, should be forwarded to the Identity provider (using SSO to obtain the cookie) while those that already have a valid access token, should be let through to the API without being forwarded to the login-page. However, valid access tokens alone should not permit the user to access the API since access should only be given to users who are also a member of the specified group(s) (--allowed-group). Since the access-tokens come from clients with different client-ids as the oauth2-proxy's client-id, I also use the --extra-jwk-issuer option as shown above. Furthermore, the email-provider is limited to company-emails.

## Your Environment

I use an internal Identity provider based on OIDC and CloudFoundry. Hence, the groups-claim is "roles". The oauth2-proxy runs on K8s as a sidecar to the application-container. After successful authentification, the sidecar forewards the request to the localhost:<app-port> uri since the sidecar runs in the same pod as the main application. I also use two separate K8s-services to access the application, one used for external requests who come through the ingress, the other service to handle requests from other pods within the same namespace.

- Oauth2-proxy version used: 7.4.3
