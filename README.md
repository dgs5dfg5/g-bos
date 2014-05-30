
### 1. 为移动端增加的表结构
##### 1.1 *T_APP_USER_LOGIN（用户登录信息）*
Name        | Type        | Nullable | Default | Comments
----------- | ----------- | -------- | ------- | --------
ID          | NUMBER      | N        |         | ID编号(PK)
USER_ID     | NUMBER      | N        |         | 用户ID
DEVICE_ID   | VARCHAR2(60)| N        |         | 设备ID
TOKEN       | VARCHAR2(80)| N        |         | 登录唯一标识(UK)
CREATE_TIME | DATE        | N        | SYSDATE | 创建时间

##### 1.2 T_APP_USER_CONFIG（应用配置信息）
Name        | Type        | Nullable | Default | Comments
----------- | ----------- | -------- | ------- | --------
ID          | NUMBER      | N        |         | ID编号(PK)
USER_ID     | NUMBER      | N        |         | 用户ID(UUK)
DEVICE_ID   | VARCHAR2(60)| N        |         | 设备ID(UUK)
CONFIG_NAME | VARCHAR2(50)| N        |         | 配置名(UUK)
CONFIG_VAL  | VARCHAR2(50)| N        |         | 配置值
UPDATE_TIME | DATE        | N        | SYSDATE | 更新时间

##### 1.3 T_APP_CAR_FOLLOW（用户关注车辆）
Name        | Type        | Nullable | Default | Comments
----------- | ----------- | -------- | ------- | --------
ID          | NUMBER      | N        |         | ID编号(PK)
USER_ID     | NUMBER      | N        |         | 用户ID(UUK)
CAR_ID      | VARCHAR2(50)| N        |         | 车辆ID(UUK)
CREATE_TIME | DATE        | N        | SYSDATE | 创建时间

#### 1.4 表设计说明
- PK:主键索引
- UK:唯一索引
- UUK:联合唯一索引

### 2. 接口设计

#### 2.1 接口相关定义说明
- 以下表示成功返回数据,其中data和token字段可有可无

	```
	{
  	  "ret": 0,
  	  "data": {...},
  	  "token": "xxx"
	}
	```

- 以下表示返回出错,msg为错误描述

	```
	{
  	  "ret": -1,
  	  "code": "xxx",
  	  "msg": "xxx"
	}
	```

- code定义说明：
	- 400:传入参数异常
	- 401:用户未验证
	- 500:服务端内部异常
	- 可自行定义更多code		
- 移动API的DOMAIN为mapi.g-bos.cn	

#### 2.2 *登录*
- 接口
	- URL:/login
	- Method:POST
	- Params:uname=xxx&pwd=xxx&dev_id=xxx或者token=xxx
	- 成功返回值:
	
		```
		{
	  	  "ret": 0,
		}
		或者
		{
	  	  "ret": 0,
  	  	  "token": "xxx"
		}
		```
- 主要逻辑
	- 通过手机端是否存在token文件来判断是否需要显示登录界面，若token文件存在，则系统加载时直接传递文件中token标识到服务端验证，不显示登录界面
	- 登录界面，输入用户名和密码，通过sql语句验证账号和密码；验证成功后记录**登录唯一标识**（用户ID＋设备号+过期时间的编码）到表T_APP_USER_LOGIN，并同时在手机端生成一个token文件（文件名自己定义）来记录该标识
	- 任何情况验证失败，则服务器端删除相应的token表记录，客户端清除token文件，并回到登录界面
	- 其他登录成功后的session设置等逻辑不在此说明，可参考web端代码
- 参考sql
	- 验证用户名和密码：SELECT * FROM T_B_USER WHERE LOGIN_NAME=? AND PWD=? AND USER_STATE=1
- web端参考代码
	- purview/controller/UserPurviewController:login
	- 移动端不需要考虑超级管理员账户

