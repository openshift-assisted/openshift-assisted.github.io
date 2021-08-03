# Setting a password for the discovery ISO
The discovery ISO generated by the Assisted Installer comes with a `core` user that has its password login disabled (although SSH access using the provided public key is still supported).

However, it can sometimes be useful (e.g. when SSH isn't working), to login to the machine directly (with physical or BMC access), using a password. This can be done using the console login prompt that is shown after the ISO boots up.

To allow that, the user must have a password set. In order to set a password, it's possible to use the Assisted Installer API which provides support for editing the discovery ISO's ignition file with custom parameters.

The following script provides an example of how to use the API in order to modify the discovery ISO in a way that sets `core`'s password to any password of your choice.

For more information about the API and its various authentication methods, see [this document](cloud.md).

# Modify password script
```
#!/bin/bash

set -euo pipefail

if ! ocm token 2>/dev/null >/dev/null; then
    echo "Failed to run 'ocm token' command, please see the assisted_service/docs/cloud.md doc for authentication information"
    exit 1
fi

## User specific configuration <-----------
WANTED_PASSWORD=mypass
TOKEN=$(ocm token)
OCM_API_ENDPOINT="https://api.openshift.com/api/" 
if true; then
    echo "Don't forget to modify the script with your cluster ID, delete this warning after you did"
    exit 1
fi
CLUSTER_ID="243e09fb-d924-42a1-bad9-b78638b767d9" # Copy from Assisted Installer URL
DEST_ISO_FILE="changed_password.iso"
###############################

function log() {
    if [[ ! $? == 0 ]]; then
        echo "Script enountered an error"
        exit 1
    fi
}

trap log EXIT

echo "Password set log" > .set_iso_password_log

DISCOVERY_IGN_URL=$OCM_API_ENDPOINT/assisted-install/v1/clusters/$CLUSTER_ID/downloads/'files?file_name=discovery.ign'

if ! curl --fail -s ${DISCOVERY_IGN_URL} -H "Authorization: Bearer $TOKEN" >/dev/null; then
    echo "Can't seem to find a discovery.ign, please generate a discovery ISO file using the UI first"
    exit 1
fi

echo Downloading the original ignition file into the ORIGINAL_IGNITION variable
ORIGINAL_IGNITION=$(curl --fail -s ${DISCOVERY_IGN_URL} -H "Authorization: Bearer $TOKEN")

echo === Original ignition === >> .set_iso_password_log
echo "$ORIGINAL_IGNITION" >> .set_iso_password_log
echo ========================= >> .set_iso_password_log
echo >> .set_iso_password_log

echo Generating a salted hash from WANTED_PASSWORD
PASS_HASH=$(mkpasswd --method=SHA-512 $WANTED_PASSWORD | tr -d '\n')

echo === Password Hash=== >> .set_iso_password_log
echo "$PASS_HASH" >> .set_iso_password_log
echo ==================== >> .set_iso_password_log
echo >> .set_iso_password_log

echo Modifying ORIGINAL_IGNITION to contain the new password hash rather than the previous one
NEW_IGNITION=$(<<< "$ORIGINAL_IGNITION" jq --arg passhash $PASS_HASH '.passwd.users[0].passwordHash = $passhash')

echo === New patched ignition === >> .set_iso_password_log
echo "$NEW_IGNITION" >> .set_iso_password_log
echo ============================ >> .set_iso_password_log
echo >> .set_iso_password_log

echo Telling service to use our patched ignition file
curl --fail -s $OCM_API_ENDPOINT/assisted-install/v1/clusters/$CLUSTER_ID/discovery-ignition -H "Authorization: Bearer $TOKEN" --request PATCH --header "Content-Type: application/json" --data @<(echo '{"config": "replaceme"}' | jq --rawfile ignition <(echo $NEW_IGNITION) '.config = $ignition')

echo "Done, please re-generate the ISO using the UI and download the newly generated ISO"
```