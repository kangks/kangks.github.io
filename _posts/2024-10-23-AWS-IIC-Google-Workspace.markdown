--- 
title: "Integrating Google Workspace as an External Identity Provider for AWS IAM Identity Center"
status: publish
layout: post
published: true
tags: 
- aws
excerpt: Use case of AWS IAM Identity Center and integrate with Google Workspace
----

## Table of Contents

  - [Overview of Centralized Identity Store Integration](#overview-of-centralized-identity-store-integration)
    - [The Importance of Centralized Identity Management](#the-importance-of-centralized-identity-management)
    - [Use Case: Mitigating Offboarding Risks](#use-case-mitigating-offboarding-risks)
    - [AWS IAM Identity Center](#aws-iam-identity-center)
  - [Step-by-Step](#step-by-step)
    - [Step 1: Google Workspace: Configure the SAML application](#step-1-google-workspace-configure-the-saml-application)
    - [Step 2: Change and setup Google Workspace as an SAML identity provider](#step-2-change-and-setup-google-workspace-as-an-saml-identity-provider)
      - [AWS Console: Configure the IdP SAML in AWS IAM Identity Center](#aws-console-configure-the-idp-saml-in-aws-iam-identity-center)
      - [Google Admin page: Configure the Service Provider details](#google-admin-page-configure-the-service-provider-details)
      - [Google Admin page: Configure the Attribute Mapping](#google-admin-page-configure-the-attribute-mapping)
      - [AWS Console: Review and confirm the use of external identity provider](#aws-console-review-and-confirm-the-use-of-external-identity-provider)
    - [Step 3: Google Admin: Enable the apps](#step-3-google-admin-enable-the-apps)
    - [Step 4: Set up IAM Identity Center automatic provisioning](#step-4-set-up-iam-identity-center-automatic-provisioning)
  - [Next step](#next-step)

### Overview of Centralized Identity Store Integration

**The Importance of Centralized Identity Management**

In today’s distributed environments, managing identities across platforms is complex and risky. A centralized identity store simplifies user access, reduces administrative burden, and enhances security. It serves as a single source of truth, ensuring that user roles and access are managed from one system. When employees leave, centralized identity management ensures that access is revoked across all systems, minimizing the risk of data breaches or unauthorized access.

**Use Case: Mitigating Offboarding Risks**

Without a centralized system, offboarding employees becomes a security liability as access may remain active across multiple platforms. By using a centralized identity store, access can be revoked instantly across all services, reducing the risk of security breaches.

**AWS IAM Identity Center**

AWS IAM Identity Center provides a default identity directory for managing AWS access, but many organizations use external providers like Google Workspace, Okta, or Microsoft Azure AD. IAM Identity Center supports these [integrations](https://docs.aws.amazon.com/singlesignon/latest/userguide/tutorials.html), enabling organizations to centralize identity management while providing seamless Single Sign-On (SSO) for AWS resources. Through this integration, administrators can automate user provisioning using SCIM and manage access in one place, enhancing security and efficiency.

## Step-by-Step

_Tips: Having two windows side by side, windows #1 on Google Workspace, windows #2 on the AWS IAM Identity Center, will be helpful in this step-by-step guides._

> This step-by-step guide augmented with screenshots and grouping by AWS Console and Google Admin for better clarity. The original documentation on AWS can be found [here](https://docs.aws.amazon.com/singlesignon/latest/userguide/gs-gwp.html).

![Side-by-Side](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_-_AWS_-_side-by-side.png)

### Step 1: Google Workspace: Configure the SAML application

1. Go to https://admin.google.com, and login using an account with super administrator privileges.

2. In the left navigation panel of your Google Admin console, choose Apps and then choose Web and Mobile Apps. ![Google Admin > Web and Mobile Apps](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google-admin-apps-mobile.png)

3. In the Add app dropdown list, select Search for apps.
![Google Admin > Web and Mobile Apps > Search for apps](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_admin_search_for_apps.png)


4. In the search box enter Amazon Web Services, then select Amazon Web Services (SAML) app from the list.
![Google Admin > Web and Mobile Apps > Search for apps > amazon web](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_Amazon_Web.png)

Keep this page of Google admin open, and open another windows for AWS IAM Identity Center. We will need to copy metadata and parameteres between the two providers.

### Step 2: Change and setup Google Workspace as an SAML identity provider

1. Go to https://console.aws.amazon.com/singlesignon/home and sign in to the IAM Identity Center console using a role with administrative permissions.

2. Choose Settings in the left navigation pane.

3. On the Settings page, choose Actions, and then choose Change identity source. ![AWS IAM Identity Console > Change Identity Source](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/AWS_IIC_Change_Identity_source.png)

3. On the Choose identity source page, select External identity provider, and then choose Next. ![AWS IAM Identity Console > Change Identity Source > External identity provider](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/AWS_IIC_Change_Identity_Source_to_External_IdentityProvider.png)

#### AWS Console: Configure the IdP SAML in AWS IAM Identity Center
5. We need to copy the following, from Google Admin to Amazon Web Services:
   1.  On the Google Identity Provider details - Amazon Web Services page:
       1. Download IdP metadata.
       2. Copy the SSO URL, Entity ID URL, and Certificate information.
   6. On the AWS IAM Identity Center
       1. Upload the IdP metadata to IdP SAML metadata
       2. Paste the values to IdP sign-on URL, IdP issuer URL, and IdP certificate.
![AWS IAM Identity Console > Update with values from Google Admin](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_-_AWS_-_side-by-side-arrows.png)

#### Google Admin page: Configure the Service Provider details

6. In the Google admin screen, select "CONTINUE" to move on to next step of configuration of Service Provider at the Google Admin ![Google Admin > configure identity provider details](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_continue_adding_Amazon_Web_Services.png)

7. Now we need to copy from AWS IAM Identity Center to Google Admin:
   1.  On the Google Admin > Service Provider:
       1. Paste the values of "IAM ACS URL" from AWS Console, to "ACS URL" in the Google Admin page;
       1. Paste the values of "IAM Issuer URL" from AWS Console, to "Entity ID" in the Google Admin page
    2. For the Name ID in the Google Admin Service Provider details page:
       1. Select "EMAIL" for the fields under Name ID;
       2. For Name ID, select Basic Information > Primary email
![Google Admin > configure identity provider details > update with values from AWS](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/AWS_IIC_to_Google_Admin_next_step.png)

8. Select "CONTINUE" at the bottom of the Google admin page.

#### Google Admin page: Configure the Attribute Mapping

1. On the Attribute Mapping page, under Attributes, choose ADD MAPPING, and then configure these fields under Google Directory attribute:

   1. For the `https://aws.amazon.com/SAML/Attributes/RoleSessionName` app attribute, select the field Basic Information, Primary Email from the Google Directory attributes. ![Google Admin > configure identity provider details > update role session name](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_Attributes_Primary_Email.png)

   1. For the `https://aws.amazon.com/SAML/Attributes/Role` app attribute, select any Google Directory attributes. A Google Directory attribute could be Department. ![Google Admin > configure identity provider details > update role](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_Attributes_Primary_Department.png)

2. Choose FINISH. ![Google Admin > configure identity provider details](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_Attributes_next.png)

3. Return to the IAM Identity Center console and choose Next. On the Review and Confirm page, review the information and then enter ACCEPT into the space provided. Choose Change identity source.

#### AWS Console: Review and confirm the use of external identity provider

1. In the AWS IAM Identity Center console, select Next. 
2. On the Review and Confirm page, review the information and then enter ACCEPT into the space provided. ![AWS IAM Identity Center > Access change to external](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/AWS_IIC_accept.png)
3. Choose Change identity source.

### Step 3: Google Admin: Enable the apps

1. In the Google Admin Console, select the newly created AWS IAM Identity Center application
2. In the User access panel, choose the down arrow.
![Google Admin > Web and Mobile Apps > User Access](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/google_admin_user_acces_down_arrow.png)
3. In Service status panel, choose ON for everyone, and then choose SAVE. ![Google Admin > Web and Mobile Apps > User Access > ON for everyone](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_service_status_enable_app_for_everyone.png)


### Step 4: Set up IAM Identity Center automatic provisioning

1. In the AWS IAM Identity Center console, choose Enable Automatic provisioning ![AWS IAM Identity Center > Enable Automatic Provisioning](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/AWS_IIC_enable_automatic_provisioning_button.png)
2. Keep the screen open ![AWS IAM Identity Center > Inbound Automatic Provisioning](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/AWS_IIC_automatic_provisioning_screen.png)

3. In the Google Admin, select the newly created AWS IAM Identity Center application
4. In the Auto-Provisiong, choose "Configure auto-provisioning" ![Google Admin > Web and Mobile Apps > amazon web > configure auto-provisioning](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_auto_provisioning.png)

5. With AWS IAM Identity Console and Google Admin windows side-by-side:
   1. Paste the value of "Access Token" in AWS IAM IDentity Console to "Access token" in Google Admin ![AWS IAM Identity Center to Google Admin access token](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/AWS_IIC_to_Google_Admin-automatic_provisioning_token.png)
   1. Paste the value of "SCIM Endpoint" in AWS IAM IDentity Console to "Endpoint URL" in Google Admin ![AWS IAM Identity Center to Google Admin endpoint url](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/AWS_IIC_to_Google_Admin-automatic_provisioning_endpoint.png)

6. In the next screen of Google Admin, verify that all mandatory IAM Identity Center attributes (those marked with *) are mapped to Google Cloud Directory attributes. If not, choose the down arrow and map to the appropriate attribute. Choose Continue. ![Google Admin > Web and Mobile Apps > mandatory IAM](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_autoprovisioning_attributes_mapping.png)

7. In Provisioning scope section, you can choose a group with your Google Workspace directory to provide access to the Amazon Web Services app. Skip this step and select Continue. ![Google Admin > Web and Mobile Apps > skip provisioning scope](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_autoprovisioning_group.png)

8. In Deprovisioning section, you can choose how to respond to different events that remove access from a user. For each situation you can specify the amount of time before deprovisioning begins to suspend and when to delete the account. ![Google Admin > Web and Mobile Apps > deporivisioning](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_autoprovisioning-deprovisioning.png)

9. Choose Finish. 

10. In the Auto-provisioning section, turn on the toggle switch ![Google Admin > Web and Mobile Apps > turn on auto-provisionng](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_autoprovisioning-turn_on.png)

11. Tottle the switch to change from Inactive to Active. ![Google Admin > Web and Mobile Apps > activate auto-provisionng](/assets/images/2024-10-23-AWS-IIC-Google-Workspace/Google_Admin_activate_auto-provisioning.png)


## Next step

1. Follow the steps in [Assign users or user groups to the application](https://kangks.github.io/2024/10/03/AWS-IAM-Identity-Center.html#step-5-assign-users-or-user-groups-to-the-application) to assign access to the users.