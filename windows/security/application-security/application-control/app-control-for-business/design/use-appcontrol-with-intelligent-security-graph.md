---
title: Authorize reputable apps with the Intelligent Security Graph (ISG)
description: Automatically authorize applications that Microsoft's ISG recognizes as having known good reputation.
ms.localizationpriority: medium
ms.date: 09/11/2024
ms.topic: how-to
---

# Authorize reputable apps with the Intelligent Security Graph (ISG)

[!INCLUDE [Feature availability note](../includes/feature-availability-note.md)]

App Control can be difficult to implement in organizations that don't deploy and manage applications through an IT-managed system. In such environments, users can acquire the applications they want to use for work, making it hard to build an effective App Control policy.

To reduce end-user friction and helpdesk calls, you can set App Control for Business to automatically allow applications that Microsoft's Intelligent Security Graph (ISG) recognizes as having known good reputation. The ISG option helps organizations begin to implement App Control even when the organization has limited control over their app ecosystem. To learn more about the ISG, see the Security section in [Major services and features in Microsoft Graph](/graph/overview-major-services).

> [!WARNING]
> Binaries that are critical to boot the system must be allowed using explicit rules in your App Control policy. Do not rely on the ISG to authorize these files.
>
> The ISG option is not the recommended way to allow apps that are business critical. You should always authorize business critical apps using explicit allow rules or by installing them with a [managed installer](configure-authorized-apps-deployed-with-a-managed-installer.md).

## How does App Control work with the ISG?

The ISG isn't a "list" of apps. Rather, it uses the same vast security intelligence and machine learning analytics that power Microsoft Defender SmartScreen and Microsoft Defender Antivirus to help classify applications as having "known good", "known bad", or "unknown" reputation. This cloud-based AI is based on trillions of signals collected from Windows endpoints and other data sources, and processed every 24 hours. As a result, the decision from the cloud can change.

App Control only checks the ISG for binaries that aren't explicitly allowed or denied by your policy, and that weren't installed by a managed installer. When such a binary runs on a system with App Control enabled with the ISG option, App Control will check the file's reputation by sending its hash and signing information to the cloud. If the ISG reports that the file has a "known good" reputation, then the file will be allowed to run. Otherwise, it will be blocked by App Control.

If the file with good reputation is an application installer, the installer's reputation will pass along to any files that it writes to disk. This way, all the files needed to install and run an app inherit the positive reputation data from the installer. Files authorized based on the installer's reputation will have the $KERNEL.SMARTLOCKER.ORIGINCLAIM kernel Extended Attribute (EA) written to the file.

App Control periodically requeries the reputation data on a file. Additionally, enterprises can specify that any cached reputation results are flushed on reboot by using the **Enabled:Invalidate EAs on Reboot** option.

## Configuring ISG authorization for your App Control policy

Setting up the ISG is easy using any management solution you wish. Configuring the ISG option involves these basic steps:

- [Ensure that the **Enabled:Intelligent Security Graph authorization** option is set in the App Control policy XML](#ensure-that-the-isg-option-is-set-in-the-app-control-policy-xml)
- [Enable the necessary services to allow App Control to use the ISG correctly on the client](#enable-the-necessary-services-to-allow-app-control-to-use-the-isg-correctly-on-the-client)

### Ensure that the ISG option is set in the App Control policy XML

To allow apps and binaries based on the Microsoft Intelligent Security Graph, the **Enabled:Intelligent Security Graph authorization** option must be specified in the App Control policy. This step can be done with the Set-RuleOption cmdlet. You should also set the **Enabled:Invalidate EAs on Reboot** option so that ISG results are verified again after each reboot. The ISG option isn't recommended for devices that don't have regular access to the internet. The following example shows both options set.

```xml
<Rules>
    <Rule>
      <Option>Enabled:Unsigned System Integrity Policy</Option>
    </Rule>
    <Rule>
      <Option>Enabled:Advanced Boot Options Menu</Option>
    </Rule>
    <Rule>
      <Option>Required:Enforce Store Applications</Option>
    </Rule>
    <Rule>
      <Option>Enabled:UMCI</Option>
    </Rule>
    <Rule>
      <Option>Enabled:Managed Installer</Option>
    </Rule>
    <Rule>
      <Option>Enabled:Intelligent Security Graph Authorization</Option>
    </Rule>
    <Rule>
      <Option>Enabled:Invalidate EAs on Reboot</Option>
    </Rule>
</Rules>
```

### Enable the necessary services to allow App Control to use the ISG correctly on the client

In order for the heuristics used by the ISG to function properly, other components in Windows must be enabled. You can configure these components by running the appidtel executable in `c:\windows\system32`.

```console
appidtel start
```

This step isn't required for App Control policies deployed over MDM, as the CSP will enable the necessary components. This step is also not required when the ISG is configured using Configuration Manager's App Control integration.

## Security considerations with the ISG option

Since the ISG is a heuristic-based mechanism, it doesn't provide the same security guarantees as explicit allow or deny rules. It's best suited where users operate with standard user rights and where a security monitoring solution like Microsoft Defender for Endpoint is used.

Processes running with kernel privileges can circumvent App Control by setting the ISG extended file attribute to make a binary appear to have known good reputation.

Also, since the ISG option passes along reputation from app installers to the binaries they write to disk, it can over-authorize files in some cases. For example, if the installer launches the app upon completion, any files the app writes during that first run will also be allowed.

## Known limitations with using the ISG

Since the ISG only allows binaries that are "known good", there are cases where the ISG may be unable to predict whether legitimate software is safe to run. If that happens, the software will be blocked by App Control. In this case, you need to allow the software with a rule in your App Control policy, deploy a catalog signed by a certificate trusted in the App Control policy, or install the software from an App Control managed installer. Installers or applications that dynamically create binaries at runtime, and self-updating applications, may exhibit this symptom.

Packaged apps aren't supported with the ISG and will need to be separately authorized in your App Control policy. Since packaged apps have a strong app identity and must be signed, it's straightforward to [authorize packaged apps](manage-packaged-apps-with-appcontrol.md) with your App Control policy.

The ISG doesn't authorize kernel mode drivers. The App Control policy must have rules that allow the necessary drivers to run.

> [!NOTE]
> A rule that explicitly denies or allows a file will take precedence over that file's reputation data. Microsoft Intune's built-in App Control support includes the option to trust apps with good reputation via the ISG, but it has no option to add explicit allow or deny rules. In most cases, customers using App Control will need to deploy a custom App Control policy (which can include the ISG option if desired) using [Intune's OMA-URI functionality](../deployment/deploy-appcontrol-policies-using-intune.md#deploy-app-control-policies-with-custom-oma-uri).
