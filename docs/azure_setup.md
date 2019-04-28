#Setting up Azure

Use the following command 

```bash
./util/xtract_azure_properties.sh ../aws-iam-aad/setup/ root-account-profile
```

It should provide an output like:

```bash
Using profile: [root-account-profile]
Reading metadata file: [../aws-iam-aad/setup//AWS_SSO_Demo.xml]
 iam-saml.saml_id successfully placed as String
 iam-saml.saml_entity_id successfully placed as String
 Azure AD Tenant Name: staging.mycompany.com
 Enter Enterprise Application Owner User: aws-sso-integration@staging.mycompany.com
 Enter Enterprise Application Owner Password:
 Azure Enterprise Application ID: xxxx-yyyyy-zzzzzz
 iam-saml.secret successfully placed as SecureString
 iam-saml.tenant_name successfully placed as String
 iam-saml.msiam_access_id successfully placed as String
 iam-saml.appId successfully placed as String
```
