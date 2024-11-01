[flows (3).json](https://github.com/user-attachments/files/17601010/flows.3.json)[flows (3).json](https://github.com/user-attachments/files/17600978/flows.3.json)# ex-0-1
Arduino Ethernet Shield 2 Raspberry Pi 4 Windows11

제목:Arduino Ethernet2 세팅 프로그램/릴레이스위치8단 Modbus프로그램
//////////시작////////////////////////////


//SPI통신을 하기때문에 내장SPI라이브러리필요
#include <SPI.h>
//설치한 이더넷 라이브러리
#include <Ethernet2.h>

//인터넷공간에서 나의 아두이노의 고유한 식별자
//지금은 별로 필요한 녀석이 아님!
byte mac[] = {
  0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED
};
//내 장치의 고유한 IP주소
IPAddress ip(172, 30, 1, 121);

//TCP서버의 포트번호가 몇인가?
//modbus tcp의 기본 포트번호
EthernetServer server(502);

//서버가 클라이언트로 일정한 간격으로 보내기위한 타이머 카운터
unsigned long send_t = 0;

//필드변수끼리 메모리를 공유한다
union{
  byte input[2];
  uint16_t output;
}converter;

//코일주소 0~7까지 8개의 주소가 있는 상황이다!
byte coil[] = {26,28,30,32,34,36,38,40,42,2,3};//어드레스맵
//HIGH == true == 1
//LOW == false == 0
bool coil_state[] = {LOW,LOW,LOW,LOW,LOW,LOW,LOW,LOW};

void setup() {
  //아두이노의 내부 사정을 시리얼모니터에 출력하는 용도
  Serial.begin(9600);

  //아두이노의 GPIO를 모두 출력으로 설정하되
  //릴레이가 로우레벨트리거이기 때문에 HIGH로 초기화한다!
  for(int i = 0;i<sizeof(coil);i++){
    pinMode(coil[i],OUTPUT);
    digitalWrite(coil[i],HIGH);
  }
  
  //Ethernet.init(CS핀번호)
  //상단에 mac주소와 ip주소로 공유기의 IP주소를 발급받는다!
  Ethernet.begin(mac, ip);
  //TCP서버 시작하기!
  server.begin();
  //공유기로부터 발급받은 IP주소를 시리얼모니터에 출력한다!
  //상단에 입력한 IP주소와 다른게 출력된다면 
  //1.중복이 되었거나
  //2.공유기가 발급해줄수없는 대역의 IP주소를 입력했거나
  //3.W5500을 연결을 잘못했거나
  Serial.print("server is at ");
  Serial.println(Ethernet.localIP());
}


void loop() {
  //서버가 클라이언트의 접속을 기다리는 부분
  EthernetClient client = server.available();

  //만약 대기중인 클라이언트가 없으면 아래 조건문은 거짓이다!
  if (client) {
    Serial.println("새로운 클라이언트 등장!");

    //무한루프를 돌리고 접속한 클라이언트가 접속을 종료하거나
    //혹은 강제로 접속이 끊어진경우 서버로 접속을 종료하고
    //대기열에있는 새로운 클라이언트의 접속을 허용한다
    while(client.connected()){
      //서버와 클라이언트가 접속을 한 상태에서 뭐할래?
      //서버가 1초간격으로 클라이언트에게 특정한 문자열을 전송한다!
      //이제부터는 클라이언트가 request를 보내기전까지는
      //서버는 어떠한 데이터도 클라이언트로 전송할 수 없음!

      //mapping table대로 명령을 수행한다!
      for(int i = 0;i<sizeof(coil);i++){
        digitalWrite(coil[i],!coil_state[i]);
      }
      
      /*
      if(millis() - send_t > 1000){
        send_t = millis();
        //이부분이 1000밀리초마다 한번씩 조건이 걸림!
        String server_msg = "{\"type\":\"server\",\"data1\":10,\"data2\":20}";
        client.println(server_msg);
      }
      */
      //클라이언트가 일정한 간격으로 보낸 데이터를 서버가 수신해서 시리얼모니터에 출력한다!
      if(client.available() > 0){
        //서버는 클라이언트가 보낸 데이터 3바이트를 읽는다
        byte req[12];
        //REQ
        client.readBytes(req,sizeof(req)); //이부분에서 req에 데이터가 대입됨!
        /*
        Serial.print("REQ : ");
        for(int i = 0;i<sizeof(req);i++){
           Serial.print(req[i],HEX);
           Serial.print(", ");
        }
        Serial.println();
        */
        //데이터해석하기!
        //수신받은 데이터를 modbus_tcp구조체에 카피하기

        converter.input[1] = req[0];
        converter.input[0] = req[1];
        uint16_t msg_id = converter.output;
        converter.input[1] = req[2];
        converter.input[0] = req[3];
        uint16_t type = converter.output;
        converter.input[1] = req[4];
        converter.input[0] = req[5];
        uint16_t len = converter.output;
        uint8_t id = req[6];
        uint8_t fc = req[7];
        converter.input[1] = req[8];
        converter.input[0] = req[9];
        uint16_t addr = converter.output;
        converter.input[1] = req[10];
        converter.input[0] = req[11];
        uint16_t cmd = converter.output;
        /*
        Serial.print("메시지번호:");
        Serial.println(msg_id,HEX);
        Serial.print("프로토콜타입:");
        Serial.println(type,HEX);
        Serial.print("데이터길이:");
        Serial.println(len,HEX);
        Serial.print("국번:");
        Serial.println(id,HEX);
        Serial.print("펑션코드:");
        Serial.println(fc,HEX);
        */

        if(fc == 0x01){
          //read coils
          /*
          Serial.print("시작주소:");
          Serial.println(addr,HEX);
          Serial.print("코일갯수:");
          Serial.println(cmd,HEX);
          */
          byte nockanda_coil = 0;
          for(int i = 0;i<sizeof(coil);i++){
            nockanda_coil = nockanda_coil | (coil_state[i]<<i);  
          }
          byte res[] = {req[0],req[1],req[2],req[3],0x00,0x04,id,fc,0x01,nockanda_coil};
          client.write(res,sizeof(res));
        }else if(fc == 0x05){
          //write single coil
          /*
          Serial.print("코일주소:");
          Serial.println(addr,HEX);
          Serial.print("제어명령:");
          Serial.println(cmd,HEX);
          */
          if(cmd == 0xFF00){
            //ON
            //digitalWrite(coil[addr],HIGH);
            coil_state[addr] = HIGH;
          }else{
            //OFF
            //digitalWrite(coil[addr],LOW);
            coil_state[addr] = LOW;
          }
          //응답
          client.write(req,sizeof(req));
        }
        
      }
    }
    //클라이언트가 접속을 종료한부분!
    client.stop();
    Serial.println("클라이언트와 접속이 해제됨!");
  }
}



//////////////////끝///////////////////////////



[1]node-red Modbus 프로그램 인공지능 MQTT uiLED 등등..



/////////////시작////////////////////



 [Upload[
   
    
    
    {
        "id": "3cba2ff2f06b7801",
        "type": "tab",
        "label": "플로우 1",
        "disabled": false,
        "info": "",
        "env": []
    },
    {
        "id": "0ebe45b38a5f16d6",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "04e05dec6601139d",
        "order": 0,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 690,
        "y": 560,
        "wires": [
            [
                "77ea7acc87c0ef27",
                "023dfd303157b468"
            ]
        ]
    },
    {
        "id": "2346ec309ee0814c",
        "type": "switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "property": "payload",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "ON",
                "vt": "str"
            },
            {
                "t": "eq",
                "v": "OFF",
                "vt": "str"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 290,
        "y": 200,
        "wires": [
            [
                "b9c4e3b0264f676e"
            ],
            [
                "75edd146b5a58c1e"
            ]
        ]
    },
    {
        "id": "b9c4e3b0264f676e",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 1",
        "func": "\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 460,
        "y": 180,
        "wires": [
            [
                "defb98da217371d8",
                "3626f6a91675df87"
            ]
        ]
    },
    {
        "id": "75edd146b5a58c1e",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 2",
        "func": "\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 460,
        "y": 260,
        "wires": [
            [
                "defb98da217371d8"
            ]
        ]
    },
    {
        "id": "3626f6a91675df87",
        "type": "mqtt out",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele",
        "qos": "",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "129593e0.7e203c",
        "x": 540,
        "y": 60,
        "wires": []
    },
    {
        "id": "c94d325413e498f0",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 900,
        "y": 60,
        "wires": [
            [
                "90d690a0ffebce9a",
                "175e424108850d9d"
            ]
        ]
    },
    {
        "id": "eaa9ed4e16f2c956",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 1",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1160,
        "y": 300,
        "wires": []
    },
    {
        "id": "defb98da217371d8",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "9ec6343d08d13227",
        "order": 2,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 670,
        "y": 300,
        "wires": [
            [
                "175e424108850d9d",
                "90d690a0ffebce9a"
            ]
        ]
    },
    {
        "id": "3fc4ae8151b3b13a",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "419cffd938dae8e5",
        "order": 2,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 690,
        "y": 800,
        "wires": [
            [
                "5c7ee1f0ec095c8b",
                "e9847ca203b82a84"
            ]
        ]
    },
    {
        "id": "2719512aac871c34",
        "type": "switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "property": "payload",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "ON",
                "vt": "str"
            },
            {
                "t": "eq",
                "v": "OFF",
                "vt": "str"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 350,
        "y": 760,
        "wires": [
            [
                "e614df7b24cca553"
            ],
            [
                "3f3fa7fe1eab16fa"
            ]
        ]
    },
    {
        "id": "e614df7b24cca553",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 3",
        "func": "\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 500,
        "y": 740,
        "wires": [
            [
                "3fc4ae8151b3b13a",
                "ff4c484c20114bf1"
            ]
        ]
    },
    {
        "id": "3f3fa7fe1eab16fa",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 4",
        "func": "\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 500,
        "y": 800,
        "wires": [
            [
                "3fc4ae8151b3b13a"
            ]
        ]
    },
    {
        "id": "ff4c484c20114bf1",
        "type": "mqtt out",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele2",
        "qos": "",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "129593e0.7e203c",
        "x": 630,
        "y": 660,
        "wires": []
    },
    {
        "id": "6a0dd9b8bb03af4a",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele2",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 920,
        "y": 680,
        "wires": [
            [
                "5c7ee1f0ec095c8b",
                "e9847ca203b82a84"
            ]
        ]
    },
    {
        "id": "6b2bf7e399c59b88",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele1",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 860,
        "y": 420,
        "wires": [
            [
                "77ea7acc87c0ef27",
                "023dfd303157b468"
            ]
        ]
    },
    {
        "id": "7759aacaec9cb235",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "a99af2e5d0b2aaa4",
        "order": 0,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 670,
        "y": 1060,
        "wires": [
            [
                "441144f5113d977e",
                "04c88e1ba51c41af"
            ]
        ]
    },
    {
        "id": "7641b966abbcc64c",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele3",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 840,
        "y": 920,
        "wires": [
            [
                "441144f5113d977e",
                "04c88e1ba51c41af"
            ]
        ]
    },
    {
        "id": "8d218969e09f0c13",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "6232e555484312cf",
        "order": 2,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 710,
        "y": 1320,
        "wires": [
            [
                "7eaa4a41bdb17b4f",
                "c0649f9ac81e2c2e"
            ]
        ]
    },
    {
        "id": "b263c8c853f3984e",
        "type": "switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "property": "payload",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "ON",
                "vt": "str"
            },
            {
                "t": "eq",
                "v": "OFF",
                "vt": "str"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 370,
        "y": 1280,
        "wires": [
            [
                "cee3c53767f71468"
            ],
            [
                "6eca0171d9ae014a"
            ]
        ]
    },
    {
        "id": "cee3c53767f71468",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 5",
        "func": "\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 520,
        "y": 1260,
        "wires": [
            [
                "8d218969e09f0c13",
                "bf334bd28a6052ab"
            ]
        ]
    },
    {
        "id": "6eca0171d9ae014a",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 6",
        "func": "\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 520,
        "y": 1320,
        "wires": [
            [
                "8d218969e09f0c13"
            ]
        ]
    },
    {
        "id": "bf334bd28a6052ab",
        "type": "mqtt out",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele4",
        "qos": "",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "129593e0.7e203c",
        "x": 670,
        "y": 1180,
        "wires": []
    },
    {
        "id": "cf1a21224f077616",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele4",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 940,
        "y": 1200,
        "wires": [
            [
                "7eaa4a41bdb17b4f",
                "c0649f9ac81e2c2e"
            ]
        ]
    },
    {
        "id": "b13c62563c555edc",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "8698074accc216c0",
        "order": 0,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 710,
        "y": 1580,
        "wires": [
            [
                "bdf99af5ee06f766",
                "eb0133e04a1e37a3"
            ]
        ]
    },
    {
        "id": "93706e28be1249e3",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele5",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 880,
        "y": 1440,
        "wires": [
            [
                "bdf99af5ee06f766",
                "eb0133e04a1e37a3"
            ]
        ]
    },
    {
        "id": "658f226d402bd858",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "1916d5b2b56dbea0",
        "order": 2,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 790,
        "y": 1820,
        "wires": [
            [
                "76b6c6e5be9074af",
                "958621ede5a0311f"
            ]
        ]
    },
    {
        "id": "75de0840e3cbb4c7",
        "type": "switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "property": "payload",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "ON",
                "vt": "str"
            },
            {
                "t": "eq",
                "v": "OFF",
                "vt": "str"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 450,
        "y": 1780,
        "wires": [
            [
                "107ed9bf60970312"
            ],
            [
                "a445c16b37196c50"
            ]
        ]
    },
    {
        "id": "107ed9bf60970312",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 7",
        "func": "\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 600,
        "y": 1760,
        "wires": [
            [
                "658f226d402bd858",
                "be3f15a9ac1c7a74"
            ]
        ]
    },
    {
        "id": "a445c16b37196c50",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 8",
        "func": "\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 600,
        "y": 1820,
        "wires": [
            [
                "658f226d402bd858"
            ]
        ]
    },
    {
        "id": "be3f15a9ac1c7a74",
        "type": "mqtt out",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele6",
        "qos": "",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "129593e0.7e203c",
        "x": 730,
        "y": 1700,
        "wires": []
    },
    {
        "id": "bdb08af520ccf798",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele6",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 1020,
        "y": 1700,
        "wires": [
            [
                "76b6c6e5be9074af",
                "958621ede5a0311f"
            ]
        ]
    },
    {
        "id": "7e2f5cf184b44de0",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "8b177f8e1fccb595",
        "order": 0,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 790,
        "y": 2080,
        "wires": [
            [
                "65b2cdbfa46035d5",
                "ad4422f1eea501c2"
            ]
        ]
    },
    {
        "id": "51b39851f0c899d6",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele7",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 960,
        "y": 1940,
        "wires": [
            [
                "65b2cdbfa46035d5",
                "ad4422f1eea501c2"
            ]
        ]
    },
    {
        "id": "66976952f981833f",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "83297fde04603a3c",
        "order": 2,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 770,
        "y": 2320,
        "wires": [
            [
                "90e58967361d4ad6",
                "37e4cf70033039d4"
            ]
        ]
    },
    {
        "id": "3f21effb45336c7b",
        "type": "switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "property": "payload",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "ON",
                "vt": "str"
            },
            {
                "t": "eq",
                "v": "OFF",
                "vt": "str"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 290,
        "y": 2300,
        "wires": [
            [
                "d57fd2d054f789f6"
            ],
            [
                "d9fd8381a3b05d89"
            ]
        ]
    },
    {
        "id": "d57fd2d054f789f6",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 9",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 460,
        "y": 2260,
        "wires": [
            [
                "66976952f981833f",
                "22d6ee53868a1cdd"
            ]
        ]
    },
    {
        "id": "d9fd8381a3b05d89",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 10",
        "func": "\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 470,
        "y": 2320,
        "wires": [
            [
                "66976952f981833f"
            ]
        ]
    },
    {
        "id": "22d6ee53868a1cdd",
        "type": "mqtt out",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele8",
        "qos": "",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "129593e0.7e203c",
        "x": 650,
        "y": 2160,
        "wires": []
    },
    {
        "id": "1f7cf3aaa59c036a",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele8",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 1120,
        "y": 2160,
        "wires": [
            [
                "90e58967361d4ad6",
                "37e4cf70033039d4"
            ]
        ]
    },
    {
        "id": "106bcd2938eeacab",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "6e0c9516caac10ee",
        "order": 0,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 810,
        "y": 2560,
        "wires": [
            [
                "66c344540b4d2ba8",
                "b0075b9d76e76dcd"
            ]
        ]
    },
    {
        "id": "f433c318ba5e77ba",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele9",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 980,
        "y": 2420,
        "wires": [
            [
                "66c344540b4d2ba8",
                "b0075b9d76e76dcd"
            ]
        ]
    },
    {
        "id": "3fb6241cdb779249",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 2",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1160,
        "y": 560,
        "wires": []
    },
    {
        "id": "80e7a7662ef61b32",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 3",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1200,
        "y": 800,
        "wires": []
    },
    {
        "id": "f130589216f5b92f",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 4",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1180,
        "y": 1080,
        "wires": []
    },
    {
        "id": "5266b4e870083d3d",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 5",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1220,
        "y": 1560,
        "wires": []
    },
    {
        "id": "ea5484d68d493406",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 6",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1300,
        "y": 1820,
        "wires": []
    },
    {
        "id": "e4b735fcf69ad894",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 7",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1240,
        "y": 2080,
        "wires": []
    },
    {
        "id": "cc2ba4ca5692d987",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 8",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1300,
        "y": 2320,
        "wires": []
    },
    {
        "id": "1cd2abfdc30bae63",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 9",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1260,
        "y": 2560,
        "wires": []
    },
    {
        "id": "77ea7acc87c0ef27",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "2",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 940,
        "y": 560,
        "wires": [
            [
                "3fb6241cdb779249"
            ],
            []
        ]
    },
    {
        "id": "90d690a0ffebce9a",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "1",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 920,
        "y": 300,
        "wires": [
            [
                "eaa9ed4e16f2c956"
            ],
            []
        ]
    },
    {
        "id": "5c7ee1f0ec095c8b",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "3",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1000,
        "y": 800,
        "wires": [
            [
                "80e7a7662ef61b32"
            ],
            []
        ]
    },
    {
        "id": "441144f5113d977e",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "4",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 920,
        "y": 1060,
        "wires": [
            [
                "f130589216f5b92f"
            ],
            []
        ]
    },
    {
        "id": "7eaa4a41bdb17b4f",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "5",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1020,
        "y": 1320,
        "wires": [
            [],
            []
        ]
    },
    {
        "id": "bdf99af5ee06f766",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "6",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 960,
        "y": 1580,
        "wires": [
            [
                "5266b4e870083d3d"
            ],
            []
        ]
    },
    {
        "id": "76b6c6e5be9074af",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "7",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1100,
        "y": 1820,
        "wires": [
            [
                "ea5484d68d493406"
            ],
            []
        ]
    },
    {
        "id": "65b2cdbfa46035d5",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "8",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1040,
        "y": 2080,
        "wires": [
            [
                "e4b735fcf69ad894"
            ],
            []
        ]
    },
    {
        "id": "90e58967361d4ad6",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "9",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1040,
        "y": 2320,
        "wires": [
            [
                "cc2ba4ca5692d987"
            ],
            []
        ]
    },
    {
        "id": "66c344540b4d2ba8",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "10",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1060,
        "y": 2560,
        "wires": [
            [
                "1cd2abfdc30bae63"
            ],
            []
        ]
    },
    {
        "id": "5943fb366b8d3ea4",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "575ae74c180920c0",
        "order": 2,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 770,
        "y": 2880,
        "wires": [
            [
                "143678b41d59b6ca",
                "ac24347382178d23"
            ]
        ]
    },
    {
        "id": "8fb578e40c69c8c8",
        "type": "switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "property": "payload",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "ON",
                "vt": "str"
            },
            {
                "t": "eq",
                "v": "OFF",
                "vt": "str"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 290,
        "y": 2860,
        "wires": [
            [
                "082ae8aa1a7b41bf"
            ],
            [
                "a2d9756d93b6c9e7"
            ]
        ]
    },
    {
        "id": "082ae8aa1a7b41bf",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 11",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 460,
        "y": 2820,
        "wires": [
            [
                "5943fb366b8d3ea4",
                "7f898d44a0674432"
            ]
        ]
    },
    {
        "id": "a2d9756d93b6c9e7",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 12",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 470,
        "y": 2880,
        "wires": [
            [
                "5943fb366b8d3ea4"
            ]
        ]
    },
    {
        "id": "7f898d44a0674432",
        "type": "mqtt out",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele10",
        "qos": "",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "129593e0.7e203c",
        "x": 650,
        "y": 2720,
        "wires": []
    },
    {
        "id": "a1de81af7038fdd1",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele10",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 1130,
        "y": 2720,
        "wires": [
            [
                "143678b41d59b6ca",
                "ac24347382178d23"
            ]
        ]
    },
    {
        "id": "e183500190481323",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 11",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1300,
        "y": 2880,
        "wires": []
    },
    {
        "id": "143678b41d59b6ca",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "11",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1040,
        "y": 2880,
        "wires": [
            [
                "e183500190481323"
            ],
            []
        ]
    },
    {
        "id": "e9ccc818dd2b2997",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "84524ce7456f45e7",
        "order": 0,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 770,
        "y": 3200,
        "wires": [
            [
                "ecce429179503654",
                "c53930c2f362697d"
            ]
        ]
    },
    {
        "id": "dfb681f28b640dec",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele11",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 950,
        "y": 3060,
        "wires": [
            [
                "ecce429179503654",
                "c53930c2f362697d"
            ]
        ]
    },
    {
        "id": "c1ff252ffeb7a429",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 12",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1220,
        "y": 3200,
        "wires": []
    },
    {
        "id": "ecce429179503654",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "12",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1020,
        "y": 3200,
        "wires": [
            [
                "c1ff252ffeb7a429"
            ],
            []
        ]
    },
    {
        "id": "ee4dc1a304008a65",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "475553ad1acf571b",
        "order": 2,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 770,
        "y": 3460,
        "wires": [
            [
                "446fdffeecf1bcf3",
                "f7353a930464d768"
            ]
        ]
    },
    {
        "id": "8c68858643993948",
        "type": "switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "property": "payload",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "ON",
                "vt": "str"
            },
            {
                "t": "eq",
                "v": "OFF",
                "vt": "str"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 290,
        "y": 3440,
        "wires": [
            [
                "8c5d1c210dce035a"
            ],
            [
                "d03357bd65779290"
            ]
        ]
    },
    {
        "id": "8c5d1c210dce035a",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 13",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 460,
        "y": 3400,
        "wires": [
            [
                "ee4dc1a304008a65",
                "f31ad854f353d78d"
            ]
        ]
    },
    {
        "id": "d03357bd65779290",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 14",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 470,
        "y": 3460,
        "wires": [
            [
                "ee4dc1a304008a65"
            ]
        ]
    },
    {
        "id": "f31ad854f353d78d",
        "type": "mqtt out",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele12",
        "qos": "",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "129593e0.7e203c",
        "x": 650,
        "y": 3300,
        "wires": []
    },
    {
        "id": "0802ffb4ab0c97ab",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele12",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 1130,
        "y": 3300,
        "wires": [
            [
                "446fdffeecf1bcf3",
                "f7353a930464d768"
            ]
        ]
    },
    {
        "id": "d547e742ac4e539b",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 13",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1300,
        "y": 3460,
        "wires": []
    },
    {
        "id": "446fdffeecf1bcf3",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "13",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1040,
        "y": 3460,
        "wires": [
            [
                "d547e742ac4e539b"
            ],
            []
        ]
    },
    {
        "id": "fe7154f156137508",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "2b01439b1fc48798",
        "order": 0,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 770,
        "y": 3780,
        "wires": [
            [
                "7460b97e16fd12d6",
                "adb46b5e75ecacc3"
            ]
        ]
    },
    {
        "id": "5652f8942c173b7e",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele13",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 950,
        "y": 3640,
        "wires": [
            [
                "7460b97e16fd12d6",
                "adb46b5e75ecacc3"
            ]
        ]
    },
    {
        "id": "694ce45cac2a7804",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 14",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1220,
        "y": 3780,
        "wires": []
    },
    {
        "id": "7460b97e16fd12d6",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "14",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1020,
        "y": 3780,
        "wires": [
            [
                "694ce45cac2a7804"
            ],
            []
        ]
    },
    {
        "id": "51ce8da7634e464e",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "219065a3c0bece05",
        "order": 2,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 870,
        "y": 4040,
        "wires": [
            [
                "d1916a96a5845464",
                "5f26fae32a8ecd0e"
            ]
        ]
    },
    {
        "id": "e2fee4cef83c622f",
        "type": "switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "property": "payload",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "ON",
                "vt": "str"
            },
            {
                "t": "eq",
                "v": "OFF",
                "vt": "str"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 390,
        "y": 4020,
        "wires": [
            [
                "a60efb48dfdb2ecf"
            ],
            [
                "6aa56c5a576c9aa6"
            ]
        ]
    },
    {
        "id": "a60efb48dfdb2ecf",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 15",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 560,
        "y": 3980,
        "wires": [
            [
                "51ce8da7634e464e",
                "12d304dd97cfc30c"
            ]
        ]
    },
    {
        "id": "6aa56c5a576c9aa6",
        "type": "function",
        "z": "3cba2ff2f06b7801",
        "name": "function 16",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 570,
        "y": 4040,
        "wires": [
            [
                "51ce8da7634e464e"
            ]
        ]
    },
    {
        "id": "12d304dd97cfc30c",
        "type": "mqtt out",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele14",
        "qos": "",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "129593e0.7e203c",
        "x": 750,
        "y": 3880,
        "wires": []
    },
    {
        "id": "821afa12e36a0ba1",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele14",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 1230,
        "y": 3880,
        "wires": [
            [
                "d1916a96a5845464",
                "5f26fae32a8ecd0e"
            ]
        ]
    },
    {
        "id": "f53cc43351ec8f04",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 15",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1400,
        "y": 4040,
        "wires": []
    },
    {
        "id": "d1916a96a5845464",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "15",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1140,
        "y": 4040,
        "wires": [
            [
                "f53cc43351ec8f04"
            ],
            []
        ]
    },
    {
        "id": "e3a9034a720b07b6",
        "type": "ui_switch",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "label": "switch",
        "tooltip": "",
        "group": "6765387b5254cee8",
        "order": 0,
        "width": 0,
        "height": 0,
        "passthru": true,
        "decouple": "false",
        "topic": "topic",
        "topicType": "msg",
        "style": "",
        "onvalue": "1",
        "onvalueType": "num",
        "onicon": "",
        "oncolor": "",
        "offvalue": "0",
        "offvalueType": "num",
        "officon": "",
        "offcolor": "",
        "animate": false,
        "className": "",
        "x": 870,
        "y": 4360,
        "wires": [
            [
                "d0e725740a64bd63",
                "6f5a86bece5c9458"
            ]
        ]
    },
    {
        "id": "96bd6083104aabdf",
        "type": "mqtt in",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "topic": "KimSuHo/f/estadoRele15",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "129593e0.7e203c",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 1050,
        "y": 4220,
        "wires": [
            [
                "d0e725740a64bd63",
                "6f5a86bece5c9458"
            ]
        ]
    },
    {
        "id": "e69b2dddd0913751",
        "type": "debug",
        "z": "3cba2ff2f06b7801",
        "name": "debug 16",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": true,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "payload",
        "statusType": "auto",
        "x": 1320,
        "y": 4360,
        "wires": []
    },
    {
        "id": "d0e725740a64bd63",
        "type": "modbus-write",
        "z": "3cba2ff2f06b7801",
        "name": "",
        "showStatusActivities": false,
        "showErrors": false,
        "showWarnings": true,
        "unitid": "1",
        "dataType": "Coil",
        "adr": "16",
        "quantity": "1",
        "server": "fc533e606813e4f3",
        "emptyMsgOnFail": false,
        "keepMsgProperties": false,
        "delayOnStart": false,
        "startDelayTime": "",
        "x": 1120,
        "y": 4360,
        "wires": [
            [
                "e69b2dddd0913751"
            ],
            []
        ]
    },
    {
        "id": "175e424108850d9d",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 1,
        "group": "9ec6343d08d13227",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 670,
        "y": 140,
        "wires": []
    },
    {
        "id": "e9847ca203b82a84",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 1,
        "group": "419cffd938dae8e5",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 690,
        "y": 760,
        "wires": []
    },
    {
        "id": "023dfd303157b468",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 2,
        "group": "04e05dec6601139d",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 630,
        "y": 500,
        "wires": []
    },
    {
        "id": "04c88e1ba51c41af",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 2,
        "group": "a99af2e5d0b2aaa4",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 610,
        "y": 1000,
        "wires": []
    },
    {
        "id": "c0649f9ac81e2c2e",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 1,
        "group": "6232e555484312cf",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 710,
        "y": 1280,
        "wires": []
    },
    {
        "id": "eb0133e04a1e37a3",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 2,
        "group": "8698074accc216c0",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 650,
        "y": 1520,
        "wires": []
    },
    {
        "id": "958621ede5a0311f",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 1,
        "group": "1916d5b2b56dbea0",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 790,
        "y": 1780,
        "wires": []
    },
    {
        "id": "ad4422f1eea501c2",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 2,
        "group": "8b177f8e1fccb595",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 730,
        "y": 2020,
        "wires": []
    },
    {
        "id": "37e4cf70033039d4",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 1,
        "group": "83297fde04603a3c",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 890,
        "y": 2220,
        "wires": []
    },
    {
        "id": "b0075b9d76e76dcd",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 2,
        "group": "6e0c9516caac10ee",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 750,
        "y": 2500,
        "wires": []
    },
    {
        "id": "ac24347382178d23",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 1,
        "group": "575ae74c180920c0",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 890,
        "y": 2780,
        "wires": []
    },
    {
        "id": "c53930c2f362697d",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 2,
        "group": "84524ce7456f45e7",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 710,
        "y": 3140,
        "wires": []
    },
    {
        "id": "f7353a930464d768",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 1,
        "group": "475553ad1acf571b",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 890,
        "y": 3360,
        "wires": []
    },
    {
        "id": "adb46b5e75ecacc3",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 2,
        "group": "2b01439b1fc48798",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 710,
        "y": 3720,
        "wires": []
    },
    {
        "id": "5f26fae32a8ecd0e",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 1,
        "group": "219065a3c0bece05",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 990,
        "y": 3940,
        "wires": []
    },
    {
        "id": "6f5a86bece5c9458",
        "type": "ui_led",
        "z": "3cba2ff2f06b7801",
        "order": 2,
        "group": "6765387b5254cee8",
        "width": 0,
        "height": 0,
        "label": "",
        "labelPlacement": "left",
        "labelAlignment": "left",
        "colorForValue": [
            {
                "color": "#ffff00",
                "value": "1",
                "valueType": "num"
            },
            {
                "color": "#ff0000",
                "value": "0",
                "valueType": "num"
            }
        ],
        "allowColorForValueInMessage": false,
        "shape": "circle",
        "showGlow": true,
        "name": "",
        "x": 810,
        "y": 4300,
        "wires": []
    },
    {
        "id": "f511bebce57344ac",
        "type": "alexa-smart-home-v3",
        "z": "3cba2ff2f06b7801",
        "conf": "71b1aa46b7ec8cfb",
        "device": "31165",
        "acknowledge": true,
        "name": "침대",
        "topic": "",
        "x": 110,
        "y": 120,
        "wires": [
            [
                "7e5fa1eeca890bc6",
                "2346ec309ee0814c"
            ]
        ]
    },
    {
        "id": "27991d132043d9ae",
        "type": "alexa-smart-home-v3",
        "z": "3cba2ff2f06b7801",
        "conf": "26512a1aa5ff7b8e",
        "device": "36484",
        "acknowledge": true,
        "name": "창고",
        "topic": "",
        "x": 150,
        "y": 680,
        "wires": [
            [
                "5c74b85034a12c74",
                "2719512aac871c34"
            ]
        ]
    },
    {
        "id": "0fd32552659e0b71",
        "type": "alexa-smart-home-v3",
        "z": "3cba2ff2f06b7801",
        "conf": "26512a1aa5ff7b8e",
        "device": "36483",
        "acknowledge": true,
        "name": "전등",
        "topic": "",
        "x": 170,
        "y": 1200,
        "wires": [
            [
                "a8f255b98c903317",
                "b263c8c853f3984e"
            ]
        ]
    },
    {
        "id": "437854ea261c8917",
        "type": "alexa-smart-home-v3",
        "z": "3cba2ff2f06b7801",
        "conf": "26512a1aa5ff7b8e",
        "device": "36482",
        "acknowledge": true,
        "name": "선풍기",
        "topic": "",
        "x": 190,
        "y": 1720,
        "wires": [
            [
                "a5465819959091fa",
                "75de0840e3cbb4c7"
            ]
        ]
    },
    {
        "id": "73131fe8fccf3c01",
        "type": "alexa-smart-home-v3",
        "z": "3cba2ff2f06b7801",
        "conf": "26512a1aa5ff7b8e",
        "device": "38747",
        "acknowledge": true,
        "name": "쿨러",
        "topic": "",
        "x": 150,
        "y": 2180,
        "wires": [
            [
                "1a69c6e34e065f89",
                "3f21effb45336c7b"
            ]
        ]
    },
    {
        "id": "21ac09aac4737a92",
        "type": "alexa-smart-home-v3",
        "z": "3cba2ff2f06b7801",
        "conf": "26512a1aa5ff7b8e",
        "device": "36481",
        "acknowledge": true,
        "name": "조명",
        "topic": "",
        "x": 150,
        "y": 2740,
        "wires": [
            [
                "ac482e24a3ffc700",
                "8fb578e40c69c8c8"
            ]
        ]
    },
    {
        "id": "f8b7873e644baad2",
        "type": "alexa-smart-home-v3",
        "z": "3cba2ff2f06b7801",
        "conf": "26512a1aa5ff7b8e",
        "device": "64296",
        "acknowledge": true,
        "name": "안방",
        "topic": "",
        "x": 150,
        "y": 3320,
        "wires": [
            [
                "2695bd564aa48419",
                "8c68858643993948"
            ]
        ]
    },
    {
        "id": "7007ce91b401f88e",
        "type": "alexa-smart-home-v3",
        "z": "3cba2ff2f06b7801",
        "conf": "26512a1aa5ff7b8e",
        "device": "64297",
        "acknowledge": true,
        "name": "주방",
        "topic": "",
        "x": 250,
        "y": 3900,
        "wires": [
            [
                "577412245bb7309c",
                "e2fee4cef83c622f"
            ]
        ]
    },
    {
        "id": "7e5fa1eeca890bc6",
        "type": "alexa-smart-home-v3-state",
        "z": "3cba2ff2f06b7801",
        "conf": "fca7cf2ca8e76f9c",
        "device": "31165",
        "name": "침대",
        "x": 110,
        "y": 260,
        "wires": []
    },
    {
        "id": "5c74b85034a12c74",
        "type": "alexa-smart-home-v3-state",
        "z": "3cba2ff2f06b7801",
        "conf": "fca7cf2ca8e76f9c",
        "device": "36484",
        "name": "창고",
        "x": 150,
        "y": 780,
        "wires": []
    },
    {
        "id": "a8f255b98c903317",
        "type": "alexa-smart-home-v3-state",
        "z": "3cba2ff2f06b7801",
        "conf": "fca7cf2ca8e76f9c",
        "device": "36483",
        "name": "전등",
        "x": 170,
        "y": 1300,
        "wires": []
    },
    {
        "id": "a5465819959091fa",
        "type": "alexa-smart-home-v3-state",
        "z": "3cba2ff2f06b7801",
        "conf": "fca7cf2ca8e76f9c",
        "device": "36482",
        "name": "선풍기",
        "x": 190,
        "y": 1820,
        "wires": []
    },
    {
        "id": "1a69c6e34e065f89",
        "type": "alexa-smart-home-v3-state",
        "z": "3cba2ff2f06b7801",
        "conf": "fca7cf2ca8e76f9c",
        "device": "38747",
        "name": "쿨러",
        "x": 150,
        "y": 2280,
        "wires": []
    },
    {
        "id": "ac482e24a3ffc700",
        "type": "alexa-smart-home-v3-state",
        "z": "3cba2ff2f06b7801",
        "conf": "fca7cf2ca8e76f9c",
        "device": "36481",
        "name": "조명",
        "x": 150,
        "y": 2840,
        "wires": []
    },
    {
        "id": "2695bd564aa48419",
        "type": "alexa-smart-home-v3-state",
        "z": "3cba2ff2f06b7801",
        "conf": "fca7cf2ca8e76f9c",
        "device": "64296",
        "name": "안방",
        "x": 150,
        "y": 3420,
        "wires": []
    },
    {
        "id": "577412245bb7309c",
        "type": "alexa-smart-home-v3-state",
        "z": "3cba2ff2f06b7801",
        "conf": "fca7cf2ca8e76f9c",
        "device": "64297",
        "name": "주방",
        "x": 250,
        "y": 4000,
        "wires": []
    },
    {
        "id": "04e05dec6601139d",
        "type": "ui_group",
        "name": "Group 2",
        "tab": "585316ec747035a8",
        "order": 2,
        "disp": true,
        "width": 6
    },
    {
        "id": "129593e0.7e203c",
        "type": "mqtt-broker",
        "name": "",
        "broker": "io.adafruit.com",
        "port": "1883",
        "clientid": "",
        "autoConnect": true,
        "usetls": false,
        "protocolVersion": "4",
        "keepalive": "60",
        "cleansession": true,
        "autoUnsubscribe": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthPayload": "",
        "birthMsg": {},
        "closeTopic": "",
        "closeQos": "0",
        "closePayload": "",
        "closeMsg": {},
        "willTopic": "",
        "willQos": "0",
        "willPayload": "",
        "willMsg": {},
        "userProps": "",
        "sessionExpiry": ""
    },
    {
        "id": "9ec6343d08d13227",
        "type": "ui_group",
        "name": "Group 1",
        "tab": "585316ec747035a8",
        "order": 1,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "419cffd938dae8e5",
        "type": "ui_group",
        "name": "Group 3",
        "tab": "585316ec747035a8",
        "order": 3,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "a99af2e5d0b2aaa4",
        "type": "ui_group",
        "name": "Group 4",
        "tab": "585316ec747035a8",
        "order": 4,
        "disp": true,
        "width": 6
    },
    {
        "id": "6232e555484312cf",
        "type": "ui_group",
        "name": "Group 5",
        "tab": "585316ec747035a8",
        "order": 5,
        "disp": true,
        "width": 6
    },
    {
        "id": "8698074accc216c0",
        "type": "ui_group",
        "name": "Group 6",
        "tab": "585316ec747035a8",
        "order": 6,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "1916d5b2b56dbea0",
        "type": "ui_group",
        "name": "Group 7",
        "tab": "585316ec747035a8",
        "order": 7,
        "disp": true,
        "width": 6
    },
    {
        "id": "8b177f8e1fccb595",
        "type": "ui_group",
        "name": "Group 8",
        "tab": "585316ec747035a8",
        "order": 8,
        "disp": true,
        "width": 6
    },
    {
        "id": "83297fde04603a3c",
        "type": "ui_group",
        "name": "Group 9",
        "tab": "585316ec747035a8",
        "order": 9,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "6e0c9516caac10ee",
        "type": "ui_group",
        "name": "Group 10",
        "tab": "585316ec747035a8",
        "order": 10,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "fc533e606813e4f3",
        "type": "modbus-client",
        "name": "",
        "clienttype": "serial",
        "bufferCommands": true,
        "stateLogEnabled": false,
        "queueLogEnabled": false,
        "failureLogEnabled": true,
        "tcpHost": "172.30.1.103",
        "tcpPort": "502",
        "tcpType": "DEFAULT",
        "serialPort": "COM10",
        "serialType": "RTU-BUFFERD",
        "serialBaudrate": "115200",
        "serialDatabits": "8",
        "serialStopbits": "1",
        "serialParity": "none",
        "serialConnectionDelay": "100",
        "serialAsciiResponseStartDelimiter": "0x3A",
        "unit_id": "1",
        "commandDelay": "1",
        "clientTimeout": "1000",
        "reconnectOnTimeout": true,
        "reconnectTimeout": "2000",
        "parallelUnitIdsAllowed": true,
        "showErrors": false,
        "showWarnings": true,
        "showLogs": true
    },
    {
        "id": "575ae74c180920c0",
        "type": "ui_group",
        "name": "Group 11",
        "tab": "585316ec747035a8",
        "order": 11,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "84524ce7456f45e7",
        "type": "ui_group",
        "name": "Group 12",
        "tab": "585316ec747035a8",
        "order": 12,
        "disp": true,
        "width": 6
    },
    {
        "id": "475553ad1acf571b",
        "type": "ui_group",
        "name": "Group 13",
        "tab": "585316ec747035a8",
        "order": 13,
        "disp": true,
        "width": 6
    },
    {
        "id": "2b01439b1fc48798",
        "type": "ui_group",
        "name": "Group 14",
        "tab": "585316ec747035a8",
        "order": 14,
        "disp": true,
        "width": 6
    },
    {
        "id": "219065a3c0bece05",
        "type": "ui_group",
        "name": "Group 15",
        "tab": "585316ec747035a8",
        "order": 15,
        "disp": true,
        "width": 6
    },
    {
        "id": "6765387b5254cee8",
        "type": "ui_group",
        "name": "Group 16",
        "tab": "585316ec747035a8",
        "order": 16,
        "disp": true,
        "width": 6
    },
    {
        "id": "71b1aa46b7ec8cfb",
        "type": "alexa-smart-home-v3-conf",
        "username": "kimsuho",
        "mqttserver": "mq-red.cb-net.co.uk",
        "webapiurl": "red.cb-net.co.uk",
        "contextName": "memory"
    },
    {
        "id": "26512a1aa5ff7b8e",
        "type": "alexa-smart-home-v3-conf",
        "username": "kimsuho",
        "mqttserver": "mq-red.cb-net.co.uk",
        "webapiurl": "red.cb-net.co.uk",
        "contextName": "memory"
    },
    {
        "id": "fca7cf2ca8e76f9c",
        "type": "alexa-smart-home-v3-conf",
        "username": "kimsuho",
        "mqttserver": "mq-red.cb-net.co.uk",
        "webapiurl": "red.cb-net.co.uk",
        "contextName": "memory"
    },
    {
        "id": "585316ec747035a8",
        "type": "ui_tab",
        "name": "Tab 1",
        "icon": "dashboard",
        "order": 1
    }


    
]ing flows (3).json…]()

       
