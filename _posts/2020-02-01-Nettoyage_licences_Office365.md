---
date: 2020-02-01 00:00:00+01:00
layout: post
Author: thiagorossener
title: Nettoyage des licences Office 365
subtitle: Nettoyage des licences Office 365 directes et héritées
tags:
  - Powershell
  - Office365
  - AzureAD
categories: Blog
paginate : true
---

## Résumé de l'article précedent

Cet article fait suite à l'article [Affectation de licence Office 365](../2020-02-01-affectation_auto_licence_o365) mais peut également être lu de façon autonome.

Dans le précedent article, nous étions passé d'un mode de gestion directe des licences Office 365 (affecté via l'interface web ou script powershell) a un mode de gestion hérité (affecté par une appartenance a un groupe Active Directory)

Nous en étions arrivé au constat suivant
![Detail utilisateur](/img/Post/Affectation_Auto_Licence_O365/detail_licence.png "Detail licence")

Après modification du type d'affectation des licences Office 365, nous avions remarqué des doublons.

Le but de cette article est donc de faire le ménage dans les doublons de licence.

## Nettoyage des licences Office 365

Plusieurs étapes vont être nécessaire pour effectuer le ménage :

1. Vérifier si l'utilisateur a une licence directe affectée
2. Si oui vérifier s'il a la même licence affectée en héritée
3. Si oui suppression de la licence directe pour ne laisser que la héritée

pour effectuer chacun des 2 premiers points, j'ai préféré partir sur des fonctions à part pour éviter de surcharger la fonction principale

La fonction suivante me retourne une valeur si l'utilisateur à, dans toutes ces licences affectées, la licence passée en paramètre sinon elle return ```$null```.
Elle me sera utile dans les prochains points

```powershell
#Returns the license object corresponding to the skuId. Returns NULL if not found
function GetUserLicense
{
    Param([Microsoft.Online.Administration.User]$user, [string]$skuId, [Guid]$groupId)
    #we look for the specific license SKU in all licenses assigned to the user
    foreach ($license in $user.Licenses)
    {
        if ($license.AccountSkuId -ieq $skuId)
        {
            return $license
        }
    }
    return $null
}
```

### Vérifier si l'utilisateur à une licence directe

```powershell
function UserHasLicenseAssignedDirectly
{
    Param([Microsoft.Online.Administration.User]$user, [string]$skuId)

    $license = GetUserLicense $user $skuId

    if ($license -ne $null)
    {
        if ($license.GroupsAssigningLicense.Count -eq 0)
        {
            return $true
        }

        foreach ($assignmentSource in $license.GroupsAssigningLicense)
        {
            if ($assignmentSource -ieq $user.ObjectId)
            {
                return $true
            }
        }
        return $false
    }
    return $false
}
```

Cette fonction test simplement ;-) si pour l'utilisateur passé en paramètre, la licence passée en paramètre est affectée.
Si oui elle vérifie si elle est assignée directement.


### Vérifier si l'utilisateur à une licence héritée

```powershell
function UserHasLicenseAssignedFromThisGroup
{
    Param([Microsoft.Online.Administration.User]$user, [string]$skuId, [Guid]$groupId)

    $license = GetUserLicense $user $skuId

    if ($license -ne $null)
    {
        foreach ($assignmentSource in $license.GroupsAssigningLicense)
        {
            if ($assignmentSource -ieq $groupId)
            {
                return $true
            }
        }
        return $false
    }
    return $false
}
```

Idem dans ce cas mais avec en plus, le paramètre GroupId.

Cette fonction vérifie, si pour l'utilisateur passé en paramètre, la licence passée en paramètre est affectée et si oui est-ce que c'est par la groupe passé en paramètre.

Arrivé a ce point nous avons nos 2 fonctions pour tester si la licence est affectée en directe ou héritée.

## Nettoyage

