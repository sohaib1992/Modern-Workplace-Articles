# Migrating macOS Configuration Policies Between Intune Tenants: A Practical Guide

If you've ever been handed a folder of JSON files from another Intune tenant and told "just import these," you have probably discovered it's not as simple as it sounds. Intune's built-in **Import policy** feature only supports one specific policy type and format everything else needs a bit of engineering. This article walks through the problem, why it happens, and working PowerShell scripts to solve it.

## The Problem

When you export device configuration policies from Intune whether through the Graph API directly, or a third-party tool like [IntuneManagement](https://github.com/Micke-K/IntuneManagement) you get JSON files. Naturally, you would expect Intune's own **Create → Import policy** button to accept them back in. It does not, for two reasons.

### 1. Not all policy types are importable

The Import policy button in the Intune portal only supports **Settings Catalog** policies (`#microsoft.graph.deviceManagementConfigurationPolicy`). If your export folder contains a mix of policy types which is normal for any real macOS device management setup — most of them will simply be rejected:

| Policy type | Import policy button supports it? |
|---|---|
| Settings Catalog | ✅ Yes |
| Custom (`.mobileconfig` payload) | ❌ No — must be uploaded as a Custom profile |
| VPN | ❌ No — must be rebuilt manually |
| Wi-Fi | ❌ No — must be rebuilt manually |
| Compliance policies | ❌ No |
| Shell scripts | ❌ No — separate upload flow entirely |

### 2. Even Settings Catalog exports can fail

If your JSON came from a raw Graph API export (rather than the portal's own **Export JSON** button), it usually contains extra metadata the importer doesn't expect: OData annotations (`@odata.id`, `@odata.editLink`, `children@odata.type`), read-only fields like `createdDateTime`, and sometimes UTF-16 encoding instead of UTF-8. The result is a generic, unhelpful error: *"There was an issue importing the policy, please try again."* Retrying doesn't help the file needs reshaping first.

## Step 1: Work Out What You're Actually Dealing With

Before converting anything, sort your files by type. This single script reads every JSON file in a folder and prints its Graph `@odata.type`, so you know exactly what you are working with:

```powershell
Get-ChildItem "C:\PolicyExport\*.json" | ForEach-Object {
    try   { $j = Get-Content $_.FullName -Raw -Encoding Unicode | ConvertFrom-Json }
    catch { $j = Get-Content $_.FullName -Raw -Encoding UTF8    | ConvertFrom-Json }

    [PSCustomObject]@{
        File        = $_.Name
        Type        = $j.'@odata.type'
        HasPayload  = [bool]$j.payload
        DisplayName = if ($j.displayName) { $j.displayName } else { $j.name }
    }
} | Sort-Object Type | Format-Table -AutoSize
```

This handles both UTF-16 and UTF-8 automatically (the `try/catch` falls back if the first encoding guess fails) and gives you a clean breakdown for example, a mix of Settings Catalog policies, a handful of Custom profiles, and maybe one VPN or Wi-Fi config.

## Step 2: Extract Custom Profiles (.mobileconfig)

Custom configuration profiles in Intune are really just a wrapper around an Apple `.mobileconfig` file, base64-encoded inside the JSON's `payload` field. You can't import these through the portal at all — you have to extract the underlying `.mobileconfig` and upload it as a **Custom** profile template.

```powershell
$folder = "C:\PolicyExport"
$out    = "C:\PolicyExport\Extracted"
New-Item -ItemType Directory -Path $out -Force | Out-Null

Get-ChildItem "$folder\*.json" | ForEach-Object {
    try   { $json = Get-Content $_.FullName -Raw -Encoding Unicode | ConvertFrom-Json }
    catch { $json = Get-Content $_.FullName -Raw -Encoding UTF8    | ConvertFrom-Json }

    if (-not $json.payload) { return }   # not a Custom profile — skip

    # Name the output after the source JSON file to avoid collisions —
    # payloadFileName is often reused across multiple profiles
    $file = Join-Path $out ($_.BaseName + ".mobileconfig")
    [System.IO.File]::WriteAllBytes($file, [Convert]::FromBase64String($json.payload))

    Write-Host "Extracted: $($_.BaseName).mobileconfig  ->  $($json.displayName)" -ForegroundColor Green
    Write-Host "  Payload name : $($json.payloadName)"
    Write-Host "  Channel      : $($json.deploymentChannel)"
}
```

**A gotcha worth knowing:** the script names each output file after the *source JSON filename*, not the `payloadFileName` field inside it. Multiple profiles frequently share a generic payload filename (e.g. `profile.mobileconfig`), and if you name outputs after that field instead, later files silently overwrite earlier ones — you'll end up with far fewer files than you started with and no error to tell you why.

Once extracted, upload each `.mobileconfig` via **Devices → macOS → Configuration → Create → Templates → Custom**, using the printed *Payload name* and *Channel* (device or user) values.

## Step 3: Convert Settings Catalog Policies for the Import Button

This is the type the portal *can* import but only if the JSON matches the exact shape Intune's own export produces. A raw Graph export needs its OData annotations stripped first. This script does that recursively, regardless of how deeply nested the settings structure is:

```powershell
function Clean-Node($node) {
    if ($node -is [System.Collections.IEnumerable] -and $node -isnot [string]) {
        return ,@($node | ForEach-Object { Clean-Node $_ })
    }
    if ($node -is [PSCustomObject]) {
        $o = [ordered]@{}
        foreach ($p in $node.PSObject.Properties) {
            $k = $p.Name
            # Drop annotation keys such as children@odata.type, @odata.id, edit/nav links
            if ($k -like "*@odata*" -and $k -ne "@odata.type") { continue }
            # Drop @odata.type only where a genuine Intune export omits it
            if ($k -eq "@odata.type" -and $p.Value -in @(
                "#microsoft.graph.deviceManagementConfigurationSetting",
                "#microsoft.graph.deviceManagementConfigurationGroupSettingValue",
                "#microsoft.graph.deviceManagementConfigurationChoiceSettingValue")) { continue }
            $o[$k] = Clean-Node $p.Value
        }
        return [PSCustomObject]$o
    }
    return $node
}

$folder = "C:\PolicyExport"
$out    = "C:\PolicyExport\ImportReady"
New-Item -ItemType Directory -Path $out -Force | Out-Null

Get-ChildItem "$folder\*.json" | ForEach-Object {
    try   { $j = Get-Content $_.FullName -Raw -Encoding Unicode | ConvertFrom-Json }
    catch { $j = Get-Content $_.FullName -Raw -Encoding UTF8    | ConvertFrom-Json }

    if ($j.'@odata.type' -ne '#microsoft.graph.deviceManagementConfigurationPolicy') { return }

    $clean = [ordered]@{
        name              = $j.name
        description       = $j.description
        platforms         = $j.platforms
        technologies      = $j.technologies
        roleScopeTagIds   = @("0")
        settings          = Clean-Node $j.settings
        templateReference = Clean-Node $j.templateReference
    }

    $file = Join-Path $out ($_.BaseName + ".json")
    [System.IO.File]::WriteAllText($file, ($clean | ConvertTo-Json -Depth 100), (New-Object System.Text.UTF8Encoding $false))
    Write-Host "Ready: $($_.BaseName).json" -ForegroundColor Green
}
```

A couple of things that matter here:

- **`-Depth 100`** — settings catalog JSON nests deeply (setting → group → child setting → value), and PowerShell's default `ConvertTo-Json` depth of 2 will silently truncate it, producing a broken file with no error.
- **`roleScopeTagIds`** is reset to `["0"]` (default) because scope tag IDs from a source tenant almost never exist in the destination tenant.
- Always **test with one simple policy first** (pick something small with few settings) before batch-importing the rest — that way, if a particular policy type still fails, you find out on one file instead of eighteen.

To import: **Devices → macOS → Configuration → Create → Import policy**, and select a file from the `ImportReady` folder.

## Step 4: Handle the Leftovers Manually

VPN, Wi-Fi, and Compliance policy JSON files don't have an import path in the portal at all — not because of encoding or formatting, but because Intune simply doesn't offer one for these types. The practical approach is to open the JSON, read off the settings values, and re-enter them through the relevant policy template in the portal. Tedious for a handful of files, but far more reliable than fighting an importer that was never built for this policy type.

## Summary

| What you have | What to do |
|---|---|
| Settings Catalog JSON, portal-exported format | Import policy button, as-is |
| Settings Catalog JSON, raw Graph export | Strip OData metadata first (script above), then Import policy |
| Custom profile JSON (`.mobileconfig` payload) | Extract payload (script above), upload via Templates → Custom |
| VPN / Wi-Fi / Compliance JSON | Rebuild manually using the JSON as a reference |
| Any file that won't parse | Check encoding — Graph exports are often UTF-16, not UTF-8 |

The underlying lesson: "export" and "import" only mean "copy/paste" when both ends were built to talk to each other. Crossing tenants, tools, or Graph API versions usually means at least one translation step in between — and once you know the shape of the problem, it's a fast fix rather than a mystery.
