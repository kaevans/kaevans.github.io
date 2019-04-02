# Using WSE 2.0 to Enable Interoperable Kerberos Authentication

July 21, 2004

The KerberosToken that ships with WSE 2.0 is a beautiful thing.  It is proof of the thing that I love most about WSE 2.0: It serves as Microsoft’s vehicle for delivering supported advanced web services implementations out of the cycle of other products.  For example, not only do you get the functionality of WS-Policy before it is finalized by a standards group, you get the support of Microsoft behind the product.  And if you run into a feature that you wish were supported, the SDK is completely open and extensible, allowing you to introduce whatever functionality you desire.

There was an important point in that previous paragraph that I glossed over.  Some of the WS-* implementations in WSE 2.0 are based on specs that aren’t fully baked yet.  WS-Security was finalized by OASIS, but they only finalized the Username Token and the X.509 Token profiles:  they did not yet finalize the profile for Kerberos tokens yet.  Luckily, there was baked-in support for Kerberos in WSE 2.0 to allow everyone to work with it and determine how Kerberos authentication fits with their architecture.  It works beautifully between a WSE 2.0 client and a WSE 2.0 server using Active Directory.

It turns out that, after some experimentation, the KerberosToken in WSE 2.0 does not currently provide for simple interop with other platforms.  If you need that interoperability today via a GSS-API AP_REQ ticket, then you can achieve it using the SSPI APIs in the Win32 API.

I am no Keith Brown, so I went looking to see where someone had already wrapped the SSPI APIs with managed code.  Sure enough, Michel Barnett wrote a series of articles based on creating an SSPI library with managed code that were demonstrated with a .NET Remoting solution.  After a quick inspection, I saw that this library was neatly packaged and could be used with a custom WSE token as well.  Just download the SSPI library, set a reference to the SSPI DLL, and use the following code. 

Disclaimers:  The following code is UNSUPPORTED.  I make no guarantee whatsoever of the following code, it is shown only as a demonstration of hooking stuff together.  Use at your own risk, it is highly suggested that you read the articles accompanying the SSPI code to understand the implications of using these APIs.  Questions concerning this code will likely be redirected to either the originating SSPI articles or the CustomBinarySecurityToken example in the WSE 2.0 samples.

The Custom Kerberos SSPI Token

````csharp
using System;
using System.Xml;
using System.Text;
using Microsoft.Web.Services2;
using Microsoft.Web.Services2.Security;
using Microsoft.Web.Services2.Security.Tokens;
using Microsoft.Web.Services2.Security.Utility;
using Microsoft.Samples.Security.SSPI;

namespace XmlAdvice.Security.Tokens.SSPI
{
 /// <summary>
 /// Custom Kerberos token using SSPI.  References project from article
 /// http://msdn.microsoft.com/library/default.asp?url=/library/en-us/dndotnet/html/remsspi.asp
 /// </summary>
 public sealed class KerberosSspiToken : BinarySecurityToken, IIssuedToken
 {
  #region Fields  
  
  // life time of this token
  private LifeTime _lifeTime = null;
  #endregion


  public KerberosSspiToken(string serviceMachineName, string domain) : this(string.Concat("host/",serviceMachineName, "@",domain))
  {

  }

  public KerberosSspiToken(string targetPrincipal): base(KerberosSspiTokenNames.ValueType,KerberosSspiTokenNames.TokenType)
  { 

   //Lifetime is 8 hours
   this.LifeTime = new LifeTime(DateTime.Now, 8 * 60 * 60 );
   ClientCredential credential = null;
   ClientContext context = null;
   try
   {
    credential = new ClientCredential(Credential.Package.Kerberos);      
    //Get the client Kerberos ticket
    context = new ClientContext(credential, targetPrincipal, ClientContext.ContextAttributeFlags.None ); 
    //If the authentication succeeded, store the value in the .RawData property
    this.RawData = context.Token;    
   }
   finally
   {
    if(null != context) context.Dispose();
    if(null != credential) credential.Dispose();
   }   
  }

  public KerberosSspiToken(XmlElement element) : base(element, KerberosSspiTokenNames.ValueType,KerberosSspiTokenNames.TokenType)
  {  
   this.LoadXml(element);
   ServerCredential credential = null;
   ServerContext context = null;
   try
   {
    //Obtain the credential to authenticate
    credential = new ServerCredential(Credential.Package.Kerberos);
    //Authenticate the provided Kerberos ticket (contained in this.RawData)
    context = new ServerContext(credential,this.RawData); 
    System.Diagnostics.Debug.Assert(null != this.RawData,"this.RawData == null");
   }
   finally
   {
    if(null != context) context.Dispose();
    if(null != credential) credential.Dispose();
   }
  }


