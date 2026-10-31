pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        APP_NAME  = 'demo-api'
        DEPLOY_DIR = '/opt/demo-app'
        APP_PORT  = '8081'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'echo "Commit: $(git rev-parse --short HEAD)"'
            }
        }

        stage('Test') {
            steps {
                sh 'go version'
                sh 'go test ./...'
            }
        }

        stage('Build Linux Binary') {
            steps {
                sh '''
                    set -eu
                    mkdir -p dist

                    COMMIT="$(git rev-parse --short HEAD)"
                    BUILD_TIME="$(date -u +%Y-%m-%dT%H:%M:%SZ)"

                    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \\
                    go build -trimpath \\
                      -ldflags "-s -w -X main.version=${BUILD_NUMBER} -X main.commit=${COMMIT} -X main.buildTime=${BUILD_TIME}" \\
                      -o "dist/${APP_NAME}" .

                    test -s "dist/${APP_NAME}"
                    file "dist/${APP_NAME}"
                '''
            }
        }

        stage('Deploy to Local Linux Runtime') {
            steps {
                sh '''
                    set -eu
                    mkdir -p "$DEPLOY_DIR/backup"

                    # Keep a copy of the currently deployed binary, if one exists.
                    if [ -f "$DEPLOY_DIR/$APP_NAME" ]; then
                        cp "$DEPLOY_DIR/$APP_NAME" \
                          "$DEPLOY_DIR/backup/${APP_NAME}-build-${BUILD_NUMBER}-previous"
                    fi

                    # Put the new binary in place atomically.
                    install -m 755 "dist/$APP_NAME" "$DEPLOY_DIR/${APP_NAME}.new"

                    # Stop the previous process gracefully, if it is running.
                    if [ -s "$DEPLOY_DIR/${APP_NAME}.pid" ]; then
                        OLD_PID="$(cat "$DEPLOY_DIR/${APP_NAME}.pid")"
                        OLD_EXE=""
                        if [ -e "/proc/$OLD_PID/exe" ]; then
                            OLD_EXE="$(readlink "/proc/$OLD_PID/exe" 2>/dev/null || true)"
                        fi

                        # Only stop the PID if it still belongs to this demo app.
                        # This avoids killing an unrelated process if a stale PID was reused.
                        if [ "$OLD_EXE" = "$DEPLOY_DIR/$APP_NAME" ]; then
                            kill -TERM "$OLD_PID" || true
                            STOPPED=0
                            for attempt in $(seq 1 10); do
                                if ! kill -0 "$OLD_PID" 2>/dev/null; then
                                    STOPPED=1
                                    break
                                fi
                                sleep 1
                            done
                            if [ "$STOPPED" -ne 1 ] && kill -0 "$OLD_PID" 2>/dev/null; then
                                kill -KILL "$OLD_PID" || true
                            fi
                        fi
                        rm -f "$DEPLOY_DIR/${APP_NAME}.pid"
                    fi

                    mv -f "$DEPLOY_DIR/${APP_NAME}.new" "$DEPLOY_DIR/$APP_NAME"

                    # Start the newly deployed binary as a background process.
                    nohup "$DEPLOY_DIR/$APP_NAME" \\
                      > "$DEPLOY_DIR/${APP_NAME}.log" 2>&1 < /dev/null &
                    NEW_PID=$!
                    echo "$NEW_PID" > "$DEPLOY_DIR/${APP_NAME}.pid"
                    echo "Started ${APP_NAME} with PID ${NEW_PID}"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -eu
                    HEALTH_OK=0

                    for attempt in $(seq 1 20); do
                        if curl -fsS "http://127.0.0.1:${APP_PORT}/health"; then
                            HEALTH_OK=1
                            break
                        fi
                        sleep 1
                    done

                    if [ "$HEALTH_OK" -ne 1 ]; then
                        echo "Health check failed. Application log:"
                        cat "$DEPLOY_DIR/${APP_NAME}.log" || true
                        exit 1
                    fi

                    echo "Deployed application version:"
                    curl -fsS "http://127.0.0.1:${APP_PORT}/version"
                    echo
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment lab succeeded.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs and the deployed application log.'
        }
    }
}
