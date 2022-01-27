---
layout: post
title: "Integrating Wazuh and kube-bench"
---

I am close to take my CKS exam, and decided to have some fun with `kube-bench` and `Wazuh`.

<img src="https://github.com/aquasecurity/kube-bench/raw/main/docs/images/kube-bench.png" alt="kube-bench" width=500px height=auto>

*Disclaimer: This blogpost only walks through how I built the image and integration. This was a side project that I made for pure fun.*


Some time ago I wanted to play with `Wazuh` on a Kubernetes deployment, but we don't have a Kubernetes variant available yet (besides the Wazuh Manager), so I decided to create a barebone Docker image that you can find here:
* [Wazuh Docker Agent on Github](https://github.com/fhielpos/wazuh-docker-agent)
* [Wazuh Docker Agent on Dockerhub](https://hub.docker.com/r/fhielpos/wazuh-agent)


This image doesn't do much as you don't have much to monitor inside the agent itself, so the best would be to follow the [sidecar pattern](https://medium.com/bb-tutorials-and-thoughts/kubernetes-learn-sidecar-container-pattern-6d8c21f873d). This would be to use the Wazuh agent as a sidecar container to ship logs to a Wazuh manager.


With this in mind, I decided to give it a try with [kube-bench, a tool from Aquasecurity](https://github.com/aquasecurity/kube-bench) to run `CIS Benchmarks` on Kubernetes nodes. They have a [job definition](https://github.com/aquasecurity/kube-bench/blob/main/job.yaml) that runs `kube-bench` and then you have to run `kubectl logs kube-bench-pod-name` to retrieve the logs, but how could we send this to Wazuh?


First things first, I needed to have the logs on `JSON` format, so I went to the `help` page to look at the possible arguments, and there they were my lifesavers:

```
Flags:
      --json                             Prints the results as JSON
      --outputfile string                Writes the JSON results to output file
```

But there was a small problem... For some reason, when running the command from a container `--json` did work correctly, but `--outputfile` did not, so I was struggling to send the output to a file. Long story short, I ended up modifying the job `command` and `args` to redirect the output manually:

```yaml
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:0.6.3
          command: ["/bin/sh"]
          args: ["-c","kube-bench --json > /var/log/kube-bench/kube-bench.json"]
```

Defnitely not ideal, but gets the job done for this little PoC.

Now, to share the file across containers I decided to use an `emptyDir`, this volumes only live as long as the pod, so they provide a good ephemeral shared storage. With this I decided to start modifying the `job` by also adding the `Wazuh` container:


```yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    metadata:
      labels:
        app: kube-bench
    spec:
      hostPID: true
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:0.6.3
          command: ["/bin/sh"]
          args: ["-c","kube-bench --json > /var/log/kube-bench/kube-bench.json"]
          volumeMounts:
            # KUBE-BENCH VOLUMES IGNORED FOR NOW
            - name: kube-bench-output
              mountPath: /var/log/kube-bench
        - name: wazuh-agent
          image: fhielpos/wazuh-agent:latesT
          volumeMounts:
            - name: kube-bench-output
              mountPath: /var/log/kube-bench
          env:
            - name: WAZUH_MANAGER_IP
              # CHANGE FOR THE HOSTNAME OR IP OF YOUR WAZUH-MANAGER
              value: wazuh
      restartPolicy: Never
      volumes:
            # KUBE-BENCH VOLUMES IGNORED FOR NOW
        - name: kube-bench-output
          emptyDir: {}
```

So far so good, right? But now, how will Wazuh read that file that we are writing? We have two options:

* Using a localfile
* Using directly the Wazuh socket

Both options were completely complex to implement, one from the management perspective (localfile), and the other one from every perspective (socket). However, after analyzing the output of `kube-bench`, I decided that I would need to process the log before sending it to Wazuh, so I would need to write some Python:


After some hours and a lot of debugging, here is what I made:

```python
#!/usr/bin/env python

import json
import time
from socket import socket, AF_UNIX, SOCK_DGRAM

socketAddr = '/var/ossec/queue/sockets/queue'

def send_event(msg):
    try:
        #print('Sending {} to {} socket.'.format(msg, socketAddr))
        string = '1:kube-bench:{}'.format(msg)
        sock = socket(AF_UNIX, SOCK_DGRAM)
        sock.sendto(string.encode(), socketAddr)
    except:
        print("Error sending message to Wazuh socket.")

finished = False
retries = 0

while not finished and retries != 5:
    try:
        with open('/var/log/kube-bench/kube-bench.json', 'r') as result:
            json_output = json.loads(result.read())
            for scan in json_output['Controls']:
                for test in scan['tests']:
                    for result in test['results']:
                        result['node_type'] = scan['node_type']
                        result['policy'] = scan['text']
                        result['section_description'] = test['desc']
                        msg = {}
                        msg['integration'] = 'kube-bench'
                        msg['kube_bench'] = result
                        send_event(json.dumps(msg))
        finished = True
        
        # Give time to Wazuh to send the messages before killing the container
        time.sleep(60)
    except:
        retries += 1
        if retries != 5:
            print("kube-bench output file not found. Sleeping 30 seconds.")
            time.sleep(30)
        else:    
            print("kube-bench output not found. Max attempts reached. Exiting")
```

It's not that complex, it basically decomposes the massive `kube-bench` JSON output into multiple small logs and send them to the `Wazuh` socket for analyzis.


Now, the trickiest part: running the integration, and only the integration, from the Wazuh container. This was a complete nightmware that made me just re-work the complete Docker image.


I decided to move away from `S6` as this container needs to be killed once the integration has finished, and also because for some reason the `SIGTERM` command sent by `S6` to the container was also affecting other containers. I deleted the `S6` directories and created a simpler `entrypoint` script with the possibility to pass arguments to it so I can run the script.


With all these already done, I built the new image and finished the job definition:
* [Wazuh Docker Agent on Github](https://github.com/fhielpos/wazuh-docker-agent)
* [Wazuh Docker Agent on Dockerhub](https://hub.docker.com/r/fhielpos/wazuh-agent)
* [Wazuh and kube-bench integration job](https://github.com/fhielpos/wazuh-docker-agent/blob/latest/pocs/kube-bench.yaml)

I only built the image with the integration for `Wazuh 4.2.4` and under the `latest` tag on Dockerhub. As I did modify the whole image structure, I prefer to leave the older versions as they are for now. You can build your own image if you need to.


And last, but not least, the Wazuh rule to make this work:

I am only interested on the checks with `FAIL` status. So this is my rule:

```xml
  <rule id="100002" level="5">
    <decoded_as>json</decoded_as>
    <field name="integration">kube-bench</field>
    <field name="kube_bench.status">FAIL</field>
    <description>kube-bench failed check: $(kube_bench.test_desc)</description>
    <options>no_full_log</options>
  </rule>
```

If you would like to catch `ALL` the events, you can use a rule like this:
```xml
  <rule id="100002" level="3">
    <decoded_as>json</decoded_as>
    <field name="integration">kube-bench</field>
    <description>kube-bench check: [$(kube_bench.status)] - $(kube_bench.test_desc)</description>
    <options>no_full_log</options>
  </rule>
```

Before running the job, you should modify the `WAZUH_MANAGER_IP` variable from the `Wazuh` container and replace it with your manager IP, if you already run `Wazuh` on Kubernetes, probably your endpoint already responds to just `wazuh`.

```shell
curl -so kube-bench.yaml https://raw.githubusercontent.com/fhielpos/wazuh-docker-agent/latest/pocs/kube-bench.yaml
```

```
          env:
            - name: WAZUH_MANAGER_IP
            # Your manager IP here:
              value: wazuh
```

Then you can finally apply it:

```shell
kubectl apply -f kube-bench.yaml
```

If everything is correctly set (Wazuh variables and the rules), you should be receiving events on your manager. There are some caviats though:

* These `Wazuh` containers are completely ephemeral, meaning that they will be registered and only live a couple of minutes top, this will consume agent IDs that may or may not be a problem in the future.
* The script is a bit raw still, I would like to have it on Python3 with better exception handling, but that also means updating the image which is another work in progress.
* The Docker image weights ~300MB.
* This integration was tested in a single node Kubernetes local environment. I did not test it on cloud providers nor multiple node clusters.
* The `CIS Benchmarks` are not the same across Kubernetes versions, you should read `kube-bench` documentation and adapt the job accordingly.
