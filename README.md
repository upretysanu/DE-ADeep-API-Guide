# AirDeep System Integration Document
##  작성 이력
| 날짜 | 버전 | 내용 | 작성자 |
|:--------|:---------|:---------|:-------|
|2020-12-30| 0.1.0 | - 최초 작성     | 유수던</br>이창주 |
|2021-01-14| 0.2.0 | - 다이어그램 수정</br> - Device 등록 Wrapper API 추가</br> - Server URL 변경</br>- Time API 추가   | 유수던</br>이창주 |
|2021-01-28| 0.3.0 | - HTTP API, MQTT API 가이드 개선  | 유수던</br>이창주 |
|2021-01-29| 0.3.1 | - 목차 이름 개선  | 이창주 |
|2021-02-04| 0.3.2 | - Client Attribute 변경 사항</br>　1. lastFirmwareVersion -> firmwareVersion </br> 　2. firmwareVersion 의 값 타입 변경 Number -> String </br>　3. timeZone 추가 | 이창주 |
|2021-02-05| 0.3.3 | - RPC 테스트 웹 페이지 추가 <br> 　MQTT-API-2.RpcCommand(서버사이드 RPC 테스트) | 이창주 |
|2021-02-08| 0.3.4 | - RPC 테스트 웹 페이지  브라우저 설정 가이드 추가 <br> 　MQTT-API-2.RpcCommand(서버사이드 RPC 테스트) | 이창주 |
|2021-02-09| 0.3.5 | - 응답 TOPIC 통일화 TELEMETRY_TOPIC </br> - 응답 Json Key 변경 <br> - Shard Attribute - remoteLogLevel 추가</br> - UploadTelemetryLogData.md 추가 | 유수던</br>이창주 |

</br>

## 1. 소개
### 1.1 목적
    본 문서의 내용은 AQS 서비스 연동 엔터페이스를 제공하기 위한 규격이다. 

### 1.2 적용 범위 
     본 문서의 내용은 AQS 서비스 제공 계획의 변경 및 관련 표준 규격의 갱신등에 다라 추후 보완 및 변경 될 수 있다.
     본 문서에서 기술되지 않은 부분에 대한 기능 구현 시 반드시 협의하여 구현하여야 한다.

### 1.3 프로토콜
      - HTTP/1.1
      - TLS 1.2
      - MQTT

### 1.4 용어 정의 및 약어
      - API : Application Program Interface
      - AQS : Air Quality Services
      - REST : Representational State Transfer

****
</br>

## 2. 기본 정보
### 2.1 서버 정보
  | 서버 | URL |   설명 |
  |:-----|:----|:-------|
  |테스트 서버|https://dev-api-airdeep.rtdata.co.kr| 개발기 |
  |상용 서버| https://api-airdeep.rtdata.co.kr | 상용기 | 
</br>


****
</br>

## 3.  서비스 연동
### 3.1 서비스 연동 콜 플로우
* 시퀀스 다이어그램

  ```plantuml
  @startuml
    participant "AirDeep[AQS]" order 1
    participant "AirDeep[REST Server]" order 2
    participant "AirDeep[MQTT Server]" order 3
    hide footbox
   
    group 1. HTTP - 서버 시간 가져오기
        "AirDeep[AQS]" -> "AirDeep[REST Server]" : Get Current Time
        "AirDeep[AQS]" <- "AirDeep[REST Server]" : Response(time)
        "AirDeep[AQS]" <- "AirDeep[AQS]" : Set Time
    end
    group 2. HTTP - 디바이스 등록
      "AirDeep[AQS]" -> "AirDeep[REST Server]" : Device Register
      "AirDeep[AQS]" <- "AirDeep[REST Server]"  : Response( Token, MQTT Host )
    end
    
    group 3. MQTT - Session Establish, Telemetry Data, Command(ATTRIBUTE CHANGED, RPC)
      group 3.1 MQTT - Session
        "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : MQTT Connection
        "AirDeep[AQS]" <--> "AirDeep[MQTT Server]" : Ack(Connected)
      end

      == MQTT Session Established ==

     group 3.2 MQTT - 'Update - ATTRIBUTE CHANGED' (Client Side - firmwareVersion, lastBootingTime, timeZone)
     "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 2.1) Publish - ATTRIBUTES_TOPIC
     end

     group 3.3 MQTT - 'Command - ATTRIBUTE CHANGED' (Server Side)
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Subscribe - ATTRIBUTE_TOPIC
       "AirDeep[AQS]" <- "AirDeep[MQTT Server]" : Publish Data(OnMessage) - CHANGED ATTRIBUTE VALUE
       "AirDeep[AQS]" <- "AirDeep[AQS]" : Update Device
     end

     group 3.4 MQTT - 'Command - RPC'
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Subscribe - RPC_REQUEST_TOPIC
       "AirDeep[AQS]" <- "AirDeep[MQTT Server]" : Publish Data(OnMessage) - RPC Command
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Publish ACK
       "AirDeep[AQS]" <- "AirDeep[AQS]" : Control Process
     end

     group 3.5 MQTT - Upload Telemetry Device Log Data
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Upload Device Log Data (Publish TELEMETRY_TOPIC)
     end

     group 3.6 MQTT - Upload Telemetry Sensor Data
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Upload Telemetry Data (Publish TELEMETRY_TOPIC)
     end

      == MQTT Session Established ==
    end
  @enduml
  ```

