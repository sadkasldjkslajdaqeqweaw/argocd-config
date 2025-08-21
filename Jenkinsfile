pipeline {
    agent any
    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: '代码分支（如master、dev）')
        string(name: 'ENV', defaultValue: 'prod', description: '部署环境（prod/test）')
        string(name: 'REPLICAS', defaultValue: '3', description: '部署副本数')
        string(name: 'CONFIG_REPO_BRANCH', defaultValue: "release/${ENV}", description: '配置仓库目标环境分支')
    }

    environment {
        // CI 相关配置
        GITLAB_PROJECT_URL = "https://git.carizon.work/bpm/carizon-bpm.git"
        PROJECT_NAME = "${env.JOB_NAME.tokenize('/').last()}" // 从Jenkins工程名提取应用名
        TIMESTAMP = "${java.time.LocalDateTime.now().format(java.time.format.DateTimeFormatter.ofPattern('yyyyMMddHHmmss'))}"
        DOCKER_IMAGE = "jfrog.carizon.work/devops-template/${PROJECT_NAME}:${params.BRANCH}_${TIMESTAMP}"
        PROJECT_DIR = "${WORKSPACE}"
        TARGET_DIR = "${PROJECT_DIR}/target"
        DOCKERFILE_PATH = "${PROJECT_DIR}/src/main/docker/Dockerfile"
        
        // CD 相关配置
        CONFIG_REPO = "https://git.carizon.work/bpm/cd-config-repo.git" // CD配置仓库地址
        DEPLOY_BRANCH = "deploy/${PROJECT_NAME}-${BUILD_NUMBER}-${TIMESTAMP}" // 临时部署分支
        NAMESPACE = "cosmic-${ENV}" // 目标命名空间
        ARGOCD_APP_NAME = "${PROJECT_NAME}-${ENV}" // ArgoCD应用名
    }

    options {
        skipDefaultCheckout(true) // 自定义代码检出步骤
        timeout(time: 60, unit: 'MINUTES') // 构建超时时间
        buildDiscarder(logRotator(daysToKeepStr: '15', numToKeepStr: '20')) // 构建历史保留策略
    }

    stages {
        // ====== CI 阶段：代码构建与镜像推送 ======
        stage('清理工作空间') {
            steps {
                echo "开始清理工作空间..."
                cleanWs()
            }
        }

        stage('克隆代码') {
            steps {
                cloneGitRepository()
            }
        }

        stage('Maven 构建') {
            steps {
                buildMavenProject()
            }
        }

        stage('Docker 构建与推送') {
            steps {
                buildAndPushDockerImage()
            }
        }

        // ====== CD 阶段：配置更新与部署触发 ======
        stage('拉取配置仓库') {
            steps {
                cloneConfigRepository()
            }
        }

        stage('替换配置模板变量') {
            steps {
                replaceConfigVariables()
            }
        }

        stage('提交临时部署分支') {
            steps {
                commitDeployBranch()
            }
        }

        stage('创建PR到环境分支') {
            steps {
                createMergeRequest()
            }
        }
    }

    post {
        always {
            echo "构建流程结束，清理工作空间..."
            cleanWs()
        }
        success {
            echo " 构建成功！镜像地址：${DOCKER_IMAGE}，部署PR已创建"
            // 可选：发送通知到企业微信/Slack
            // slackSend channel: '#devops', color: 'good', message: "[${PROJECT_NAME}] 构建成功: ${DOCKER_IMAGE}"
        }
        failure {
            echo " 构建失败，请查看日志排查问题"
            // 可选：发送失败通知
            // slackSend channel: '#devops', color: 'danger', message: "[${PROJECT_NAME}] 构建失败: 构建号 ${BUILD_NUMBER}"
        }
    }
}

// 克隆代码仓库
def cloneGitRepository() {
    echo "克隆代码仓库（分支：${params.BRANCH}）..."
    checkout([
        $class: 'GitSCM',
        branches: [[name: "*/${params.BRANCH}"]],
        userRemoteConfigs: [[url: env.GITLAB_PROJECT_URL, credentialsId: 'gitlab']],
        extensions: [[$class: 'CloneOption', depth: 1]] // 浅克隆加速
    ])
}

// Maven构建项目
def buildMavenProject() {
    echo "开始Maven构建..."
    sh """
        /usr/local/maven/bin/mvn clean install -Dmaven.test.skip=true -U
    """
    // 验证构建产物
    if (!fileExists("${TARGET_DIR}/${PROJECT_NAME}.jar")) {
        error "构建失败：未找到产物 ${TARGET_DIR}/${PROJECT_NAME}.jar"
    }
}

