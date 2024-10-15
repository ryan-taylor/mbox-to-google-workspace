# Import .mbox files to Google Workspace (formerly G Suite / Google Apps)

Modified by Ryan Taylor (<ryantaylor.codes>)

This script allows Google Workspace admins to import mbox files in bulk for their users.

**DISCLAIMER**: This is not an official Google product.

Original author: Liron Newman (<lironn@google.com>)

## Disclaimer

This modified version of the import-mailbox-to-gmail script is provided "as is", without warranty of any kind, express or implied. The author (Ryan Taylor) makes no claims about its suitability for any purpose and offers no guarantee of support. Users are advised to test thoroughly before using it in any critical environment.

While efforts have been made to ensure the script's functionality, the author is not responsible for any data loss, damage, or other issues that may arise from its use. Always back up your data before performing any import operations.

This tool is not officially supported by Google. Use at your own risk.

Users should be aware that this tool processes email data. Ensure you have the necessary permissions and comply with relevant data protection regulations when using this tool.

## Requirements

- Python 3.6 or higher
- Google API Python Client Library

## Overview

This script allows you to import multiple mbox files into Gmail accounts within your Google Workspace domain. It uses a service account for authentication, enabling bulk imports for multiple users. If you're migrating from Mozilla Thunderbird, consider using the [mail-importer](https://github.com/google/mail-importer) tool instead, which is specifically designed for Thunderbird migrations.

## Modifications

This version has been updated by Ryan Taylor (<ryantaylor.codes>) to work with the latest Google API client library and includes additional logging and error handling features. The core functionality remains the same as the original script.

## Comprehensive Guide: Importing Mailboxes to Google Workspace using Service Account

### 1. Set up Google Cloud Project

#### 1.1 Create a new project or select an existing one

```bash
gcloud projects create [PROJECT_ID] --name="[PROJECT_NAME]"
gcloud config set project [PROJECT_ID]
```

#### 1.2 Enable the Gmail API

```bash
gcloud services enable gmail.googleapis.com
```

### 2. Create and Configure Service Account

#### 2.1 Create a service account

```bash
gcloud iam service-accounts create import-mailbox-sa --display-name="Import Mailbox Service Account"
```

#### 2.2 Generate a key for the service account

```bash
gcloud iam service-accounts keys create key.json --iam-account=import-mailbox-sa@[PROJECT_ID].iam.gserviceaccount.com
```

#### 2.3 Get the service account details

```bash
gcloud iam service-accounts describe import-mailbox-sa@[PROJECT_ID].iam.gserviceaccount.com
```

### 3. Set up Domain-Wide Delegation

#### 3.1 Enable domain-wide delegation for the service account (web interface required)