  /// <summary>
  /// Compares the values of two byte arrays to determine
  /// if they are equal or not.
  /// </summary>
  /// <param name="a">The first byte array to compare.</param>
  /// <param name="b">The second byte array to compare.</param>
  /// <returns>True if they are equivalent, false otherwise.</returns>
  private bool CompareArray(byte[] a, byte[] b)
  {
   if (a != null && b != null && a.Length == b.Length)
   {
    int index = a.Length;
    while (–index > -1)
     if (a[index] != b[index])
      return false;
    return true;
   }
   else if (a == null && b == null)
    return true;
   else
    return false;
  }

  /// <summary>
  /// Return true if two tokens has the same raw data
  /// </summary>
  public override bool Equals(SecurityToken token)
  {
   if ( token == null || token.GetType() != GetType() )
    return false;
   else
   {
    KerberosSspiToken t = (KerberosSspiToken)token;
    return CompareArray(RawData, t.RawData);
   }
  }


  public override int GetHashCode()
  {
   if ( RawData != null )
    return RawData.GetHashCode();
   else
    return 0;
  }


  /// <summary>
  /// Return true if the token hasn’t expired and its
  /// creation time is not a post dated time
  /// </summary>
  public override bool IsCurrent
  {
   get
   {
    return (LifeTime == null || LifeTime.IsCurrent);
   }
  }


  public override Microsoft.Web.Services2.Security.Cryptography.KeyAlgorithm Key
  {
   get {return null;}
  }


  public override bool SupportsDataEncryption
  {
   get {return false;}
  }


  public override bool SupportsDigitalSignature
  {
   get {return false;}
  }


  /// <summary>
  /// Get/Set a life time for this binary token, including creation
  /// time and expiration time
  /// </summary>
  public LifeTime LifeTime
  {
   get {return _lifeTime;}
   set {_lifeTime = value;}
  }


  /// <summary>
  /// Get/Set the proof token for this binary token.  This is the 
  /// encrypted form of the key for the token requestor
  /// </summary>
  public RequestedProofToken ProofToken
  {
   get {return null;}
   set { }
  }

 

  #region IIssuedToken Members


  /// <summary>
  /// Not used
  /// </summary>
  public Microsoft.Web.Services2.Policy.AppliesTo AppliesTo
  {
   get {return null;}
   set { }
  }

  /// <summary>
  /// Not used
  /// </summary>
  public SecurityTokenCollection SupportingTokens
  {
   get {return null;}
   set { }
  }  

  /// <summary>
  /// Not used
  /// </summary>
  public SecurityToken BaseToken
  {
   get {return null;}
   set { }
  }

  /// <summary>
  /// Not used
  /// </summary>
  public Uri TokenIssuer
  {
   get {return null;}
   set { }
  }

  #endregion
 }
}
````

Most of the above code was required for overriding methods, and honestly I got most of it from the CustomBinarySecurityToken sample in that comes with WSE 2.0.  The rest of the solution is also very similar to the CustomBinarySecurityToken example.

The Token Manager

````csharp
using System;
using System.Xml;
using System.Security.Permissions;
using System.Security.Cryptography.Xml;

using Microsoft.Web.Services2.Security;
using Microsoft.Web.Services2.Security.Tokens;

namespace XmlAdvice.Security.Tokens.SSPI
{
 /// <summary>
 /// KerberosSspiTokenManager class defines all the operations related to this custom 
 /// binary token, such as serialization/deserialization, etc.
 /// </summary>
 [SecurityPermission(SecurityAction.Demand,Flags= SecurityPermissionFlag.UnmanagedCode)] 
 public class KerberosSspiTokenManager : Microsoft.Web.Services2.Security.Tokens.SecurityTokenManager
 {  
  /// <summary>
  /// Returns a binary token’s token type
  /// </summary>
  public override string TokenType 
  { 
   get 
   {
    return KerberosSspiTokenNames.TokenType;
   }
  }

  /// <devdoc>
  /// Returns an instance of the security token serialized to an XML format.
  /// </devdoc>
  public override SecurityToken LoadTokenFromXml(XmlElement element)
  {
   return new KerberosSspiToken(element);
  }
  
 }

}
````

Finally, I adopted the pattern from the WSE 2.0 samples that segments the namespace URIs from the rest of the implementation code.

The Token Names

````csharp
using System;

namespace XmlAdvice.Security.Tokens.SSPI
{
 /// <summary>
 /// Summary description for KerberosSspiTokenNames.
 /// </summary>
 public class KerberosSspiTokenNames
 {
  public const string NamespaceURI = "urn:XmlAdvice:KerberosSspiToken";
  public const string TokenName = "GSSInteropKerberosv5ST";
  
  //
  // Use the same qualified name for this custom token’s ValueType and TokenType
  //
  public const string ValueType = NamespaceURI + "#" + TokenName;
  public const string TokenType = ValueType;

  //The concatenated example looks like:

  //urn:XmlAdvice:KerberosSspiToken#GSSInteropKerberosv5ST
 }
}
````