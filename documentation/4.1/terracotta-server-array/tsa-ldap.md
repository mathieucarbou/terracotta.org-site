---
---
# Setting Up LDAP-Based Authentication
{toc|2:3}

## Introduction

Note: For a brief overview of Terracotta security with links to individual topics, see <a href="/documentation/4.1/bigmemorymax/security-overview">Security Overview</a>.

Instead of using the [built-in user management system](/documentation/4.1/terracotta-server-array/tsa-security#set-up-authentication), you can set up authentication and authorization for the Terracotta Server Array (TSA) based on the Lightweight Directory Access Protocol (LDAP). This allows you to use your existing security infrastructure for controlling access to Terracotta clusters.

The two types of LDAP-based authentication supported are Microsoft Active Directory and standard LDAP. In addition, LDAPS (LDAP over SSL) is supported.

**NOTE:** Terracotta servers must be [configured to use SSL](tsa-security) before any Active Directory or standard LDAP can be used.

**NOTE:** This topic assumes that the reader has knowledge of standard LDAP concepts and usage.

## Configuration Overview
Active Directory and standard LDAP are configured in the &lt;auth> section of each server's configuration block:

    <servers secure="true">
      <server host="172.16.254.1" name="server1">
        ...
        <security>
          ...
          <auth>
            <realm>...</realm>
            <url>...</url>
            <user>...</user>
          </auth>
        </security>
    ...
    </server>

Active Directory and standard LDAP are configured using the &lt;realm> and &lt;url> elements; the &lt;user> element is used for [connections between Terracotta servers](tsa-security#keychain) and is not required for LDAP-related configuration.

For presentation, the URLs used in this document use line breaks. **Do not use line breaks** when creating URLs in your configuration.

### Realms and Roles
The setup for LDAP-based authentication and authorization uses Shiro realms to map user groups to one of the following two roles:

* admin &ndash; The user with the admin role is the initial user who sets up security. Thereafter, the user with the admin role performs system functions such as shutting down servers, clearing or deleting caches and cache managers, and reloading configuration.
* terracotta &ndash; The operator role is required to log in to the TMC, so even a user with the admin role must have the operator role. Thereafter, the person with the operator role can connect to the TMC and add connections.

### URL Encoding
Certain characters used in the LDAP URL must be encoded, unless [wrapped in a CDATA construct](#cdata). Characters that may be required in an LDAP URL are described below:

* & (ampersand) &ndash; Encode as %26.
* { (left brace) &ndash; Encode as %7B.
* } (right brace) &ndash; Encode as %7D.
* Space &ndash; Encode as %20. *Spaces must always be encoded*, even if wrapped in CDATA.
* = (equals sign) &ndash; Does not require encoding.


## Active Directory Configuration
Specify the realm and URL in the &lt;security> section of the Terracotta configuration as follows:

    <auth>
      <realm>com.tc.net.core.security.ShiroActiveDirectoryRealm</realm>
      <url>ldap://admin_user@server_address:server_port/searchBase=search_domain%26
           groupBindings=groups_to_roles</url>
      <user></user>
    </auth>

Note the value of the &lt;realm> element, which must specify the correct class (or Shiro security realm) for Active Directory. The components of the URL are defined in the following table.

| Component | Description |
|:-------|:------------|
| ldap:// | For the scheme, use either `ldap://` or `ldaps://`. |
| admin\_user | The name of a user with sufficient rights in Active Directory to perform a search in the domain specified by searchBase. The password for this user password must be stored in the Terracotta keychain used by the Terracotta server, using as key the root of the LDAP URI, `ldap://admin_user@server_name:server_port`, with no trailing slash ("/"). |
| server\_address:server\_port | The IP address or resolvable fully qualified domain name of the server, and the port for Active Directory. |
| searchBase | Specifies the Active Directory domain to be searched. For example, if the Active Directory domain is reggae.jamaica.org, then the format is `searchBase=dc=reggae,dc=jamaica,dc=org`. |
| groupBindings | Specifies the mappings between Active Directory groups and Terracotta roles. For example, `groupBindings=Domain%20Admins=admin,Users=terracotta` maps the  Active Directory groups "Domain Admins" and "Users" to the "admin" and "terracotta" Terracotta roles, respectively. To be mapped, the named Active Directory groups must be part of the domain specified in searchBase; all other groups (including those with the specified names) in other domains are ignored. |

For example:

    <auth>
      <realm>com.tc.net.core.security.ShiroActiveDirectoryRealm</realm>
      <url>ldap://bmarley@172.16.254.1:389?searchBase=dc=reggae,dc=jamaica,dc=org%26
           groupBindings=Domain%20Admins=admin,Users=terracotta</url>
      <user></user>
    </auth>

## Standard LDAP Configuration
Specify the realm and URL in the &lt;security> section of the Terracotta configuration as follows:

    <auth>
      <realm>com.tc.net.core.security.ShiroLdapRealm</realm>
      <url>ldap://directory_manager@myLdapServer:636?
           userDnTemplate=cn=%7B0%7D,ou=users,dc=mycompany,dc=com%26
           groupDnTemplate=cn=%7B0%7D,ou=groups,dc=mycompany,dc=com%26
           groupAttribute=uniqueMember%26
           groupBindings=bandleaders=admin,bandmembers=terracotta</url>
      <user></user>
    </auth>

Note the value of the &lt;realm> element, which must specify the correct class (or Shiro security realm) for Active Directory. The components of the URL are defined in the following table.

| Component | Description |
|:-------|:------------|
| ldap:// | For the scheme, use either `ldap://` or `ldaps://`. |
| directory\_manager | The name of a user with sufficient rights on the LDAP server to perform searches. No user is required if anonymous lookups are allowed. If a user is required, the user's password must be stored in the Terracotta keychain, using as key the root of the LDAP URL, `ldap://admin_user@server_name:server_port`, with no trailing slash ("/"). |
| server\_address:server\_port | The IP address or resolvable fully qualified domain name of the server, and the LDAP server port. |
| userDnTemplate | Specifies user-template values. See the example below. |
| groupDnTemplate | Specifies group-template values. See the example below. |
| groupAttribute | Specifies the LDAP group attribute whose value uniquely identifies a user. By default, this is "uniqueMember". See the example below. |
| groupBindings | Specifies the mappings between LDAP groups and Terracotta roles. For example, `groupBindings=bandleaders=admin,bandmembers=terracotta` maps the LDAP groups "bandleaders" and "bandmembers" to the "admin" and "terracotta" Terracotta roles, respectively. |

For example:

    <auth>
      <realm>com.tc.net.core.security.ShiroLdapRealm</realm>
      <url>ldap://dizzy@172.16.254.1:636?
           userDnTemplate=cn=%7B0%7D,ou=users,dc=mycompany,dc=com%26
           groupDnTemplate=cn=%7B0%7D,ou=groups,dc=mycompany,dc=com%26
           groupAttribute=uniqueMember%26
           groupBindings=bandleaders=admin,bandmembers=terracotta</url>
      <user></user>
    </auth>

This implies the LDAP directory structure is set up similar to the following:

    + dc=com  
        + dc=mycompany  
            + ou=groups  
                + cn=bandleaders  
                      | uniqueMember=dizzy
                      | uniqueMember=duke
                + cn=bandleaders
                      | uniqueMember=art
                      | uniqueMember=bird


If, however, the the LDAP directory structure is set up similar to the following:

    + dc=com  
        + dc=mycompany  
            + ou=groups  
                + cn=bandleaders  
                      | musician=dizzy
                      | musician=duke
                + cn=bandleaders
                      | musician=art
                      | musician=bird

then the value of groupAttribute should be "musician".

<a id="cdata"></a>
## Using the CDATA Construct
To avoid encoding the URL, wrap it in a CDATA construct as shown:

    <url><![CDATA[ldap://dizzy@172.16.254.1:636?
         userDnTemplate=cn={0},ou=users,dc=mycompany,dc=com&
         groupDnTemplate=cn={0},ou=groups,dc=mycompany,dc=com&
         groupAttribute=uniqueMember&
         groupBindings=bandleaders=admin,bandmembers=terracotta]]></url>
