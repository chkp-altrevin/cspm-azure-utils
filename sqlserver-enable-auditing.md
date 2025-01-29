## Enable SQL Server Auditing
If you want to enrich your third partys you may want to enable.
To find login audit logs for an Azure SQL server using the Azure CLI, you'll need to ensure that auditing is enabled on the server and then query the relevant logs. Below is a step-by-step guide with detailed examples.

---

### **Step 1: Enable Azure SQL Auditing (if not already enabled)**

Azure SQL Server must have auditing enabled to log events like logins. To check and enable auditing:

#### Enable Server-Level Auditing
```bash
az sql server audit-policy update \
    --resource-group <RESOURCE_GROUP_NAME> \
    --server <SQL_SERVER_NAME> \
    --state Enabled \
    --storage-account <STORAGE_ACCOUNT_NAME>
```

- Replace `<RESOURCE_GROUP_NAME>` with your Azure resource group name.
- Replace `<SQL_SERVER_NAME>` with the name of your SQL server.
- Replace `<STORAGE_ACCOUNT_NAME>` with the name of your Azure Storage account where logs will be stored.

---

### **Step 2: Enable Diagnostics for SQL Server**

Enable diagnostic settings to send logs to a Log Analytics workspace or storage account.

```bash
az monitor diagnostic-settings create \
    --name "SqlServerDiagnostics" \
    --resource <SQL_SERVER_RESOURCE_ID> \
    --workspace <LOG_ANALYTICS_WORKSPACE_ID> \
    --logs '[{"category": "SQLSecurityAuditEvents", "enabled": true}]'
```

- Replace `<SQL_SERVER_RESOURCE_ID>` with the resource ID of your SQL server. You can find it using:
  ```bash
  az sql server show --name <SQL_SERVER_NAME> --resource-group <RESOURCE_GROUP_NAME> --query id -o tsv
  ```
- Replace `<LOG_ANALYTICS_WORKSPACE_ID>` with the ID of your Log Analytics workspace.

---

### **Step 3: Query Audit Logs**

If the logs are being sent to a Log Analytics workspace, you can query them using Azure CLI. The specific category to look for is `SQLSecurityAuditEvents`.

#### Query the Logs in Log Analytics Workspace
```bash
az monitor log-analytics query \
    --workspace <LOG_ANALYTICS_WORKSPACE_ID> \
    --query "AzureDiagnostics | where Category == 'SQLSecurityAuditEvents' and TimeGenerated > ago(1d)" \
    --timespan P1D
```

- Replace `<LOG_ANALYTICS_WORKSPACE_ID>` with your workspace ID.
- Modify the `TimeGenerated > ago(1d)` to change the time span as needed (e.g., `ago(7d)` for the last 7 days).

---

### **Step 4: Filter for Login Events**

To specifically look for login-related events, refine your query:

```bash
az monitor log-analytics query \
    --workspace <LOG_ANALYTICS_WORKSPACE_ID> \
    --query "AzureDiagnostics | where Category == 'SQLSecurityAuditEvents' and LogicalServerName_s == '<SQL_SERVER_NAME>' and EventPrimaryName_s == 'Login' and TimeGenerated > ago(1d)" \
    --timespan P1D
```

- Replace `<SQL_SERVER_NAME>` with the name of your SQL server.
- Add additional filters if needed (e.g., user names or specific IP addresses).

---

### **Step 5: (Optional) Query Logs in Storage Account**

If logs are stored in a Storage Account, you can download and analyze them:

1. List available blobs in the storage container:
   ```bash
   az storage blob list \
       --account-name <STORAGE_ACCOUNT_NAME> \
       --container-name <CONTAINER_NAME> \
       --output table
   ```

2. Download the desired log file:
   ```bash
   az storage blob download \
       --account-name <STORAGE_ACCOUNT_NAME> \
       --container-name <CONTAINER_NAME> \
       --name <BLOB_NAME> \
       --file <LOCAL_FILE_NAME>
   ```

3. Open and analyze the downloaded file (usually in JSON format).

---

### Example

1. Enable auditing:
   ```bash
   az sql server audit-policy update \
       --resource-group MyResourceGroup \
       --server myserver123 \
       --state Enabled \
       --storage-account mystorageaccount
   ```

2. Enable diagnostic settings:
   ```bash
   az monitor diagnostic-settings create \
       --name "SqlServerDiagnostics" \
       --resource $(az sql server show --name myserver123 --resource-group MyResourceGroup --query id -o tsv) \
       --workspace $(az monitor log-analytics workspace show --resource-group MyResourceGroup --name MyLogWorkspace --query customerId -o tsv) \
       --logs '[{"category": "SQLSecurityAuditEvents", "enabled": true}]'
   ```

3. Query login events for the past day:
   ```bash
   az monitor log-analytics query \
       --workspace $(az monitor log-analytics workspace show --resource-group MyResourceGroup --name MyLogWorkspace --query customerId -o tsv) \
       --query "AzureDiagnostics | where Category == 'SQLSecurityAuditEvents' and LogicalServerName_s == 'myserver123' and EventPrimaryName_s == 'Login' and TimeGenerated > ago(1d)" \
       --timespan P1D
   ```

---

This approach ensures you can find and analyze Azure SQL Server login audit logs effectively.