#### [**3.1.1 현재 시간 가져오기**](./docs/HTTP-API-1.GetCurrentTime.md)
   - 목적 : 디바이스는 서버 시간 정보를 가져온 후 디바이스에 시간 정보를 셋팅한다.
  
  </br>

#### [**3.1.2 디바이스 등록**](./docs/HTTP-API-1.GetCurrentTime.md)
   - 목적 : 디바이스는 최초 구동시 디바이스 등록 과정을 하여 Access Token(credentialsId) 및 MQTT Host 정보를 가져온다.
  
  </br>

#### [**3.1.3 MQTT Connect**](./docs/MQTT-API-1.AttributeChangeCommand.md)
   - 목적 : [3.1 디바이스 등록] 과정에서의 얻은 MQTT Host 정보를 바탕으로 MQTT Session 을 수립한다.
  
  </br>

#### [**3.1.4 Command - ATTRIBUTE CHANGED(Client Side)**](./docs/MQTT-API-1.AttributeChangeCommand.md)
   - 목적 : 디바이스는 자신의 정보를 서버에 전달하여 저장을 하거나, 업데이트 할 수 있다.
  
  </br>

#### [**3.1.5 Command - ATTRIBUTE CHANGED(Server Side)**](./docs/MQTT-API-1.AttributeChangeCommand.md)
   - 목적 : 서버와 디바이스는 SHARD-ATTRIBUTE(uploadFrequency) 를 공유하고 있으며, 서버에서 SHARD-ATTRIBUTE(uploadFrequency) 의 변경을 통해 제어를 한다.
  
  </br>

#### [**3.1.6 Command - RPC**](./docs/MQTT-API-2.RpcCommand.md)
   - 목적 : 서버는 디바이스와 MQTT Session 수립 후 RPC Command 를 단말에 전달하여 디바이스 제어를 한다.
  
  </br>

#### [**3.1.7 Upload Telemetry Device Log Data**](./docs/MQTT-API-3.UploadTelemetryLogData.md)
   - 목적 : 디바이스는 서버와 MQTT Session 수립 후 'Shard Attribute - remoteLogLevel' 에 따라 Log Data를 서버에 전달한다.

#### [**3.1.8 Upload Telemetry Sensor Data**](./docs/MQTT-API-4.UploadTelemetrySensorData.md)
   - 목적 : 디바이스는 서버와 MQTT Session 수립 후 Sensor Data 를 서버에 전달한다.
  </br>
  
### 3.2  서비스 연동 API
- 디바이스는 서버 시간 정보를 가져온 후 디바이스에 시간 정보를 셋팅한다.
- 디바이스는 최초 구동시 디바이스 등록 과정을 하여 Access Token 및 MQTT Host 정보를 가져온다.
- 시퀀스 다이어그램

  ```plantuml
  @startuml
   participant "AirDeep[AQS]" order 1
   participant "AirDeep[REST Server]" order 2
   hide footbox
   group 1. HTTP - 현재 시간 가져오기
       "AirDeep[AQS]" -> "AirDeep[REST Server]" : Get Current Time
       "AirDeep[AQS]" <- "AirDeep[REST Server]" : Response(time)
       "AirDeep[AQS]" <- "AirDeep[AQS]" : Set Time
   end
   group 2. HTTP - 디바이스 등록
     "AirDeep[AQS]" -> "AirDeep[REST Server]" : Device Register
     "AirDeep[AQS]" <- "AirDeep[REST Server]"  : Response( token, mqtt host info )
   end

  @enduml
  ```

</br>

