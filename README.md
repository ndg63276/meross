MEROSS LOGIN
============

import requests
import time
import hashlib
import json
import base64
import uuid

_SECRET = '23x17ahWarFH6w29'

timestamp_millis = int(round(time.time() * 1000))
nonce = ''

parameters = {'email': email, 'password': password}
jsonstring = json.dumps(parameters)
login_params = str(base64.b64encode(jsonstring.encode('utf8')), 'utf8')
datatosign = '%s%s%s%s' % (_SECRET, timestamp_millis, nonce, login_params)

m = hashlib.md5()
m.update(datatosign.encode('utf8'))
md5hash = m.hexdigest()

payload = {
	'params': login_params,
	'sign': md5hash,
	'timestamp': timestamp_millis,
	'nonce': nonce
}


url = 'https://iot.meross.com/v1/Auth/Login'
r = requests.post(url, data=payload)

token = r.json()['data']['token']
userid = r.json()['data']['userid']
key = r.json()['data']['key']

headers = {"Authorization": "Basic "+token}
url2 = 'https://iot.meross.com/v1/Device/devList'
r2 = requests.post(url2, headers=headers, data=payload)
device_uuid = r2.json()['data'][1]['uuid']  # device 1

import paho.mqtt.client as mqtt

m2 = hashlib.md5()
m2.update(('API'+str(uuid.uuid4())).encode('utf8'))
appId = m2.hexdigest()

user_topic = f"/app/{userid}/subscribe"
client_response_topic = f"/app/{userid}-{appId}/subscribe"
device_request_topic = f"/appliance/{device_uuid}/subscribe"

def on_connect(client, userdata, flags, rc):
    print("Connected with result code "+str(rc))
    client.subscribe([(user_topic, 0), (client_response_topic, 0)], qos=1)

# The callback for when a PUBLISH message is received from the server.
def on_message(client, userdata, msg):
    print(msg.topic+" "+str(msg.payload))

m3 = hashlib.md5()
m3.update((userid+key).encode('utf8'))
md5hash3 = m3.hexdigest()

clientId = 'app:' + appId
client = mqtt.Client(clientId)
client.on_connect = on_connect
client.on_message = on_message
client.tls_set()
client.username_pw_set(userid, md5hash3)
domain = r2.json()['data'][1]['domain']
client.connect(domain, 2001, 30)
client.loop_start()   # loop_forever()


def build_message(method, namespace, payload):
	randomstring = '7NO3ENYZRO39LZXS'
	m4 = hashlib.md5()
	m4.update(randomstring.encode('utf8'))
	messageId = m4.hexdigest().lower()
	timestamp = int(round(time.time()))
	m5 = hashlib.md5()
	strtohash = "%s%s%s" % (messageId, key, timestamp)
	m5.update(strtohash.encode("utf8"))
	signature = m5.hexdigest().lower()
	data = {
		"header":
			{
				"from": client_response_topic,
				"messageId": messageId,  # Example: "122e3e47835fefcd8aaf22d13ce21859"
				"method": method,  # Example: "GET",
				"namespace": namespace,  # Example: "Appliance.Control.ToggleX",
				"payloadVersion": 1,
				"sign": signature,  # Example: "b4236ac6fb399e70c3d61e98fcb68b74",
				"timestamp": timestamp,
				'triggerSrc': 'Android'
			},
		"payload": payload
	}
	strdata = json.dumps(data)
	return strdata.encode("utf-8"), messageId


method = "SET"  # or "GET"
namespace = "Appliance.Control.ToggleX"
channel = 0
payload = {"togglex": {"onoff": 0, "channel": channel}}
message, messageId = build_message(method, namespace, payload)
client.publish(topic=device_request_topic, payload=message, qos=1)


client.loop_stop()

=============
JS Equivalent
=============
_SECRET = '23x17ahWarFH6w29'
timestamp_millis = new Date().getTime()
nonce = ''
parameters = {'email': email, 'password': password}
jsonstring = JSON.stringify(parameters)
login_params = btoa(jsonstring)
datatosign = _SECRET + timestamp_millis + nonce + login_params
md5hash = CryptoJS.MD5(datatosign).toString()

...

var device = user_info.devices[1]
var domain = device.domain
var appId = CryptoJS.MD5('API'+uuidv4()).toString()

var clientId = 'app:' + appId
var userid = user_info["meross_userid"]
var key = user_info["meross_key"]
var password = CryptoJS.MD5(userid+key).toString()

user_topic = "/app/"+userid+"/subscribe"
client_response_topic = "/app/"+userid+"-"+appId+"/subscribe"


function onConnect() {
  console.log("onConnect");
  client.subscribe(user_topic);
  client.subscribe(client_response_topic);
}

function onConnectionLost(responseObject) {
  if (responseObject.errorCode !== 0) {
    console.log("onConnectionLost:"+responseObject.errorMessage);
  }
}

function onMessageArrived(message) {
  console.log("onMessageArrived:"+message.payloadString);
}

client = new Paho.MQTT.Client(domain, 2001, clientId);
client.onConnectionLost = onConnectionLost;
client.onMessageArrived = onMessageArrived;
var options = {
	onSuccess: onConnect,
	useSSL: true,
	userName: userid,
	password: password,
}
client.connect(options);


//  message = new Paho.MQTT.Message("Hello");
//  message.destinationName = "World";
//  client.send(message);

##############
# Fails as meross doesn't seem to allow mqtt over websocket