#### 2.3 接口验证说明
- 请求除登录外的任意接口，需要额外加上token验证参数，token参数在登录验证成功后由服务器端生成,有效时长默认为3天
- 服务器端使用自定义的加解密函数来生成token和解密收到的token
- 服务器端收到客户端发送的token参数后，先解密，然后验证是否过期，若过期就删除登录信息表中对应记录，销毁session，并返回401（客户端收到401就显示登录界面），若发现即将过期（即过期时间－当前时间<=5分钟），则任意接口在处理自身业务逻辑的同时，有义务再生成一个新的token，更新对应表记录的token字段，并放到接口返回值中，另外客户端收到服务器端任何响应，需要检测是否包含token，有则更新本地token文件

#### 2.4 注销
- 接口
	- URL:/logout
	- Method:GET
	- 除token外的参数:无
	- 成功返回值:
	
		```
		{
	  	  "ret": 0,
		}
		```
- 主要逻辑
	- 删除T_APP_USER_LOGIN中对应记录，销毁session，成功返回后手机端需要删除token文件

#### 2.5 *获取不规范驾驶行为统计数据*
- 接口
	- URL:/behavior/stat
	- Method:GET
	- 除token外的参数:start_date=xxxx-xx-xx&end_date=xxxx-xx-xx
	- 成功返回值:
	
		```
		{
  		  "ret": 0,
  		  "data": {
    		"osn": "xxx",
    		"orn": "xxx",
    		"sun": "xxx",
    		"sdn": "xxx",
    		"oin": "xxx",
    		"ian": "xxx",
    		"cn": "xxx",
    		"fden": "xxx",
    		"mden": "xxx"
  		  }
		}
		```
	- 说明：
		- osn:超速分数
		- orn:超转分数
		- sun:急加速分数
		- sdn:急减速分数
		- oin:过长怠速分数
		- ian:怠速空调分数
		- cn:空档滑行分数
		- fden:前门带速开门分数
		- mden:中门带速开门分数
- 主要逻辑
	- 调用ORG_REPORT.GetOrgAbnormalStats存储过程
- 参考web端代码
	- purview/controller/CompanyController:getOrgAbStatsList
	- purview/service/imp/CompanyDBService:getOrgAbStatsResultList
	
#### 2.6 *获取驾驶评分排名*
- 接口
	- URL:/behavior/score
	- Method:GET
	- 除token外的参数:start_date=xxxx-xx-xx&end_date=xxxx-xx-xx&sort=[asc|desc]&page=N&pagesize=N
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": [
		    {
		      "carno": "xxx",
		      "score": "xxx"
		    },
		    ...
		  ]
		}
		```
	- 说明：
		- carno:车牌号
		- score:驾驶评分
- 参考web端代码
	- drivescorechart/controller/DriveScoreChartController:getDriveScoreChartDetailDateGD
	- drivescorechart/service/imp/DriveScoreChartDBService:getDriveScoreChartDetailDate
	- drivescorechart/controller/DriveScoreChartController:getDriveScoreChartDetailDateDG
	- drivescorechart/service/imp/DriveScoreChartDBService:getDriveScoreChartDetailLessDate

#### 2.7 *获取不规范驾驶行为统计数据，反向评分排名*
- 接口
	- URL:/behavior/scoreandstat
	- Method:GET
	- 除token外的参数:start_date=xxxx-xx-xx&end_date=xxxx-xx-xx&pagesize=N
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": {
		    "osn": "xxx",
		    "orn": "xxx",
		    "sun": "xxx",
		    "sdn": "xxx",
		    "oin": "xxx",
		    "ian": "xxx",
		    "cn": "xxx",
		    "fden": "xxx",
		    "mden": "xxx",
		    "best": [
			  {
		      	"carno": "xxx",
		      	"score": "xxx"
		      },
			  ...
			]
		  }
		}
		```
	- 说明:参考接口2.5, 2.6
	
