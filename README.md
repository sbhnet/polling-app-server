# polling-app-server
- 使用动态jenkins slave pipline 方式构建polling 服务端镜像，需要建立持久化存储的mysql
- 使用helm进行部署，需要先建立chart，然后手动测试是否成功，升级时使用--set 来替换镜像tag，完成升级部署
