# Sample Drive Folder Structure

When a client is marked signed, this workflow automatically creates the folder structure below in Google Drive — nothing here needs to be created by hand.

```
Jamie Torres — MATTER-1024/
├── Correspondence/
├── Documents/
├── Discovery/
└── Billing/
```

The parent folder is named from the client's name and matter ID. The four subfolders shown are the default set and are created automatically inside it.

## Customizing the subfolder list

Set the `ONBOARDING_DRIVE_SUBFOLDERS` variable in n8n to a comma-separated list of your own subfolder names — no code changes needed. For example:

```
Correspondence,Documents,Discovery,Billing,Court Filings
```
