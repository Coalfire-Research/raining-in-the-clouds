# Software Configuration of Simulator Tools

> ℹ️ This documentation assumes that the necessary Git repositories of tools have already been cloned/unzipped into the current directory. In a few cases `sudo` privileges will be required, but most commands do not require it.

No Burp or HTTP MitM intercept should be running at this point.

---

### Baseline Prep

Tested on Ubuntu 24.04 (9/2025) for x86_64 and ARM.

```shell
cat /etc/os-release

echo $SHELL

pwd

# Triggers password prompt and typically caches
sudo -l
```

### OpenStack

```shell
pushd pen-testing-cloud-apis/
```

```shell
sudo apt --assume-yes install xfsprogs
sudo modprobe -v xfs

sudo apt --assume-yes install python3-swiftclient

cd documentation/server_setup/OpenStack/

sudo bash lab-server_openstack_install.bash

# Ignore the SSL errors or Unable to find XXX config section messages
```

```shell
# Check that base service is responding

curl --insecure --include https://localhost:8888/healthcheck

swift --insecure --auth=https://localhost:8888/auth/v1.0 -U system:root -K testpass --verbose stat

# If not, try manually restart the OpenStack Swift service with
# sudo swift-init all restart

```

##### Reset all existing data, if any

```shell
swift --insecure -A https://localhost:8888/auth/v1.0 -U system:root -K testpass delete fileuploads

swift --insecure -A https://localhost:8888/auth/v1.0 -U account1:normal -K expected delete deptdocs

swift --insecure -A https://localhost:8888/auth/v1.0 -U account2:somebody -K else delete research 

swift --insecure -A https://localhost:8888/auth/v1.0 -U codeerror:unexpecteduser -K shouldnothappen delete warez
```

##### Recreate test data

```shell
cd openstack-demo/demo-testdata/


echo $USER > sample_object.txt

# setup XSS example

swift --insecure -A https://localhost:8888/auth/v1.0 -U system:root -K testpass upload fileuploads sample_object.txt

swift --insecure -A https://localhost:8888/auth/v1.0 -U system:root -K testpass upload fileuploads sugarskull-2019_orig.png

# setup IAM examples

# Base check that accounts work

swift --insecure -A https://localhost:8888/auth/v1.0 -U account1:normal -K expected list
swift --insecure -A https://localhost:8888/auth/v1.0 -U account2:somebody -K else list
swift --insecure -A https://localhost:8888/auth/v1.0 -U codeerror:unexpecteduser -K shouldnothappen list

# Setup default uploads
swift --insecure -A https://localhost:8888/auth/v1.0 -U account1:normal -K expected upload deptdocs sugarskull-2019_orig.png
swift --insecure -A https://localhost:8888/auth/v1.0 -U account1:normal -K expected upload deptdocs sample_object.txt

swift --insecure -A https://localhost:8888/auth/v1.0 -U account2:somebody -K else upload research super-secret-doc-for-account2-only.txt
swift --insecure -A https://localhost:8888/auth/v1.0 -U account2:somebody -K else upload research "initials profile picture - small.png"
swift --insecure -A https://localhost:8888/auth/v1.0 -U account2:somebody -K else upload research "fyi emoji.png"

# Hacker account
swift --insecure -A https://localhost:8888/auth/v1.0 -U codeerror:unexpecteduser -K shouldnothappen upload warez pumpkin.JPG
```

```shell
# Return to base directory

popd
```

#### Web UI Simulator App

You will want to either use the `screen` command or open a new, dedicated terminal tab so this process continues to run where you can monitor it

```shell
cd pen-testing-cloud-apis/

cd documentation/server_setup/OpenStack/

python3 xss_python_swift_rest_api_server.py 9080

# Leave the terminal running
```

---

### Cloud AuthoriZation Trainer

You will want a dedicated terminal tab (or `screen`) to run and keep open the simulator server for CAZT.

```shell
pushd cazt/
```

```shell
python3 -m venv venv
source venv/bin/activate

pip3 install -r requirements.txt
# It is normal if no requirements are needed. Steps included for possible future additions.


cd simulator/

bash x509/generate-self-signed.bash

python3 main_http_endpoint_server.py
```

Keep this terminal tab open and go into another terminal tab (or `screen`). This assumes the gcloud CLI client was already installed.

```shell
pushd cazt/

cd trainee/cloud-clients/gcloud/

sudo python3 install-cazt-into-gcloud-cli.py

gcloud cazt --help
```

```shell
# Return to base directory

popd
```

##### Populate sample data
Verify the simulator is functional and populate with sample data:

```shell
# There is an optional way to trust the HTTP MitM (Burp) CA. Reference the cazt git repo for details.
gcloud config set auth/disable_ssl_validation  True


gcloud cazt create \
    --api-endpoint-overrides=https://cazt.us-texas-9.cloud.localtest.me:8443/uat \
    --account=cazt_scen0_Setup-Any@000000001111 \
    --format json \
    --name=MyMoggy \
    --activity-log-object-storage=moggylitterbox-000000001111

gcloud cazt create \
    --api-endpoint-overrides=https://cazt.us-texas-9.cloud.localtest.me:8443/uat \
    --account=cazt_scen0_Setup-Any@000000002222  \
    --format json \
    --name=NotMyMoggy \
    --activity-log-object-storage=moggylitterbox-000000002222

gcloud cazt run-activity \
    --api-endpoint-overrides=https://cazt.us-texas-9.cloud.localtest.me:8443/uat \
	--account=cazt_scen0_Setup-Any@000000001111 \
	--format json \
	--arn=arn:cloud:cazt:us-texas-9:000000001111:MyMoggy
```

---

### PaaS Cloud Goat

This requires that you already have a Salesforce Developer Edition account.

Following the instructions at https://github.com/rbeede/paas-cloud-goat/blob/main/Documentation/INSTALL.md

---

## Next

[Client Tools Setup](client_tool_setup.md)