### 3.2.1 REST API
  | Name | HTTP Method | Request URI | Description |
  |:--------- | :--------- | :--------- | :--------- |
  | [**getCurrentTime**](./docs/HTTP-API-1.GetCurrentTime.md)| **GET** | /v1/infos/time |   서버의 시간 정보를 가져온다. |
  | [**register**](./docs/HTTP-API-2.DeviceRegister.md) | **POST** | /v1/devices/register |  서버에 디바이스 등록을 시도한다.</br> 등록 완료 후 서버의 응답으로 Access Token(credentialsId), MQTT Host 를 수신한다. |

  
</br>

### 3.2.2 MQTT API
 - MQTT API 를 사용하기 위해서는 반드시 **디바이스 등록과정** 이 선행 되어야 한다.
 - AQS 디바이스는 **디바이스 등록과정** 을 통해 얻어진 MQTT Host 정보로 MQTT Host 와 MQTT Session Establish 되어야 한다.
 - 제어 
   - MQTT Session 수립 후 서버가 RPC Command 를 단말에 전달하여 디바이스 제어를 할 수 있다.
   - MQTT Session 수립 후 서버가 SHARD-ATTRIBUTE(uploadFrequency) 변경하여 센서 데이터 업로드 주기를 설정 할 수 있다.
   - MQTT Session 수립 후 서버가 SHARD-ATTRIBUTE(remoteLogLevel) 변경하여 단말이 서버로 업로드 하는 로그 레벨을 설정 할 수 있다.
 - 시퀀스 다이어그램
 
   ```plantuml
   @startuml
     participant "AirDeep[AQS]" order 1
     participant "AirDeep[MQTT Server]" order 2
     hide footbox
      group 1. MQTT - Connection
      "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : MQTT Connection
      "AirDeep[AQS]" <--> "AirDeep[MQTT Server]"  : Ack(Connected)
     end
     == MQTT Session Established ==
     group 2.1 MQTT - 'Command - ATTRIBUTE CHANGED' (Server Side)
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Subscribe - ATTRIBUTE_TOPIC
       "AirDeep[AQS]" <- "AirDeep[MQTT Server]" : Publish Data - CHANGED ATTRIBUTE VALUE
       "AirDeep[AQS]" <- "AirDeep[AQS]" : on_message
     end

     group 2.2 MQTT - 'Command - RPC'
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Subscribe - RPC_REQUEST_TOPIC
       "AirDeep[AQS]" <- "AirDeep[MQTT Server]" : Publish Data - RPC Command
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Publish ACK
       "AirDeep[AQS]" <- "AirDeep[AQS]" : on_message
     end

     group 2.3 MQTT - Upload Telemetry Device Log Data
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Upload Telemetry Device Log Data (Publish TELEMETRY_TOPIC)
     end

     group 2.4 MQTT - Upload Telemetry Sensor Data
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : Upload Telemetry Sensor Data (Publish TELEMETRY_TOPIC)
     end

     == MQTT Session Established ==
   @enduml
   ```

</br>

 - MQTT API 리스트 
    | Method | Topic | Description |
    |:------------- |:------------- |:-------------|
    | **ATTRIBUTES_TOPIC(Client-Side)** | **v1/devices/me/attributes** | 디바이스가 디바이스의 속성정보 서버로 전송한다. |
    | **ATTRIBUTES_TOPIC(Server-Side)** | **v1/devices/me/attributes** | 서버가 Shared 속성 정보 변경 후 변경된 정보를 디바이스로 전달한다.  |
    | **ATTRIBUTES_REQUEST_TOPIC** | **v1/devices/me/attributes/request/1** | 디바이스는 서버가 가지고 있는 디바이스의 정보를 요청한다. |
    | **ATTRIBUTES_REQUEST_TOPIC** | **v1/devices/me/attributes/request/1** | 서버는 요청 받은 디바이스의 정보를 디바이스에 전달한다. |
    | **RPC_REQUEST_TOPIC** | **v1/devices/me/rpc/request/+** | 서버는 RPC 명령어 디바이스로 전달한다.  |
    | **TELEMETRY_TOPIC**(RPC 응답) | **v1/devices/me/telemetry** | 디바이스는 서버로 센서 데이터를 전달한다. </br>　***JsonKey 로 응답을 구분한다.***|
    | **TELEMETRY_TOPIC**(로그 전달) | **v1/devices/me/telemetry** | 디바이스는 서버로 로그 데이터를 전달한다. </br>　***JsonKey: remoteLogMessage*** |
    | **TELEMETRY_TOPIC**(센서 값 전달) | **v1/devices/me/telemetry** | 디바이스는 서버로 센서 데이터를 전달한다. |
</br></br>

***