```powershell
function Remove-DirectO365License
{
    <#
    .SYNOPSIS
        Office 365 license removal
    .DESCRIPTION
        Removes directly affected Office 365 licenses when there is an legacy Office 365 license
    .PARAMETER GroupNames
        Azure AD group in which users are located
    .PARAMETER Licenses
        Office 365 licenses to process
    .PARAMETER Credential
        A user account with the necessary rights to read an Azure Active Directory
    .PARAMETER Force
        Force the deletion of the direct license on all users of the group even if a legacy license does not exist
    .EXAMPLE
        PS C:\> Remove-DirectO365License -GroupNames "Office365-E3" -Licenses "ENTERPRISEPACK"
        Removes the Office 365 E3 direct license on all users in the Office365-E3 group if a legacy ENTERPRISEPACK license exists
    .EXAMPLE
        PS C:\> Remove-DirectO365License -GroupNames "Office365-E3" -Licenses "ENTERPRISEPACK" -Force
        Force the deletion of the Office 365 E3 direct license on all users of the Office365-E3 group even if a legacy ENTERPRISEPACK license does not exist
    .EXAMPLE
        PS C:\> Remove-DirectO365License -GroupNames "Office365-E3","Office365-F1" -Licenses "ENTERPRISEPACK"
        Removes the Office 365 E3 direct license on all users in the Office365-E3 and Office365-F1 group if a legacy ENTERPRISEPACK license exists
    #>
    [CmdletBinding()]
    param (
        [parameter(Mandatory = $True)]
        [System.String[]]$GroupNames,
        [parameter(Mandatory = $True)]
        [System.String[]]$Licenses,
        [parameter(Mandatory = $false)]
        [pscredential]$Credential,
        [parameter(Mandatory = $false)]
        [Switch]$Force
    )

    begin
    {
        Write-Verbose "Connection MSOL service"
        if (!($PSBoundParameters['Credential']))
        {
            $Credential = Get-Credential -Message "Enter an account with the necessary rights on AzureAD"
        }
        $null = Connect-MsolService -Credential $Credential -Verbose:$false

        Write-Verbose "Connection AzureAD service"
        $null = Connect-AzureAD -Credential $Credential -Verbose:$false

    }

    process
    {
        foreach ($GroupName in $GroupNames)
        {
            $groupID = (Get-AzureADGroup -SearchString "$GroupName" | Select-Object Objectid).objectid

            $AllUsers = Get-MsolGroupMember -All -GroupObjectId $groupId | Get-MsolUser -ObjectId { $_.ObjectId }

            Foreach ($user in $AllUsers)
            {
                #$user = $_;
                $operationResult = "";

                foreach ($skuId in $Licenses)
                {
                    Write-verbose "check if Direct license $($skuId) exists on the user $($user.UserPrincipalName)"
                    if (UserHasLicenseAssignedDirectly $user $skuId)
                    {
                        if ($Force)
                        {
                            Write-Verbose "remove the direct license $($skuId) from user $($user.UserPrincipalName)"
                            Set-MsolUserLicense -ObjectId $user.ObjectId -RemoveLicenses $skuId
                            $operationResult = "Force Removed direct license $($skuId) from user $($user.UserPrincipalName)."
                        }
                        else
                        {
                            Write-verbose "check if the license $($skuId) is assigned from this group, as expected"
                            if (UserHasLicenseAssignedFromThisGroup $user $skuId $groupId)
                            {
                                Write-Verbose "remove the direct license $($skuId) from user $($user.UserPrincipalName)"
                                Set-MsolUserLicense -ObjectId $user.ObjectId -RemoveLicenses $skuId
                                $operationResult = "Removed direct license $($skuId) from user $($user.UserPrincipalName)."
                            }
                            else
                            {
                                $operationResult = "User $($user.UserPrincipalName) does not inherit this license $($skuId) from this group. License removal was skipped."
                            }
                        }
                    }
                    else
                    {
                        $operationResult = "User $($user.UserPrincipalName) has no direct license $($skuId) to remove. Skipping."
                    }

                    #format output
                    New-Object Object |
                    Add-Member -NotePropertyName UserId -NotePropertyValue $user.ObjectId -PassThru |
                    Add-Member -NotePropertyName OperationResult -NotePropertyValue $operationResult -PassThru
                }
            }
        }
    }

    end
    {

    }
}
```

Dans le fonction principale, je passe en paramètre :

1. le nom du(es) groupe(s) que je veux traiter (```GroupNames```)
2. le nom de(s) licence(s) que je veux supprimer (```Licenses```)


```powershell
$Arguments = @{
  GroupNames = "Office365-E3"
  Licenses = "ENTERPRISEPACK"
  credential = $CredAdmin
}
PS> Remove-DirectO365License @Arguments -Verbose
```

Dans ce cas je prends chaque utilisateur du groupe ```Office365-E3```, je vérifie s'il possède la licence ```ENTERPRISEPACK```affectée directement, si oui je vérifie s'il la également via le groupe et enfin si oui je supprime la licence affectée directement et ne laisse que la licence héritée.

Cela semble un peu compliqué à première vue mais le principe est assez simple dans le fond :-)