- Go to [Google Cloud Console](https://console.cloud.google.com/)
- Navigate to "IAM & Admin" > "Service Accounts"
- Find your service account and click on it
- Click "Edit" at the top of the page
- Enable the "Enable Google Workspace Domain-wide Delegation" option
- Save changes

#### 3.2 Configure OAuth consent screen (web interface required)

- Go to [Google Cloud Console](https://console.cloud.google.com/)
- Navigate to "APIs & Services" > "OAuth consent screen"
- Choose "Internal" for user type
- Fill in the required information and save

#### 3.3 Set up domain-wide delegation in Google Workspace Admin Console (web interface required)

- Go to [Google Workspace Admin Console](https://admin.google.com/)
- Navigate to Security > API Controls > Domain-wide Delegation
- Click "Add new"
- Enter the Client ID (oauth2ClientId from service account details)
- For OAuth scopes, enter:

  ```plaintext
  https://www.googleapis.com/auth/gmail.insert,https://www.googleapis.com/auth/gmail.labels
  ```

- Click "Authorize"

For more details on delegating authority to a service account, refer to:  
<https://developers.google.com/identity/protocols/oauth2/service-account?authuser=4#delegatingauthority>

### 4. Prepare the Environment

#### 4.1 Install required Python libraries

```bash
pip install google-auth google-auth-oauthlib google-auth-httplib2 google-api-python-client
```

#### 4.2 Clone the import-mailbox-to-gmail repository

```bash
git clone https://github.com/google/import-mailbox-to-gmail.git
cd import-mailbox-to-gmail
```

#### 4.3 Organize your MBOX files

```bash
mbox_directory/
├── user1@domain.com/
│   ├── Inbox.mbox
│   └── Archive.mbox
└── user2@domain.com/
    ├── Inbox.mbox
    └── Archive.mbox
```

### 5. Run the Import Script

#### 5.1 Basic command

```bash
python import-mailbox-to-gmail.py --json key.json --dir /path/to/mbox_directory
```

#### 5.2 Bash script (save as `import_mailboxes.sh`)

```bash
#!/bin/bash

PROJECT_ID="[YOUR_PROJECT_ID]"
MBOX_DIR="/path/to/mbox_directory"

# Activate service account
gcloud auth activate-service-account --key-file=key.json

# Run import script
python import-mailbox-to-gmail.py --json key.json --dir "$MBOX_DIR"

# Deactivate service account
gcloud auth revoke
```

#### 5.3 Fish script (save as `import_mailboxes.fish`)

```fish
#!/usr/bin/env fish

set PROJECT_ID "[YOUR_PROJECT_ID]"
set MBOX_DIR "/path/to/mbox_directory"

# Activate service account
gcloud auth activate-service-account --key-file=key.json

# Run import script
python import-mailbox-to-gmail.py --json key.json --dir $MBOX_DIR

# Deactivate service account
gcloud auth revoke
```

Make the scripts executable:

```bash
chmod +x import_mailboxes.sh
chmod +x import_mailboxes.fish
```

Run the scripts:

```bash
./import_mailboxes.sh
```

or

```fish
./import_mailboxes.fish
```

### 6. Troubleshooting

- If you encounter authentication errors, verify that domain-wide delegation is properly set up and the correct scopes are authorized.
- Check the log file (default: `import-mailbox-to-gmail-[PID].log`) for detailed error messages.
- Ensure the `key.json` file contains the correct `client_email`:

```bash
cat key.json | grep client_email
```

### 7. Clean Up

After a successful import, consider revoking the service account key:

```bash
gcloud iam service-accounts keys delete [KEY_ID] --iam-account=import-mailbox-sa@[PROJECT_ID].iam.gserviceaccount.com
```

Replace placeholders (e.g., `[PROJECT_ID]`, `[KEY_ID]`) with your actual values throughout this guide.

### Options and Notes

- Use the `-v` or `--verbose` flag for detailed logging.
- Use the `--from_message` parameter to start the upload from a particular message. This allows you to resume an upload if the process previously stopped.  
  e.g. `python import-mailbox-to-gmail.py --from_message 74336`
- If any of the folders have a ".mbox" extension, it will be dropped when creating the label for it in Gmail.
- To import mail from Apple Mail.app, make sure you export it first—raw Apple Mail files can't be imported. You can export a folder by right-clicking it in Apple Mail and choosing "Export Mailbox".
- This script can import nested folders. In order to do so, preserve the email folders' hierarchy when exporting them as mbox files. In Apple Mail.app, this can be done by expanding all subfolders, selecting both parent and subfolders at the same time, and exporting them by right-clicking the selection and choosing "Export Mailbox".
- If any of the folders have a ".mbox" extension and a file named "mbox" in them, the contents of the "mbox" file will be imported to the label named as the folder. This is how Apple Mail exports are structured.

### Best Practices

1. Always use a virtual environment to isolate the project dependencies from your system-wide Python installation.
2. Keep your `key.json` file secure and never share it or commit it to version control.
3. Regularly update the script and its dependencies to ensure you have the latest security patches and features.
4. Before running the script on production data, test it with a small subset of data on a test account.
5. Monitor the log files during the import process to catch and address any issues early.
6. Back up your mbox files before starting the import process.
7. If possible, run the script on a machine with a stable and fast internet connection to avoid interruptions during the import process.
8. Consider using a `requirements.txt` file to manage dependencies:

   ```bash
   pip freeze > requirements.txt
   ```

   Then, you can install dependencies in a new environment with:

   ```bash
   pip install -r requirements.txt
   ```

9. If you're using version control, consider adding a `.gitignore` file to exclude the virtual environment directory and sensitive files

 like `key.json`.

## License

Original work Copyright 2015 Google Inc.
Modified work Copyright 2023 Ryan Taylor

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   <http://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

This project includes software developed by Google Inc., modified by Ryan Taylor.
All modifications are released under the same Apache License 2.0.

## Acknowledgements

Special thanks to Liron Newman and Google Inc. for the original script, which has been invaluable in email migration processes.

### Contributing to the Modified Version

This project has been significantly modified by Ryan Taylor. If you wish to contribute to this modified version:

1. Please ensure your changes are compatible with the existing modifications.
2. Clearly document any new changes or features you add.
3. Follow the existing code style and documentation practices.
4. Submit a pull request with a clear description of your changes and their purpose.

Note that while this project is based on Google's original work, contributions to this modified version should be directed to the current maintainer, Ryan Taylor.
