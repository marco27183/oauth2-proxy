<!--- Provide a general summary of your changes in the Title above -->

## Description

The session state is now automatically updated for sessions which do not contain role/group information yet.

## Motivation and Context

<!--- Why is this change required? What problem does it solve? -->
I want to be able to use the oauth2-proxy in front of some K8s application and access this application in various ways:
1. Access the application via the url using some browser. This case works fine without the change.
2. Access the application from JupyterHub and/or code using the Authorization-code flow. The authorization code flow is required since  access to the application is only granted to specific users which are part of a group and hence m2m cannot be used. Since this access flow requires SSO, we handle the retrieval of the access-token and refresh-token separately and call the service API behind the oauth2-proxy with a JWT Bearer access-token.

The access-token does not contain any information about the users groups but they can easily be retrieved using the /userinfo endpoint of our OIDC Provider using the access-token. One further complication is the fact that the groups are 
In addition, there are multiple different Client-IDs in use for different access-methods (e.g. browser-access and JupyterHub utilize different client credentials).
Now, the issue is that using the --allowed-groups configuration-option and accessing the Service-API through the oauth2-Proxy, the request gets blocked since the session state has no roles. However, when accessing the service using the browser, it works fine and only users with the respective role are admitted.

As far as I can tell, the issue is that the roles/groups are never retrieved from the /userinfo endpoint and therefore, the request cannot be admitted. I also checked other providers and in the case of gitlab, the enrichment is implemented as I would expect it.

In order to get it to work for me, I changed the oidc.go as well as the oauth2_proxy.go files and this solved my problem (i.e. access is controlled by the user roles independent of the access method). However, since I'm not too familiar with all other providers and the full code-base of this proxy, I would greatly appreciate some feedback and I'm happy to work on this issue to correctly implement this functionality.

<!--- If it fixes an open issue, please link to the issue here. -->

## How Has This Been Tested?

<!--- Please describe in detail how you tested your changes. -->
- All test cases have been run (go test for each submodule)
- Testing extensively with our OIDC provider
<!--- Include details of your testing environment, and the tests you ran to -->
<!--- see how your change affects other areas of the code, etc. -->

## Checklist:

<!--- Go over all the following points, and put an `x` in all the boxes that apply. -->
<!--- If you're unsure about any of these, don't hesitate to ask. We're here to help! -->

- [ ] My change requires a change to the documentation or CHANGELOG.
- [ ] I have updated the documentation/CHANGELOG accordingly.
- [ ] I have created a feature (non-master) branch for my PR.
