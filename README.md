
# OneScape Recipe Framework

## Overview
* OneScape 의 여러 서비스들을 한번에 실행 하거나 일정 이벤트에 따라 실행
* Action 은 실행할 서비스들을 정의
* Trigger는 Action을 실행할 조건을 설정
* Trigger + Action => Recipe

## Trigger
* 액션을 실행할 조건들을 명시  

### Trigger Schema

    {
    "$schema": "http://json-schema.org/draft-04/schema#",
    "description": "OneScape Rule Schema",
    "type": "object",   
    "required":["owner", "rule", "dest", "action", "waitMin", "again"],
    
	"properties":{
		"owner":{"type" : "string"},
		"rule":{"type" : "string"},
		"dest":{"type" : "string"},
		"again":{"type" : "boolean"},
		"action":{ "type" : "integer", "minimum" : 1},
		"waitMin":{ "type" : "integer", "minimum" : 1}
   	    }  
    } 

### Parameters
Param | Desc
------|------------------
owner | owner of trigger
rule  | 실행 할 룰 설정
dest  | 룰 이 true 일때 실행할 목적지. 예를들어 객실 번호, 디바이스 아이디 등
again | trigger의 재사용 여부. 1 = 재사용, 0 = 1회만 사용
action| 실행할  action 번호
waitMin| 액션 실행후 다시 실행될 때 까지 지연 시간 설정. 이설정이 필요한 이유는, again 을 사용할 때 특정 조건이 맞으면 반복해서 실행되는 것을 방지하고자 함. 즉, keytag이 1이면 커튼을  오픈. 이라고 할때 keytag이 계속 1 상태일때 커튼을 반복해서 오픈하는것을 방지

__waitMin__

	function isTriggerAvail($trigger)
	{
		$last = $lastActionTime;	
		$last->add(new \DateInterval('PT' . $waitMin . 'M')); // add minutes
		$cur = new \DateTime('NOW');
		
		if($cur < $last) 
			return false;

		return true;
	}

### API
API | Add or Update Trigger
----|-------------------------------
Endpoint | POST  https://one-scape.com/recipe/trigger
Authentication | Basic Auth

**Param**

	{
		"id": 1, 
		"owner" : "hotelgallery",
		"rule"  : "declare(sc) and sc-keytag = 1",
		"dest" : "all",
		"again" : false,
		"action" : 1,
		"waitMin" : 5	
	}  
> id 가 없으면 add , 있으면 update


## Rule
* 룰은 일련의 조건을 설정하여 평가. true or false 로 액션 실행
* Rule Engine은 Binder를 호출하여 rule script를 평가

__ex)키택이 꽂혀있고 현재 시각이 7시면__

> declare(sc) and sc-keytag = 1 and declare(dt) and dt-time = ‘07:00’


## OneScape Binders
* Binder는 OneScape Framwork 규격에 따라 생성된 서비스 객체
* StayCore, Hue, Sensibo, Weather 등 


### DateTime Binder
* 시간 속성에 따라 조건을 설정
* 특정 시각, 요일, 월 등 

**1월 1일 오후 1시 이면**
> declare(dt) and dt-month = 1 and dt-day = 1 and dt-time = ‘13:00’ 

**매월 1일 오후 1시 이면**
> declare(dt) and dt-day = 1 and dt-time = ‘13:00’ 

**오늘이 일요일 또는 월요일 이면**
> declare(dt) and dt-weekday in [‘sun’, ‘mon’]

**이번달이 6,7,8월 중에 속하는지**
> declare(dt) and dt-month in [6,7,8]

Context | Desc | Examples
--------|------|------------
dt   |   Declare |  declare(dt)
dt-time | current time   | ‘07:00’, ‘08:21’
dt-min | current minute | 23, 1, 3
dt-weekend | 요일 | 'sun', 'mon', 'tue', 'wed','thu','fri', 'sat'. 
dt-day | 일 | 5일이면. dt-day = 5
dt-month | 월 | 1월 이면 dt-month = 1

__오늘이 일요일 또는 월요일 이면__
> dt-weekday in [‘sun’, ‘mon’]

__오늘이 월요일 이면__
> dt-weekday = ‘sun’

__오전 7시가 넘었으면__
> dt_time > '07:00'

__오전 10시 와 12시 사이__
> dt_time > '10:00' and dt_time < '12:00'

### iStay® StayCore™ Binder
* iStay® 객실용 Binder
* OneScape™ GRMS(Guest Room Management Service)의 Rule 기반 서비스

__키택이 꽂혀 있고 현재 방의 온도가 25도 이상이면__
> declare(sc) and sc-keytag = 1 and sc-cur-temp > ‘25.0’

__방 청소 요청 상태이면__
> declare(sc) and sc-mur = 1

Context | Desc | Examples
--------|------|----------
sc  | Binder Declare |  declare(sc)
sc-keytag | keytag | sc-keytag = 1  ( or 0)
sc-dnd | do not disturb | sc-dnd = 1 (or 0)
sc-mur | makeup room | sc-mur = 1 (or 0)
sc-emr | emergency state | sc-emr = 1(or 0)
sc-user-temp | current user temperature | sc-user-temp > ‘30.0’
sc-cur-temp | current room temperature | sc-cur-temp > ‘24.5’
sc-light1 (~ 16) | light status | sc-light1 = 0 (or 1)
sc-curtain1 (~ 4) | curtain status | sc-curtain1 = 1, __1:open , 2: close 3:stop__
sc-fan_power | fan power status | sc-fan_power = 1 (or 0)
sc-fan_manual | Fan 제어모드(자동/수동) 상태 | sc-fan_manual = 1 (or 0)
sc-fan_speed | Fan Speed | sc-fan-speed = 1, __0: Stop, 1: Low, 2:Mid, 3:High__
sc-fan-valve | Fan value staus | sc-fan-valve = 1, __0: 닫힘 1:열림__

