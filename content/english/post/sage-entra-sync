+++
author = "Grant Houser"
title = "Syncing Employees from Sage to Entra ID"
date = "2026-02-06"
description = "Syncing Employees from Sage to Entra ID"
tags = ["sage", "entra", "powershell", "azure-automation", "sql"]
categories = ["integration", "automation", "onboarding"]
+++

I recently wrapped up a project that solved a common operational headache: ensuring our managers have seamless access to company resources without manual overhead. This was an excellent opportunity to expand my automation toolkit, as it marked my first time leveraging Azure Automation Accounts.

While I’ve historically used Power Automate and Azure Data Factory for integrations, Azure Automation proved to be a highly straightforward and scalable service for this type of identity synchronization. It’s a tool I expect to lean on heavily moving forward.

&nbsp;
## 1. The Objective
The goal was simple: identify all managers within our accounting software (SAGE)—our "source of truth"—and ensure they have corresponding Entra ID logins for SharePoint and OneDrive. By building a bridge between SAGE and Entra, we ensure that access is granted (and revoked) automatically based on employment status.</p>

&nbsp;
## 2. Architecting the Bridge
Since I had already established a pipeline between SAGE and an Azure SQL database for previous onboarding integrations, the scope of this project focused on linking that database to Entra ID.
        
### Data Preparation
I created a dedicated table to manage manager information. To ensure "sticky" logins—where a username stays with an employee even if they leave and return—I opted for a persistent table rather than a dynamic view. I used the immutable 32-bit employee key from SAGE as the primary identifier.

| Employee Key | User Name | Email | FirstName | LastName | Department | IsActive |
        
### Logic & Scripting
I developed a stored procedure to identify managers and handle "upserts"—updating existing records or inserting new ones as hiring occurs. For the synchronization logic, I wrote a PowerShell runbook within Azure Automation. The script performs the following:
* **Authenticates with Microsoft Graph and the Azure SQL database.**
* **Executes the stored procedure to refresh the manager list.**
* **Compares the database records against existing Entra users.**
* **Creates new accounts or updates/disables existing ones based on the IsActive flag.**

&nbsp;
## 3. Security and Permissions
Adhering to the principle of least privilege, I configured the following:
* **Managed Identity:** The Automation Account was granted User.ReadWrite.All and Directory.ReadWrite.All permissions within Entra ID.
* **Database Access:** Created a specific SQL login for the automation service, restricted to executing the necessary stored procedure and reading the managers table.

&nbsp;
## 4. Results & Future Scalability
The sync is now scheduled to run every morning. This automation ensures that as soon as a manager is terminated in Sage, their access to company resources is disabled by the next business day—significantly improving our security posture and reducing manual IT tickets.

Looking ahead, this project serves as a blueprint for wider integrations. I plan to expand this to handle automated email provisioning through our web host and eventually use it as a foundation for a B2C Entra tenant to provide login capabilities for our entire workforce.