#### 2.8 *获取节油率排名*
- 接口
	- URL:/oil/save/rank
	- Method:GET
	- 除token外的参数:start_date=xxxx-xx-xx&end_date=xxxx-xx-xx&sort=[asc|desc]&page=N&pagesize=N
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": [
		    {
		      "carno": "xxx",
		      "oilsave": "xxx"
		    },
		    ...
		  ]
		}
		```
	- 说明：
		- carno:车牌号
		- oilsave:节油率%

- 参考web端代码
	- avageperoil/controller/AvagePerOilController:getAvagePerOilDetailDate
	- avageperoil/service/imp/AvagePerOilDBService:getgetAvagePerOilDetailDate
	
#### 2.9 *获取油耗统计数据*
- 接口
	- URL:/oil/stat
	- Method:GET
	- 除token外的参数:start_date=xxxx-xx-xx&end_date=xxxx-xx-xx
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": {
		    "mileage": "xxx",
		    "oilCost": "xxx",
		    "averageOilCost": "xxx",
		    "lowestOilCost": "xxx",
		    "highestOilCost": "xxx",
		    "highestSavingOilRank": "xxx",
		    "lowestSavingOilRank": "xxx"
		  }
		}
		```
	- 说明：
		- mileage:行驶里程Km
		- oilCost:行驶油耗L/100KM
		- averageOilCost:平均油耗L/100KM
		- lowestOilCost:最低油耗L/100KM
		- highestOilCost:最高油耗L/100KM
		- highestSavingOilRank:最高节油率%
		- lowestSavingOilRank:最低节油率%
- 主要逻辑
	- 查询表T_O_ORG_DAY_INFO
- 参考web端代码
	- homepage/controller/HomepageController:loadKanbanData
	- homepage/dao/imp/HomepageDAO:loadKanbanData
	
#### 2.10 *获取油耗统计数据和节油率排名*
- 接口
	- URL:/oil/rankandstat
	- Method:GET
	- 除token外的参数:pagesize=N&start_date=xxxx-xx-xx&end_date=xxxx-xx-xx
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": {
		    "statistics": {
		      "mileage": "xxx",
		      "oilCost": "xxx",
		      "averageOilCost": "xxx",
		      "lowestOilCost": "xxx",
		      "highestOilCost": "xxx",
		      "highestSavingOilRank": "xxx",
		      "lowestSavingOilRank": "xxx"
		    },
		    "rank": [
		      {
		        "carno": "xxx",
		        "oilsave": "xxx"
		      },
		      ...
		    ]
		  }
		}
		```
	- 说明：参考接口2.8, 2.9
	
#### 2.11 关注车辆
- 接口
	- URL:/car/follow
	- Method:POST
	- 除token外的参数:car_id=N
	- 成功返回值:
	
		```
		{
	  	  "ret": 0
		}
		```
- 主要逻辑
	- 插入一条记录到T_APP_CAR_FOLLOW

#### 2.12 取消关注车辆
- 接口
	- URL:/car/unfollow
	- Method:POST
	- 除token外的参数:car_id=N
	- 成功返回值:
	
		```
		{
	  	  "ret": 0
		}
		```
- 主要逻辑
	- 从T_APP_CAR_FOLLOW删除相应的记录
	
#### 2.13 *获取关注的车辆列表*
- 接口
	- URL:/user/carfollowlist
	- Method:GET
	- 除token外的参数:page=N&page_size=N
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": [
		    {
		      "carId": "xxx",
		      "carno": "xxx",
		      "status": "xxx",
		      "tick": "xxx",
		      "address": "xxx",
		      "gpsSpd": "xxx",
		      "speed": "xxx",
		      "rev": "xxx",
		      "mileage": "xxx"
		    },
		    ...
		  ]
		}
		```	
	- 说明：
		- carId:车辆ID
		- carno:车牌号
		- status:行车状态
		- tick:上报时间
		- address:地理位置
		- gpsSpd:GPS车速
		- speed:里程表车速
		- rev:转速
		- mileage:总里程
