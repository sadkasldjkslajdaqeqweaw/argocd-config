master 分支：仅维护基础模板和环境目录结构，禁止直接提交部署变更。
环境分支：如 release/prod（生产）、release/test（测试），作为 ArgoCD 监控的目标分支，仅通过 PR 从临时部署分支合并。
临时部署分支：如 deploy/mservice-${BUILD_NUMBER}，由 Jenkins 动态创建，用于存放本次部署的配置变更，通过 PR 合并到环境分支。