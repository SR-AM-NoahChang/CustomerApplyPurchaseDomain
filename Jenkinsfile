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
                    error("❌ 無法從 /tmp/exported_env.json 中取得 PD_WORKFLOW_ID")
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
                            curl -s -X GET '${BASE_URL}/workflow_api/adm/workflows/${workflowId}/jobs' \\
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
                            script: """
                                curl -s -X PATCH '${BASE_URL}/workflow_api/adm/jobs/status' \\
                                    -H 'Accept: application/json' \\
                                    -H 'X-API-Key: ${ADM_KEY}' \\
                                    -H 'Content-Type: application/json' \\
                                    -d '{"id":${job.job_id},"status":"success"}'
                            """,
                            returnStdout: true
                        ).trim()
                        echo "📬 PATCH 回應：${patchResponse}"
                        patchedJobs << "- ${jobNameMap.get(job.name, job.name)}"
                    }

                    def recheckResponse = sh(
                        script: """
                            curl -s -X GET '${BASE_URL}/workflow_api/adm/workflows/${workflowId}/jobs' \\
                                -H "X-API-Key: ${ADM_KEY}" \\
                                -H "Accept: application/json" \\
                                -H "Content-Type: application/json"
                        """,
                        returnStdout: true
                    ).trim()

                    def recheckJobs = readJSON text: recheckResponse
                    def remainingFailures = recheckJobs.findAll { it.status == 'failure' }
                    def remainingBlocked = recheckJobs.findAll { it.status == 'blocked' }
                    def stillPending = recheckJobs.findAll { !(it.status in ['success', 'running', 'failure', 'blocked']) }

                    if (remainingFailures || remainingBlocked) {
                        def failedDetails = remainingFailures.collect { "- ${jobNameMap.get(it.name, it.name)} - ❌failure" }
                        def blockedDetails = remainingBlocked.collect { "- ${jobNameMap.get(it.name, it.name)} - 🔒blocked" }
                        def allIssues = (failedDetails + blockedDetails).join("\\n")

                        def message = """{
                            "cards": [{
                                "header": {
                                    "title": "🚨 Jenkins - 廳主買域名項目資料 (Job狀態檢查 - 異常)",
                                    "subtitle": "Workflow ID: ${workflowId}",
                                    "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                                },
                                "sections": [{
                                    "widgets": [
                                        {
                                            "textParagraph": {
                                                "text": "🌐 環境: <b>${envName}</b>\\n🔗 BASE_URL: ${BASE_URL}"
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
                                        },
                                        {
                                            "textParagraph": {
                                                "text": "<b>🚨 異常 Job：</b><br>${allIssues.replace('"', '\\"').replaceAll("\\n", "<br>")}"
                                            }
                                        }
                                    ]
                                }]
                            }]
                        }"""

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
                            def successPatchedMessage = """{
                                "cards": [{
                                    "header": {
                                        "title": "✅ 廳主買域名項目資料 (Job狀態檢查) 成功",
                                        "subtitle": "部分 Job 有人工 PATCH 處理",
                                        "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                                    },
                                    "sections": [{
                                        "widgets": [
                                            {
                                                "textParagraph": {
                                                    "text": "🌐 環境: <b>${envName}</b>\\n🔗 BASE_URL: ${BASE_URL}"
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
                                                    "text": "<b>🛠 有人工 PATCH Job：</b><br>${patchedJobs.join("<br>").replace('"', '\\"')}"
                                                }
                                            }
                                        ]
                                    }]
                                }]
                            }"""

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

                    def timeoutMessage = """{
                        "cards": [{
                            "header": {
                                "title": "⏰ 廳主買域名項目資料 (Job狀態檢查) 輪詢超時失敗",
                                "subtitle": "Workflow Timeout",
                                "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                            },
                            "sections": [{
                                "widgets": [
                                    {
                                        "textParagraph": {
                                            "text": "🌐 環境: <b>${envName}</b>\\n🔗 BASE_URL: ${BASE_URL}"
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
                    }"""

                    writeFile file: 'payload.json', text: timeoutMessage

                    withEnv(["WEBHOOK_URL=${WEBHOOK_URL}"]) {
                        sh 'curl -k -X POST -H "Content-Type: application/json" -d @payload.json "$WEBHOOK_URL"'
                    }

                    error("⏰ Workflow Timeout，已通知 webhook")
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
            
                    def maxRetries = 10
                    def delaySeconds = 300
                    def retryCount = 0
                    def success = false
            
                    while (retryCount < maxRetries) {
                    def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('Asia/Taipei'))
                    echo "🔄 第 ${retryCount + 1} 次輪詢 workflow 狀態（${timestamp}）..."
            
                    def response = sh(
                        script: """
                        curl -s -X GET '${BASE_URL}/workflow_api/adm/workflows/${workflowId}/jobs' \\
                            -H "X-API-Key: ${ADM_KEY}" \\
                            -H "Accept: application/json" \\
                            -H "Content-Type: application/json"
                        """,
                        returnStdout: true
                    ).trim()
            
                    echo "🔎 API 回應：${response}"
            
                    def json = readJSON text: response
            
                    def failedJobs = json.findAll { it.status == 'failure' }
                    def blockedJobs = json.findAll { it.status == 'blocked' }
                    def pendingJobs = json.findAll { !(it.status in ['success', 'running', 'failure', 'blocked']) }
            
                    if (failedJobs || blockedJobs) {
                        def failedDetails = failedJobs.collect { "- ${jobNameMap.get(it.name, it.name)} - ❌failure" }
                        def blockedDetails = blockedJobs.collect { "- ${jobNameMap.get(it.name, it.name)} - 🔒blocked" }
                        def allIssues = (failedDetails + blockedDetails).join("\\n")
            
                        echo "🚨 偵測到異常 Job：\n${allIssues.replace('\\n', '\n')}"
            
                    writeFile file: 'payload.json', text: """{
                        "cards": [{
                            "header": {
                            "title": "🚨 刪除域名項目資料 (Job狀態檢查 - 異常)",
                            "subtitle": "Workflow: ${workflowId}",
                            "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                            },
                            "sections": [
                                {
                                "widgets": [
                                    {
                                    "keyValue": {
                                        "topLabel": "🌐 測試環境 :",
                                        "content": "${envName}"
                                    }
                                    },
                                    {
                                    "keyValue": {
                                        "topLabel": "🔗 BASE_URL :",
                                        "content": "${BASE_URL}"
                                    }
                                    },
                                    {
                                    "textParagraph": {
                                        "text": "────────────────────────────"
                                    }
                                    },
                                    {
                                    "textParagraph": {
                                        "text": "<b>自動化Job :</b><br>${allIssues.replace('"', '\\"').replaceAll('\\n', '<br>')}"
                                    }
                                    }

                                ]
                                }
                            ]
                            }]
                        }]
                        }"""
            
                        withEnv(["WEBHOOK_URL=${WEBHOOK_URL}"]) {
                        sh 'curl -k -X POST -H "Content-Type: application/json" -d @payload.json "$WEBHOOK_URL"'
                        }
            
                        error("❌ 偵測到異常 Job（已通知 webhook）")
                    }
            
                    if (pendingJobs.isEmpty()) {
                        echo "✅ 所有 Job 已完成，提前結束輪詢"
                        success = true
                        break
                    }
            
                    retryCount++
                    echo "⏳ 尚有 ${pendingJobs.size()} 個未完成 Job，等待 ${delaySeconds} 秒後進行下一次輪詢..."
                    sleep time: delaySeconds, unit: 'SECONDS'
                    }
            
                    if (!success) {
                    echo "⏰ 超過最大重試次數（${maxRetries} 次），workflow 未完成"
            
                    writeFile file: 'payload.json', text: """{
                        "cards": [{
                        "header": {
                            "title": "⏰ 除域名項目資料 (Job狀態檢查) 輪詢超時失敗",
                            "subtitle": "Workflow Timeout",
                            "imageUrl": "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/postman-icon.png"
                        },
                        "sections": [{
                            "widgets": [
                            {
                                "textParagraph": {
                                "text": "🌐 測試環境: <b>${envName}</b>\\n🔗 BASE_URL: ${BASE_URL}"
                                }
                            },
                            {
                                "textParagraph": {
                                    "text": "────────────────────────────"
                                }
                                },
                            {
                                "keyValue": {
                                "topLabel": "Workflow ID",
                                "content": "${workflowId}"
                                }
                            }
                            ]
                        }]
                        }]
                    }"""
            
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
    ENV_FILE = "${env.WORKSPACE}/environments/DEV.postman_environment.json"
    WEBHOOK_URL = credentials('GOOGLE_CHAT_WEBHOOK')
    BASE_URL = "http://maid-cloud.vir999.com"
    ADM_KEY = credentials('DEV_ADM_KEY')
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

    stage('申請廳主買域名') {
      steps {
        script {
          def testData = readJSON file: "${COLLECTION_DIR}/申請廳主買域名_testdata.json"

          // 定義函式：讀取 exported_env.json 裡指定的 key 值
          def readExportedEnvVariable = { filePath, key ->
            def envData = readJSON file: filePath
            def value = null
            if (envData?.values) {
              envData.values.each { item ->
                if (item.key == key) {
                  value = item.value
                }
              }
            }
            return value
          }

          testData.eachWithIndex { dataRow, index ->
            def testLabel = "資料${index + 1}"
            def tmpDataFile = "${WORKSPACE}/data_${index + 1}.json"

            writeJSON file: tmpDataFile, json: [dataRow]

            stage("${testLabel} - 申請域名") {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh """
                  newman run '${COLLECTION_DIR}/申請廳主買域名.postman_collection.json' \
                    --environment "${ENV_FILE}" \
                    --export-environment "/tmp/exported_env.json" \
                    --iteration-data "${tmpDataFile}" \
                    --insecure \
                    --reporters cli,json,html,junit,allure \
                    --reporter-json-export "${REPORT_DIR}/Apply_${index + 1}.json" \
                    --reporter-html-export "${HTML_REPORT_DIR}/Apply_${index + 1}.html" \
                    --reporter-junit-export "${REPORT_DIR}/Apply_${index + 1}.xml" \
                    --reporter-allure-export "${ALLURE_RESULTS_DIR}"
                """
              }
              // 讀取 workflow_id 並設定成環境變數
              if (fileExists("/tmp/exported_env.json")) {
                def PD_WORKFLOW_ID = readExportedEnvVariable("/tmp/exported_env.json", "PD_WORKFLOW_ID")
                echo "📤 從 exported_env.json 讀取的 PD_WORKFLOW_ID: ${PD_WORKFLOW_ID}"
                if (PD_WORKFLOW_ID) {
                  env.PD_WORKFLOW_ID = PD_WORKFLOW_ID
                } else {
                  echo "⚠️ exported_env.json 未包含 PD_WORKFLOW_ID"
                }
              } else {
                echo "❌ 找不到 exported_env.json"
              }
            }

            stage("${testLabel} - 檢查申請 Job 狀態") {
              checkCustomerApplyPurchaseDomainJobStatus()
            }

            stage("${testLabel} - 刪除域名") {
              def collectionPath = "${COLLECTION_DIR}/清除測試域名.postman_collection.json"
              if (fileExists(collectionPath)) {
                echo "🧹 執行清除測試域名 Collection"
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                  sh """
                    newman run "${collectionPath}" \
                      --environment "${ENV_FILE}" \
                      --export-environment "/tmp/exported_env.json" \
                      --insecure \
                      --reporters cli,json,html,junit,allure \
                      --reporter-json-export "${REPORT_DIR}/Delete_${index + 1}.json" \
                      --reporter-html-export "${HTML_REPORT_DIR}/Delete_${index + 1}.html" \
                      --reporter-junit-export "${REPORT_DIR}/Delete_${index + 1}.xml" \
                      --reporter-allure-export "${ALLURE_RESULTS_DIR}"
                  """
                }
              }
            }

            stage("${testLabel} - 檢查刪除 Job 狀態") {
              DeleteDomainJobStatus()
            }
          }
        }
      }
    }

    stage('Publish HTML Reports') {
      steps {
        script {
          def files = sh(script: "ls ${HTML_REPORT_DIR}", returnStdout: true).trim().split('\n')
          def indexHtml = new File("${HTML_REPORT_DIR}/01_report.html")
          indexHtml.text = "<html><body><h1>測試報告列表</h1><ul>\n"
          files.each { file ->
            if (file.endsWith('.html')) {
              indexHtml << "<li><a href='${file}'>${file}</a></li>\n"
            }
          }
          indexHtml << "</ul></body></html>\n"
        }

        publishHTML(target: [
          reportDir: "${HTML_REPORT_DIR}",
          reportFiles: 'CustomerApplyPurchaseDomain_report.html',
          reportName: '整合 HTML 測試報告',
          allowMissing: true,
          alwaysLinkToLastBuild: true,
          keepAll: true
        ])
      }
    }

    stage('Allure Report') {
      steps {
        allure([
          includeProperties: false,
          jdk: '',
          results: [[path: "${ALLURE_RESULTS_DIR}"]]
        ])
      }
    }
  }

  post {
    always {
      script {
        def buildResult = currentBuild.currentResult
        def statusEmoji = buildResult == 'SUCCESS' ? '✅' :
                          buildResult == 'FAILURE' ? '❌' :
                          buildResult == 'UNSTABLE' ? '⚠️' :
                          buildResult == 'ABORTED' ? '🚫' : '❔'

        def statusText = buildResult == 'SUCCESS' ? '成功(SUCCESS)' :
                         buildResult == 'FAILURE' ? '失敗(FAILURE)' :
                         buildResult == 'UNSTABLE' ? '不穩定(UNSTABLE)' :
                         buildResult == 'ABORTED' ? '已終止(ABORTED)' : '未知'

        def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('Asia/Taipei'))

        def message = """
        {
          \"cards\": [
            {
              \"header\": {
                \"title\": \"${statusEmoji} Jenkins Pipeline 執行結果\",
                \"subtitle\": \"專案：${env.JOB_NAME} (#${env.BUILD_NUMBER})\",
                \"imageUrl\": \"https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/jenkins-icon.png\",
                \"imageStyle\": \"AVATAR\"
              },
              \"sections\": [
                {
                  \"widgets\": [
                    {
                      \"keyValue\": {
                        \"topLabel\": \"狀態\",
                        \"content\": \"${statusText}\"
                      }
                    },
                    {
                      \"keyValue\": {
                        \"topLabel\": \"完成時間\",
                        \"content\": \"${timestamp}\"
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
        withEnv(["WEBHOOK=${WEBHOOK_URL}"]) {
          sh 'curl -k -X POST -H "Content-Type: application/json" -d @payload.json "$WEBHOOK"'
        }
      }
    }
  }
}
