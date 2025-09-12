# Client Tools Setup

- You repeat these steps anytime you open a new terminal.
- It is assumed that you have your HTTP MitM proxy tool (i.e. Burp Suite) already running.
- Be aware that if you are attempting to run Burp/clients outside of where the simualted servers are running or on a different VM then you may need to bind Burp to a different network interface
  - Example: Using WSL but running Burp outside of WSL (on the host) can result in network communication issues
  - It is recommended to run everything from inside your WSL or VM

## Environment Setup

```shell
export https_proxy=http://localhost:8080

gcloud config set auth/disable_ssl_validation  True
```

For an alternative gcloud CA trust store setup see https://github.com/rbeede/pen-testing-cloud-apis/blob/main/documentation/client_setup/Burp_linux.md

## Verify Clients Work

### OpenStack Swift

```shell
swift --insecure --auth=https://localhost:8888/auth/v1.0 -U system:root -K testpass --verbose stat
```

You should see two requests in your proxy.

### Google Cloud (CAZT)

```shell
gcloud cazt create \
    --api-endpoint-overrides=https://cazt.gcloud.localtest.me:8443/uat \
    --account=cazt_scen1_QA_specific@123456789012 \
    --format json \
    --name=MyMoggy \
    --activity-log-object-storage=moggylitterbox-123456789012

gcloud cazt list \
    --api-endpoint-overrides=https://cazt.gcloud.localtest.me:8443/uat \
    --account=cazt_scen1_QA_wildcard@123456789012 \
    --format json
```

### PaaS Cloud Goat (Salesforce)

Starting your web browser from the terminal it should pickup the https_proxy variable. If not, ensure that your browser is sending traffic to your proxy. Note that it is technically possible to solve the lab exercises with just the web browser developer tools, but most people prefer using the proxy tool.

---

### Next

[Sample Exercises](exercises.md)
