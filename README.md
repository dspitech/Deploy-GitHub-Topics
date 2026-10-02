# Set-ReposTopics

Script PowerShell pour appliquer, fusionner et normaliser les topics d'un ou plusieurs dépôts GitHub via l'API REST, sans passer par l'interface web.

## Sommaire

1. Présentation
2. Prérequis
3. Authentification
4. Installation
5. Utilisation
6. Paramètres
7. Fonctionnement
8. Règles de GitHub sur les topics
9. Application à plusieurs dépôts
10. Dépannage
11. Sécurité
12. Structure recommandée du dépôt

---

## 1. Présentation

GitHub utilise les topics pour classer les dépôts et améliorer leur visibilité dans la recherche. Les définir à la main dépôt par dépôt est long et peu reproductible. Ce script permet de :

- appliquer une liste de topics à un dépôt en une commande ;
- utiliser une liste par défaut (thématique homelab et observabilité) quand aucune liste n'est fournie ;
- fusionner la liste avec les topics déjà présents au lieu de les écraser ;
- respecter automatiquement la limite de 20 topics ;
- récupérer le jeton d'authentification depuis la CLI GitHub (`gh`) ou depuis le gestionnaire d'identifiants Git, sans jamais le saisir en clair.

Il peut s'utiliser de deux façons : comme script autonome (`Set-ReposTopics.ps1`) ou comme fonction permanente chargée dans le profil PowerShell.

---

## 2. Prérequis

| Élément | Détail |
|---|---|
| Système | Windows 10 ou 11 (fonctionne aussi sous PowerShell 7 sur Linux et macOS) |
| PowerShell | 5.1 ou supérieur |
| Git | Installé et configuré avec un gestionnaire d'identifiants |
| GitHub CLI | Recommandé (`winget install GitHub.cli`) |
| Compte GitHub | Droits d'administration sur les dépôts ciblés |
| Réseau | Accès HTTPS à `api.github.com` |

Vérifier les versions installées :

```powershell
$PSVersionTable.PSVersion
git --version
gh --version
```

Si l'exécution de scripts est bloquée, autoriser les scripts locaux pour l'utilisateur courant :

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

---

## 3. Authentification

Le script cherche un jeton dans cet ordre :

1. la CLI GitHub : `gh auth token` ;
2. le gestionnaire d'identifiants Git : `git credential fill`.

Méthode recommandée :

```powershell
winget install GitHub.cli
gh auth login
```

Choisir `GitHub.com`, le protocole `HTTPS`, puis l'authentification par navigateur.

Vérifier que la connexion est active :

```powershell
gh auth status
```

Permissions nécessaires pour modifier les topics :

- jeton classique : portée `repo` (ou `public_repo` pour des dépôts publics uniquement) ;
- jeton à granularité fine : permission `Administration` en écriture sur les dépôts concernés.

---

## 4. Installation

Deux options au choix. Elles peuvent coexister.

### Option A : script autonome

Créer le fichier `Set-ReposTopics.ps1` dans un dossier de votre choix, par exemple `~/OneDrive/Bureau/` ou `~/bin/`, avec le contenu suivant :

```powershell
<#
.SYNOPSIS
    Ajoute des topics à un repo GitHub via l'API REST.

.EXAMPLE
    .\Set-ReposTopics.ps1 -Repo "azure-ad-graylog-observability" `
        -Topics terraform,azure,graylog,grafana

.EXAMPLE
    # Utiliser les topics par défaut (homelab et observabilité)
    .\Set-ReposTopics.ps1 -Repo "mon-autre-repo"
#>
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string]$Repo,

    [string]$Owner = "dspitech",

    [string[]]$Topics,

    [switch]$Merge   # fusionne avec les topics existants au lieu d'écraser
)

# Topics par défaut si non fournis
if (-not $Topics) {
    $Topics = @(
        "terraform","azure","infrastructure-as-code","graylog","grafana",
        "prometheus","alertmanager","siem","log-management","observability",
        "monitoring","active-directory","windows-server","dns","winlogbeat",
        "docker-compose","opensearch","azure-key-vault","devsecops","homelab"
    )
}

