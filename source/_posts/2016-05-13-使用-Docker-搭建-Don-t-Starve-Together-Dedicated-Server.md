---
title: 使用 Docker 搭建 Don't Starve Together Dedicated Server
date: 2016-05-13 06:36:21
categories:
- '应用实践'
- 'Self-Hosting'
tags:
- 'Docker'
- 'Linux'
---
了解过docker之后我决定练一下手（本来是因为有些人想玩DST所以才决定的），于是就拿饥荒联机服务器(以下简称dst服务器)来做练手作

Google到了Docker Hub里面有现成的DST docker镜像，感谢jamesits。介绍地址:[Docker Hub ： DST Dedicated Server](https://hub.docker.com/r/jamesits/don-t-starve-together-dedicated-server/)。

安装docker的步骤网上很多，我就不介绍了。安装完docker之后还得安装docker-compose。我的DST服务器数据放在`/srv/dst/`，以下例子都用这个路径。

镜像作者使用Docker Compose，所以只要在打算让dst服务器保存数据的目录下新建文件并粘贴以下代码

	overworld-server:
	  image: jamesits/don-t-starve-together-dedicated-server:latest
	  restart: always
	  ports:
	  - 10999:10999/udp
	  - 8766:8766/udp
	  - 27016:27016/udp
	  volumes:
	  - ./server_config:/data/DoNotStarveTogether

保存成名为docker-compose.yml的文件，在`/srv/dst/`下以root权限启动`docker-compose up`，即可自动下载并启动饥荒服务器。但是这样子服务器并不会真正启动起来，还需要写一下配置才能够跑起来。

`Ctrl+C`停掉服务器，会发现自动生成好的配置文件目录`/srv/dst/server_config/`。进入`/srv/dst/server_config/Cluster_1/`，新建一个`cluster.ini`文件，并在里面写配置：

	[NETWORK]
	
	cluster_name = 服务器的名称
	cluster_description = 服务器描述
	cluster_intention = 服务器的类型 [cooperative | social | competitive | madness]
	cluster_password = 密码，可选
	
	server_port = 10999 服务器的端口，建议不要修改
	max_players = 20 最大玩家数量，1-64
	pvp = false 是否允许pvp，玩家对打
	game_mode = survival 游戏模式 [endless | survival | wilderness]
	tick_rate = 30 服务器的帧率，越高越fantasy不过对服务器和带宽要求高
	connection_timeout = 3000
	server_save_slot = 1 服务器存档读取，一般不用改
	pause_when_empty = true 这个虽然是对应“当服务器没人时停止服务器”但是并没有生效
	dedicated_lan_server = true 是否允许局域网联机
	
写好配置之后，要获取服务端的令牌。进入DST客户端之后，点Play登陆，然后点右下角的Account，页面里找到生成Token的地方（右侧的名字可以随便写），然后把生成的Token写进`/srv/dst/server_config/Cluster_1/cluster_token.txt`里保存。

如果不需要mod的话，到这里就可以回到`/srv/dst`里`docker-compose up`了，在后面加`-d`可以让其在后台运行。

如果要加mod，那么需要编辑`/srv/dst/server_config/`里面的`dedicated_server_mods_setup.lua`文件。在里面一行添加一个mod。

	ServerModSetup("mod1-id")
	ServerModSetup("mod2-id")
	ServerModSetup("mod3-id")

Mod ID可以在创意工坊里面查到。进入mod页面后拷贝一下链接。找个地方粘贴一下

	http://steamcommunity.com/sharedfiles/filedetails/?id=681368916
	
id后面跟着的就是了。

写好之后保存。现在还没能启用，如果启动服务器的话，只会下载列表里的Mod而不会启动。要启动的话有两种方式，一种是强制启动，但是这种方式一般只在开发mod的时候使用，不推荐。另外一种是常规的启动方式，能给mod写配置（如何写配置请参考下面给出的链接）。

进入`/srv/dst/server_config/Cluster_1/Master/`，新建一个`modoverrides.lua`，在里面写

	return{
	 ["workshop-id1"] = { enabled = true },
	 ["workshop-id2"] = { enabled = true },
	 ["workshop-id3"] = { enabled = true }
	}

id替换成要启动的mod的id，保存后建议给文件添加可执行权限

	chmod +x modoverrides.lua

现在启动服务器后，就如你所愿啦。

黑名单，白名单以及管理员名单列表放在

更详尽的服务器设置和Mod设置请参考 [Guides/Don’t Starve Together Dedicated Servers](http://dont-starve-game.wikia.com/wiki/Guides/Don’t_Starve_Together_Dedicated_Servers) 和 [Dedicated Server w/Mods [Problem]](http://forums.kleientertainment.com/topic/59506-dedicated-server-wmods-problem/)