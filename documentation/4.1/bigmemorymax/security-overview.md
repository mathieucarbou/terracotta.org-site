---
---
# Security Overview

{toc|2:3}

## Introduction
Security can be applied to both authentication (such as login credentials) and authorization (the privileges of specific roles).

We recommend that you plan and implement a security strategy that encompasses all of the points of potential vulnerability in your environment, including, but not necessarily limited to, your application servers (Terracotta clients), Terracotta servers in the TSA, the Terracotta Management Console (TMC), and any BigMemory .NET or C++ clients.

**Note**: Terracotta does not encrypt the data on its servers, but applying your own data encryption is another possible security measure.

## Links
 * Terracotta Server Array (TSA) using SSL, LDAP, JMX
    * <a href="/documentation/4.1/terracotta-server-array/managing-security">Cluster Security (basic)</a>
    * <a href="/documentation/4.1/terracotta-server-array/tsa-security">Securing Terracotta Clusters (advanced)</a>
    * <a href="/documentation/4.1/terracotta-server-array/tsa-ldap">Setting Up LDAP-Based Authentication</a>
    * <a href="/documentation/4.1/terracotta-server-array/tsa-crypt-keychain">Using Encrypted Keychains</a>  
 * Terracotta Client (your application)
    * <a href="/documentation/4.1/terracotta-server-array/tsa-security#enabling-ssl-on-terracotta-clients">Enabling SSL on Terracotta Clients</a>
   * <a href="/documentation/4.1/terracotta-server-array/tsa-crypt-keychain#clients-automatically-reading-the-keychain-password">Clients Automatically Reading the Keychain Password</a>          
 * Terracotta Management Console (TMC)
    * LDAP, Microsoft Active Directory, Secured Sockets Layer (SSL) &mdash; see <a href="/documentation/4.1/tms/tms-security">Terracotta Management Console Security Setup</a>
    * Identity Assertion &mdash; see <a href="/documentation/4.1/tms/tms-security#basic-connection-security">Basic Connection Security</a>
 * BigMemory Max monitoring without the TMC
    * <a href="/documentation/4.1/bigmemorymax/operations/jmx#ssl-secured-jmx-monitoring">SSL-Secured JMX Monitoring</a>      
 * BigMemory .NET and C++ clients
    * <a href="/documentation/4.1/cross-language/clc-security">Cross-Language Connector SSL Security</a>
