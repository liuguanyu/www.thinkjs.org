## 线上部署

### 使用 pm2 管理服务

pm2 是一款专业管理 Node.js 服务的模块，非常建议在线上使用。使用 pm2 需要以全局的方式安装，如： `sudo npm install -g pm2`。安装完成后，命令行下会有 pm2 命令。

创建项目时，会在项目目录下创建名为 `pm2.json` 的配置文件，内容类似如下：

```js
{
  "apps": [{
    "name": "demo",
    "script": "www/production.js",
    "cwd": "/Users/welefen/Develop/git/thinkjs/demo",
    "max_memory_restart": "1G",
    "autorestart": true,
    "node_args": [],
    "args": [],
    "env": {
      
    }
  }]
}
```

将 `cwd` 配置值改为线上真实的项目路径，然后在项目目录下使用下面的命令来启动/重启服务：

```sh
pm2 startOrGracefulReload pm2.json
```

pm2 详细的配置请见 <http://pm2.keymetrics.io/docs/usage/application-declaration/>。

### 使用 nginx 做反向代理

创建项目时，会在项目目录创建一个名为 `nginx.conf` 的 nginx 配置文件。配置文件内容类似如下：

```nginx
server {
    listen 80;
    server_name localhost;
    root /Users/welefen/Develop/git/thinkjs/demo/www;
    set $node_port 8360;

    index index.js index.html index.htm;
    if ( -f $request_filename/index.html ){
        rewrite (.*) $1/index.html break;
    }
    if ( !-f $request_filename ){
        rewrite (.*) /index.js;
    }
    location = /index.js {
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_pass http://127.0.0.1:$node_port$request_uri;
        proxy_redirect off;
    }
    location = /production.js {
        deny all;
    }

    location = /testing.js {
        deny all;
    }
    location ~ /static/ {
        etag         on;
        expires      max;
    }
}   

```

将 `server_name localhost` 里的 localhost 修改为对应的域名，将 `set $node_port 8360` 里的 8360 修改和项目里监听的端口一致。

修改完成后，将该配置文件拷贝到 nginx 的配置文件目录中，然后通过 `nginx -s reload` 命令 reload 配置，这样就可以通过域名访问了。

线上建议开启配置 `proxy_on`，这样就可以禁止直接通过 IP + 端口来访问。修改配置文件 `src/common/config/env/production.js`，如：

```js
export default {
  proxy_on: true
}
```

### 关闭静态资源处理的配置

为了方便开发，ThinkJS 是支持处理静态资源请求的。但代码部署在线上时，是用 nginx 来处理静态资源请求的，这时可以关闭 ThinkJS 里处理静态资源请求的功能来提高性能。

可以在配置文件 `src/common/config/env/production.js` 中加入如下的配置：

```js
export default {
  resource_on: false
}
```

### 使用 cluster

线上可以开启 cluster 功能达到利用多核 CPU 来提升性能，提高并发处理能力。

可以在配置文件 `src/common/config/env/production.js` 中加入如下的配置：

```js
export default {
  cluster_on: true
}
```