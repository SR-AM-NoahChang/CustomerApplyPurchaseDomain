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

        withEnv(["ADM_KEY=${ADM_KEY}"]) {
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
                finalJobList = jobs.collect {
                    "- ${jobNameMap.get(it.name, it.name)}：${it.status}"
                }

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

            // 發送 Webhook（顯示最終狀態）
            def message = """
            {
                "cards": [{
                    "header": {
                        "title": "ℹ️申請廳主買域名 (Job狀態檢查)",
                        "subtitle": "Workflow 輪詢完成",
                        "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                    },
                    "sections": [{
                        "widgets": [{
                            "textParagraph": {
                                "text": "🌐 環境: <b>${envName}</b>\\n🔗 BASE_URL: ${BASE_URL}\\n🆔 Workflow_Id: <b>${workflowId}</b>\\n🏷️ Domain: <b>${domains.join(', ')}</b>\\n-----------------------------------\\n Job 狀態：<br>${finalJobList.join('<br>').replace('\"','\\\\\"')}"
                            }
                        }]
                    }]
                }]
            }
            """
            writeFile file: 'payload.json', text: message
            sh 'curl -k -X POST -H "Content-Type: application/json" -d @payload.json "${WEBHOOK_URL}"'
        }
    }
}


// 取得刪除域名項目資料 (輪詢Job狀態檢查)
def DeleteDomainJobStatus() {
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
        def workflowId = exported.values.find { it.key == 'DD_WORKFLOW_ID' }?.value
       
        if (!workflowId) {
            error("❌ 無法從 /tmp/exported_env.json 中取得 DD_WORKFLOW_ID")
        }

        echo "📌 取得 workflowId：${workflowId}"

        def maxRetries = 20
        def delaySeconds = 180
        def retryCount = 0
        def success = false
        def initialJobList = ""
        def patchedJobs = []

        while (retryCount < maxRetries) {
                def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('Asia/Taipei'))
                echo "🔄 第 ${retryCount + 1} 次輪詢 workflow 狀態（${timestamp}）..."

                def response = sh(
                    script: """
                        curl -s -X GET "${BASE_URL}/workflow_api/adm/workflows/${workflowId}/jobs" \\
                            -H "X-API-Key: ${ADM_KEY}" \\
                            -H "Accept: application/json" \\
                            -H "Content-Type: application/json"
                    """,
                    returnStdout: true
                ).trim()

                    def json = readJSON text: response

                    if (retryCount == 0) {
                        initialJobList = json.collect {
                            "- ${jobNameMap.get(it.name, it.name)} - ${it.status}"
                        }.join("\\n")
                    }

                    def failedJobs = json.findAll { it.status == 'failure' }
                    def blockedJobs = json.findAll { it.status == 'blocked' }
                    def pendingJobs = json.findAll { !(it.status in ['success', 'running', 'failure', 'blocked']) }

                    for (job in failedJobs) {
                        echo "🛠 人工調整 Job ID: ${job.job_id} (${job.name}) 為 success"
                        def patchResponse = sh(
                            script: '''
                                curl -s -X PATCH '${BASE_URL}/workflow_api/adm/jobs/status' \\
                                    -H 'Accept: application/json' \\
                                    -H 'X-API-Key: ${ADM_KEY}' \\
                                    -H 'Content-Type: application/json' \\
                                    -d '{"id":${job.job_id},"status":"success"}'
                            ''',
                            returnStdout: true
                        ).trim()
                        echo "📬 PATCH 回應：${patchResponse}"
                        patchedJobs << "- ${jobNameMap.get(job.name, job.name)}"
                    }

                    def recheckResponse = sh(
                        script: '''
                            curl -s -X GET '${BASE_URL}/workflow_api/adm/workflows/${workflowId}/jobs' \\
                                -H "X-API-Key: ${ADM_KEY}" \\
                                -H "Accept: application/json" \\
                                -H "Content-Type: application/json"
                        ''',
                        returnStdout: true
                    ).trim()

                    def recheckJobs = readJSON text: recheckResponse
                    def remainingFailures = recheckJobs.findAll { it.status == 'failure' }
                    def remainingBlocked = recheckJobs.findAll { it.status == 'blocked' }
                    def stillPending = recheckJobs.findAll { !(it.status in ['success', 'running', 'failure', 'blocked']) }

                    if (remainingFailures || remainingBlocked) {
                        def failedDetails = remainingFailures.collect {
                            def jobName = jobNameMap.get(it.name, it.name) 
                            def msg = it.message?.trim()
                            "- ${jobName}${msg ? "（${msg}）" : ""} - ❌"
                        }
                        def blockedDetails = remainingBlocked.collect {
                            def jobName = jobNameMap.get(it.name, it.name)
                            def msg = it.message?.trim()
                            "- ${jobName}${msg ? "（${msg}）" : ""} - 🔒"
                        }

                        def allIssues = (failedDetails + blockedDetails).join("\\n")

                        def domains = recheckJobs*.domain.findAll { it }
                        def uniqueDomains = domains.unique().join(', ')

                        def message = '''{
                            "cards": [{
                                "header": {
                                    "title": "🚨 刪除域名項目資料 (Job狀態檢查 - 異常)",
                                    "subtitle": "Workflow Failed",
                                    "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                                },
                                "sections": [{
                                    "widgets": [
                                        {
                                            "textParagraph": {
                                                "text": "🌐 環境: <b>${envName}</b>\\n🔗 BASE_URL: ${BASE_URL}\\n🆔 Workflow_Id: <b>${workflowId}</b>\\n🏷️ Domain: <b>${uniqueDomains}</b>"
                                            }
                                        },
                                        {
                                            "textParagraph": {
                                                "text": "────────────────────────────"
                                            }
                                        },
                                        {
                                            "textParagraph": {
                                                "text": "<b>🔍 初始 Job 狀態：</b><br>${initialJobList.replace('"', '\\"').replaceAll("\\n", "<br>")}"
                                            }
                                        },
                                         {
                                            "textParagraph": {
                                                "text": "────────────────────────────"
                                            }
                                        },
                                        {
                                            "textParagraph": {
                                                "text": "<b>🛠 已 PATCH Job：</b><br>${patchedJobs.isEmpty() ? "（無）" : patchedJobs.join("<br>").replace('"', '\\"')}"
                                            }
                                        },
                                        {
                                            "textParagraph": {
                                                "text": "────────────────────────────"
                                            }
                                        },
                                        {
                                            "textParagraph": {
                                                "text": "<b>🚨 異常 Job：</b><br>${allIssues.replace('"', '\\"').replaceAll("\\n", "<br>")}"
                                            }
                                        }
                                    ]
                                }]
                            }]
                        }'''

                        writeFile file: 'payload.json', text: message

                        withEnv(["WEBHOOK=${WEBHOOK_URL}"]) {
                             sh 'curl -k -X POST -H "Content-Type: application/json" -d @payload.json "$WEBHOOK"'
                        }

                        error("❌ 異常 Job 偵測後仍存在（已通知 webhook）") 
                    }

                    if (stillPending.isEmpty()) {
                        echo "✅ 所有 Job 已完成，提前結束輪詢"
                        success = true

                        // ✅ 新增成功時 webhook 通知
                        if (!patchedJobs.isEmpty()) {
                            def domains = recheckJobs*.domain.findAll { it }
                            def uniqueDomains = domains.unique().join(', ')
                            def successPatchedMessage = '''{
                                "cards": [{
                                    "header": {
                                        "title": "✅ 刪除域名項目資料 (Job狀態檢查) 成功",
                                        "subtitle": "部分 Job 有人工 PATCH 處理",
                                        "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                                    },
                                    "sections": [{
                                        "widgets": [
                                            {
                                                "textParagraph": {
                                                    "text": "🌐 環境: <b>${envName}</b>\\n🔗 BASE_URL: ${BASE_URL}\\n🆔 Workflow_Id: <b>${workflowId}</b>\\n🏷️ Domain: <b>${uniqueDomains}</b>"
                                            }
                                            },
                                            {
                                                "textParagraph": {
                                                    "text": "────────────────────────────"
                                                }
                                            },
                                            {
                                                "textParagraph": {
                                                    "text": "<b>🔍 初始 Job 狀態：</b><br>${initialJobList.replace('"', '\\"').replaceAll("\\n", "<br>")}"
                                                }
                                            },
                                            {
                                                "textParagraph": {
                                                    "text": "────────────────────────────"
                                                }
                                            },
                                            {
                                                "textParagraph": {
                                                    "text": "<b>🛠 有人工 PATCH Job：</b><br>${patchedJobs.join("<br>").replace('"', '\\"')}"
                                                }
                                            }
                                        ]
                                    }]
                                }]
                            }'''

                            writeFile file: 'payload.json', text: successPatchedMessage

                            withEnv(["WEBHOOK=${WEBHOOK_URL}"]) {
                                sh 'curl -k -X POST -H "Content-Type: application/json" -d @payload.json "$WEBHOOK"'
                            }

                        }

                        break
                    }

                    retryCount++
                    echo "⏳ 尚有 ${stillPending.size()} 個未完成 Job，等待 ${delaySeconds} 秒後進行下一次輪詢..."
                    sleep time: delaySeconds, unit: 'SECONDS'
                }
            if (!success) {
                    echo "⏰ 超過最大重試次數（${maxRetries} 次），workflow 未完成"

                    def timeoutMessage = '''{
                        "cards": [{
                            "header": {
                                "title": "⏰ 刪除域名項目資料 (Job狀態檢查) 輪詢超時失敗",
                                "subtitle": "Workflow Timeout",
                                "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                            },
                            "sections": [{
                                "widgets": [
                                    {
                                        "textParagraph": {
                                            "text": "🌐 環境: <b>${envName}</b>\\n🔗 BASE_URL: ${BASE_URL}\\n🆔 Workflow_Id: <b>${workflowId}</b>"
                                        }
                                    },
                                    {
                                        "textParagraph": {
                                            "text": "────────────────────────────"
                                        }
                                    },
                                    {
                                        "textParagraph": {
                                            "text": "<b>🔍 初始 Job 狀態：</b><br>${initialJobList.replace('"', '\\"').replaceAll("\\n", "<br>")}"
                                        }
                                    },
                                    {
                                        "textParagraph": {
                                            "text": "<b>🛠 已 PATCH Job：</b><br>${patchedJobs.isEmpty() ? "（無）" : patchedJobs.join("<br>").replace('"', '\\"')}"
                                        }
                                    }
                                ]
                            }]
                        }]
                    }'''

                    writeFile file: 'payload.json', text: timeoutMessage

                    withEnv(["WEBHOOK_URL=${WEBHOOK_URL}"]) {
                        sh 'curl -k -X POST -H "Content-Type: application/json" -d @payload.json "$WEBHOOK_URL"'
                    }

                    error("⏰ Workflow Timeout，已通知 webhook")
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
                    bash -c "echo \"✅ 當前 Git commit：$(git rev-parse HEAD)\"; \
                             echo \"📝 Commit 訊息：$(git log -1 --oneline)\""
                '''
            }
        }

        stage('申請廳主買域名') {
            steps {
                script {
                    def testData = readJSON file: "${COLLECTION_DIR}/STAGING_申請域名_測試資料.json"

                    def readExportedEnvVariable = { filePath, key ->
                        def envData = readJSON file: filePath
                        return envData?.values?.find { it.key == key }?.value
                    }

                    // 逐筆測試資料執行
                    testData.eachWithIndex { dataRow, index ->
                        def testLabel = "資料${index + 1}"
                        def tmpDataFile = "${WORKSPACE}/data_${index + 1}.json"
                        writeJSON file: tmpDataFile, json: [dataRow]

                        stage("${testLabel} - 申請域名") {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                def currentEnvFile = "${WORKSPACE}/environments/current_env_${index + 1}.json"

                                sh """
                                    mkdir -p ${WORKSPACE}/environments
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

                        stage("${testLabel} - 檢查申請 Job 狀態") {
                            checkCustomerApplyPurchaseDomainJobStatus()
                        }
                    }
                }
            }
        }

    }

    post {
        always {
            script {
                currentBuild.result = 'SUCCESS'
                def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('Asia/Taipei'))
                def testResults = readJSON file: "${REPORT_DIR}/all_test_results.json"
                def orderedKeys = testResults.keySet().toList()

                def resultLines = orderedKeys.collect { domain ->
                    def result = testResults[domain]
                    def emoji = (result == "SUCCESS") ? "🟢" :
                                (result == "FAILURE") ? "🔴" : "⚪️"
                    return "${emoji} ${domain}：${result}"
                }.join("<br>")

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
                                        {
                                            "keyValue": {
                                                "topLabel": "整體狀態",
                                                "content": "成功 (SUCCESS)"
                                            }
                                        },
                                        {
                                            "keyValue": {
                                                "topLabel": "完成時間",
                                                "content": "${timestamp}"
                                            }
                                        },
                                        {
                                            "textParagraph": {
                                                "text": "<b>各筆資料結果：</b><br>${resultLines}"
                                            }
                                        }
                                    ]
                                }
                            ]
                        }
                    ]
                }
                """

                writeFile file: 'payload.json', text: message
                sh """
                    curl -k -X POST \
                        -H "Content-Type: application/json" \
                        -d @payload.json \
                        "$WEBHOOK_URL"
                """

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




