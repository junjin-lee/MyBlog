## Centos下载与安装
先安装mysql与git（git版本大于2.1）
```
wget -O gitea https://dl.gitea.io/gitea/1.3.2/gitea-1.3.2-linux-amd64 

chmod +x gitea
```
启动gitea 
```
nohup ./gitea web &
```

服务器配置文件路径 
/root/custom/conf/app.ini

## gitea备份与恢复
先转到git用户的权限: su git. 再Gitea目录运行 ./gitea dump。一般会显示类似如下的输出 
```
2016/12/27 22:32:09 Creating tmp work dir: /tmp/gitea-dump-417443001
2016/12/27 22:32:09 Dumping local repositories.../home/git/gitea-repositories
2016/12/27 22:32:22 Dumping database...
2016/12/27 22:32:22 Packing dump files...
2016/12/27 22:32:34 Removing tmp work dir: /tmp/gitea-dump-417443001
2016/12/27 22:32:34 Finish dumping in file gitea-dump-1482906742.zip
```
Restore Command
```
unzip gitea-dump-1482906742.zip
cd gitea-dump-1482906742
unzip gitea-repo.zip
mv gitea-repo/* /var/lib/gitea/repositories/
mysql -u$USER -p$PASS $DATABASE <gitea-db.sql
```
