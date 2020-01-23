---
layout: post
Author: "Laurent LIENHARD"
date: 2020-03-01T00:00:00+01:00
title: "Copie d'un profile UPD (VHDX)"
subtitle: "Déplacement d'un profile UPD"
tags: [Powershell, RDS, FICHIER]
categories: [RDS]
bigimg: [ { src: "/img/Header/RDP.jpg" }]
published: false
draft: true
---

## Demande initiale

Lors de l'une de mes missions, qui consistait à découper une collection de serveur RDS en 2 nouvelles collections plus petites, j'ai eu besoin de déplacer les profiles UPD des utilisateurs qui sont stockés sous forme de fichier .vhdx.

En effet, 2 collections de serveurs de bureau à distance, ne peuvent pas partager le même fichier de profile.
Ils sont donc stockés dans des répertoires différents.

En résumé on va partir de :

Collection1 ==========> \\\srv-profil\profile1

Pour arriver a :

Collection2 ==========> \\\srv-profil\profile2

Collection3 ==========> \\\srv-profil\profile3

Pour faire cela j'avais besoin d'une fonction qui pouvait copier les profiles de tous les utilisateurs d'un site donné mais également d'un seul utilisateur pour effectuer les rattrapages en cas d'erreur ;-)

Les profiles des utilisateurs stockés au format vhdx sur le partage ```\\\srv-profil\profile1``` ne sont pas stockés avec le nom de l'utilisateur en clair (sinon ce serait trop facile ;-) ) mais avec le SID de l'utilisateur sour la forme ```UVHD-S-1-5-21-3419836203-728636838-671359659-4189.vhdx```

Il faut donc également une fonction pour convertir en SID un nom d'utilisateur.

## Conversion en SID

```powershell
function Convert-User2Sid
{
    [CmdletBinding()]
    param (
        [parameter(Mandatory = $True)]
        [System.String]$Domain,
        [parameter(Mandatory = $True)]
        [System.String]$SamAccountName
    )

    begin
    {
    }

    process
    {
        $objUser = New-Object System.Security.Principal.NTAccount($Domain, $SamAccountName)
        $strSID = $objUser.Translate([System.Security.Principal.SecurityIdentifier])
    }

    end
    {
        return $strSID.Value
    }
}
```



Ici j'utilise la classe .net "System.Security.Principal.SecurityIdentifier" pour traduire le nom de l'utilisateur en SID

j'appelle la fonction de cette facon :

```powershell
PS> Convert-User2Sid -Domain "contoso.local" -SamAccountName "test.user"
PS> S-1-5-21-3419836203-728636838-671359659-4189
```

## Copie des profiles

Une fois la fonction pour la conversion en SID des noms des utilisateurs, on peut passer à la fonction qui va déplacer mes fichiers vhdx de profile.

Pour rappelle, je veux que cette fonction :

1. Puisse déplacer tous les profiles de tous les utilisateurs d'un site (en résumé une liste d'utilisateur)
2. Puisse déplacer le profile d'un seul utilisateur
3. Puisse supprimer (ou non) le profile dans son répertoire d'origine

Commençons donc par les paramètres de cette fonction.

```powershell
function Copy-VHDXProfile
{
    <#
    .SYNOPSIS
        Copy the VHDX profile
    .DESCRIPTION
        Copies a user's VHDX profile from a source directory to a destination directory
    .PARAMETER Site
        Name of the site to be searched for in ExtensionAttribute1
    .PARAMETER SamAccountName
        The user's samaccountname
    .PARAMETER OldPath
        The directory from where the vhdx profile must be copied (ended with \)
    .PARAMETER NewPath
        The directory to which the vhdx profile should be copied (ended with \)
    .PARAMETER Remove
        Deletion of the original VHDX profile
    #>
    [CmdletBinding(DefaultParameterSetName = "BySite")]
    param (
        [parameter(Mandatory = $True, ParameterSetName = "BySite")]
        [System.String]$Site,
        [parameter(Mandatory = $True, ParameterSetName = "ByName")]
        [System.String[]]$SamAccountName,
        [parameter(Mandatory = $True)]
        [System.String]$OldPath,
        [parameter(Mandatory = $True)]
        [System.String]$NewPath,
        [parameter(Mandatory = $false)]
        [Switch]$Remove
    )

}
```