# Récupération du token (gh si disponible, sinon git credential)
function Get-GitHubToken {
    if (Get-Command gh -ErrorAction SilentlyContinue) {
        $t = gh auth token 2>$null
        if ($t) { return $t.Trim() }
    }
    $credInput = "protocol=https`nhost=github.com`n"
    $line = $credInput | git credential fill 2>$null |
            Select-String '^password=(.+)$'
    if ($line) { return $line.Matches.Groups[1].Value.Trim() }
    return $null
}

$token = Get-GitHubToken
if (-not $token) {
    throw "Aucun token GitHub trouvé. Installez gh (winget install GitHub.cli) et lancez 'gh auth login', ou configurez un credential helper Git."
}

$headers = @{
    "Authorization"        = "Bearer $token"
    "Accept"               = "application/vnd.github+json"
    "X-GitHub-Api-Version" = "2022-11-28"
    "User-Agent"           = "PowerShell"
}

$uri = "https://api.github.com/repos/$Owner/$Repo/topics"

# Fusion optionnelle avec les topics existants
if ($Merge) {
    try {
        $current = (Invoke-RestMethod -Uri $uri -Headers $headers).names
        $Topics  = @($current + $Topics | Select-Object -Unique)
    } catch {
        Write-Warning "Impossible de lire les topics existants : $($_.Exception.Message)"
    }
}

# Limite GitHub : 20 topics maximum
if ($Topics.Count -gt 20) {
    Write-Warning "Plus de 20 topics fournis ($($Topics.Count)). Troncature à 20."
    $Topics = @($Topics | Select-Object -First 20)
}

# Envoi
try {
    $body = @{ names = @($Topics) } | ConvertTo-Json
    $result = Invoke-RestMethod -Method Put -Uri $uri -Headers $headers `
                                -Body $body -ContentType "application/json"
    Write-Host "[OK] Topics appliqués sur $Owner/$Repo :" -ForegroundColor Green
    $result.names | ForEach-Object { Write-Host "   - $_" }
} catch {
    Write-Host "[ERREUR] $($_.Exception.Message)" -ForegroundColor Red
    if ($_.ErrorDetails.Message) {
        Write-Host $_.ErrorDetails.Message -ForegroundColor Yellow
    }
}
```

Adapter la valeur par défaut de `$Owner` à votre compte.

Pour appeler le script sans le préfixe `.\`, le placer dans un dossier du `PATH` :

```powershell
# Créer un dossier personnel pour les scripts
New-Item -ItemType Directory -Force -Path "$HOME\bin" | Out-Null

# Ajouter ce dossier au PATH utilisateur (persistant)
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
if ($userPath -notlike "*$HOME\bin*") {
    [Environment]::SetEnvironmentVariable("Path", "$userPath;$HOME\bin", "User")
}

# Prise en compte immédiate dans la session courante
$env:Path += ";$HOME\bin"
```

Copier ensuite `Set-ReposTopics.ps1` dans `$HOME\bin`.

### Option B : fonction permanente dans le profil PowerShell

Cette option rend la commande disponible dans toutes les sessions.

Ouvrir le profil (le créer s'il n'existe pas) :

```powershell
if (-not (Test-Path $PROFILE)) { New-Item -Path $PROFILE -ItemType File -Force }
notepad $PROFILE
```

Coller la fonction suivante :

```powershell
function Set-ReposTopics {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, Position=0)]
        [string]$Repo,

        [string]$Owner = "dspitech",

        [Parameter(Position=1)]
        [string[]]$Topics,

        [switch]$Merge
    )

    if (-not $Topics) {
        $Topics = @(
            "terraform","azure","infrastructure-as-code","graylog","grafana",
            "prometheus","alertmanager","siem","log-management","observability",
            "monitoring","active-directory","windows-server","dns","winlogbeat",
            "docker-compose","opensearch","azure-key-vault","devsecops","homelab"
        )
    }

    # Token
    $token = $null
    if (Get-Command gh -ErrorAction SilentlyContinue) {
        $token = (gh auth token 2>$null)
    }
    if (-not $token) {
        $line = ("protocol=https`nhost=github.com`n" | git credential fill 2>$null |
                 Select-String '^password=(.+)$')
        if ($line) { $token = $line.Matches.Groups[1].Value }
    }
    if (-not $token) { throw "Aucun token GitHub trouvé (gh ou git credential)." }
    $token = $token.Trim()

    $headers = @{
        "Authorization"        = "Bearer $token"
        "Accept"               = "application/vnd.github+json"
        "X-GitHub-Api-Version" = "2022-11-28"
        "User-Agent"           = "PowerShell"
    }
    $uri = "https://api.github.com/repos/$Owner/$Repo/topics"

    if ($Merge) {
        try {
            $cur    = (Invoke-RestMethod -Uri $uri -Headers $headers).names
            $Topics = @($cur + $Topics | Select-Object -Unique)
        } catch {}
    }
    if ($Topics.Count -gt 20) { $Topics = @($Topics | Select-Object -First 20) }

    $body = @{ names = @($Topics) } | ConvertTo-Json
    $r = Invoke-RestMethod -Method Put -Uri $uri -Headers $headers `
                           -Body $body -ContentType "application/json"
    Write-Host "[OK] $Owner/$Repo : $($r.names.Count) topics" -ForegroundColor Green
    $r.names | ForEach-Object { Write-Host "   - $_" }
}
```

