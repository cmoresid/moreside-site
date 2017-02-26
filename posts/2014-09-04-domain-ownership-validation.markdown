---
title: Domain Ownership Validation
tags: C#, .NET, Misc
---

At my current position, I was tasked with creating a component which would determine 
whether or not an individual owned or at the very least had administrative rights to a 
domain name; that is, to perform domain ownership validation. Why would one want to 
perform domain ownership validation? If one was providing a service which would send 
emails on behalf of a company with the domain name in question, for example, you would 
want to verify that whomever had signed up for the service with the domain name actually 
had the authority to do so.  
  
At first glance, the most simple way would be to send an email to the individual who 
is listed as the administrator contact in the WHOIS database. However, there are a few 
different issues which become glaringly apparent once you perform a couple WHOIS lookups 
on a few different domains.  
  
Many individuals opt for WHOIS Privacy Protection which is a service that many domain 
name registers provide that generally replaces all the publicly visible contact details 
with alternate contact information so that when a WHOIS query is performed on the domain, 
an alternate mailing address, email address and phone number are displayed. The alternative 
email address should forward any messages to the actual email address the registerer used. 
However, in my experience, this has been spotty at best. To complicate things further, some 
domain authorities no longer post registration details of individuals associated with 
particular domains, such as .ca domains.  
  
So… scratch the WHOIS lookup method…  
  
Another approach one could take is to have the individual upload a HTML document containing a 
unique token associated with the individual’s domain to his/her web server. After the document is 
uploaded, our system could then check to see if that document exists and whether or not the token 
inside the document matches the one in our system. Great! This sounds easy enough to do. One problem 
though: what if the user does not have a web server associated with that 
domain? Crap… moving on…  
  
I weighed the pros and cons of a couple other methods, but I finally settled on a method which 
involved an individual adding a TXT DNS record to his/her domain name server. Here is a brief 
overview of the approach:  
  
1. When an individual asks to register a domain on our system, we will generate a 
unique token that is associated with the domain name registration request. We then need 
to store this validation token somewhere for future use (probably in a database).  
2. The individual must place this unique token in a TXT record that is associated 
with this domain.  
3. Once the TXT record has been created, the user will be able to click 
a ‘Verify Domain’ button in our system.  
4. Our system will then proceed to retrieve all TXT records associated 
with the domain. The system will then try to find the record that the 
customer inserted.  
5. If the record is found and the validation token matches the one we have 
in our system, the domain is verified!  

This approach works rather nicely; however, it is a little technical for the average 
user. Most people wouldn’t know how to add a DNS record (let alone even know what a 
DNS record is!) to their domain name server. One could help mitigate this complexity by 
preparing an email template containing the relevant information that a user could copy/paste 
into an email and send to their DNS provider.  
  
Here is some sample code for generating a unique validation token:  
  
```cs
public class DomainValidationTokenUtil {
    public static readonly int TOKEN_LENGTH = 50;        
 
    public static string GenerateToken()
    {
        char[] universeOfDiscourse = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890".ToCharArray();
            
        RNGCryptoServiceProvider crypto = new RNGCryptoServiceProvider();
        byte[] data = new byte[TOKEN_LENGTH];
        crypto.GetNonZeroBytes(data);
 
        StringBuilder result = new StringBuilder(TOKEN_LENGTH);
        foreach (byte b in data)
        {
            result.Append(universeOfDiscourse[b % (universeOfDiscourse.Length - 1)]);
        }
            
        return result.ToString();
    }
}
```
  
As far as I know, the .NET framework does not have a built-in mechanism for 
nicely retrieving DNS records, so instead of writing our own, let us use a 3rd party 
library such as [ARSoft.Tools.NET](http://arsofttoolsnet.codeplex.com/) or one found on 
[CodeProject](http://www.codeproject.com/Articles/23673/DNS-NET-Resolver-C). Personally, 
I used the library on CodeProject. Here is some sample code for retrieving the DNS 
records, parsing the records, and performing the validation:  
  
```cs
public interface IDNSProvider {
    IList<string> GetTXTRecords(string domainName);
}
```

```cs
using Heijden.DNS;
 
public class DNSProvider : IDNSProvider
{
    private readonly Resolver _resolver;
 
    public DNSProvider()
    {
        _resolver = new Resolver();
        _resolver.Recursion = true;
        _resolver.UseCache = true;
        _resolver.DnsServer = "8.8.8.8"; // Google Public DNS
 
        _resolver.TimeOut = 1000;
        _resolver.Retries = 3;
        _resolver.TransportType = TransportType.Udp;
    }
 
    public IList<string> GetTXTRecords(string domainName)
    {
        IList<string> records = new List<string>();
        const QType qType = QType.TXT;
        const QClass qClass = QClass.IN;
 
        Response response = _resolver.Query(domainName, qType, qClass);
 
        foreach (RecordTXT record in response.RecordsTXT)
        {
            records.Add(record.ToString());
        }
 
        return records;
    }
}
```

And here is how you could perform the validation:  

```cs
public class DomainValidationService
{
    private IDNSProvider _dnsProvider;
 
    public DomainValidationService(IDNSProvider dnsProvider)
    {
        _dnsProvider = dnsProvider;
    }
 
    public bool ValidateDomainOwnership(string domainName, string validationToken)
    {
        // Retrieve TXT records
        IList<string> txtRecords = _dnsProvider.GetTXTRecords(domainName);
 
        // If there are no TXT records, the site is not verified yet.
        if (txtRecords.Count == 0)
            return false;
 
        // Check to see if validation token exists in any of the TXT records found
        // and return the validation token if found.
        string validationTokenInTXTRecord = FindValidationTokenInTXTRecords(txtRecords, validationToken);
 
        // No TXT record found containing a validation token.
        if (validationTokenInTXTRecord == null)
            return false;
 
        if (validationTokenInTXTRecord.Length != 50)
            throw new Exception("Invalid validation token in TXT record.");
 
        if (!validationTokenInTXTRecord.Equals(validationToken))
            throw new Exception("Validation token in database does not match token in TXT record.");
 
        // Everything must be OK, domain has been validated!
        return true;
    }
 
    public string GenerateTXTRecordValue()
    {
        return string.Format("domain-verification={0}", DomainVerificationTokenUtil.GenerateToken());
    }
 
    private string FindValidationTokenInTXTRecords(IList<string> txtRecords, string validationToken)
    {
        string txtValidationToken = null;
 
        foreach (string txtRecord in txtRecords)
        {
            if (txtRecord.StartsWith("domain-verification"))
                return txtRecord.Split('=')[1];
        }
 
        return txtValidationToken;
    }
}
```

Thanks for reading and hopefully this will be helpful if you need to do any sort of 
domain ownership validation on your systems!  
  
Cheers,  
  
Connor Moreside