J'utilise ici la notion de ```ParameterSetName``` puisque je veux pouvoir traiter en entrée soit une liste d'utilisateur soit un utilisateur seul

Comme vous pouvez le remarquer cette fonction est plutôt faite pour cette mission en particulier car, par exemple, je cherche le site dans l'attribut ```ExtensionAttribute1``` car c'est là que se trouver l'information lors de ma mission.

Naturellement cette fonction peut, assez facilement, être généralisée ;-)

Ajoutons maintenant le traitement du ```ParameterSetName```

```powershell
        switch ($PSCmdlet.ParameterSetName)
        {
            "BySite"
            {
                $Filter = "*$Site*"
                $AllUsers = Get-ADUser -filter * -Properties SamAccountName, ExtensionAttribute1 | Select-Object SamAccountName, ExtensionAttribute1 | Where-Object { $_.extensionAttribute1 -like $Filter }
            }
            "ByName"
            {
                $AllUsers = @()
                foreach ($name in $SamAccountName)
                {
                    $Filter = "*$name*"
                    $AllUsers += (Get-ADUser -Filter * -Properties SamAccountName, ExtensionAttribute1 | Select-Object SamAccountName, ExtensionAttribute1 | where-object { $_.SamAccountName -like $Filter })
                }
            }
        }
```

Le but ici est de renvoyé l'ensemble des utilisateurs à traiter dans la variable ```$AllUsers``` afin de pouvoir la traiter dans la suite de la fonction. Ceci me permet d'avoir un traitement identique pour tous les cas de figures dans la fonction.

Cette variable ```$AllUsers``` contient pour chaque entrée : le SamAccountName et la valeur de ExtensionAttribute1 de chaque utilisateur à traiter.

Une fois ma variable ```$AllUsers``` défini la suite du traitement consiste simplement à passer sur chaque entrée de cette variable et d'effectuer la copie du profile VHDX.

```powershell
    $Count = 0
        foreach ($User in $AllUsers)
        {
            $SID = Convert-User2SID -Domain  $env:USERDNSDOMAIN -SamAccountName $user.SamAccountName
            Write-Verbose "current treatment of $($user.SamAccountName) with SID $($SID)"

            $OldProfile = $OldPath + "UVHD-" + $SID + ".vhdx"
            if (Test-Path -Path $OldProfile)
            {
                Write-Verbose "Copy $($OldProfile) to $($NewPath)"
                Copy-Item -Path $OldProfile -Destination $NewPath -Force -Confirm:$false
                if ($Remove)
                {
                    Remove-Item -Path $OldProfile -Force -Confirm:$false
                }
            }
            else
            {
                Write-Warning "no vhdx profile found for $($OldProfile) "
            }

            $Count += 1
            Write-Verbose "$($Count) user(s) out of $($AllUsers.Count) treated "
        }
```

les étapes :

1. Conversion de l'utilisateur en SID
2. Construction du nom du fichier pour l'ancien profile
3. Copie du fichier vers le nouveau dossier
4. Suppression de l'ancien fichier si demandé

Au final l'utilisation de cette fonction se fait de la façon suivante :
1. Par site
```powershell
Copy-VHDXProfile -Site "Londres" -OldPath "\\srv-profil\profile1\" -NewPath "\\srv-profil\profile2" -verbose
```
2. Par Utilisateur
```powershell
PS> Copy-VHDXProfile -SamAccountName "test.user" -OldPath "\\srv-profil\profile1\" -NewPath "\\srv-profil\profile2" -verbose
```

Avec l'option ```-Remove``` pour forcer la suppression de l'ancien profile vhdx.