# *Appendix A. HTTP STATUS CODE 정의*
  | HTTP STATUS CODE | 설명 |
  |:-----------------|:-----|
  |200(OK)| 정상적으로 처리 |
  |400(Bad Request)| 파라미터 유효성 검사 중 에러 |
  |401(Unauthorized)| API 인증 실패 |
  |403(Forbidden)| API 접근 권한 오류 |
  |404(Not Found)| 요청 API 가 존재하지 않거나 사용자가 경우 (리소스가 없는 경우) |
  |500(Internal Server Error)|서버 내부 로직등의 예외적인 에러 |
  |503(Service Unavailable)|서버 유지보수, 과부하등으로 요청 처리를 할 수 없는 경우|
  |504(Gateway Timeout)| 요청 후 응답 Timeout 이 발생한 경우|
  
  <br>


* A1.1 성공 응답 Body
  ```
  요청에 대하여 성공인 경우는 HTTP Status code를 200(성공), 응답 Body 값은 각 각의 API 에서 정의된 값으로 반환한다.
  ```

* 실패 응답 Body
  ```
  실패인 경우 HTTP Status Code 값을 4xx, 5xx 반환하며, 응답 Body 값은 각 각의 API 에서 정의된 값으로 반환한다. (응답 Body에 정의된 msgType 값을 통하여 상세 에러내역을 확인 할 수 있다)
  ```
</br>

# *Appendix B. MQTT STATUS CODE 정의*
  | RESPONSE CODE | 설명 |
  |:-----------------|:-----|
  |0 (OK)| 정상적으로 처리 |
  |1 (Connection Refused: Unacceptable protocol version)| 프로토콜 에러( mqtt version : mqttv31, mqttv311에 따른 broker와 client간 version 틀렸을때 오류) |
  |2 (Connection Refused: Identifier rejected)| 인증실패 |
  |3 (Connection Refused: Server Unavailable)| connection 후 서버다운되어 연동실패 |
  |4 (Connection Refused: Bad username or password)| mqtt id/pwd 틀림 |
  |5 (Connection Refused: Authorization error)|access token 틀림(connection은 되나 pub/sub 시 오류코드발생|
  |6 (Connection lost or bad)| 요청 후 응답 Timeout 이 발생한 경우|
  
  <br>


* A1.1 성공 응답 Body
  ```
  요청에 대하여 성공인 경우는 mqtt response code (rc)를 0(성공), topic 및 payload 값으로 추가작업한다.
  ```

* 실패 응답 Body
  ```
  실패인 경우 mqtt response code (rc) 0외 값으로 반환된다.
  ```
</br>

# *Appendix C. Sample 소스*
- [**Python Sample 소스**](#python-sample-소스)
  - [**필요한 lib**](#필요한-lib)
  - [**소스 파일**](#소스-파일)
  - [**HOST 정보**](#host-정보)
  - [**MQTT Sample 실행 방법**](#mqtt-sample-실행-방법)
  - [**REST Sample 실행 방법**](#rest-sample-실행-방법)

</br>

## Python Sample 소스
## 필요한 lib
  - python 설치 후 필요한 lib 설치해야함

    | Lib | Description |
    | :--- | :--------- |
    | `time` | 해당 thread를 지연하기 위한 |
    | `datetime` | 센서정보 측정된 timestamp |
    | `random` | random 데이터 사용함  |
    | `json` | josn으로 변경 또는 string으로 변경시 사용|
    | `paho.mqtt.client` | mqtt 연동시 사용 |
    | `threading` | threading |

</br>

## 소스 파일
  - Sample 소스로 아래 파일들이 제공된다.
    | File | Description  |
    | :--- | :----------- |
    | `login.py` | username & password으로 mobideep 서버에 로그인하는 모듈 |
    | `provisioning.py` | 디바이스 및 telemetry 데이터 관리 API 모듈 |
    | `simple-mqtt-emul.py` | 2 step 디바이스 등록 및 MQTT 연동 후  데이터 전송  |
    | `mqtt-emul.py` | 디바이스 등록 및 MQTT 연동 후  데이터 전송  |
    | `rest-emul.py` | 디바이스 등록 및 데이터 전송|

</br>

## HOST 정보 (**HOST 정보 변경 예정**)
```json
AIRDEEP_HTTP_HOST = "https://api-airdeep.rtdata.co.kr"  
AIRDEEP_MQTT_HOST = "mqtt-airdeep.rtdata.co.kr"
AIRDEEP_MQTT_PORT= 1883
username = "Airdeep 에서 제공한 사용자 아이디"
password ="Airdeep 에서 제공한 사용자 아이디"
```
## Onestep device 등록 및 MQTT Sample 실행 방법

```
python simple-mqtt-emul.py 
```
명령어가 나오는대로 파라미터 입력: username, password, deviceName