- 主要逻辑
	- 根据当前登录的用户ID获取关注的车辆ID列表
	- 根据车辆ID从T_B_CAR_INFO表查询车牌号
	- 根据车辆ID从T_B_CAR_TERMINAL表获取当前在使用的终端设备ID
	- 调用com.higer.memcached.monitor.MonitorGPS:batchLastGpsInfo方法获取车辆实时数据
	
#### 2.14 获取单车实时状态
- 接口
	- URL:/car/status
	- Method:GET
	- 除token外的参数:car_id=xxx
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": {
		    "carId": "xxx",
		    "carno": "xxx",
		    "status": "xxx",
		    "tick": "xxx",
		    "address": "xxx",
		    "gpsSpd": "xxx",
		    "speed": "xxx",
		    "rev": "xxx",
		    "mileage": "xxx",
		    "swithcInfo": "xxx",
		    "dayMileage": "xxx"
		  }
		}
		```
	- 说明：
		- carId:车辆ID
		- carno:车牌号
		- status:行车状态
		- tick:上报时间
		- address:地理位置
		- gpsSpd:GPS车速
		- speed:里程表车速
		- rev:转速
		- mileage:总里程
		- switchInfo:开关量(比如前门关，中门开之类)
		- dayMileage:当日里程
- 参考web端代码
	- com.higer.memcached.monitor.MonitorGPS:batchLastGpsInfo方法获取车辆实时数据
	- com.higer.web.monitor.controller.MapController:getIntradayMileage获得当日里程
	
#### 2.15 向单车发送消息
- 接口
	- URL:/car/sendmsg
	- Method:POST
	- 除token外的参数:car_id=xxx&content=xxx&type=N
	- 成功返回值:
	
		```
		{
	  	  "ret": 0
		}
		```
	- 说明:
		- type:值为0或1，1表示语音播报
- 参考web端代码
	- com.higer.web.sendinfomation.controller.SendInfomationController:sendInfomation
	
#### 2.16 获取最后一次发送到某单车的消息
- 接口
	- URL:/car/lastmsg
	- Method:GET
	- 除token外的参数:car_id=xxx
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": {
		    "carno": "xxx",
		    "sendTime": "xxx",
		    "content": "xxx",
		    "state": "xxx"
		  }
		}
		```
	- 说明:
		- carno:车牌号
		- sendTime:发送时间
		- content:发送内容
		- state:发送状态
- 参考web端代码
	- com.higer.web.sendinfomation.controller.SendInfomationController:sendInfomationList
	- com.higer.web.sendinfomation.service.imp.SendInfomationDBService:findInfoManageListByUserId

#### 2.17 单车拍照
- 接口
	- URL:/car/takephoto
	- Method:GET
	- 除token外的参数:car_id=xxx
	- 成功返回值:
	
		```
		{
	  	  "ret": 0
		}
		```
- 主要逻辑：
	- 这里只调用拍照指令即可，不去等待。后续逻辑为：照片成功返回后，会存入服务器文件，并且在数据库中存放文件路径，需要把车辆ID，车牌号，照片路径等信息推送到手机端	
- 参考web端代码
	- com.higer.web.jcc.task.PhotoTask:doPhoto

