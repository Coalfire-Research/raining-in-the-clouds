# Exercises

A walkthrough of some sample exercises is provided next. Each respective cloud API simulator also contains additional exercises for you to explore outside of this content.

Each exercise assumes that you have already setup the simulator servers and the respective client tools.

---

<details>
<summary>Cross-site Scripting with REST APIs</summary>

![image](xss-exercise-sample-screenshot.png)

From a web browser (connect to your HTTP MitM Proxy, Burp) navigate to:
> `http://localhost:9080/REST/API/endpoint.cgi`

Observe that the page renders a listing of files store in a backend OpenStack Swift (Object Storage). In the first release for customers they could use the upload functionality of this page to upload files to the server for storage.

Observe that if you attempt to upload a file with a filename that does not comply to the restrictions that you are blocked. Feel free to try repeating a few malicious payloads via Burp in an attempt to inject XSS on the list page. During the first pen test the service was not found vulnerable to XSS.

##### Customer Version 2
Months later customers demanded a way to more easily bulk upload millions of their data files. The application was updated to allow customers to use the OpenStack cloud REST APIs to perform bulk uploads via the Swift service.

Our subsequent pen test will uncover that an attacker can bypass the filename character input validation and cause XSS to be exploited on the application.

### Useful concepts

1. The Swift CLI for the OpenStack SDK and REST API
   - https://docs.openstack.org/ocata/cli-reference/swift.html
1. Observe: Object Storage != Filesystem Storage nor the same limitations
   - https://docs.openstack.org/api-ref/object-store/index.html#objects

### Baseline Start (QA)

We start by only performing legitimate, expected behaviors (i.e. inputs) of the functionality. This tells us if the service we are testing actually works as well as what a valid request looks like.

```shell
echo $USER > sample_object.txt

swift --insecure -A https://localhost:8080/auth/v1.0 -U system:root -K testpass upload fileuploads ./sample_object.txt

swift --insecure -A https://localhost:8080/auth/v1.0 -U system:root -K testpass upload fileuploads /etc/os-release
```

Observe on the application website (GUI) that the file uploads now appear.

You can also observe the traffic in your HTTP MitM proxy (Burp). You will noticed that there are two calls. One to authenticate with the client credentials and obtain a token. Another to perform the actual upload. In this case we are not attacking the authentication or authorization but instead already have credentials.

### Malicious Input Injection

We observed previously that uploading files via the web browser (HTML/HTTP form upload) resulted in strict validation of the permitted filenames. Is that the case with the standard SDK of the OpenStack REST API?

```shell
swift --insecure -A https://${LAB_OPENSTACK_IP}:8080/auth/v1.0 -U system:root -K testpass copy fileuploads sample_object.txt -d '/fileuploads/easytest<script>alert("you been pwned")</script>forme'
```

Now observe the web UI result. You should see a simple (persistent) XSS payload execute.

Object Storage in the cloud is not strictly file storage. The key name of the object can be a wide range of byte sequences and not strictly characters.

ℹ️ In this example we used an API to copy an item already in the object storage but with a new key name. You could have also uploaded another file with a malicious name. The lesson is that you want to look for multiple possible API calls that were not considered by the application developers to cause unexpected behavior.
</details>

---

<details>
<summary>Authorization Bypass - Privilege Escalation</summary>

### Useful concepts

- Exercise scenario details
  - https://github.com/Coalfire-Research/cazt/blob/main/documentation/lab_manual/scenarios/07-impersonation.md

Identity and access management (IAM) controls include policies that define permissions for the caller of the expected actions (APIs) and resources (IDs). These permissions can allow things or deny things based on a variety of conditional states.

A configuration mistake in a policy by the client/customer/tenant can result in unauthorized access to data or the ability to use the service in unwanted ways. This typically falls on the customer-shared responsiblity.

However, the cloud (or service) provider must also ensure that their systems correctly interept both the policy documents and how it is applied to the user inputs. Failure to do so could result in unauthorized access to the tenant's data or privilege escalation.

### Baseline Start (QA)

From a terminal (connected to your proxy) run the following baseline (QA) command to verify that the service's API is working as expected:

```shell
gcloud cazt pet-sitter \
    --api-endpoint-overrides=https://cazt.gcloud.localtest.me:8443/uat \
    --account=cazt_scen7_impersonation@000000001111 \
    --format json \
    --arn=arn:cloud:iam:us-texas-9:000000001111:CareForPets
```

You'll observe a success response from the service. This is because the psuedo-policy allows access to CareForPets. Reference https://github.com/Coalfire-Research/cazt/blob/main/trainee/iam_policies/cazt_scen7_impersonation.json

### Malicious Input Injection

The goal is to bypass the IAM policy through a flaw in the cloud vendor's (responsibility) system to achieve FullAdmin access.

Attempt the following API REST call:

```shell
gcloud cazt pet-sitter \
    --api-endpoint-overrides=https://cazt.gcloud.localtest.me:8443/uat \
    --account=cazt_scen7_impersonation@00000000111 \
    --format json \
    --arn=arn:cloud:iam:us-texas-9:00000000111:FullAdmin
```

Note:
1. You must not change the `--account` value
   1. The attack is against authori**z**ation, not the authentication
   1. The attacker has only their own credentials, not anothers
   1. Do not change the HTTP authentication header either
1. Verify that the request is visible in your HTTP MitM proxy (i.e. Burp)
1. The response should indicate that your request to the API was denied

##### Goal

The goal is to get a success response like the following:
```json
{
  "Message": "00000000111 using impersonation arn:cloud:iam:us-texas-9:00000000111:FullAdmin"
}
```

##### Attack Methodologies

1. Identify the input to attack
   - In this case "arn"
1. Attempt fuzzing of the input value
   - Encode some characters with URL character encoding escapes
   - Try adding extra spaces at the beginning or end
   - Duplicate the key+value in the JSON
   - Change the value from a single string to an array of values
     - One with the legit value CareForPets, the other FullAdmin
     - Which value is used for authorization versus the business logic?
   - Are the key names or values case sensitive?

##### Solution

```http
TODO add example
```

```http
TODO add example
```

In this case the software bug in the API was that the policy authorization rules were applied with case sensitivity in parts that were case insensitive.

</details>

---
