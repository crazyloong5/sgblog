# 三更博客系统

基于Springboot + Vue3 开发的前后端分离博客



## 本地运行

1. MySQL版本为`8.0.27`，npm版本为`9.4.0`，node版本为`v16.18.0`
2. SQL 文件位于根目录下的`sql`，将其中的数据导入到本地数据库中
3. 修改后端配置文件中的数据库等连接信息，项目中使用到的关于阿里云、腾讯云功能和第三方授权登录等需要自行开通（这里使用七牛云OSS）
4. 需要启动Redis服务
5. 前端分别`npm install`、`npm run dev`
6. 项目启动后，管理员账号需要自行创建，`user`表中`type`设置为`1`

 

## 技术介绍

**前端：** Vue3 + Vue Router + Axios + Element Plus + Echarts

**后端：** SpringBoot + SpringSecurity + Mysql + Redis + Swagger2 + MyBatisPlus + EasyExcel



## 开发环境

| 开发工具              | 说明               |
| --------------------- | ------------------ |
| IDEA                  | Java 开发工具 IDE  |
| VSCode                | Vue 开发工具 IDE   |
| sqlyog                | MySQL 远程连接工具 |
| Redis Desktop Manager | Redis 远程连接工具 |
| Xshell                | Linux 远程连接工具 |
| Xftp                  | Linux 文件上传工具 |



## 项目展示

前台：

![9c07e1dd84654f8557687b0d6c8b95ed](https://picbed.sinarcsinx.fun/markdown/202412222137630.png)

![a7386c5b79a5acea413dd8dd0ac39ca](https://picbed.sinarcsinx.fun/markdown/202412222141180.png)

后台：

![80f85ebddfaa40ab8d44e7908dd10381](https://picbed.sinarcsinx.fun/markdown/202412222136018.png)

![63d1c927989d2af9343b634fa055bf2](https://picbed.sinarcsinx.fun/markdown/202412222139872.png)

![7a8c0e3e21eba8a4a48b26c605319f7](https://picbed.sinarcsinx.fun/markdown/202412222144815.png)