Recharger le profil :

```powershell
. $PROFILE
```

---

## 5. Utilisation

### Avec le script autonome

```powershell
# Topics par défaut
.\Set-ReposTopics.ps1 -Repo "azure-ad-graylog-observability"

# Topics personnalisés
.\Set-ReposTopics.ps1 -Repo "mon-repo" -Topics terraform,azure,docker

# Fusion avec les topics existants
.\Set-ReposTopics.ps1 -Repo "mon-repo" -Merge

# Autre propriétaire
.\Set-ReposTopics.ps1 -Repo "un-repo" -Owner "autre-user"
```

### Avec la fonction du profil

```powershell
# Topics par défaut
Set-ReposTopics azure-ad-graylog-observability

# Topics personnalisés
Set-ReposTopics mon-repo -Topics terraform,azure

# Fusion
Set-ReposTopics mon-repo -Merge
```

### Résultat attendu

```
[OK] dspitech/mon-repo : 4 topics
   - terraform
   - azure
   - docker
   - homelab
```

---

## 6. Paramètres

| Paramètre | Type | Obligatoire | Défaut | Description |
|---|---|---|---|---|
| `-Repo` | string | Oui | aucun | Nom du dépôt (sans le propriétaire) |
| `-Owner` | string | Non | `dspitech` | Propriétaire du dépôt (utilisateur ou organisation) |
| `-Topics` | string[] | Non | Liste par défaut de 20 topics | Topics à appliquer, séparés par des virgules |
| `-Merge` | switch | Non | désactivé | Ajoute les topics aux existants au lieu de les remplacer |

---

## 7. Fonctionnement

Le script appelle l'endpoint REST suivant :

```
PUT https://api.github.com/repos/{owner}/{repo}/topics
```

Étapes :

1. Si aucun topic n'est fourni, la liste par défaut est utilisée.
2. Le jeton est lu depuis `gh` ou `git credential fill`.
3. Avec `-Merge`, un appel `GET` sur le même endpoint lit les topics existants, puis les deux listes sont fusionnées sans doublon.
4. La liste est tronquée à 20 éléments si nécessaire.
5. Un appel `PUT` envoie la liste finale au format `{ "names": [ ... ] }`.
6. La réponse de l'API est affichée pour confirmer le résultat.

Point important : la méthode `PUT` remplace la liste complète. Sans `-Merge`, les topics existants non listés sont supprimés.

---

## 8. Règles de GitHub sur les topics

