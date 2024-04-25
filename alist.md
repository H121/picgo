在shell命令行中输入以下命令
```shell
docker pull xhofe/alist
docker run -d --restart=always -v /etc/alist:/opt/alist/data -p 5244:5244 --network=leanote_nw --name="alist" xhofe/alist:latest
docker exec -it alist bash
cd data/
vi config.json
# 修改site_url 以便nginx进行配置
"site_url": "/alist",
docker restart alist
ufw allow 5244/tcp
```
修改nginx
```shell
cd ~/nginx
vim nginx.conf
# 增加以下内容
 upstream alist{
        server 10.0.8.10:5244;
    }

location /alist {
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header Host $http_host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header Range $http_range;
          proxy_set_header If-Range $http_if_range;
          proxy_redirect off;
          proxy_pass http://alist/alist;
          # the max size of file to upload
          client_max_body_size 20000m;
        }
docker restart nginx
```
!!! 记得看端口是否放开【防火墙】
挂载网盘
先打开管理->存储
同时看官方文档
注意要启用web代理，选择使用代理地址


AList玩法
```shell
# 忘记密码的查询
docker exec -it alist ./alist admin
# 取消二次认证
docker exec -it alist ./alist cancel2fa
```

下载RaiDrive[挂载到本地的工具]
选择NAS WebDAV 输入对应的账号密码，ip+端口+/alist/dav

参看连接
https://www.bilibili.com/video/BV15A41127bQ/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=0e40790a7fcabf3d036dedebb701912c
https://www.bilibili.com/video/BV1sj411w7Wm/?spm_id_from=333.788.top_right_bar_window_history.content.click
https://399s.com/7565.html
https://www.mfpud.com/topics/10886/