# Guide pour désactiver les applications au démarrage sur Windows

Ce guide explique comment empêcher les applications de s'ouvrir automatiquement au démarrage de Windows. Il couvre des méthodes graphiques simples et des commandes PowerShell pour un contrôle avancé.

## Sommaire
- [Guide pour désactiver les applications au démarrage sur Windows](#guide-pour-désactiver-les-applications-au-démarrage-sur-windows)
  - [Sommaire](#sommaire)
  - [Pourquoi le faire](#pourquoi-le-faire)
  - [Méthode 1 — Gestionnaire des tâches](#méthode-1--gestionnaire-des-tâches)
  - [Méthode 2 — Paramètres Windows](#méthode-2--paramètres-windows)
  - [Méthode 3 — PowerShell (avancé)](#méthode-3--powershell-avancé)
    - [3.1 Sauvegarde des clés de registre](#31-sauvegarde-des-clés-de-registre)
    - [3.2 Nettoyage du dossier de démarrage](#32-nettoyage-du-dossier-de-démarrage)
    - [3.3 Script global (HKCU) — suppression automatique et sûre](#33-script-global-hkcu--suppression-automatique-et-sûre)
    - [3.4 Vérification](#34-vérification)
  - [Méthode 4 — Tâches planifiées et services](#méthode-4--tâches-planifiées-et-services)
  - [Précautions](#précautions)
  - [Conclusion et ressources](#conclusion-et-ressources)
  - [👤 Auteur](#-auteur)

## Pourquoi le faire
- Améliorer la vitesse de démarrage.  
- Réduire la consommation de CPU, RAM et batterie.  
- Garder le contrôle sur ce qui s’exécute automatiquement.

## Méthode 1 — Gestionnaire des tâches
1. Appuyez sur `Ctrl + Shift + Esc`.  
2. Ouvrez l’onglet **Démarrage**.  
3. Clic droit sur une application → **Désactiver**.  

Simple et sûr — recommandé pour la plupart des utilisateurs.

## Méthode 2 — Paramètres Windows
1. Ouvrez **Paramètres** (Win + I).  
2. Allez dans **Applications > Démarrage**.  
3. Désactivez les applications indésirables.  

Interface moderne, pratique sur Windows 10/11.

## Méthode 3 — PowerShell (avancé)
> Faites une sauvegarde et testez d’abord sur votre compte utilisateur (HKCU) avant de modifier le registre système (HKLM). Les opérations sur HKLM nécessitent des droits administrateur.

### 3.1 Sauvegarde des clés de registre
Exécutez dans une invite de commandes (en tant qu’administrateur si vous ciblez HKLM) :
```cmd
reg export HKCU\Software\Microsoft\Windows\CurrentVersion\Run "%USERPROFILE%\Documents\run_backup_HKCU.reg"
reg export HKLM\Software\Microsoft\Windows\CurrentVersion\Run "%USERPROFILE%\Documents\run_backup_HKLM.reg"
```

### 3.2 Nettoyage du dossier de démarrage
Dans PowerShell (exécuté en tant qu'utilisateur) :
```powershell
$startupPath = [Environment]::GetFolderPath("Startup")
Get-ChildItem -Path $startupPath -Force -ErrorAction SilentlyContinue
# Pour supprimer manuellement :
# Remove-Item "$startupPath\*" -Force -Recurse -ErrorAction SilentlyContinue
```

### 3.3 Script global (HKCU) — suppression automatique et sûre
Le script suivant :
- sauvegarde la clé HKCU\...\Run,
- liste les entrées et le contenu du dossier Startup,
- demande confirmation,
- supprime toutes les entrées HKCU Run et vide le dossier Startup si vous confirmez.

Copiez-collez dans une console PowerShell (non administrateur pour HKCU). Pour agir sur HKLM, exécutez PowerShell en tant qu’administrateur et adaptez la clé cible.

```powershell
# Script global : sauvegarde + suppression (HKCU)
# filepath: c:\Users\Nelson Bandos\Desktop\Block_apps_autostart_with_Windows\scripts\clear_startup_HKCU.ps1

$timestamp = (Get-Date).ToString("yyyyMMdd_HHmmss")
$backupDir = Join-Path $env:USERPROFILE "Documents\startup_backups"
New-Item -Path $backupDir -ItemType Directory -Force | Out-Null

# Sauvegarde de la clé HKCU Run
$regBackupFile = Join-Path $backupDir "run_backup_HKCU_$timestamp.reg"
reg export "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" $regBackupFile > $null 2>&1

# Chemin du dossier Startup
$startupPath = [Environment]::GetFolderPath("Startup")

Write-Host "Sauvegarde créée : $regBackupFile" -ForegroundColor Green
Write-Host "`nEntrées actuelles dans HKCU:\Software\Microsoft\Windows\CurrentVersion\Run :" -ForegroundColor Cyan
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -ErrorAction SilentlyContinue | Format-List

Write-Host "`nFichiers présents dans le dossier Startup ($startupPath) :" -ForegroundColor Cyan
Get-ChildItem -Path $startupPath -Force -ErrorAction SilentlyContinue | ForEach-Object { Write-Host $_.Name }

# Confirmation utilisateur
$confirm = Read-Host "`nSupprimer TOUTES les entrées HKCU Run et VIDER le dossier Startup ? (O/N)"
if ($confirm -match '^[Oo]') {
    # Récupérer les propriétés (noms) et les supprimer
    $runProps = @()
    try {
        $runProps = (Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -ErrorAction Stop) |
                    Get-Member -MemberType NoteProperty | Select-Object -ExpandProperty Name
    } catch { $runProps = @() }

    foreach ($p in $runProps) {
        Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name $p -ErrorAction SilentlyContinue
    }

    # Vider le dossier Startup
    try {
        Remove-Item -Path (Join-Path $startupPath '*') -Force -Recurse -ErrorAction SilentlyContinue
    } catch { }

    Write-Host "`nSuppression terminée. Sauvegarde : $regBackupFile" -ForegroundColor Green
} else {
    Write-Host "`nOpération annulée. Aucun changement effectué." -ForegroundColor Yellow
}
```

Remarques :
- Ce script cible uniquement HKCU (compte utilisateur courant). Il n’exige pas d’élévation.
- Pour appliquer la même logique à HKLM (tous les utilisateurs), exécutez en tant qu’administrateur et remplacez la clé par HKLM:\Software\Microsoft\Windows\CurrentVersion\Run — attention, cela peut impacter des composants système ou logiciels d’entreprise.
- Conservez la sauvegarde avant toute modification. Pour restaurer : double-cliquez sur le fichier .reg ou utilisez reg import.

### 3.4 Vérification
Lister les entrées restantes (HKCU) :
```powershell
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"
```
Vérifier le dossier Startup :
```powershell
Get-ChildItem -Path ([Environment]::GetFolderPath("Startup")) -Force
```

## Méthode 4 — Tâches planifiées et services
Certaines applications utilisent des tâches planifiées ou des services pour se lancer.

- Lister les tâches configurées pour le démarrage :
```powershell
Get-ScheduledTask | Where-Object { $_.Triggers -ne $null -and ($_.Triggers | Where-Object { $_.AtStartup -eq $true }) }
```
- Lister les services configurés en démarrage automatique :
```powershell
Get-Service | Where-Object { $_.StartType -eq "Automatic" }
```
Désactivez uniquement les services non essentiels et vérifiez l’impact.

## Précautions
- Ne désactivez pas les services critiques de Windows.  
- Sauvegardez toujours les clés de registre avant modification.  
- Préférez tester sur HKCU avant HKLM.  
- Pour toute opération risquée, créez un point de restauration système.

## Conclusion et ressources
En combinant ces méthodes, vous gardez le contrôle des applications qui se lancent au démarrage et améliorez les performances du PC.

Ressources officielles Microsoft :
- Support Microsoft (gestion des applications de démarrage) : https://support.microsoft.com/windows  
- Registre Windows (documentation) : https://learn.microsoft.com/windows/win32/sysinfo/registry  
- Planificateur de tâches (documentation) : https://learn.microsoft.com/windows/win32/taskschd/  
- Documentation PowerShell : https://learn.microsoft.com/powershell/

## 👤 Auteur

**Nelson Bandos**   
🔗 GitHub : [@Nelson2410](https://github.com/Nelson2410)  
📅 Date : Janvier 2026  
📍 Localisation : Abidjan, Côte d'Ivoire.