// 构建并推送Docker镜像
def buildAndPushDockerImage() {
    echo "构建Docker镜像：${DOCKER_IMAGE}"
    script {
        if (!fileExists(env.DOCKERFILE_PATH)) {
            error "未找到Dockerfile：${DOCKERFILE_PATH}"
        }
        
        // 构建镜像
        sh """
            cd ${TARGET_DIR}
            docker build -t ${DOCKER_IMAGE} -f ${DOCKERFILE_PATH} .
        """
        
        // 推送镜像到仓库
        withCredentials([usernamePassword(
            credentialsId: 'jfrog-registry-credentials',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            sh """
                docker login -u ${DOCKER_USER} -p ${DOCKER_PASS} jfrog.carizon.work
                docker push ${DOCKER_IMAGE}
                docker logout jfrog.carizon.work
            """
        }
    }
}

// 克隆配置仓库并创建临时部署分支
def cloneConfigRepository() {
    echo "克隆配置仓库并创建临时分支 ${DEPLOY_BRANCH}..."
    sh """
        git clone ${CONFIG_REPO} ${WORKSPACE}/config-repo
        cd ${WORKSPACE}/config-repo
        git checkout ${CONFIG_REPO_BRANCH}  // 基于环境分支创建临时分支
        git checkout -b ${DEPLOY_BRANCH}
        git config user.name "Jenkins CI"
        git config user.email "jenkins@carizon.work"
    """
}

// 替换配置模板中的变量
def replaceConfigVariables() {
    echo "替换配置模板变量..."
    script {
        // 计算配置哈希（用于变更检测）
        CONFIG_HASH = sh(
            script: "echo -n '${DOCKER_IMAGE}-${BUILD_NUMBER}' | sha256sum | cut -d' ' -f1",
            returnStdout: true
        ).trim()
        
        // 替换Deployment模板变量
        sh """
            cd ${WORKSPACE}/config-repo/base
            sed -i "s|\${APP_NAME}|${PROJECT_NAME}|g" deployment.template.yaml
            sed -i "s|\${NAMESPACE}|${NAMESPACE}|g" deployment.template.yaml
            sed -i "s|\${REPLICAS}|${REPLICAS}|g" deployment.template.yaml
            sed -i "s|\${IMAGE_TAG}|${params.BRANCH}_${TIMESTAMP}|g" deployment.template.yaml
            sed -i "s|\${BUILD_NUMBER}|${BUILD_NUMBER}|g" deployment.template.yaml
            sed -i "s|\${ENV}|${ENV}|g" deployment.template.yaml
        """
        
        // 替换ConfigMap模板变量
        sh """
            cd ${WORKSPACE}/config-repo/base
            sed -i "s|\${APP_NAME}|${PROJECT_NAME}|g" configmap.template.yaml
            sed -i "s|\${CONFIG_HASH}|${CONFIG_HASH}|g" configmap.template.yaml
            sed -i "s|\${NAMESPACE}|${NAMESPACE}|g" configmap.template.yaml
        """
    }
}

// 提交临时部署分支到配置仓库
def commitDeployBranch() {
    echo "提交临时部署分支到配置仓库..."
    withCredentials([usernamePassword(
        credentialsId: 'gitlab',
        usernameVariable: 'GIT_USER',
        passwordVariable: 'GIT_PASS'
    )]) {
        sh """
            cd ${WORKSPACE}/config-repo
            git add .
            git commit -m "[${PROJECT_NAME}] 自动部署更新：${DOCKER_IMAGE}（构建号：${BUILD_NUMBER}）"
            git push https://${GIT_USER}:${GIT_PASS}@${CONFIG_REPO.replace('https://', '')} ${DEPLOY_BRANCH}
        """
    }
}

// 创建PR到环境分支（如release/prod）
def createMergeRequest() {
    echo "创建PR到目标环境分支 ${CONFIG_REPO_BRANCH}..."
    withCredentials([string(credentialsId: 'gitlab-api-token', variable: 'GITLAB_TOKEN')]) {
        sh """
            curl --request POST \
              --url "https://git.carizon.work/api/v4/projects/bpm%2Fcd-config-repo/merge_requests" \
              --header "Private-Token: ${GITLAB_TOKEN}" \
              --header "Content-Type: application/json" \
              --data '{
                "source_branch": "${DEPLOY_BRANCH}",
                "target_branch": "${CONFIG_REPO_BRANCH}",
                "title": "[${PROJECT_NAME}] 部署PR（${ENV}环境，构建号${BUILD_NUMBER}）",
                "description": "镜像地址：${DOCKER_IMAGE}\n代码分支：${BRANCH}\n副本数：${REPLICAS}"
              }'
        """
    }
}
