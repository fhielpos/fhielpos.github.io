---
layout: post
title: "Setting up a Wazuh lab - Part 3 [A bit of customization and Telegram bot]"
---

Hello there, in this section of the journey to create my exposed Wazuh lab, I will be making this lab a bit more **"secure"**.

<img src="assets/img/telegram.png" alt="Telegram" width=500px height=auto>

I have already configured custom `users` and `passwords` on {% post_url 2021-11-21-wazuh-lab-part1 %}. However, there are a couple of things I want to make sure first:

* The Wazuh API is not accessible from outside the VM.
* The Elasticsearch API is not accessible from outside the VM.
* I can identify the logins to the machine.


Let's get started:

## Checking Elasticsearch API exposure

The `network.host` setting from the `/etc/elasticsearch/elasticsearch.yml` file allows us to set at which IP the **Elasticsearch API** should listen. In this case, I will set it to `localhost` or `127.0.0.1` instead of `0.0.0.0` to avoid connections outside of the VM:

```yaml
network.host: 127.0.0.1
```

And you need to restart to apply the changes:

```bash
systemctl restart elasticsearch
```

Additionally, I could set a **Firewall rule** to add an extra layer of security, but in this case, I still want this machine to be insecure "by default".


## Checking Wazuh API exposure

Similar to the `network.host` setting on Elasticsearch, Wazuh has it's own `host` setting on `/var/ossec/api/configuration/api.yaml`. In this case, we need to uncomment the `host: 0.0.0.0.0` and replace it with `localhost` as we did before:

```yaml
host: 127.0.0.1
```

After that, we can test the configuration with:

```bash
/var/ossec/bin/wazuh-apid -t
```

In case there are any issues, you will see an error like the following:
```log
Configuration not valid: 2000 - Some parameters are not expected in the configuration file (WAZUH_PATH/api/configuration/api.yaml): Additional properties are not allowed ('hostt' was unexpected).
```

If no output is shown, the config is correct, and you can restart Wazuh to apply the changes.

```bash
systemctl restart wazuh-manager
```

## Alerting on SSH logins

When logging in to the VM I get an `Authentication success` alert with ID `5715` from Wazuh, so I decided to create a **Telegram bot** to send me a message with the details from the alert.

Creating a Telegram bot is not much complicated. Basically, you need to talk with [BotFather](https://telegram.me/BotFather) and request a TOKEN. Once you have the Bot created you need to start a conversation with it and retrieve the `chatId` for the Python integration. I based my script on [the one on this post](https://medium.com/@jesusjimsa_12801/integrating-telegram-with-wazuh-4d8db91025f) I just made some modifications to add extra data.


### The Python script

Wazuh integrations should be located on `/var/ossec/integrations` and follow the `custom-NAME` name pattern, with no extension. [Here, you can read more about custom integrations on Wazuh](https://wazuh.com/blog/how-to-integrate-external-software-using-integrator/).


Here is the script that will be located in `/var/ossec/integrations/custom-telegram`:

```python
#!/usr/bin/env python3
import sys
import json
import logging

try:
    import requests
    import ipinfo
except Exception:
    print("Module requests or ipinfo missing. Make sure to install them with: pip3 install requests && pip3 install ipinfo")
    sys.exit(1)

# Define logging
logging.basicConfig(level="INFO", filename="/var/ossec/logs/integrations.log")
logger = logging.getLogger(__name__)

# Important stuff
CHAT_ID = "CHANGE THIS"
IPINFO_TOKEN = "CHANGE THIS"

def create_message(alert_json):
    # Get alert information
    title = alert_json['rule']['description'] if 'description' in alert_json['rule'] else ''
    description = alert_json['full_log'] if 'full_log' in alert_json else ''
    description.replace("\\n", "\n")
    alert_level = alert_json['rule']['level'] if 'level' in alert_json['rule'] else ''
    rule_id = alert_json['rule']['id'] if 'rule' in alert_json else ''
    agent_name = alert_json['agent']['name'] if 'name' in alert_json['agent'] else ''
    agent_id = alert_json['agent']['id'] if 'id' in alert_json['agent'] else ''
    user = alert_json['data']['dstuser'] if 'dstuser' in alert_json['data'] else ''
    ip = alert_json['data']['srcip'] if 'srcip' in alert_json['data'] else ''

    if 'srcip' in alert_json['data']:
        ipinfo_handler = ipinfo.getHandler(IPINFO_TOKEN)
        ipinfo_data = ipinfo_handler.getDetails(ip)

    msg_text = f"""
*{title}*
_{description}_
*User:* {user}
*IP:* {ip}
*Location:* {ipinfo_data.city}, {ipinfo_data.country}
*Rule:* {rule_id} (Level {alert_level})
*Agent:* {agent_name} ({agent_id})
"""


    msg_data = {}
    msg_data['chat_id'] = CHAT_ID
    msg_data['text'] = msg_text
    msg_data['parse_mode'] = 'markdown'

    # Debug information
    logger.debug(f"Message sent: {msg_data}")

    return json.dumps(msg_data)



# Get hook from config
hook_url = sys.argv[3]

# Open alert json
with open(sys.argv[1], 'r') as alert_file:
    alert_json = json.loads(alert_file.read())

# Send the request
msg_data = create_message(alert_json)
headers = {'Content-Type': 'Application/json', 'Accept-Charset': 'UTF-8'}
response = requests.post(hook_url, headers=headers, data=msg_data)

# Debug information
if response.status_code != 200:
    logger.error(f"Failed to send message. Response code: {response.status_code} .")
    logger.debug(f"{response.content}")
else:
    logger.info("Message sent")

sys.exit(0)
```

As you can see, I am using `logging` module to write into `/var/ossec/logs/integrations.log` and the `ipinfo` module to obtain `GeoLocation` data of the IP accessing to the machine.

### Configuring the integration

To configure our custom integration, we need to set the following in `/var/ossec/etc/ossec.conf`:

```xml
<integration>
    <name>custom-telegram</name>
    <rule_id>5715</rule_id>
    <hook_url>https://api.telegram.org/botYOUR-TOKEN/sendMessage</hook_url>
    <alert_format>json</alert_format>
</integration>
```

Here we are:
* Calling the `custom-telegram` script.
* Telling Wazuh to forward `5715` alerts to the script.
* Passing the Telegram `hook_url` to the script.
* Using the `json` format for the alert received by the alert.


After a `Wazuh` restart, we can log into the VM and see if we receive an alert:

<img src="assets/img/telegram_notification.png" alt="Telegram notification" width=500px height=auto>