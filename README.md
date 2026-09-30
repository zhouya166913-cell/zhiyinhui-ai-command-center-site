# 智隐会 · AI经营指挥中心

大屏展示网页。程序、样式和案例内容均内嵌在 `index.html`，适用于静态网站托管和浏览器离线打开。

此仓库仅包含展示网页；开发源码保存在单独的私有仓库。网页中的客户实践按项目资料和历史反馈展示，公开标杆保留原文来源。运行指标为本地预置场景，未接入企业生产后台。

页面无需登录，不上传或保存诊断表单内容。

## 部署

正式站点使用 Jenkins 的 **Pipeline script from SCM** 模式读取仓库根目录的 `Jenkinsfile`。

- 发布分支：`codex/site-publish`
- 流水线文件：`Jenkinsfile`
- 目标域名：`command.zhiyinhui.top`
- 目标目录：`/var/www/zhiyinhui-command-center`

流水线会校验静态文件，按 Jenkins 构建号创建独立版本目录，通过原子切换 `current` 发布，并在健康检查失败时恢复上一版本。Nginx 和 HTTPS 属于服务器的一次性基础设施配置，不由流水线修改。
