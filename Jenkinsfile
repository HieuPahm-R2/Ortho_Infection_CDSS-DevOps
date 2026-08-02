pipeline {
    agent any

    parameters {
        string(name: 'DOCKERHUB_REPO', defaultValue: 'hieupahmet', trim: true, description: 'Docker Hub repository namespace')
        string(name: 'COMPONENT_BRANCH', defaultValue: 'main', trim: true, description: 'Branch to checkout for every component repo')
        string(name: 'BACKEND_REPO_URL',  defaultValue: 'https://github.com/108-PJIOrthoGen/Backend_Server.git',  trim: true, description: 'Git URL for the Spring backend')
        string(name: 'FRONTEND_REPO_URL', defaultValue: 'https://github.com/108-PJIOrthoGen/Frontend_Client.git', trim: true, description: 'Git URL for the React frontend')
        string(name: 'RAG_REPO_URL',      defaultValue: 'https://github.com/108-PJIOrthoGen/Rag_Agentic.git',     trim: true, description: 'Git URL for the RAG service')
        string(name: 'EXTRACT_REPO_URL',  defaultValue: 'https://github.com/108-PJIOrthoGen/Extract_Images.git',  trim: true, description: 'Git URL for Extract Images')
        string(name: 'DEPLOY_PATH', defaultValue: '/opt/pji-advisor', trim: true, description: 'Local directory on the Jenkins host where docker compose runs')
        booleanParam(name: 'RUN_TESTS', defaultValue: false, description: 'Run unit tests (skip for now until test infrastructure is in place)')
        booleanParam(name: 'RUN_SONAR', defaultValue: false, description: 'Run SonarQube analysis')
        booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Deploy locally after pushing images (Jenkins must run on the same host as the docker daemon)')
    }

    environment {
        // Component checkout dirs (siblings of the Infras_Devops content at workspace root)
        BACKEND_DIR  = 'Backend_Server'
        FRONTEND_DIR = 'Frontend_Client'
        RAG_DIR      = 'Rag_Agentic'
        EXTRACT_DIR  = 'Extract_Images'
        DOCKER_BUILDKIT = '1'
        // Mirror params -> env so shell steps can read them on the FIRST build (before
        // Jenkins has registered the new parameters block). Elvis (?:) supplies defaults.
        BACKEND_REPO_URL  = "${params.BACKEND_REPO_URL  ?: 'https://github.com/108-PJIOrthoGen/Backend_Server.git'}"
        FRONTEND_REPO_URL = "${params.FRONTEND_REPO_URL ?: 'https://github.com/108-PJIOrthoGen/Frontend_Client.git'}"
        RAG_REPO_URL      = "${params.RAG_REPO_URL      ?: 'https://github.com/108-PJIOrthoGen/Rag_Agentic.git'}"
        EXTRACT_REPO_URL  = "${params.EXTRACT_REPO_URL  ?: 'https://github.com/108-PJIOrthoGen/Extract_Images.git'}"
        COMPONENT_BRANCH  = "${params.COMPONENT_BRANCH  ?: 'main'}"
        DOCKERHUB_REPO    = "${params.DOCKERHUB_REPO    ?: 'hieupahmet'}"
        DEPLOY_PATH       = "${params.DEPLOY_PATH       ?: '/opt/pji-advisor'}"
    }

    triggers {
        // Poll the SCM (Infras_Devops) every 5 minutes. Component repos must be triggered
        // separately if you want auto-rebuild on every code push — see README for options.
        pollSCM('H/5 * * * *')
    }

    options {
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                // checkout scm pulls Infras_Devops at workspace root (Caddyfile, Caddyfile.prod,
                // docker/, Jenkinsfile etc. are siblings of Backend_Server/, Frontend_Client/, ...)
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.SAFE_BRANCH = (env.BRANCH_NAME ?: 'manual').replaceAll('[^A-Za-z0-9_.-]', '-')
                    env.IMAGE_TAG = "${env.SAFE_BRANCH}-${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                }

                // Clone the four component repos as siblings of Infras_Devops content.
                // Use the github-pat credential so private org repos are accessible.
                withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh '''
                        set -eu
                        clone_or_pull() {
                          local url="$1" dir="$2" branch="$3"
                          # Inject token into HTTPS URL (works for private org repos)
                          local auth_url="$(echo "$url" | sed -E "s|https://|https://${GH_USER}:${GH_TOKEN}@|")"
                          if [ -d "$dir/.git" ]; then
                            echo "[refresh] $dir"
                            git -C "$dir" remote set-url origin "$auth_url"
                            git -C "$dir" fetch --depth 1 origin "$branch"
                            git -C "$dir" reset --hard "origin/$branch"
                          else
                            echo "[clone] $dir from $url"
                            git clone --depth 1 --branch "$branch" "$auth_url" "$dir"
                          fi
                          # Scrub token back out so logs/inspectors don't leak it
                          git -C "$dir" remote set-url origin "$url"
                        }
                        clone_or_pull "${BACKEND_REPO_URL}"  "${BACKEND_DIR}"  "${COMPONENT_BRANCH}"
                        clone_or_pull "${FRONTEND_REPO_URL}" "${FRONTEND_DIR}" "${COMPONENT_BRANCH}"
                        clone_or_pull "${RAG_REPO_URL}"      "${RAG_DIR}"      "${COMPONENT_BRANCH}"
                        clone_or_pull "${EXTRACT_REPO_URL}"  "${EXTRACT_DIR}"  "${COMPONENT_BRANCH}"
                    '''
                }
            }
        }

        stage('Validate Layout') {
            steps {
                sh '''
                    # Infras_Devops content lives at workspace root after `checkout scm`
                    test -f Caddyfile
                    test -f Caddyfile.prod
                    test -f maintenance.html
                    test -f assets/pog-logo.png
                    test -f docker/docker-compose.yml
                    test -f docker/observability/alertmanager/alertmanager.yml
                    test -f docker/observability/prometheus/prometheus.yml
                    test -f docker/observability/prometheus/prometheus.local.yml
                    test -f docker/observability/prometheus/rules/pji-alerts.yml
                    test -f docker/observability/prometheus/tests/pji-alerts.test.yml
                    test -f docker/observability/alloy/config.alloy
                    test -f docker/observability/otel-collector/config.yml
                    test -f docker/observability/jaeger/config.yml
                    test -f docker/observability/grafana/provisioning/datasources/datasources.yml
                    test -f docker/observability/grafana/provisioning/dashboards/provider.yml
                    test -f docker/observability/grafana/provisioning/dashboards/json/production-overview.json
                    # Component repos cloned by the Checkout stage above
                    test -f "${BACKEND_DIR}/pom.xml"
                    test -f "${BACKEND_DIR}/Dockerfile"
                    test -f "${FRONTEND_DIR}/package.json"
                    test -f "${FRONTEND_DIR}/Dockerfile"
                    test -f "${FRONTEND_DIR}/nginx.conf"
                    test -f "${RAG_DIR}/pyproject.toml"
                    test -f "${RAG_DIR}/Dockerfile"
                    test -f "${EXTRACT_DIR}/pyproject.toml"
                    test -f "${EXTRACT_DIR}/Dockerfile"
                '''
            }
        }

        stage('Validate Observability') {
            steps {
                sh '''
                    set -eu
                    root_dir="$(pwd)"
                    discord_test_secret="$(mktemp "$root_dir/.alertmanager-validation.XXXXXX")"
                    trap 'rm -f "$discord_test_secret"' EXIT
                    printf '%s' 'https://discord.com/api/webhooks/validation/placeholder' > "$discord_test_secret"
                    chmod 644 "$discord_test_secret"

                    docker compose -f docker/docker-compose.yml config --no-interpolate --quiet
                    docker compose -f docker-buildlocal.yml config --no-interpolate --quiet

                    docker run --rm \
                      -v "$root_dir/docker/observability/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro" \
                      -v "$root_dir/docker/observability/prometheus/rules:/etc/prometheus/rules:ro" \
                      --entrypoint /bin/promtool \
                      prom/prometheus:v3.13.1 \
                      check config /etc/prometheus/prometheus.yml

                    docker run --rm \
                      -v "$root_dir/docker/observability/prometheus:/workspace:ro" \
                      -w /workspace/tests \
                      --entrypoint /bin/promtool \
                      prom/prometheus:v3.13.1 \
                      test rules pji-alerts.test.yml

                    docker run --rm \
                      -v "$root_dir/docker/observability/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro" \
                      -v "$discord_test_secret:/run/secrets/discord_webhook_url:ro" \
                      --entrypoint /bin/amtool \
                      prom/alertmanager:v0.28.1 \
                      check-config /etc/alertmanager/alertmanager.yml

                    docker run --rm \
                      -v "$root_dir/docker/observability/otel-collector/config.yml:/etc/otelcol/config.yml:ro" \
                      otel/opentelemetry-collector-contrib:0.153.0 \
                      validate --config=/etc/otelcol/config.yml

                    docker run --rm \
                      -v "$root_dir/Caddyfile.prod:/etc/caddy/Caddyfile:ro" \
                      caddy:2-alpine \
                      caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile

                    jq -e . docker/observability/grafana/provisioning/dashboards/json/production-overview.json >/dev/null
                '''
            }
        }

        stage('Test') {
            when {
                expression { return params.RUN_TESTS }
            }
            parallel {
                stage('Backend') {
                    steps {
                        dir("${env.BACKEND_DIR}") {
                            sh './mvnw -B test -Dspring.profiles.active=test'
                        }
                    }
                    post {
                        always {
                            junit testResults: "${env.BACKEND_DIR}/target/surefire-reports/*.xml", allowEmptyResults: true
                        }
                    }
                }

                stage('Frontend') {
                    steps {
                        dir("${env.FRONTEND_DIR}") {
                            sh '''
                                npm ci
                                npm run build
                            '''
                        }
                    }
                }

                stage('RAG Service') {
                    steps {
                        dir("${env.RAG_DIR}") {
                            sh '''
                                python3 -m pip install --upgrade pip uv
                                uv sync --frozen --dev
                                uv run pytest --junitxml=test-results.xml -v
                            '''
                        }
                    }
                    post {
                        always {
                            junit testResults: "${env.RAG_DIR}/test-results.xml", allowEmptyResults: true
                        }
                    }
                }

                stage('Extract Images') {
                    steps {
                        dir("${env.EXTRACT_DIR}") {
                            sh '''
                                python3 -m pip install --upgrade pip uv
                                uv sync --frozen --dev
                                uv run pytest --junitxml=test-results.xml -v || true
                            '''
                        }
                    }
                    post {
                        always {
                            junit testResults: "${env.EXTRACT_DIR}/test-results.xml", allowEmptyResults: true
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            when {
                expression { return params.RUN_SONAR }
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    dir("${env.BACKEND_DIR}") {
                        sh './mvnw -B sonar:sonar -Dsonar.projectKey=pji-backend -Dsonar.projectName="PJI Backend"'
                    }
                    dir("${env.FRONTEND_DIR}") {
                        sh '''
                            npm ci
                            npx sonar-scanner \
                              -Dsonar.projectKey=pji-frontend \
                              -Dsonar.projectName="PJI Frontend" \
                              -Dsonar.sources=src
                        '''
                    }
                    dir("${env.RAG_DIR}") {
                        sh '''
                            npx sonar-scanner \
                              -Dsonar.projectKey=pji-rag-service \
                              -Dsonar.projectName="PJI RAG Service" \
                              -Dsonar.sources=app \
                              -Dsonar.python.version=3.11
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression { return params.RUN_SONAR }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Images') {
            parallel {
                stage('Backend Image') {
                    steps {
                        dir("${env.BACKEND_DIR}") {
                            sh """
                                docker build \
                                  -t ${params.DOCKERHUB_REPO}/pji-backend:${env.IMAGE_TAG} \
                                  -t ${params.DOCKERHUB_REPO}/pji-backend:latest \
                                  .
                            """
                        }
                    }
                }

                stage('Frontend Image') {
                    steps {
                        dir("${env.FRONTEND_DIR}") {
                            sh '''
                                set -eu
                                TURNSTILE_SITE_KEY=""
                                if [ -f "${DEPLOY_PATH}/.env" ]; then
                                  TURNSTILE_SITE_KEY="$(awk -F= '/^VITE_TURNSTILE_SITE_KEY=/ {print substr($0, index($0, "=") + 1); found=1; exit} END {if (!found) print ""}' "${DEPLOY_PATH}/.env" | tr -d '\\r')"
                                fi
                                docker build \
                                  --build-arg VITE_BACKEND_URL=/ \
                                  --build-arg VITE_ACL_ENABLE=true \
                                  --build-arg VITE_TURNSTILE_SITE_KEY="${TURNSTILE_SITE_KEY}" \
                                  -t "${DOCKERHUB_REPO}/pji-frontend:${IMAGE_TAG}" \
                                  -t "${DOCKERHUB_REPO}/pji-frontend:latest" \
                                  .
                            '''
                        }
                    }
                }

                stage('RAG Image') {
                    steps {
                        dir("${env.RAG_DIR}") {
                            sh """
                                docker build \
                                  -t ${params.DOCKERHUB_REPO}/pji-rag-service:${env.IMAGE_TAG} \
                                  -t ${params.DOCKERHUB_REPO}/pji-rag-service:latest \
                                  .
                            """
                        }
                    }
                }

                stage('Extract Images Image') {
                    steps {
                        dir("${env.EXTRACT_DIR}") {
                            sh """
                                docker build \
                                  -t ${params.DOCKERHUB_REPO}/pji-extract-api:${env.IMAGE_TAG} \
                                  -t ${params.DOCKERHUB_REPO}/pji-extract-api:latest \
                                  -t ${params.DOCKERHUB_REPO}/pji-extract-worker:${env.IMAGE_TAG} \
                                  -t ${params.DOCKERHUB_REPO}/pji-extract-worker:latest \
                                  .
                            """
                        }
                    }
                }
            }
        }

        stage('Push Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        set -eu
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        # Push with retry — slow networks sometimes need multiple attempts to
                        # complete the upload of large Python wheels (torch, transformers, etc.).
                        # Sequential pushes avoid saturating bandwidth with parallel uploads.
                        push_with_retry() {
                          local image="$1"
                          local attempts=0
                          local max=3
                          until docker push "$image"; do
                            attempts=$((attempts + 1))
                            if [ "$attempts" -ge "$max" ]; then
                              echo "FAILED to push $image after $max attempts"
                              return 1
                            fi
                            echo "Push of $image failed, retrying in 10s ($attempts/$max)..."
                            sleep 10
                          done
                        }

                        for repo in pji-backend pji-frontend pji-rag-service pji-extract-api pji-extract-worker; do
                          push_with_retry "${DOCKERHUB_REPO}/${repo}:${IMAGE_TAG}"
                          push_with_retry "${DOCKERHUB_REPO}/${repo}:latest"
                        done

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                expression { return params.DEPLOY }
            }
            steps {
                sh '''
                    set -eu
                    test -d "${DEPLOY_PATH}" || { echo "DEPLOY_PATH ${DEPLOY_PATH} does not exist"; exit 1; }
                    test -f "${DEPLOY_PATH}/.env" || { echo "${DEPLOY_PATH}/.env is missing — create it manually before first deploy"; exit 1; }

                    mkdir -p "${DEPLOY_PATH}/docker/observability"
                    mkdir -p "${DEPLOY_PATH}/docker/init-db"
                    mkdir -p "${DEPLOY_PATH}/backups"   # postgres-backup-local writes here
                    # Production uses a tunnel-mode Caddyfile (HTTP-only on :80, no Let's Encrypt).
                    # NOTE: Infras_Devops content lives at the workspace root after `checkout scm`,
                    # so paths are NOT prefixed with Infras_Devops/.
                    # Caddy config DIR: Caddyfile + maintenance page + assets live together in
                    # ${DEPLOY_PATH}/caddy and are bind-mounted as /etc/caddy (read-only).
                    # The maintenance toggle is a flag file in the same dir (checked per-request,
                    # no Caddy reload needed):
                    #   ON : touch ${DEPLOY_PATH}/caddy/maintenance.on
                    #   OFF: rm -f  ${DEPLOY_PATH}/caddy/maintenance.on
                    # Deploys intentionally do NOT touch maintenance.on — maintenance state
                    # survives redeploys until toggled off explicitly.
                    mkdir -p "${DEPLOY_PATH}/caddy/assets"
                    cp Caddyfile.prod      "${DEPLOY_PATH}/caddy/Caddyfile"
                    cp maintenance.html    "${DEPLOY_PATH}/caddy/maintenance.html"
                    cp assets/pog-logo.png "${DEPLOY_PATH}/caddy/assets/pog-logo.png"
                    rm -f "${DEPLOY_PATH}/Caddyfile"   # legacy single-file location from older deploys
                    rm -rf "${DEPLOY_PATH}/docker/signoz" # legacy observability stack configs
                    cp docker/docker-compose.yml "${DEPLOY_PATH}/docker-compose.yml"
                    cp -r docker/observability/. "${DEPLOY_PATH}/docker/observability/"
                    if [ -d docker/init-db ] && [ -n "$(ls -A docker/init-db 2>/dev/null)" ]; then
                      cp -r docker/init-db/. "${DEPLOY_PATH}/docker/init-db/"
                    fi

                    # Bind mount the caddy config dir on the server (no Docker Desktop fileshare cache there).
                    # The source compose uses an external `pji_caddy_config` volume as a Docker Desktop workaround.
                    sed -i 's|pji_caddy_config:/etc/caddy:ro|./caddy:/etc/caddy:ro|' "${DEPLOY_PATH}/docker-compose.yml"
                    sed -i '/^  pji_caddy_config:$/,/^    external: true$/d' "${DEPLOY_PATH}/docker-compose.yml"

                    cd "${DEPLOY_PATH}"
                    # Resolve the production .env and verify required secret files
                    # before pulling or recreating any container.
                    docker compose config --quiet
                    # Caddy and the separately managed cloudflared container use
                    # this external network for a private container-to-container
                    # origin route (`http://caddy:80`).
                    docker network inspect cloudflare-net >/dev/null 2>&1 || \
                      docker network create --driver bridge cloudflare-net >/dev/null
                    # Pull only the images we build/push ourselves (the rest are public images
                    # like postgres:16-alpine, redis:7-alpine, prometheus/loki/grafana/jaeger, etc. — compose pulls
                    # them automatically on first `up`).
                    DOCKERHUB_REPO="${DOCKERHUB_REPO}" IMAGE_TAG="${IMAGE_TAG}" \
                      docker compose pull pji-backend pji-frontend pji-rag-service pji-extract-api pji-extract-worker caddy
                    # Bring up the full stack including Prometheus, Alertmanager,
                    # Loki, Grafana, Jaeger, Alloy, and the OTel collector.
                    DOCKERHUB_REPO="${DOCKERHUB_REPO}" IMAGE_TAG="${IMAGE_TAG}" \
                      docker compose up -d --remove-orphans
                '''
            }
        }

        stage('Smoke Test') {
            when {
                expression { return params.DEPLOY }
            }
            steps {
                sh '''
                    set -eu
                    for container in \
                      pji-backend pji-frontend pji-rag-service pji-caddy \
                      pji-postgres-exporter pji-redis-exporter \
                      pji-alertmanager pji-prometheus pji-loki \
                      pji-docker-socket-proxy pji-alloy \
                      pji-jaeger pji-otel-collector pji-grafana; do
                      tries=0
                      while [ "$tries" -lt 30 ]; do
                        status="$(docker inspect --format='{{if .State.Health}}{{.State.Health.Status}}{{else}}{{.State.Status}}{{end}}' "$container" 2>/dev/null || true)"
                        if [ "$status" = "healthy" ] || [ "$status" = "running" ]; then
                          break
                        fi
                        tries=$((tries + 1))
                        sleep 5
                      done
                      status="$(docker inspect --format='{{if .State.Health}}{{.State.Health.Status}}{{else}}{{.State.Status}}{{end}}' "$container" 2>/dev/null || true)"
                      if [ "$status" != "healthy" ] && [ "$status" != "running" ]; then
                        echo "Container $container is not healthy: $status"
                        docker logs --tail 30 "$container" || true
                        exit 1
                      fi
                    done

                    docker exec pji-prometheus \
                      wget --spider -q http://localhost:9090/-/ready
                    docker exec pji-alertmanager \
                      wget --spider -q http://localhost:9093/-/ready
                    docker exec pji-loki \
                      wget --spider -q http://localhost:3100/ready
                    docker exec pji-docker-socket-proxy \
                      wget -qO- http://docker-socket-proxy:2375/_ping | grep -q OK
                    docker exec pji-docker-socket-proxy \
                      wget -qO- http://docker-socket-proxy:2375/containers/json >/dev/null
                    post_response="$(docker exec pji-docker-socket-proxy \
                      wget -S -O /dev/null --post-data='' \
                      http://docker-socket-proxy:2375/containers/json 2>&1 || true)"
                    echo "$post_response" | grep -q '403 Forbidden'
                    docker exec pji-jaeger \
                      wget --spider -q http://localhost:13133/status
                    docker exec pji-otel-collector \
                      wget --spider -q http://localhost:13133
                    docker exec pji-grafana \
                      wget --spider -q http://localhost:3000/api/health
                    docker inspect pji-caddy \
                      --format '{{json .NetworkSettings.Networks}}' | grep -q '"cloudflare-net"'

                    rules_json="$(docker exec pji-prometheus \
                      wget -qO- http://localhost:9090/api/v1/rules)"
                    echo "$rules_json" | grep -q '"name":"PjiTargetDown"'
                    echo "$rules_json" | grep -q '"name":"PjiBackendHighServerErrorRate"'

                    # Retry the public-facing curl — Caddy may briefly 5xx right after startup
                    # before its first upstream probe completes.
                    # When the maintenance flag is set, the expected answer is the 503
                    # maintenance page instead of the 200 SPA.
                    success=0
                    for i in $(seq 1 12); do
                      code=$(curl -s -o /dev/null -w '%{http_code}' -H 'Host: 108pog.site' http://localhost/ || true)
                      if [ "$code" = "200" ] || [ "$code" = "304" ]; then
                        success=1
                        echo "Caddy returned $code after $i attempt(s)"
                        break
                      fi
                      if [ "$code" = "503" ] && [ -f "${DEPLOY_PATH}/caddy/maintenance.on" ]; then
                        success=1
                        echo "Caddy returned 503 — expected, maintenance mode is ON"
                        break
                      fi
                      echo "[$i/12] caddy returned $code, retrying in 5s..."
                      sleep 5
                    done
                    if [ "$success" != "1" ]; then
                      echo "Caddy never returned 200 — last code: $code"
                      docker logs --tail 30 pji-caddy || true
                      exit 1
                    fi

                    unknown_code="$(curl -s -o /dev/null -w '%{http_code}' \
                      -H 'Host: origin.invalid' http://localhost/ || true)"
                    if [ "$unknown_code" != "421" ]; then
                      echo "Caddy accepted an unknown Host with status $unknown_code"
                      exit 1
                    fi
                '''
            }
        }
    }

    post {
        always {
            sh '''
                docker image rm "${DOCKERHUB_REPO}/pji-backend:${IMAGE_TAG}" 2>/dev/null || true
                docker image rm "${DOCKERHUB_REPO}/pji-frontend:${IMAGE_TAG}" 2>/dev/null || true
                docker image rm "${DOCKERHUB_REPO}/pji-rag-service:${IMAGE_TAG}" 2>/dev/null || true
                docker image rm "${DOCKERHUB_REPO}/pji-extract-api:${IMAGE_TAG}" 2>/dev/null || true
                docker image rm "${DOCKERHUB_REPO}/pji-extract-worker:${IMAGE_TAG}" 2>/dev/null || true
            '''
            cleanWs()
        }
    }
}