- 20 topics maximum par dépôt.
- Lettres minuscules, chiffres et tirets uniquement.
- 50 caractères maximum par topic.
- Le premier caractère doit être une lettre ou un chiffre.
- Pas d'espaces, de points ni de tirets bas.

Valider une liste avant envoi (n'affiche que les topics invalides) :

```powershell
$Topics | Where-Object { $_ -notmatch '^[a-z0-9][a-z0-9-]{0,49}$' }
```

---

## 9. Application à plusieurs dépôts

Avec la CLI GitHub, la fonction peut être appliquée à tous les dépôts d'un compte. Commencer par lister les dépôts pour vérifier la cible :

```powershell
gh repo list dspitech --limit 100 --json name --jq '.[].name'
```

Puis appliquer les topics en mode fusion, sans écraser l'existant :

```powershell
gh repo list dspitech --limit 100 --json name --jq '.[].name' |
    ForEach-Object { Set-ReposTopics $_ -Topics homelab,devops -Merge }
```

Pour exclure les dépôts archivés ou les forks, ajouter les options `--no-archived` et `--source` à `gh repo list`.

Tester d'abord sur un seul dépôt avant d'exécuter la boucle complète.

---

## 10. Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| `Aucun token GitHub trouvé` | Aucune session `gh`, aucun identifiant Git | Lancer `gh auth login` |
| `404 Not Found` | Nom de dépôt ou propriétaire incorrect, ou jeton sans accès au dépôt privé | Vérifier `-Repo` et `-Owner`, contrôler les droits du jeton |
| `403 Forbidden` | Permissions insuffisantes | Utiliser un jeton avec la portée `repo` ou la permission `Administration` |
| `422 Unprocessable Entity` | Topic invalide (majuscule, espace, caractère interdit) | Corriger avec la règle de la section 8 |
| `401 Unauthorized` | Jeton expiré ou révoqué | Relancer `gh auth login` ou `gh auth refresh` |
| Le script ne se lance pas | Politique d'exécution restrictive | `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` |
| `Set-ReposTopics` introuvable | Profil non rechargé | `. $PROFILE` ou ouvrir un nouveau terminal |
| Les anciens topics ont disparu | Exécution sans `-Merge` | Relancer avec les topics souhaités, ou utiliser `-Merge` à l'avenir |
| Un seul topic en échec | Désérialisation JSON d'un élément unique en chaîne | Utiliser la version de ce document, qui force un tableau avec `@()` |

Afficher le détail d'une erreur de l'API :

```powershell
$_.ErrorDetails.Message
```

---

## 11. Sécurité

- Ne jamais écrire un jeton dans le script, le profil ou un dépôt.
- Ne pas afficher le jeton dans la console ni dans les journaux (`Write-Host $token` à proscrire).
- Préférer `gh auth login`, qui stocke le jeton dans le coffre d'identifiants du système.
- Utiliser un jeton à granularité fine, limité aux dépôts nécessaires, plutôt qu'un jeton classique à large portée.
- Révoquer immédiatement un jeton exposé : GitHub, Settings, Developer settings, Personal access tokens.
- Si le dépôt de scripts est public, vérifier qu'il ne contient ni jeton ni chemin personnel sensible avant chaque commit.

---

## 12. Structure recommandée du dépôt

```
github-topics-manager/
|-- README.md
|-- LICENSE
|-- Set-ReposTopics.ps1
|-- profile/
|   `-- Set-ReposTopics.function.ps1
`-- docs/
    `-- exemple-sortie.png
```

Nom de dépôt suggéré : `github-topics-manager`

Description suggérée pour le champ GitHub :

> Script PowerShell pour appliquer, fusionner et normaliser les topics de dépôts GitHub via l'API REST, avec authentification automatique par gh ou git credential.

Topics suggérés : `powershell` `github-api` `github-cli` `automation` `devops` `windows` `scripting` `repository-management`

---

## Licence

MIT. Ajouter un fichier `LICENSE` à la racine du dépôt.
