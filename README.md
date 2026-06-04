# ArtFetchDeploy

ArtFetchDeploy 是 `/Users/wyn/code/ArtFetch` 的全新本地 DevOps 项目，用于从本地完成 ArtFetch 的预检、构建、离线制品打包、上传、部署、健康检查、回滚和审计。

当前需求文档：

- [ArtFetch DevOps 项目需求文档](docs/requirements-artfetch-devops.md)

核心边界：

- ArtFetch 仓库只作为源码输入，不依赖、不调用 ArtFetch 下的 GitHub Actions workflow 或旧部署脚本。
- 第一版使用 ArtFetchDeploy 仓库内脚本作为唯一自动化事实来源，Jenkins 以后最多作为调用脚本的外壳。
- 本地构建完整离线发布包，并通过 SSH/SCP 部署到服务器。
- 生产服务器只消费 ArtFetchDeploy 生成的离线制品，不执行源码构建或 registry 拉取。
- 本仓库不得保存生产密钥、数据库 dump、Cookie、对象存储密钥、SSH 私钥或 GitHub token。