#### 2.18 获取单车历史轨迹和告警点
- 接口
	- URL:/car/historytack
	- Method:POST
	- 除token外的参数:car_id=xxx&start_time=xxx&end_time=xxx
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": {
		    "history": [
		      {
		        "lat": "xxx",
		        "lng": "xxx",
		        "direct": "xxx",
		        "speed": "xxx",
		        "rev": "xxx",
		        "tick": "xxx"
		      },
		      ...
		    ],
		    "alarm": [
		      {
		        "s_t": "xxx",
		        "e_t": "xxx",
		        "event": "xxx",
		        "topV": "xxx",
		        "lat": "xxx",
		        "lng": "xxx"
		      },
		      ...
		    ]
		  }
		}
		```
	- 说明:
		- lat:纬度
		- lng:经度
		- direct:方向
		- speed:车速
		- rev:转速
		- tick:上报时间
		- s_t:告警开始时间
		- e_t:告警结束时间
		- event:告警事件名称
		- topV:最大值
- 参考web端代码
	- com.higer.web.monitor.controller.MapController:getGPSInfoByTime
	- com.higer.web.monitor.controller.MapController:getBadDriverEvent
	- com.higer.web.monitor.controller.MapController:getOpenDoorEvent
	
#### 2.19 获取车辆实时位置信息
- 接口
	- URL:/car/position
	- Method:POST
	- 除token外的参数:car_ids=xxx,xxx,xxx,xxx,...，当car_id为0时，表示获取该账号下所有车辆实时位置信息
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": {
		    "total": "xxx",
		    "online": {
		      "on_cnt": "xxx",
		      "off_cnt": "xxx",
		      "cars": [
		        {
		          "car_id": "xxx",
		          "car_no": "xxx",
		          "lat": "xxx",
		          "lng": "xxx",
		          "direct": "xxx",
		          "speed": "xxx",
		          "rev": "xxx"
		        },
		        ...
		      ]
		    },
		    "outline": {
		      "cnt": "xxx",
		      "cars": [
		        {
		          "car_id": "xxx",
		          "car_no": "xxx",
		          "lat": "xxx",
		          "lng": "xxx",
		          "direct": "xxx"
		        },
		        ...
		      ]
		    }
		  }
		}
		```
	- 说明:
		- lat:纬度
		- lng:经度
		- direct:方向
		- speed:车速
		- rev:转速
		- on_cnt:行车在线车辆数
		- off_cnt:停车在线车辆数
		- cnt:离线车辆数
		- total:总车辆数
- 参考web端代码
	- com.higer.memcached.monitor.MonitorGPS:batchLastGpsInfo方法获取车辆实时数据

#### 2.20 获取告警提示聚合数据
- 接口
	- URL:/alarm/stat
	- Method:GET
	- 除token外的参数:month=xxxx-xx&type=[1|2|3|4],1表示超速,2表示超转,3表示急加速,4表示急减速
	- 成功返回值:
	
		```
		{
  		  "ret": 0,
  		  "data": {
    		"daily": [
      		  {
        		"day": "xxx",
        		"day_num": "xxx"
      		  },
      		  ...
    		],
    		"rank": [
      		  {
        		"car_id": "xxx",
        		"carno": "xxx",
        		"num": "xxx"
      		  },
      		  ...
    		]
  		  }
		}
		```
	- 说明:
		- day:具体日期，精确到日
		- day_num:某日总违章次数
		- car_id:车辆ID
		- carno:车牌号
		- num:单车当月总违章次数
- web端部分参考代码
	- com.higer.web.speedstatis.controller.SpeedingController:getSpeedingDateS
	- com.higer.web.speedstatis.controller.SpeedingController:getSpeedingListForDay
	
#### 2.21 获取告警提示排行榜
- 接口
	- URL:/alarm/rank
	- Method:GET
	- 除token外的参数:page=N&pagesize=N&month=xxxx-xx&type=[1|2|3|4],1表示超速,2表示超转,3表示急加速,4表示急减速
	- 成功返回值:
	
		```
		{
  		  "ret": 0,
  		  "data": [
    		{
      		  "car_id": "xxx",
      		  "carno": "xxx",
      		  "num": "xxx"
    		},
    		...
  		  ]
		}
		```
	- 说明:
		- car_id:车辆ID
		- carno:车牌号
		- num:单车当月总违章次数
