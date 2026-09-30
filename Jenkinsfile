pipeline {
    agent any

    environment {
        DEPLOY_ROOT = '/var/www/zhiyinhui-command-center'
        DOMAIN = 'command.zhiyinhui.top'
    }

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir()
                checkout scm

                sh '''
                    set -eu
                    git rev-parse HEAD
                    git status --short
                '''
            }
        }

        stage('Validate') {
            steps {
                sh '''
                    set -eu

                    test -f index.html
                    test -s index.html

                    FILE_SIZE="$(wc -c < index.html)"
                    if [ "$FILE_SIZE" -lt 100000 ]; then
                        echo "index.html 文件异常，大小只有 ${FILE_SIZE} 字节"
                        exit 1
                    fi

                    grep -q '<title>智隐会 · AI经营指挥中心</title>' index.html
                    grep -q 'id="root"' index.html

                    test -d "${DEPLOY_ROOT}/releases"
                    test -w "${DEPLOY_ROOT}"
                    test -w "${DEPLOY_ROOT}/releases"

                    echo "静态文件和部署目录校验通过"
                    echo "文件大小：${FILE_SIZE} 字节"
                    sha256sum index.html
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -eu

                    RELEASE_DIR="${DEPLOY_ROOT}/releases/${BUILD_NUMBER}"
                    TEMP_LINK="${DEPLOY_ROOT}/.current-${BUILD_NUMBER}"
                    PREVIOUS_RELEASE="$(readlink "${DEPLOY_ROOT}/current" 2>/dev/null || true)"

                    if [ -e "$RELEASE_DIR" ]; then
                        echo "停止：版本目录已经存在：$RELEASE_DIR"
                        exit 1
                    fi

                    if [ -e "$TEMP_LINK" ] || [ -L "$TEMP_LINK" ]; then
                        echo "停止：临时链接已经存在：$TEMP_LINK"
                        exit 1
                    fi

                    install -d -m 0755 "$RELEASE_DIR"
                    install -m 0644 index.html "$RELEASE_DIR/index.html"

                    git rev-parse HEAD > "$RELEASE_DIR/REVISION"

                    {
                        echo "build=${BUILD_NUMBER}"
                        echo "revision=$(git rev-parse HEAD)"
                        echo "deployed_at=$(date -Iseconds)"
                    } > "$RELEASE_DIR/BUILD_INFO"

                    printf '%s\n' "$PREVIOUS_RELEASE" > "$WORKSPACE/.previous_release"

                    ln -s "releases/${BUILD_NUMBER}" "$TEMP_LINK"
                    mv -Tf "$TEMP_LINK" "${DEPLOY_ROOT}/current"

                    echo "已发布版本：${BUILD_NUMBER}"
                    echo "上一版本：${PREVIOUS_RELEASE:-无}"
                    readlink "${DEPLOY_ROOT}/current"
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    set -eu

                    HEALTH_FILE="$WORKSPACE/deployed.html"

                    if curl \
                        --http1.1 \
                        --noproxy '*' \
                        --resolve "${DOMAIN}:443:127.0.0.1" \
                        --fail \
                        --silent \
                        --show-error \
                        --max-time 20 \
                        "https://${DOMAIN}/" \
                        --output "$HEALTH_FILE" \
                        && grep -q '<title>智隐会 · AI经营指挥中心</title>' "$HEALTH_FILE" \
                        && grep -q 'id="root"' "$HEALTH_FILE"
                    then
                        echo "HTTPS 健康检查通过"
                        wc -c "$HEALTH_FILE"
                    else
                        echo "健康检查失败，开始回滚"

                        PREVIOUS_RELEASE="$(cat "$WORKSPACE/.previous_release")"

                        if [ -n "$PREVIOUS_RELEASE" ] \
                            && [ -f "${DEPLOY_ROOT}/${PREVIOUS_RELEASE}/index.html" ]
                        then
                            ROLLBACK_LINK="${DEPLOY_ROOT}/.rollback-${BUILD_NUMBER}"

                            if [ -e "$ROLLBACK_LINK" ] || [ -L "$ROLLBACK_LINK" ]; then
                                echo "停止：回滚临时链接已存在"
                                exit 1
                            fi

                            ln -s "$PREVIOUS_RELEASE" "$ROLLBACK_LINK"
                            mv -Tf "$ROLLBACK_LINK" "${DEPLOY_ROOT}/current"

                            echo "已回滚到：$PREVIOUS_RELEASE"
                        else
                            ACTIVE_RELEASE="$(readlink "${DEPLOY_ROOT}/current" 2>/dev/null || true)"

                            if [ "$ACTIVE_RELEASE" = "releases/${BUILD_NUMBER}" ]; then
                                unlink "${DEPLOY_ROOT}/current"
                            fi

                            echo "没有上一版本，已撤销本次发布链接"
                        fi

                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        success {
            echo "部署成功：https://${DOMAIN}"
        }

        failure {
            echo '部署失败，请查看失败阶段；线上版本已按规则回滚。'
        }

        always {
            archiveArtifacts artifacts: 'deployed.html', allowEmptyArchive: true
        }
    }
}
