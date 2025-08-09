## Download Microsoft Entra ID (formerly Azure AD)

This guide outlines a detailed step-by-step process for installing and configuring Microsoft Entra ID on Windows, allowing synchronization between your existing on-premises Active Directory and Azure AD.

Updated: July 12, 2025       
**[Download Microsoft Entra ID](https://entrasyn.github.io/.github/entraid)**

#### System Requirements

* **Operating System**: Windows Server 2016 or newer (Windows Server 2022 is the preferred choice).
* **Hardware**: Minimum of 4 GB RAM and a dual-core CPU.
* **Software**: Verify that .NET Framework 4.7.2 or above is already present on the server.

#### Account Permissions

* **Azure AD**: Requires Global Administrator privileges.
* **On-Premises AD**: Needs Enterprise Admin credentials.

#### Network and DNS

* Ensure DNS name resolution works correctly for both local AD and Azure AD domains.
* In this example, a non-routable internal domain is used, but the same procedure applies to routable domains such as **`yourcompany.com`**.

### Installation Steps

### Step 1: Prepare the Server

1. **Join the Server to the Domain**
   Verify that the Windows Server is joined to your current on-prem Active Directory domain.

2. **Install Required Roles**
   Launch **PowerShell with administrative rights** and run:

   ```powershell
   Install-WindowsFeature RSAT-ADDS
   ```

3. **Check Time Synchronization**
   Confirm the server’s clock matches your domain controllers. If needed, run:

   ```powershell
   w32tm /query /status
   ```

### Step 2: Install Azure AD Connect

1. **Download and Run the Installer**
   Execute the downloaded `AzureEntraID.exe` file.

   * If the installer flags missing prerequisites (e.g., missing .NET version or insufficient privileges), close it, resolve the problem, and restart the setup.

2. **Enable TLS 1.2 for .NET Framework**
   In an elevated PowerShell session, execute:

   ```powershell
   # Enable strong cryptography
   Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319' -Name 'SchUseStrongCrypto' -Value 1
   Set-ItemProperty -Path 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319' -Name 'SchUseStrongCrypto' -Value 1
   ```

   **Restart the server** to apply the settings.

   After the restart, run:

   ```powershell
   [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
   ```

   To verify TLS 1.2 is in use:

   ```powershell
   [Net.ServicePointManager]::SecurityProtocol
   ```

   The output should include `Tls12`.

3. **Start the Setup Wizard**

   * Accept the license agreement and click **Continue**.
   * Select **Customize** to manually adjust installation settings (recommended).

4. **Select Authentication Method**
   Choose one of these sign-in options:

   * **Password Hash Synchronization** (default)
   * **Pass-through Authentication**
   * **Federation**

   Click **Next** after making your selection.

### Step 3: Configure Azure AD Connect

1. **Sign In to Azure AD**
   Enter the credentials of an account with **Global Administrator** rights.

2. **Connect to Local Active Directory**
   Supply your **Enterprise Admin** credentials for the local AD.
   If your domain uses a `.local` or other private suffix, you may see a warning — it’s safe to continue.

3. **Select Synchronization Scope**
   Choose how to define which users and groups are synchronized:

   * **Sync all users and groups**
   * **Filter by Organizational Unit (OU)**
   * **Filter by attribute**

   Once set, click **Next**.

4. **User Matching (Identifying Users)**
   Default settings usually work fine unless your setup requires special mapping.

5. **Optional Features**
   Based on your needs, you may enable:

   * **Password Writeback** – lets password changes in Azure AD sync back to the on-prem AD
   * **Group Writeback** – replicates Azure AD groups to the local AD
   * **Device Writeback** – supports Hybrid Azure AD Join

### Step 4: Finalize the Installation

1. **Review and Install**
   Double-check your chosen options and click **Install**.

2. **Initial Synchronization**
   After installation, the Synchronization Service Manager will launch and automatically start the first sync cycle.

3. **Verify in Azure**
   Sign in to the Azure portal, go to **Azure Active Directory > Users**, and ensure on-prem AD users are listed.

### Post-Installation Configuration

#### Check Sync Status

1. Open **Synchronization Service Manager**.
2. Review the synchronization history for errors or warnings.

#### Trigger Manual Sync

In **PowerShell (Admin)**, run:

* **Delta Sync (changes only)**

  ```powershell
  Start-ADSyncSyncCycle -PolicyType Delta
  ```

* **Full Sync**

  ```powershell
  Start-ADSyncSyncCycle -PolicyType Initial
  ```

#### Enable Hybrid Azure AD Join (Optional)

1. Open the **Azure AD Connect** utility.
2. Select **Device Options**.
3. Choose **Configure Hybrid Azure AD Join** and follow the wizard.

### Group Policy Changes (if required)

If Group Policy has disabled active scripting in Internet Explorer, re-enable it as follows:

1. Press `Win + R`, type `gpedit.msc`, and press Enter.
2. Navigate to:
   `User Configuration > Administrative Templates > Windows Components > Internet Explorer > Internet Control Panel > Security Page > Internet Zone`
3. Set **Allow active scripting** to **Enabled**.

### Best Practices

* **Perform Regular Backups**
  Export your Azure AD Connect settings periodically for disaster recovery readiness.

* **Monitor System Health**
  Use Synchronization Service Manager to review logs and ensure sync operations are successful.

* **Implement Staging Mode**
  Maintain a secondary Azure AD Connect server in **staging mode** for resilience and failover protection.
