# MQTT API - Airdeep.UploadTelemetryLogData

## **목적**
  - 디바이스는 서버로부터 수신된 'remoteLogLevel' 를 기준으로 Telemetry Log Data 를 업로드 한다.

## **선행 조건**
  - 장치 등록 
  - 디바이스 등록 과정에서 수신받은 MQTT 호스트 정보
  - Access Token (deviceCredentialsId)

</br>

## **Protocol**
- MQTT

</br>

## **Host Name**
- [디바이스 등록 과정](./HTTP-API-DeviceRegister.md)에서 수신받은 MQTT 호스트 정보

</br>

## **MQTT TOPIC**
NO | TOPIC    | Publish | Subscribe | Description
:------: | :------ | :---------: | :----------: | :----------
1  | v1/devices/me/telemetry  | O | X | **Publish** </br> 서버에 telemetry Log data 를 전달한다.
2 | v1/devices/me/attributes/request/1 | O | X | **Publish**</br> 서버에 저장된 Attribute(Client, Shard) 값을 요청한다. 
3  | v1/devices/me/attributes/response/1  | X | X | 2번 요청의 응답 값</br> on_message 콜백으로 응답 값이 수신된다. 

</br>

## **Payload**
NO | TOPIC    | Send(Publish) | Recv(on_message)
:------: | :------ | :--------- | :----------
1  | v1/devices/me/telemetry  | O [(Log Telemetry)](#Log-Telemetry) | X
2  | v1/devices/me/attributes/response/1  | X | { "firmwareVersion": "5.0", "lastBootingTime": timestamp(sec), timeZone: "Asia/Seoul" },"shared":{"uploadFrequency": 1, "remoteLogLevel": 1} }
</br>

## Log Telemetry
```json
{ 
  "remoteLogMessage" : {
    "ts" : timestamp,
    "logLevel" : 1,
    "logString": "xxxx"
  }
}
```

## **Payload 설명**
Key        |  Value | Description 
:----------|:-----------------|:------------------
ts| Unix timestamp(Sec) | json 을 구성하여 데이터를 전달한 시간 
logLevel| Number | 로그 레벨
logString | String | 로그 스트링


</br>

## **시퀀스 다이어그램**

1. Telemetry Log Data 업로드
    - 디바이스 업데이트 후 펌웨어 값 전달
   ```plantuml
   @startuml
     participant "AirDeep[AQS]" order 1
     participant "AirDeep[MQTT Server]" order 2
     hide footbox

     group 1. MQTT - Connection
      "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : MQTT Connection(MQTT_HOST, MQTT_PORT, ACCESS_TOKEN(deviceCredentialsId))
      "AirDeep[AQS]" <- "AirDeep[MQTT Server]"  : Ack(Connected)
     end

     == MQTT Session Established ==
     group 2. MQTT - Shared Attribute 값 수신 후 적용
     "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 1.1) Publish - ATTRIBUTES_REQUEST_TOPIC(서버에 저장된 Attribute(Client, Shard) 값 요청)
     "AirDeep[AQS]" <- "AirDeep[MQTT Server]" : 1.2) ATTRIBUTES_RESPONSE_TOPIC( Receive Attribute (Client(firmwareVersion, lastBootingTime, timeZone), Shard(uploadFrequency, remoteLogLevel)) )
     "AirDeep[AQS]" <- "AirDeep[AQS]" : 1.3 on_message
     note left: remoteLogLevel 설정
     end

     group 3. MQTT - Upload Telemetry Log Data
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 2.1) Publish Log Data - (TELEMETRY_TOPIC) 
       note left: 설정된 RemoteLogLevel에 따라 Log 전달
     end
     
     == MQTT Session Established ==
   @enduml
   ```
</br></br>


## **Python Example**
```python
import requests
import json
from datetime import datetime

uploadFrequency = 1

ATTRIBUTES_TOPIC = "v1/devices/me/attributes"
ATTRIBUTES_REQUEST_TOPIC = "v1/devices/me/attributes/request/1"
ATTRIBUTES_RESPONSE_TOPIC = "v1/devices/me/attributes/response/1"

# When device is connected to AirDeep server
def on_connect(client, userdata, rc, *extra_params):

    # 1.1) Attribute(Client, Shared) 값 요청한다.
    client.publish(ATTRIBUTES_REQUEST_TOPIC , json.dumps({"clientKeys": "firmwareVersion,lastBootingTime","sharedKeys": "uploadFrequency"}))

    # 2.1) 서버에서 Device의 Shared Attribute 변경 시, 변경된 값을 받기 위하여 Subscribe 요청.
    client.subscribe(ATTRIBUTES_TOPIC)


def on_message(client, userdata, msg):
    global remoteLogLevel

    payload = json.loads(msg.payload)

    # 1.2) 서버로 부터 요청한 Attribute(Client, Shard) 값을 수신한다.
    if msg.topic == ATTRIBUTES_RESPONSE_TOPIC:
        if payload["shared"]["remoteLogLevel"]  > 0:
            remoteLogLevel = payload["shared"]["remoteLogLevel"]
        return

    # 2.2) 서버에서 Shard Attribute 변경 시 수신된다.
    if msg.topic == ATTRIBUTES_TOPIC:
        if payload["remoteLogLevel"] > 0:
            remoteLogLevel = payload["remoteLogLevel"]
        return
    
# log data
log_data = {
    'ts' : 0,
    'logLevel' : 1,
    'logString': 'xxxx'
}

try:
    timestamp_info = provisioning.getUTCTimestamp(AIRDEEP_HTTP_HOST, random.randint(0, 100), "UTC")
    ts = timestamp_info.get("msg").get("timestamp")
    while True:

        log_data['ts'] = ts
        log_data['logLevel'] = 1
        log_data['logString'] = "Hello Log World"
        
        client.publish(TELEMETRY_TOPIC, json.dumps({"remoteLogMessage": log_data}))

  ```


