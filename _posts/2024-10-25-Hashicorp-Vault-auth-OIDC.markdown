--- 
title: "Configuring Google Workspace as an OIDC identity provider for Hashicorp Vault"
social:
  name: Dr Richard Kang
  links:
    - https://www.linkedin.com/in/kangks/
---

# Table of Contents

1. [Overview](#Overview)
2. [Business Outcome](#business-outcome)
2. [Prerequisites](#prerequisites)
3. [Step 1: Setting Up the OAuth Consent Screen](#step-1-setting-up-the-oauth-consent-screen)
4. [Step 2: Creating OAuth 2.0 Client IDs](#step-2-creating-oauth-2.0-client-ids)
5. [Step 3: Setting Up HashiCorp Vault](#step-3-setting-up-hashicorp-vault)
6. [Step 4: Enabling OIDC Authentication](#step-4-enabling-oidc-authentication)
7. [Step 5: Configuring the OIDC Role](#step-5-configuring-the-oidc-role)
8. [Step 6: Logging in with OIDC](#step-6-logging-in-with-oidc)
9. [Conclusion](#conclusion)

### Overview

This blog post serves as a comprehensive guide to integrating Google Workspace as an OpenID Connect (OIDC) identity provider with HashiCorp Vault. By leveraging OIDC, organizations can streamline authentication for users accessing Vault, enhancing security and user experience. The guide provides detailed, step-by-step instructions covering the OAuth configuration in Google Workspace, setting up a Vault server in development mode, configuring OIDC authentication, and enabling the necessary callback URIs. The tutorial aims to simplify the process of implementing a secure, standards-compliant authentication method, making it easier for teams to authenticate and manage access to critical secrets in Vault.

### Prerequisites
The administrator-level access to the organization’s Google Workspace account is required to create a new project, or permissions to create and configure OAuth 2.0 credentials for an existing project. 

### Step 1: Fill out the OAuth consent screen
1. In the Google Cloud console, go to  menu > APIs & Services > [OAuth consent screen](https://console.cloud.google.com/apis/credentials/consent).
2. Select project, or create new project if does not have any. ![GCP console > choose project](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/gcp_console_select_project.png)
3. From the left navigation ment, select APIs and services > OAuth consent screen . ![GCP console > choose APIs](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/gcp_console-api-oauth_screen.png)
3. In the user type for your app, choose "Internal", then click Create. ![GCP console > choose Internal](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/gcp_console-api-oauth_screen-internal.png)
3. Complete the app registration form, then click Save and Continue.
4. On the Scopes page leave everything blank and choose Save and Continue.

The OAuth consent screen for the selected project is now configured.![GCP console > consent created](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/gcp_console-api-oauth_screen-completed.png)

### Step 2: Creates OAuth 2.0 Client IDs
1 In the Google Cloud consle, go to APIs & Services > [Credentials](https://console.cloud.google.com/apis/credentials).
2. Choose "CREATE CREDENTIALS" from the top, and choose "OAuth client ID" ![GCP console > OAuth Client ID](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/gcp_console-api-credentials-create_credentials.png)
3. In the "Applcation type", choose "Web application" ![GCP console > Web application](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/gcp_console-api-credentials-create_credentials-web_application.png)
4. Enter the client ID name.
5. In the "Authorised redirect URIs", add these 2 URI. Details of these 2 URIs can be found in [Step 4](#step-5-setting-up-oidc-role)
   1. http://localhost:8200/ui/vault/auth/oidc/oidc/callback
   2. http://localhost:8400/oidc/callback
6. Choose "CREATE" ![GCP console > CREATE](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/gcp_console-api-credentials-create_credentials-web_application-create.png)
5. Screen showing that OAuth client created. Keep the screen open because we will need the information when creating the OAuth Auth for Vault ![GCP console > OAuth Client Created](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/oauth_client_created.png)

### Step 3: Setting up Hashicorp Vault

> Note: In this tutorial, a localhost Vault will be used.

1. Open a shell terminal, and starts the Vault server with command `vault server -dev`
1. The following messages will show that the server is up
    ```
    WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
    and starts unsealed with a single unseal key. The root token is already
    authenticated to the CLI, so you can immediately begin using Vault.

    You may need to set the following environment variables:

        $ export VAULT_ADDR='http://127.0.0.1:8200'

    The unseal key and root token are displayed below in case you want to
    seal/unseal the Vault or re-authenticate.

    Unseal Key: <REDACTED>
    Root Token: hvs.<REDACTED>

    Development mode should NOT be used in production installations!
    ```
1. Open another shell terminal, set the environment variable for the VAULT endpoint:
    ```
    $ export VAULT_ADDR='http://127.0.0.1:8200'
    ```

### Step 4: Enable OIDC Auth

1. Keep the VAULT token as environment variable
    ```
    $ export VAULT_TOKEN=hvs.<REDACTED>
    ```    
1. Enable OIDC auth for Vault
    ```
    $ vault auth enable oidc
    Success! Enabled oidc auth method at: oidc/
    ```

### Step 5: Setting up OIDC Role
1. Create a role for the OIDC default role. Explainations of each parameter:
   1. The `user_claim` has to be either `sub` or `email` for Google Workspace queries to [success](https://developer.hashicorp.com/vault/docs/auth/jwt/oidc-providers/google#role). 
   2. The two `allowed_redirect_uris`:
      1. `http://localhost:8250/oidc/callback` is a [temporary local webserver on port 8250](https://developer.hashicorp.com/vault/docs/auth/jwt#redirect-uris) to receive a redirect to localhost, when  `vault login -method=oidc` is being executed.
      2. `${VAULT_ADDR}/ui/vault/auth/oidc/oidc/callback` is the callback from Google Workspace with the token
   3. `user_claim` is to uniquely identify the user.
   4. `groups_claim` is to to uniquely identify the set of groups to which the user belongs.
   5. `verbose_oidc_logging` Log received OIDC tokens and claims if the non-production server was started with `export VAULT_LOG_LEVEL=debug`.
    ```
    $ vault write auth/oidc/role/default \
    user_claim="email" \
    groups_claim="groups" \
    allowed_redirect_uris=http://localhost:8250/oidc/callback,${VAULT_ADDR}/ui/vault/auth/oidc/oidc/callback \
    verbose_oidc_logging=true
    Success! Data written to: auth/oidc/config
    ```
1. Configure the Vault OIDC auth with Google OIDC information. Reference to the screen, the mapping as followings: ![GCP console > OAuth client created](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/oauth_client_created.png)
   * oidc_discovery_url="https://accounts.google.com"
   * oidc_client_id="Client ID"
   * oidc_client_secret="Client Secret"
    ```
    vault write auth/oidc/config \
    oidc_discovery_url=https://accounts.google.com \
    oidc_client_id=<MASKED>.apps.googleusercontent.com \
    oidc_client_secret=GOCSPX-<MASKED>mpHMn \
    default_role=default
    ```


### Step 6: Login with OIDC

1. Login with OIDC method.
   1. The `port=8250` is the local server that `vault login` creates for the Google Workspace will redirect to. 
    ```
    % vault login -method=oidc port=8400 role=default
    Complete the login via your OIDC provider. Launching browser to:

        https://accounts.google.com/o/oauth2/v2/auth?client_id=630608949624-tb6c9lhieuotajo4r8oe0to0ek42nul7.apps.googleusercontent.com&code_challenge=D4VghdGdyoC-8Uaz8BMvictrP3pyDIV6TbmaQAdHkuY&code_challenge_method=S256&nonce=n_TB7BCUd8bzMpm8KSnZxL&redirect_uri=http%3A%2F%2Flocalhost%3A8400%2Foidc%2Fcallback&response_type=code&scope=openid+email+profile&state=st_jIhTjKFTbVMs8k8aoHbb


    Waiting for OIDC authentication to complete...
    ```
1. A browser should pop up for the Google login
1. Once browser login successful, the vault will display the token:
    ```
    Success! You are now authenticated. The token information displayed below
    is already stored in the token helper. You do NOT need to run "vault login"
    again. Future Vault requests will automatically use this token.

    Key                  Value
    ---                  -----
    token                hvs.<OIDC TOKEN>
    token_accessor       <REDACTED>
    token_duration       768h
    token_renewable      true
    token_policies       ["default"]
    identity_policies    []
    policies             ["default"]
    token_meta_role      default
    ```

Note: If the port mismatch with OAuth callback, Google workspace will throw Error 400: ![GCP console > error 400](/assets/images/2024-10-25-Hashicorp-Vault-auth-OIDC/vault_login_oidc-port-mismatch.png)

## Conclusion

Configuring Google Workspace as an OpenID Connect (OIDC) provider for HashiCorp Vault enhances the security, scalability, and usability of your organization’s authentication processes. By following the steps outlined, teams can leverage Google’s identity management to streamline secure access to Vault, reducing the risks associated with multiple credentials and manual access controls. This setup allows organizations to provide consistent, controlled access to sensitive data while integrating seamlessly with existing identity providers. Adopting OIDC with Vault not only strengthens compliance and security but also promotes operational efficiency, empowering teams to focus on core tasks within a unified, secure infrastructure.