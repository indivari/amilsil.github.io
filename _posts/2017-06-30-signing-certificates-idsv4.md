---
layout: post
title: Generating and Installing Signing-Certificates for Identity Server
tags:
- identityserver
- certificates
- makecert
- pvk2pfx
---

A development implementation of an Identity Server (found in almost all examples online) uses a **Temporary Signing Certificate** to sign the JWT tokens. A temporary key is created every time the identity server is restarted. A new signing certificate makes all the tokens generated before invalid. In a production environment however, you want the tokens to be valid after a re-deploy of the identity server. So the signing certificate should be constant. In this post I'd show you how to create and use such self signed certificates.

## Getting the Tools Ready

We are going to use `makecert` and `pvk2pfx` utilities to create a pair of certificates. These two executables are found in "Windows Software Development Kit for Windows 8.1", downloadable from [here](https://developer.microsoft.com/en-us/windows/downloads/windows-8-1-sdk).

The executables are found at `C:\Program Files (x86)\Windows Kits\8.1\bin\x86`.
Open a command prompt and navigate to this folder. We are going to execute some commands there.

## Generating x509 Certificates Ready for Identity Server

We will need a *Certificate Authority* and a *Personal Information Exchange* (that contains a *Private Key* and a *Public Key*). 

- The **Private Key** is used to sign the JWT tokens
- The **Public Key** is exposed to the clients, so that they can validate the JWTs.

1. Create a Certificate Authority and a Private Key

> `makecert -n "CN=IdentityServerCN" -a sha256 -sv IdentityServer4Auth.pvk -r IdentityServer4Auth.cer`

2. Store the Private Key and the Public Key in a Personal Information Exchange

> `pvk2pfx -pvk IdentityServer4Auth.pvk -spc IdentityServer4Auth.cer -po -pfx IdentityServer4Auth.pfx`

Now you should have the following three files.

- IdentityServer4Auth.pvk
- IdentityServer4Auth.cer
- IdentityServer4Auth.pfx

Let's go ahead and use them. 

## Installing and Using Certificates

Identity Server 4 can retrieve these certificates in two alternative ways. 

1. Grab them from the certificate file directly
2. Grab them from the MMC (Microsoft Management Console)

### Using Certs from File

The most simplest. You can save the `pfx` file on a folder. Let's say `C:\workspace\keys\`.

I have created an extension to configure the certificates.

**SigninKeyExtension.cs**

```csharp 
public static void AddCertificateFromFile(
    this IIdentityServerBuilder builder, 
    IConfigurationSection options, 
    ILogger logger)
{
    var keyFilePath = options.GetValue<string>(KeyFilePath);
    var keyFilePassword = options.GetValue<string>(KeyFilePassword);

    if (File.Exists(keyFilePath))
    {
        logger.LogDebug($"SigninCredentialExtension adding key from file {keyFilePath}");
        
        // You can simply add this line in the Startup.cs if you don't want an extension. 
        // This is neater though ;)
        builder.AddSigningCredential(new X509Certificate2(keyFilePath, keyFilePassword));
    }
    else
    {
        logger.LogError($"SigninCredentialExtension cannot find key file {keyFilePath}");
    }
}
```

Here's the config above code uses.

**appsettings.json**

```json
"SigninKeyCredentials": {
    "KeyFilePath": "C:\workspace\keys\IdentityServer4Auth.pfx",
    "KeyFilePassword": "ourpassword"
  }
```

**Startup.cs**
```csharp
var identityServerBuilder = services.AddIdentityServer()
    .AddSigninCredentialFromConfig(Configuration.GetSection("SigninKeyCredentials"), Logger);
```

### Installing Certs in Local Machine using MMC

Certificates are sensitive files and thus keeping them in the file system may not be a good idea. A better way to approach that is to import the certificate to the *Machine Certificate Store* using MMC.

1. Start -> Run -> mmc.exe enter
2. File -> Add or Remove Snap-ins -> Certificates -> Add -> Computer Account -> OK
3. Import the certificate (.cer) to personal -> Trusted Root Certification Authorities)
4. Import the pfx, **with exportable private key support**, to personal -> certificates.

Now you have installed the certificates to the Machine Certificate Store.

Simplar to above, I have created an extension to configure this. 

**SigninKeyExtension.cs**

```csharp
private static void AddCertificateFromStore(
    IIdentityServerBuilder builder, 
    IConfigurationSection options, 
    ILogger logger)
{
    var keyIssuer = options.GetValue<string>(KeyStoreIssuer);
    logger.LogDebug($"SigninCredentialExtension adding key from store by {keyIssuer}");

    X509Store store = new X509Store(StoreName.My, StoreLocation.LocalMachine);
    store.Open(OpenFlags.ReadOnly);

    var certificates = store.Certificates.Find(X509FindType.FindByIssuerName, keyIssuer, true);

    if (certificates.Count > 0)
        builder.AddSigningCredential(certificates[0]);
    else
        logger.LogError("A matching key couldn't be found in the store");
}
```

**appsettings.json**

```json
 "SigninKeyCredentials": {
    "KeyStoreIssuer": "IdentityServerCN"
  }
```

**Startup.cs**

```csharp
var identityServerBuilder = services.AddIdentityServer()
    .AddCertificateFromStore(Configuration.GetSection("SigninKeyCredentials"), Logger);
```