### Rule examples

__방의 온도가 25도 이상이고 6,7,8,9 월 이면__
> declare(sc) and sc-cur-temp > 25 and declare(dt) and dt-month in [6,7,8,9]

__방의 온도가 25도 이하이고 7,8월 이고 22:00 이후 이면__
> declare(sc) and sc-cur-temp > 25 and declare(dt) and dt-month in [7,8] and dt-time > “22:00”

## Action
* Action 은 미리 설정된 일련의 서비스의 집합

### Action Schema
    {
        "$schema": "http://json-schema.org/draft-04/schema#",
        "description": "OneScape Action Schema",
        "type": "object",   
        "required":["actions"],
    
	"properties":{				

        "actions":{
        	"type":"array",
        	"items":{
	        	"oneOf":[
	        		{"$ref":"#/definitions/interval"},
	        		{"$ref":"#/definitions/hue"},
	        	    {"$ref":"#/definitions/tv"},
	        	    {"$ref":"#/definitions/sc"}
	        	]
        	}
        }
    },
    "definitions":{
    	"interval":{
    		"properties":{
    			"type": { "enum": [ "interval" ] },
    			"sleep" : {
    				"type" : "integer",
    				"minimum" : 1,
    				"maximum" : 86400
    			}
    		},
    		"required": ["type", "sleep"]
    	},    
    	"hue":{
    		"properties":{
    			"type": { "enum": [ "hue" ] },
    			"on": { "enum": [ true, false ] },   			
     			"hue" : {
    				"type" : "integer",
    				"minimum" : 0,
    				"maximum" : 65535
    			},  
      			"sat" : {
    				"type" : "integer",
    				"minimum" : 0,
    				"maximum" : 254
    			},   			
      			"bri" : {
    				"type" : "integer",
    				"minimum" : 1,
    				"maximum" : 254
    			},   
      			"ct" : {
    				"type" : "integer",
    				"minimum" : 153,
    				"maximum" : 500
    			},     			  			 			   			 			
    			"group" : {
    				"type":"integer",
     				"minmum" : 0,
    				"maximum" : 1000 
    			}     		
    		},
    		"required": ["type"]
    	},
     	"tv":{
    		"properties":{
    			"type": { "enum": [ "tv" ] },
    			"control": { "enum": [ "on" ] },
    			"channel" : "integer",
    			"index" : {
    				"type":"integer",
     				"minmum" : 0,
    				"maximum" : 3 
    			}   			
    		},
    		"required": ["type"]
    	},   	
     	"sc":{
    		"properties":{
    			"type": { "enum": [ "sc" ] },
    			"item": { "enum": [ "light", "lightAll", "temp","dnd","curtain", "curtainAll","fcuPower","fcuSpeed","door" ] },
				"param1":"string",
				"param2":"string"  			
    		},
    		"required": ["type","item","param1"]
    	}  
    	   		
    }
     
} 


### Action Objects

### StayCore Object

Property | Value(s) | Type | Min | Max | Required
---------|----------|------|-----|-----|------------
type     | sc       | string | | | yes
item     | light, lightAll, temp, dnd, curtain, curtainAll, fcuPower, fcuSpeed, door | string | | | yes
param1   | [iStay Api](https://docs.google.com/presentation/d/1Y4rDaI_K76YXsHT-XA0BLKiTKz-7fQFPYMs32Ivug2M/edit#slide=id.g229fbcd557_1_21) | string | | | yes
param1   | same as above | string | | |

### Hue Object

Property | Value(s) | Type | Min | Max | Required
---------|----------|------|-----|-----|------------
type     | hue       | string | | | yes
on       | turn on or off       | boolean | | |
hue       | light's hue value      | integer |  0| 65535|
sat       | light's saturation value      | integer |  0| 254|
bri       | light's brightness value      | integer |  1| 254|
ct       | light's color temperature value      | integer |  153| 500|
group       | hue group id     | integer |  0| 1000|

### TV Object
Property | Value(s) | Type | Min | Max | Required
---------|----------|------|-----|-----|------------
type     | tv       | string | | | yes
control     | turn on / off       | enum {"on", "off"} | | |
channel     | tv channel      | integer | | |
index     | TV's index      | integer | 0 | 4 |


### Interval Object

Property | Value(s) | Type | Min | Max | Required
---------|----------|------|-----|-----|------------
type     | interval       | string | | | yes
sleep     | minutes to sleep       | integer | 1 | 86400|
channel     | tv channel      | integer | | |

### Examples

	{
	"actions": [
	{
	  "type": "hue",
	  "hue": 40000,
	  "sat": 12,
	  "bri": 255,
	  "on": true,
	  "group": 0
	},
	{
	  "type": "interval",
	  "sleep": 5
	},
	{
	  "type": "tv",
	  "channel": 24
	}
	]}
