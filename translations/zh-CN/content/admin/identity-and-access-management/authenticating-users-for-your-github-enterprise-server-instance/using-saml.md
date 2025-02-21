---
title: 使用 SAML
redirect_from:
  - /enterprise/admin/articles/configuring-saml-authentication
  - /enterprise/admin/articles/about-saml-authentication
  - /enterprise/admin/user-management/using-saml
  - /enterprise/admin/authentication/using-saml
  - /admin/authentication/using-saml
  - /enterprise/admin/authentication/authenticating-users-for-your-github-enterprise-server-instance/using-saml
intro: 'You can configure SAML single sign-on (SSO) for {% data variables.product.product_name %}, which allows users to authenticate through a SAML identity provider (IdP) to access your instance.'
versions:
  ghes: '*'
type: how_to
topics:
  - Accounts
  - Authentication
  - Enterprise
  - Identity
  - SSO
---

## About SAML for {% data variables.product.product_name %}

SAML SSO allows people to authenticate and access {% data variables.product.product_location %} through an external system for identity management.

SAML 是一种基于 XML 的身份验证和授权标准。 When you configure SAML for {% data variables.product.product_location %}, the external system for authentication is called an identity provider (IdP). Your instance acts as a SAML service provider (SP). For more information, see [Security Assertion Markup Language](https://en.wikipedia.org/wiki/Security_Assertion_Markup_Language) on Wikipedia.

{% data reusables.enterprise_user_management.built-in-authentication %}

## 支持的 SAML 服务

{% data reusables.saml.saml-supported-idps %}

{% ifversion ghes > 3.3 %}

If your IdP supports encrypted assertions, you can configure encrypted assertions on {% data variables.product.product_name %} for increased security during the authentication process.

{% endif %}

{% data reusables.saml.saml-single-logout-not-supported %}

## 使用 SAML 时的用户名考量因素

每个 {% data variables.product.prodname_ghe_server %} 用户名都由 SAML 响应中的以下断言之一决定，这些断言按优先级从高到低排列的顺序为：

- 自定义用户名属性（如果定义且存在）
- `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` 断言（如果存在）
- `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` 断言（如果存在）
- `NameID` 元素

即使其他属性存在，也需要 `NameID` 元素。

将在 `NameID` 与 {% data variables.product.prodname_ghe_server %} 用户名之间创建映射，`NameID` 应持久、唯一，并且在用户生命周期内不会发生变化。

{% note %}

**注**：如果在 IdP 上更改某用户的 `NameID`，该用户在尝试登录到您的 {% data variables.product.prodname_ghe_server %} 实例时会看到错误消息。 {% ifversion ghes %}To restore the user's access, you'll need to update the user account's `NameID` mapping. 更多信息请参阅“[更新用户的 SAML `NameID`](#updating-a-users-saml-nameid)”。{% else %} 更多信息请参阅“[错误：另一个用户已拥有该帐户](#error-another-user-already-owns-the-account)”。{% endif %}

{% endnote %}

{% data reusables.enterprise_management_console.username_normalization %}

{% data reusables.enterprise_management_console.username_normalization_sample %}

{% data reusables.enterprise_user_management.two_factor_auth_header %}
{% data reusables.enterprise_user_management.external_auth_disables_2fa %}

## SAML 元数据

The service provider metadata for {% data variables.product.product_location %} is available at `http(s)://[hostname]/saml/metadata`.

要手动配置您的身份提供程序，断言使用者服务 (ACS) URL 为 `http(s)://[hostname]/saml/consume`。 它使用 `urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST` 绑定。

## SAML 属性

以下属性可用。 You can change the attribute names in the [management console](/enterprise/{{ currentVersion }}/admin/guides/installation/accessing-the-management-console/), with the exception of the `administrator` attribute.

| 默认属性名称        | 类型 | 描述                                                                                                              |
| ------------- | -- | --------------------------------------------------------------------------------------------------------------- |
| `NameID`      | 必选 | 持久用户标识符。 可以使用任意持久名称标识符格式。 除非提供备用断言之一，否则将为 {% data variables.product.prodname_ghe_server %} 用户名使用 `NameID` 元素。 |
| `管理员`         | 可选 | 如果值为“true”，用户将被自动升级为管理员。 任何其他值或不存在的值会将用户降级为普通用户帐户。                                                              |
| `用户名`         | 可选 | {% data variables.product.prodname_ghe_server %} 用户名。                                                         |
| `full_name`   | 可选 | 用户的个人资料页面上显示的姓名。 用户可以在配置后更改他们的姓名。                                                                               |
| `emails`      | 可选 | 用户的电子邮件地址。 可以指定多个。                                                                                              |
| `public_keys` | 可选 | 用户的 SSH 公钥。 可以指定多个。                                                                                             |
| `gpg_keys`    | 可选 | 用户的 GPG 密钥。 可以指定多个。                                                                                             |

To specify more than one value for an attribute, use multiple `<saml2:AttributeValue>` elements.

```
<saml2:Attribute FriendlyName="public_keys" Name="urn:oid:1.2.840.113549.1.1.1" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
    <saml2:AttributeValue>ssh-rsa LONG KEY</saml2:AttributeValue>
    <saml2:AttributeValue>ssh-rsa LONG KEY 2</saml2:AttributeValue>
</saml2:Attribute>
```

## 配置 SAML 设置

{% data reusables.enterprise_site_admin_settings.access-settings %}
{% data reusables.enterprise_site_admin_settings.management-console %}
{% data reusables.enterprise_management_console.authentication %}
1. 选择 **SAML**。

   ![Screenshot of option to enable SAML authentication in management console](/assets/images/enterprise/management-console/auth-select-saml.png)
1. {% data reusables.enterprise_user_management.built-in-authentication-option %}

   ![Screenshot of option to enable built-in authentication outside of SAML IdP](/assets/images/enterprise/management-console/saml-built-in-authentication.png)
1. 或者，要启用非请求响应 SSO，请选择 **IdP initiated SSO**。 默认情况下，{% data variables.product.prodname_ghe_server %} 将向 IdP 发回 `AuthnRequest`，回复非请求身份提供程序 (IdP) 发起的请求。

   ![Screenshot of option to enable IdP-initiated unsolicited response](/assets/images/enterprise/management-console/saml-idp-sso.png)

   {% tip %}

   **Note**: We recommend keeping this value **unselected**. You should enable this feature **only** in the rare instance that your SAML implementation does not support service provider initiated SSO, and when advised by {% data variables.contact.enterprise_support %}.

   {% endtip %}

1. 如果您**不**希望 SAML 提供程序为 {% data variables.product.product_location %} 上的用户确定管理员权限，请选择 **Disable administrator demotion/promotion（禁用管理员降级/升级）**。

   ![Screenshot of option to enable option to respect the "administrator" attribute from the IdP to enable or disable administrative rights](/assets/images/enterprise/management-console/disable-admin-demotion-promotion.png)
1. Optionally, to allow {% data variables.product.product_location %} to send and receive encrypted assertions to and from your SAML IdP, select **Require encrypted assertions**. For more information, see "[Enabling encrypted assertions](#enabling-encrypted-assertions)."

   ![Screenshot of "Enable encrypted assertions" checkbox within management console's "Authentication" section](/assets/images/help/saml/management-console-enable-encrypted-assertions.png)

   {% warning %}

   **Warning**: Incorrectly configuring encrypted assertions can cause all authentication to {% data variables.product.product_location %} to fail.

   - You must ensure that your IdP supports encrypted assertions and that the encryption and key transport methods in the management console match the values configured on your IdP. You must also provide {% data variables.product.product_location %}'s public certificate to your IdP. For more information, see "[Enabling encrypted assertions](#enabling-encrypted-assertions)."

   - Before enabling encrypted assertions, {% data variables.product.company_short %} recommends testing encrypted assertions in a staging environment, and confirming that SAML authentication functions as you expect. 更多信息请参阅“[设置暂存实例](/admin/installation/setting-up-a-github-enterprise-server-instance/setting-up-a-staging-instance)”。

   {% endwarning %}
1. In the **Single sign-on URL** field, type the HTTP or HTTPS endpoint on your IdP for single sign-on requests. 此值由您的 IdP 配置提供。 If the host is only available from your internal network, you may need to [configure {% data variables.product.product_location %} to use internal nameservers](/enterprise/{{ currentVersion }}/admin/guides/installation/configuring-dns-nameservers/).

   ![Screenshot of text field for single sign-on URL](/assets/images/enterprise/management-console/saml-single-sign-url.png)
1. Optionally, in the **Issuer** field, type your SAML issuer's name. This verifies the authenticity of messages sent to {% data variables.product.product_location %}.

   ![Screenshot of text field for SAML issuer URL](/assets/images/enterprise/management-console/saml-issuer.png)
1. In the **Signature Method** and **Digest Method** drop-down menus, choose the hashing algorithm used by your SAML issuer to verify the integrity of the requests from {% data variables.product.product_location %}. Specify the format with the **Name Identifier Format** drop-down menu.

   ![Screenshot of drop-down menus to select signature and digest method](/assets/images/enterprise/management-console/saml-method.png)
1. 在 **Verification certificate（验证证书）**下，单击 **Choose File（选择文件）**并选择用于验证 IdP 的 SAML 响应的证书。

   ![Screenshot of button for uploading validation certificate from IdP](/assets/images/enterprise/management-console/saml-verification-cert.png)
1. Modify the SAML attribute names to match your IdP if needed, or accept the default names.

   ![Screenshot of fields for entering additional SAML attributes](/assets/images/enterprise/management-console/saml-attributes.png)

{% ifversion ghes > 3.3 %}

## Enabling encrypted assertions

To enable encrypted assertions, your SAML IdP must also support encrypted assertions. You must provide {% data variables.product.product_location %}'s public certificate to your IdP, and configure encryption settings that match your IdP.

{% warning %}

**Warning**: Incorrectly configuring encrypted assertions can cause all authentication to {% data variables.product.product_location %} to fail. {% data variables.product.company_short %} strongly recommends testing your SAML configuration in a staging environment. For more information about staging instances, see "[Setting up a staging instance](/admin/installation/setting-up-a-github-enterprise-server-instance/setting-up-a-staging-instance)."

{% endwarning %}

1. Configure SAML for {% data variables.product.product_location %}. For more information, see "[Configuring SAML settings](#configuring-saml-settings)."
{% data reusables.enterprise_installation.ssh-into-instance %}
1. Run the following command to output {% data variables.product.product_location %}'s public certificate.
   
        openssl pkcs12 -in /data/user/common/saml-sp.p12 -nokeys -passin pass:
1. In the output, copy the text beginning with `-----BEGIN CERTIFICATE-----` and ending with `-----END CERTIFICATE-----`, and paste the output into a plaintext file.
1. Sign into your SAML IdP as an administrator.
1. In the application for {% data variables.product.product_location %}, enable encrypted assertions.
   - Note the encryption method and key transport method.
   - Provide the public certificate from step 3.
{% data reusables.enterprise_site_admin_settings.access-settings %}
{% data reusables.enterprise_site_admin_settings.management-console %}
{% data reusables.enterprise_management_console.authentication %}
1. Select **Require encrypted assertions**.

   ![Screenshot of "Enable encrypted assertions" checkbox within management console's "Authentication" section](/assets/images/help/saml/management-console-enable-encrypted-assertions.png)
1. To the right of "Encryption Method", select the encryption method for your IdP from step 5.

   ![Screenshot of "Encryption Method" for encrypted assertions](/assets/images/help/saml/management-console-encrypted-assertions-encryption-method.png)
1. To the right of "Key Transport Method", select the key transport method for your IdP from step 5.

   ![Screenshot of "Key Transport Method" for encrypted assertions](/assets/images/help/saml/management-console-encrypted-assertions-key-transport-method.png)
1. 单击 **Save settings（保存设置）**。
{% data reusables.enterprise_site_admin_settings.wait-for-configuration-run %}

{% endif %}

## Updating a user's SAML `NameID`

{% data reusables.enterprise_site_admin_settings.access-settings %}
2. 选择 **SAML**。 ![网站管理员设置中的"All users（所有用户）"侧边栏项目](/assets/images/enterprise/site-admin-settings/all-users.png)
3. 在用户列表中，点击您想要更新其 `NameID` 映射的用户名。 ![实例用户帐户列表中的用户名](/assets/images/enterprise/site-admin-settings/all-users-click-username.png)
{% data reusables.enterprise_site_admin_settings.security-tab %}
5. 在“Update SAML NameID（更新 SAML 名称 ID）”右侧，单击 **Edit（编辑）**。 ![SAML 身份验证](/assets/images/enterprise/site-admin-settings/update-saml-nameid-edit.png)
6. 在“NameID（名称 ID）”字段中，为用户键入新的 `NameID`。 ![键入了名称 ID 的模态对话框中的"名称 ID"字段](/assets/images/enterprise/site-admin-settings/update-saml-nameid-field-in-modal.png)
7. 单击 **Update NameID（更新名称 ID）**。 ![模态中更新的名称 ID 下的"Update NameID（更新名称 ID）"按钮](/assets/images/enterprise/site-admin-settings/update-saml-nameid-update.png)

## 撤销 {% data variables.product.product_location %} 的权限

如果您将某个用户从您的身份提供程序中移除，还必须手动挂起他们。 否则，他们仍可以继续使用访问令牌或 SSH 密钥进行身份验证。 更多信息请参阅“[挂起和取消挂起用户](/enterprise/admin/guides/user-management/suspending-and-unsuspending-users)”。

## 响应消息的要求

响应消息必须满足以下要求：

- `<Destination>` 元素必须在根响应文档上提供，而且只有在根响应文档签署后才匹配 ACS URL。 如果断言已签名，它将被忽略。
- `<Audience>` 元素必须始终作为 `<AudienceRestriction>` 元素的一部分提供。 It must match the `EntityId` for {% data variables.product.prodname_ghe_server %}. 这是 {% data variables.product.prodname_ghe_server %} 实例的 URL，如 `https://ghe.corp.example.com`。
- Each assertion in the response **must** be protected by a digital signature. 签署各个 `<Assertion>` 元素或签署 `<Response>` 元素可以实现此操作。
- `<NameID>` 元素必须作为 `<Subject>` 元素的一部分提供。 可以使用任意持久名称标识符格式。
- `Recipient` 属性必须存在并设为 ACS URL。 例如：

```xml
<samlp:Response ...>
  <saml:Assertion ...>
    <saml:Subject>
      <saml:NameID ...>...</saml:NameID>
      <saml:SubjectConfirmation ...>
        <saml:SubjectConfirmationData Recipient="https://ghe.corp.example.com/saml/consume" .../>
      </saml:SubjectConfirmation>
    </saml:Subject>
    <saml:AttributeStatement>
      <saml:Attribute FriendlyName="USERNAME-ATTRIBUTE" ...>
        <saml:AttributeValue>monalisa</saml:AttributeValue>
      </saml:Attribute>
    </saml:AttributeStatement>
  </saml:Assertion>
</samlp:Response>
```

## SAML 身份验证

{% data variables.product.prodname_ghe_server %} 在  _/var/log/github/auth.log_ 的身份验证日志中为失败的 SAML 身份验证记录错误消息。 关于 SAML 响应要求的更多信息，请参阅“[响应消息要求](#response-message-requirements)”。

### Error: "Another user already owns the account"（错误：“其他用户已拥有该帐户”）

When a user signs in to {% data variables.product.prodname_ghe_server %} for the first time with SAML authentication, {% data variables.product.prodname_ghe_server %} creates a user account on the instance and maps the SAML `NameID` to the account.

当用户再次登录时，{% data variables.product.prodname_ghe_server %} 会比较帐户的 `NameID` 映射与 IdP 的响应。 如果 IdP 响应中的 `NameID` 不再与 {% data variables.product.prodname_ghe_server %} 对用户预期的 `NameID` 匹配， 登录将失败。 用户将看到以下消息。

> 另一个用户已经拥有该帐户。 请让您的管理员检查身份验证日志。

该消息通常表示此人的用户名或电子邮件地址已在 IdP 上更改。 {% ifversion ghes %}Ensure that the `NameID` mapping for the user account on {% data variables.product.prodname_ghe_server %} matches the user's `NameID` on your IdP. 更多信息请参阅“[更新用户的 SAML `NameID`](#updating-a-users-saml-nameid)”。{% else %}如需更新 `NameID` 映射的帮助，请联系 {% data variables.contact.contact_ent_support %}。{% endif %}

### Error: Recipient in SAML response was blank or not valid（错误：SAML 响应中的收件人为空或无效）

如果 `Recipient` 与 {% data variables.product.prodname_ghe_server %} 实例的 ACS URL 不匹配，则当用户尝试验证时，身份验证日志中将显示以下两条错误消息之一：

```
Recipient in the SAML response must not be blank.
```

```
Recipient in the SAML response was not valid.
```

Ensure that you set the value for `Recipient` on your IdP to the full ACS URL for your {% data variables.product.prodname_ghe_server %} instance. 例如，`https://ghe.corp.example.com/saml/consume`。

### Error: "SAML Response is not signed or has been modified"（错误：“SAML 响应未签名或已修改”）

如果您的 IdP 未对 SAML 响应进行签名，或者签名与内容不匹配，则身份验证日志中将显示以下错误消息。

```
SAML Response is not signed or has been modified.
```

确保为 IdP 上的 {% data variables.product.prodname_ghe_server %} 应用程序配置签名的断言。

### Error: "Audience is invalid" or "No assertion found"（错误：“受众无效”或“未找到断言”）

如果 IdP 的响应缺少 `Audience` 的值或者该值不正确，则身份验证日志中将显示以下错误消息。

```shell
Audience is invalid. Audience is invalid. Audience attribute does not match your_instance_url
```

确保对您的 {% data variables.product.prodname_ghe_server %} 实例将 IdP 上的 `Audience` 值设为 `EntityId`，这是 {% data variables.product.prodname_ghe_server %} 实例的完整 URL。 For example, `https://ghe.corp.example.com`.
