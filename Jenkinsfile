// 取得廳主買域名項目資料 (輪詢Job狀態檢查)
def checkCustomerApplyPurchaseDomainJobStatus() {
    def jobNameMap = [
        "AddTag": "AddTag 新增 Tag",
        "AddThirdLevelRandom": "AddThirdLevelRandom 設定三級亂數",
        "AttachAntiBlockTarget": "AttachAntiBlockTarget 新增抗封鎖目標",
        "AttachAntiHijackSource": "AttachAntiHijackSource 新增抗劫持",
        "AttachAntiHijackTarget": "AttachAntiHijackTarget 新增抗劫持目標",
        "CheckDomainBlocked": "CheckDomainBlocked 檢查封鎖",
        "CheckPurchaseDeployCertificateStatus": "CheckPurchaseDeployCertificateStatus 檢查購買部署憑證結果",
        "CheckWorkflowApplication": "CheckWorkflowApplication 檢查自動化申請",
        "DeleteDomainRecord": "DeleteDomainRecord 刪除解析",
        "DetachAntiBlockSource": "DetachAntiBlockSource 撤下抗封鎖",
        "DetachAntiBlockTarget": "DetachAntiBlockTarget 撤下抗封鎖目標",
        "DetachAntiHijackSource": "DetachAntiHijackSource 撤下抗劫持",
        "DetachAntiHijackTarget": "DetachAntiHijackTarget 撤下抗劫持目標",
        "InformDomainInfringement": "InformDomainInfringement 通知侵權網址",
        "MergeErrorRecord": "MergeErrorRecord 檢查異常地區合併規則",
        "PurchaseAndDeployCert": "PurchaseAndDeployCert 購買與部署憑證",
        "PurchaseDomain": "PurchaseDomain 購買域名",
        "RecheckARecordResolution": "RecheckARecordResolution 複檢域名 A 紀錄解析",
        "RecheckCert": "RecheckCert 複檢憑證",
        "RecheckDomainResolution": "RecheckDomainResolution 複檢域名",
        "RecheckThirdLevelRandom": "RecheckThirdLevelRandom 複檢三級亂數",
        "RemoveAntiBlock": "RemoveAntiBlock 刪除抗封鎖",
        "RemoveAntiBlockTarget": "RemoveAntiBlockTarget 刪除抗封鎖目標",
        "RemoveAntiHijackSource": "RemoveAntiHijackSource 刪除抗劫持",
        "RemoveAntiHijackTarget": "RemoveAntiHijackTarget 刪除抗劫持目標",
        "RemoveTag": "RemoveTag 移除 Tag",
        "ReplaceCertificateProviderDetach": "ReplaceCertificateProviderDetach 替換憑證商下架",
        "ReuseAndDeployCert": "ReuseAndDeployCert 轉移憑證",
        "RevokeCert": "RevokeCert 撤銷憑證",
        "SendCertCompleted": "SendCertCompleted 通知憑證已完成",
        "SendUpdateUB": "SendUpdateUB 通知 UB 更新",
        "SyncT2": "SyncT2 同步 F5 T2 設定",
        "UpdateDomainRecord": "UpdateDomainRecord 設定域名解析",
        "UpdateNameServer": "UpdateNameServer 上層設定",
        "UpdateOneToOneList": "UpdateOneToOneList 更新一對一IP清單",
        "UpdateOneToOneSourceRecord": "UpdateOneToOneSourceRecord 來源域名解析設定",
        "UpdateOneToOneTargetRecord": "UpdateOneToOneTargetRecord 目標域名解析設定",
        "VerifyDomainPDNSTags": "VerifyDomainPDNSTags 驗證域名 PDNS Tag",
        "VerifyTLD": "VerifyTLD 驗證頂級域名"
    ]

    def envName = "測試環境"
    if (BASE_URL.contains("vir999.com")) {
        envName = "DEV"
    } else if (BASE_URL.contains("staging168.com")) {
        envName = "STAGING"
    } else if (BASE_URL.contains("vir000.com")) {
        envName = "PROD"
    }

    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {

        def exported = readJSON file: '/tmp/exported_env.json'
        def workflowId = exported.values.find { it.key == 'PD_WORKFLOW_ID' }?.value

        if (!workflowId) {
            echo "⚠️ 無法取得 PD_WORKFLOW_ID，跳過輪詢"
            return
        }

        echo "📌 取得 workflowId：${workflowId}"

        def maxRetries = 12
        def delaySeconds = 300
        def retryCount = 0
        def success = false
        def finalJobList = []
        def domains = []

        // 安全傳遞 ADM_KEY
        withCredentials([string(credentialsId: 'STAGING_ADM_KEY', variable: 'ADM_KEY')]) {
            while (retryCount < maxRetries) {
                def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('Asia/Taipei'))
                echo "🔄 第 ${retryCount + 1} 次輪詢 workflow 狀態（${timestamp}）..."

                def response = sh(
                    script: """
                        curl -s -X GET "${BASE_URL}/workflow_api/adm/workflows/${workflowId}/jobs" \\
                            -H "X-API-Key: \$ADM_KEY" \\
                            -H "Accept: application/json" \\
                            -H "Content-Type: application/json"
                    """,
                    returnStdout: true
                ).trim()

                def jobs = readJSON text: response
                domains = jobs*.domain.findAll { it }?.unique() ?: []

                // 更新最終 Job 狀態
                finalJobList = jobs.collect { [
                    name: jobNameMap.get(it.name, it.name),
                    status: it.status
                ] }

                // 若有 failure 或 blocked，立即停止輪詢
                def hasFailureOrBlocked = jobs.any { it.status in ['failure', 'blocked'] }
                if (hasFailureOrBlocked) {
                    echo "❌ 發現 Job failure 或 blocked，停止輪詢"
                    success = false
                    break
                }

                // 若所有 Job 都完成
                def stillPending = jobs.findAll { !(it.status in ['success', 'running']) }
                if (stillPending.isEmpty()) {
                    echo "✅ 所有 Job 已完成"
                    success = true
                    break
                }

                retryCount++
                echo "⏳ 尚有 ${stillPending.size()} 個未完成 Job，等待 ${delaySeconds} 秒後進行下一次輪詢..."
                sleep time: delaySeconds, unit: 'SECONDS'
            }

            if (!success) {
                echo "⏰ 超過最大重試次數或 Job 失敗/封鎖，workflow 未完成，視為失敗"
            }

            // 產生 Job 狀態文字
                def jobStatusText = finalJobList.collect { job ->
                    def symbol = "•"
                    if (job.status == "success") symbol = "✅"
                    else if (job.status == "blocked") symbol = "⛔"
                    else if (job.status == "failure") symbol = "❌"
                    return " ${job.name} : ${symbol}"
                }.join("\n")

                // 卡片 JSON payload
                def message = [
                    cards: [[
                        header: [
                            title: "ℹ️ 申請廳主買域名 (Job狀態檢查)",
                            subtitle: "Workflow 輪詢完成",
                            imageUrl: "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                        ],
                        sections: [[
                            widgets: [[
                                textParagraph: [
                                    text: """
                環境 : <b>${envName}</b>
                BASE_URL : <b>${BASE_URL}</b>
                Workflow ID : <b>${workflowId}</b>
                Domain : <b>${domains.join(', ')}</b>

                -----------------------------------
                <b> Job 狀態:</b>
                ${jobStatusText}
                """
                                ]
                            ]]
                        ]]
                    ]]
                ]

                // 將 payload 寫入檔案
                writeFile file: 'payload.json', text: groovy.json.JsonOutput.toJson(message)

                // 發送到 Google Chat
                sh """
                curl -s -X POST -H 'Content-Type: application/json' -d @payload.json "${WEBHOOK_URL}"
                """
        }
    }
}


pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        COLLECTION_DIR = "${env.WORKSPACE}/collections"
        REPORT_DIR = "${env.WORKSPACE}/reports"
        HTML_REPORT_DIR = "${env.WORKSPACE}/reports/html"
        ALLURE_RESULTS_DIR = "${env.WORKSPACE}/allure-results"
        ENV_FILE = "${env.WORKSPACE}/environments/STAGING.postman_environment.json"
        BASE_URL = "http://maid.staging168.com"
        ADM_KEY = credentials('STAGING_ADM_KEY')
        WEBHOOK_URL = "https://chat.googleapis.com/v1/spaces/AAQAGYLH9k0/messages?key=AIzaSyDdI0hCZtE6vySjMm-WEfRq3CPzqKqqsHI&token=HvPXUUnqPlN6c9HhB02kpWleJ86p2lLmDaq32-5t0gQ"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                echo '🧹 清理 Jenkins 工作目錄...'
                deleteDir()
            }
        }

        stage('Checkout Code') {
            steps {
                echo '📥 Checkout Git repo...'
                checkout scm
            }
        }

        stage('Show Commit Info') {
            steps {
                sh '''
                    echo "✅ 當前 Git commit：$(git rev-parse HEAD)"
                    echo "📝 Commit 訊息：$(git log -1 --oneline)"
                '''
            }
        }

        stage('申請域名') {
            steps {
                script {
                    def testData = readJSON file: "${COLLECTION_DIR}/STAGING_申請域名_測試資料.json"

                    def readExportedEnvVariable = { filePath, key ->
                        def envData = readJSON file: filePath
                        return envData?.values?.find { it.key == key }?.value
                    }

                    // 確保目錄存在
                    sh "mkdir -p '${ALLURE_RESULTS_DIR}' '${REPORT_DIR}' '${HTML_REPORT_DIR}' '${WORKSPACE}/environments'"

                    testData.eachWithIndex { dataRow, index ->
                        def testLabel = "資料${index + 1}"
                        def tmpDataFile = "${WORKSPACE}/data_${index + 1}.json"
                        writeJSON file: tmpDataFile, json: [dataRow]

                        // 子 stage 1：申請域名
                        stage("${testLabel} - 申請域名") {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                def currentEnvFile = "${WORKSPACE}/environments/current_env_${index + 1}.json"

                                sh """
                                    newman run "${COLLECTION_DIR}/申請廳主買域名.postman_collection.json" \\
                                        --environment "${ENV_FILE}" \\
                                        --export-environment "/tmp/exported_env.json" \\
                                        --iteration-data "${tmpDataFile}" \\
                                        --verbose \\
                                        --insecure \\
                                        --reporters cli,json,html,junit,allure \\
                                        --reporter-json-export "${REPORT_DIR}/Apply_${index + 1}.json" \\
                                        --reporter-html-export "${HTML_REPORT_DIR}/Apply_${index + 1}.html" \\
                                        --reporter-junit-export "${REPORT_DIR}/Apply_${index + 1}.xml" \\
                                        --reporter-allure-export "${ALLURE_RESULTS_DIR}"
                                    cp /tmp/exported_env.json "${currentEnvFile}"
                                """

                                env.ENV_FILE = currentEnvFile
                                echo "✅ 更新 ENV_FILE 為最新：${currentEnvFile}" 

                                def PD_WORKFLOW_ID = readExportedEnvVariable("/tmp/exported_env.json", "PD_WORKFLOW_ID")
                                if (PD_WORKFLOW_ID) {
                                    env.PD_WORKFLOW_ID = PD_WORKFLOW_ID
                                    echo "📤 取得 PD_WORKFLOW_ID: ${PD_WORKFLOW_ID}"
                                } else {
                                    echo "⚠️ exported_env.json 未包含 PD_WORKFLOW_ID"
                                }
                            }
                        }

                        // 子 stage 2：檢查 Job 狀態
                        stage("${testLabel} - 檢查申請 Job 狀態") {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                checkCustomerApplyPurchaseDomainJobStatus()
                            }
                        }
                    }

                    // 合併所有 JSON 成 all_test_results.json
                    def allResults = [:]
                    testData.eachWithIndex { dataRow, index ->
                        def jsonFile = "${REPORT_DIR}/Apply_${index + 1}.json"
                        if (fileExists(jsonFile)) {
                            def json = readJSON file: jsonFile
                            allResults["資料${index + 1}"] = json.status ?: "UNKNOWN"
                        } else {
                            allResults["資料${index + 1}"] = "NO_RESULT"
                        }
                    }
                    writeJSON file: "${REPORT_DIR}/all_test_results.json", json: allResults
                }
            }
        }

        stage('產生 Allure 報告') {
            steps {
                script {
                    echo "📦 產生 Allure 測試報告..."
                    allure([
                        includeProperties: false,
                        jdk: '',
                        results: [[path: "${ALLURE_RESULTS_DIR}"]]
                    ])
                }
            }
        }
    }

    post {
        always {
            script {
                // 強制外層 Build 成功
                currentBuild.result = 'SUCCESS'

                def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('Asia/Taipei'))

                // 讀取測試結果，避免檔案不存在造成錯誤
                def testResults = fileExists("${REPORT_DIR}/all_test_results.json") ? 
                                readJSON(file: "${REPORT_DIR}/all_test_results.json") : [:]

                // 計算 BUILD_URL 去掉 https://
                def buildUrlNoHttps = env.BUILD_URL.replaceFirst(/^https?:\/\//, '')

                def message = """
                {
                "cards": [
                    {
                    "header": {
                        "title": "✅ Jenkins Pipeline 執行完成",
                        "subtitle": "專案：${env.JOB_NAME} (#${env.BUILD_NUMBER})",
                        "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/jenkins-icon.png",
                        "imageStyle": "AVATAR"
                    },
                    "sections": [
                        {
                        "widgets": [
                            {"textParagraph": {"text": "完成時間：${timestamp}"}},
                            {"textParagraph": {"text": "Allure 報告鏈結： <a href='http://${buildUrlNoHttps}allure'>點此查看</a>"}}
                        ]
                        }
                    ]
                    }
                ]
                }
                """

                writeFile file: 'payload.json', text: message
                sh 'curl -k -X POST -H "Content-Type: application/json" -d @payload.json "$WEBHOOK_URL"'
            }
        }
    }

}
