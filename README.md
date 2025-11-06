# Bastion-Windows-RDP-using-EntraID-To-Authenticate


Here’s how to **securely connect to an Azure Windows VM via Azure Bastion** using **Microsoft Entra ID (Azure AD)** authentication — **without exposing the VM publicly**:

---

## 🔒 **Architecture Overview**

```mermaid
flowchart LR
    subgraph Azure
        A[User Workstation<br>(Azure Portal / RDP)] -->|Entra ID Auth| B[Azure Bastion]
        B -->|Private Network (No Public IP)| C[Windows VM]
        D[Microsoft Entra ID] -->|Token-Based Auth| B
        D -->|Token Validation| C
    end
```

**Key points:**

* The VM **has no public IP**.
* Azure Bastion provides **RDP/SSH via browser** through the **Azure Portal** or **Bastion client**.
* Microsoft Entra ID handles **Single Sign-On (SSO)** authentication.
* No inbound NSG rules or open RDP (3389) ports are needed.

---

## ✅ **Step-by-Step Configuration**

### **1. Prepare Network and VM**

* Ensure the VM is deployed in a **VNet**.
* Create a **subnet named `AzureBastionSubnet`** (minimum /27).
* Confirm the VM **has no Public IP** and RDP (3389) is **not open** on the NSG.

```bash
az network nsg rule delete \
  --nsg-name MyVM-NSG \
  --name "default-allow-rdp"
```

---

### **2. Enable Entra ID Login on VM**

Use Azure CLI to enable Entra ID-based login for Windows:

```bash
az vm identity assign --name MyVM --resource-group MyRG
az vm extension set \
  --publisher Microsoft.Azure.ActiveDirectory \
  --name AADLoginForWindows \
  --resource-group MyRG \
  --vm-name MyVM
```

---

### **3. Assign Entra Role to User**

Assign the **Virtual Machine Administrator Login** or **Virtual Machine User Login** role to your user or group:

```bash
az role assignment create \
  --assignee user@domain.com \
  --role "Virtual Machine Administrator Login" \
  --scope $(az vm show --name MyVM --resource-group MyRG --query id -o tsv)
```

---

### **4. Deploy Azure Bastion**

Create Bastion in the same VNet as the VM:

```bash
az network bastion create \
  --name MyBastion \
  --public-ip-address MyBastionPIP \
  --resource-group MyRG \
  --vnet-name MyVNet \
  --sku Standard
```

> The **Bastion public IP** is for the Bastion service itself — not the VM.

---

### **5. Connect to VM via Bastion**

1. Go to **Azure Portal → VM → Connect → Bastion**
2. Select **Microsoft Entra ID** as the **Authentication Type**
3. Sign in using your **Entra credentials (SSO)**
4. The RDP session opens in the browser (HTML5 client)

---

### **6. Optional: Private Bastion (No Public IP at All)**

If you want *zero public exposure* — even for Bastion:

* Use **Bastion with Private IP**.
* Access through a **VPN** or **ExpressRoute** connection.

```bash
az network bastion create \
  --name MyPrivateBastion \
  --vnet-name MyVNet \
  --resource-group MyRG \
  --sku Standard \
  --enable-tunneling true \
  --enable-ip-connect true \
  --public-ip-address ""
```

Then connect using the **Azure Bastion native client**:

```bash
az network bastion rdp --name MyPrivateBastion --target-resource-id /subscriptions/{sub}/resourceGroups/MyRG/providers/Microsoft.Compute/virtualMachines/MyVM
```

---

## 🧠 **Best Practices**

* ✅ Use **Entra ID-only** logins for RBAC control.
* 🚫 Do **not** assign local admin passwords.
* 🔐 Enable **Conditional Access + MFA** for VM access.
* 📊 Log connections in **Azure Monitor / Log Analytics**.
* 🧱 Ensure NSG denies inbound RDP traffic (3389).

---

