# Exercises

A walkthrough of some sample exercises is provided next. Each respective cloud API simulator also contains additional exercises for you to explore outside of this content.

Each exercise assumes that you have already setup the simulator servers and the respective client tools.

---

## Cross-site Scripting with REST APIs

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

---

## 