- web端部分参考代码
	- com.higer.web.speedstatis.controller.SpeedingController:getSpeedAllCarList
	
#### 2.22 *获取故障检测聚合数据*
- 接口
	- URL:/breakdown/stat
	- Method:GET
	- 除token外的参数:pagesize=N&start_date=xxxx-xx-xx&end_date=xxxx-xx-xx
	- 成功返回值:
	
		```
		{
  		  "ret": 0,
  		  "data": {
    		"stat": {
      		  "serious": "xxx",
      		  "generl": "xxx",
      		  "no_effect": "xxx"
    		},
    	  	"serious_list": [
      		  {
        	    "car_id": "xxx",
        	    "carno": "xxx",
        	    "num": "xxx",
				"descs": [
			  	  {
				  	"desc": "xxx",
				  	"time": "xxx"
				  },
				  ...
			  	]
      		  },
      		  ...
    	  	]
  		  }
		}		
		```
	- 说明:
		- serious:严重故障次数
		- generl:一般故障次数
		- no_effect:无影响（轻微故障）次数
		- car_id:车辆ID
		- carno:车牌号
		- num:故障数量
		- desc:故障描述
		- time:故障发生时间
- web端部分参考sql
	- sqlmaps/faultreport_SqlMap.xml(selectFaultReportCount,selectFaultReportByCarId)
	
#### 2.23 *获取故障检测车辆列表*
- 接口
	- URL:/breakdown/list
	- Method:GET
	- 除token外的参数:page=N&pagesize=N&start_date=xxxx-xx-xx&end_date=xxxx-xx-xx&type=[1|2|3],1表示严重,2表示一般,3表示无影响
	- 成功返回值:
	
		```
		{
  		  "ret": 0,
  		  "data": [
    		{
      		  "car_id": "xxx",
      		  "carno": "xxx",
      		  "num": "xxx",
			  "descs": [
			  	{
				  "desc": "xxx",
				  "time": "xxx"
				},
				...
			  ]
    		},
    		...
  		  ]
		}
		```
	- 说明:
		- car_id:车辆ID
		- carno:车牌号
		- num:故障数量
		- desc:故障描述
		- time:故障发生时间
		
#### 2.24 获取app配置信息
- 接口
	- URL:/userconfig/get
	- Method:GET
	- 除token外的参数:无
	- 成功返回值:
	
		```
		{
  		  "ret": 0,
  		  "data": [
    		{
      		  "name": "xxx",
      		  "val": "xxx"
    		},
    		...
  		  ]
		}
		```
	- 说明:
		- name:配置名
		- val:配置值
		
#### 2.25 设置app配置信息
- 接口
	- URL:/userconfig/set
	- Method:POST
	- 除token外的参数:name=xxx,xxx,...&val=xxx,xxx,...
	- 成功返回值:
	
		```
		{
	  	  "ret": 0
		}
		```
		
#### 2.26 *根据车牌号获取车辆实时信息*
- 接口
	- URL:/car/position/search_by_no
	- Method:POST
	- 除token外的参数:kw=xxx
	- 成功返回值:
	
		```
		{
		  "ret": 0,
		  "data": {
		    "total": "xxx",
		    "online": {
		      "on_cnt": "xxx",
		      "off_cnt": "xxx",
		      "cars": [
		        {
		          "car_id": "xxx",
		          "car_no": "xxx",
		          "lat": "xxx",
		          "lng": "xxx",
		          "direct": "xxx",
		          "speed": "xxx",
		          "rev": "xxx"
		        },
		        ...
		      ]
		    },
		    "outline": {
		      "cnt": "xxx",
		      "cars": [
		        {
		          "car_id": "xxx",
		          "car_no": "xxx",
		          "lat": "xxx",
		          "lng": "xxx",
		          "direct": "xxx"
		        },
		        ...
		      ]
		    }
		  }
		}
		```
	- 说明: 参考2.19
		
