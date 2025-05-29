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
          catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            sh '''
              newman run "${COLLECTION_DIR}/申請廳主買域名.postman_collection.json" \
                --environment "${ENV_FILE}" \
                --export-environment "/tmp/exported_env.json" \
                --insecure \
                --reporters cli,json,html,junit,allure \
                --reporter-json-export "${REPORT_DIR}/CustomerApplyPurchaseDomain_report.json" \
                --reporter-html-export "${HTML_REPORT_DIR}/CustomerApplyPurchaseDomain_report.html" \
                --reporter-junit-export "${REPORT_DIR}/CustomerApplyPurchaseDomain_report.xml" \
                --reporter-allure-export "allure-results"
            '''
          }
        }
      }
    }

    stage('取得廳主買域名項目資料 (Job狀態檢查)') {
      steps {
        script {
          def jobNameMap = [
            "AddTag": "AddTag（新增 Tag）",
            "AddThirdLevelRandom": "AddThirdLevelRandom（設定三級亂數）",
            "AttachAntiBlockTarget": "AttachAntiBlockTarget（新增抗封鎖目標）",
            "AttachAntiHijackSource": "AttachAntiHijackSource（新增抗劫持）",
            "AttachAntiHijackTarget": "AttachAntiHijackTarget（新增抗劫持目標）",
            "CheckDomainBlocked": "CheckDomainBlocked（檢查封鎖）",
            "CheckPurchaseDeployCertificateStatus": "CheckPurchaseDeployCertificateStatus（檢查購買部署憑證結果）",
            "CheckWorkflowApplication": "CheckWorkflowApplication（檢查自動化申請）",
            "DeleteDomainRecord": "DeleteDomainRecord（刪除解析）",
            "DetachAntiBlockSource": "DetachAntiBlockSource（撤下抗封鎖）",
            "DetachAntiBlockTarget": "DetachAntiBlockTarget（撤下抗封鎖目標）",
            "DetachAntiHijackSource": "DetachAntiHijackSource（撤下抗劫持）",
            "DetachAntiHijackTarget": "DetachAntiHijackTarget（撤下抗劫持目標）",
            "InformDomainInfringement": "InformDomainInfringement（通知侵權網址）",
            "MergeErrorRecord": "MergeErrorRecord（檢查異常地區合併規則）",
            "PurchaseAndDeployCert": "PurchaseAndDeployCert（購買與部署憑證）",
            "PurchaseDomain": "PurchaseDomain（購買域名）",
            "RecheckARecordResolution": "RecheckARecordResolution（複檢域名 A 紀錄解析）",
            "RecheckCert": "RecheckCert（複檢憑證）",
            "RecheckDomainResolution": "RecheckDomainResolution（複檢域名）",
            "RecheckThirdLevelRandom": "RecheckThirdLevelRandom（複檢三級亂數）",
            "RemoveAntiBlock": "RemoveAntiBlock（刪除抗封鎖）",
            "RemoveAntiBlockTarget": "RemoveAntiBlockTarget（刪除抗封鎖目標）",
            "RemoveAntiHijackSource": "RemoveAntiHijackSource（刪除抗劫持）",
            "RemoveAntiHijackTarget": "RemoveAntiHijackTarget（刪除抗劫持目標）",
            "RemoveTag": "RemoveTag（移除 Tag）",
            "ReplaceCertificateProviderDetach": "ReplaceCertificateProviderDetach（替換憑證商下架）",
            "ReuseAndDeployCert": "ReuseAndDeployCert（轉移憑證）",
            "RevokeCert": "RevokeCert（撤銷憑證）",
            "SendCertCompleted": "SendCertCompleted（通知憑證已完成）",
            "SendUpdateUB": "SendUpdateUB（通知 UB 更新）",
            "SyncT2": "SyncT2（同步 F5 T2 設定）",
            "UpdateDomainRecord": "UpdateDomainRecord（設定域名解析）",
            "UpdateNameServer": "UpdateNameServer（上層設定）",
            "UpdateOneToOneList": "UpdateOneToOneList（更新一對一IP清單）",
            "UpdateOneToOneSourceRecord": "UpdateOneToOneSourceRecord（來源域名解析設定）",
            "UpdateOneToOneTargetRecord": "UpdateOneToOneTargetRecord（目標域名解析設定）",
            "VerifyDomainPDNSTags": "VerifyDomainPDNSTags（驗證域名 PDNS Tag）",
            "VerifyTLD": "VerifyTLD（驗證頂級域名）"
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
    
            def maxRetries = 10
            def delaySeconds = 300
            def retryCount = 0
            def success = false
    
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
                      "title": "🚨 取得廳主買域名項目資料 (Job狀態檢查 - 異常)",
                      "subtitle": "Workflow: ${workflowId}",
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
                            "text": "${allIssues.replace('"', '\\"')}"
                          }
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
                    "title": "⏰ 取得廳主買域名項目資料 (Job狀態檢查) 輪詢超時失敗",
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
      }
    }

    stage('刪除域名') {
      steps {
        script {
          def collectionPath = "${COLLECTION_DIR}/清除測試域名.postman_collection.json"
          if (fileExists(collectionPath)) {
            echo "🧹 開始執行測試資料清除 collection：清除測試域名"
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
              sh """
                newman run "${collectionPath}" \
                  --environment "${ENV_FILE}" \
                  --export-environment "/tmp/exported_env.json" \
                  --insecure \
                  --reporters cli,json,html,junit,allure \
                  --reporter-json-export "${REPORT_DIR}/DeleteDomain_cleanup_report.json" \
                  --reporter-html-export "${HTML_REPORT_DIR}/DeleteDomain_cleanup_report.html" \
                  --reporter-junit-export "${REPORT_DIR}/DeleteDomain_cleanup_report.xml" \
                  --reporter-allure-export "allure-results"
              """
            }
          } else {
            echo "⚠️ 找不到 collection 檔案：${collectionPath}，跳過清除流程"
          }
        }
      }
    }

    stage('取得刪除域名項目資料 (Job狀態檢查)') {
      steps {
        script {
          def jobNameMap = [
            "AddTag": "AddTag（新增 Tag）",
            "AddThirdLevelRandom": "AddThirdLevelRandom（設定三級亂數）",
            "AttachAntiBlockTarget": "AttachAntiBlockTarget（新增抗封鎖目標）",
            "AttachAntiHijackSource": "AttachAntiHijackSource（新增抗劫持）",
            "AttachAntiHijackTarget": "AttachAntiHijackTarget（新增抗劫持目標）",
            "CheckDomainBlocked": "CheckDomainBlocked（檢查封鎖）",
            "CheckPurchaseDeployCertificateStatus": "CheckPurchaseDeployCertificateStatus（檢查購買部署憑證結果）",
            "CheckWorkflowApplication": "CheckWorkflowApplication（檢查自動化申請）",
            "DeleteDomainRecord": "DeleteDomainRecord（刪除解析）",
            "DetachAntiBlockSource": "DetachAntiBlockSource（撤下抗封鎖）",
            "DetachAntiBlockTarget": "DetachAntiBlockTarget（撤下抗封鎖目標）",
            "DetachAntiHijackSource": "DetachAntiHijackSource（撤下抗劫持）",
            "DetachAntiHijackTarget": "DetachAntiHijackTarget（撤下抗劫持目標）",
            "InformDomainInfringement": "InformDomainInfringement（通知侵權網址）",
            "MergeErrorRecord": "MergeErrorRecord（檢查異常地區合併規則）",
            "PurchaseAndDeployCert": "PurchaseAndDeployCert（購買與部署憑證）",
            "PurchaseDomain": "PurchaseDomain（購買域名）",
            "RecheckARecordResolution": "RecheckARecordResolution（複檢域名 A 紀錄解析）",
            "RecheckCert": "RecheckCert（複檢憑證）",
            "RecheckDomainResolution": "RecheckDomainResolution（複檢域名）",
            "RecheckThirdLevelRandom": "RecheckThirdLevelRandom（複檢三級亂數）",
            "RemoveAntiBlock": "RemoveAntiBlock（刪除抗封鎖）",
            "RemoveAntiBlockTarget": "RemoveAntiBlockTarget（刪除抗封鎖目標）",
            "RemoveAntiHijackSource": "RemoveAntiHijackSource（刪除抗劫持）",
            "RemoveAntiHijackTarget": "RemoveAntiHijackTarget（刪除抗劫持目標）",
            "RemoveTag": "RemoveTag（移除 Tag）",
            "ReplaceCertificateProviderDetach": "ReplaceCertificateProviderDetach（替換憑證商下架）",
            "ReuseAndDeployCert": "ReuseAndDeployCert（轉移憑證）",
            "RevokeCert": "RevokeCert（撤銷憑證）",
            "SendCertCompleted": "SendCertCompleted（通知憑證已完成）",
            "SendUpdateUB": "SendUpdateUB（通知 UB 更新）",
            "SyncT2": "SyncT2（同步 F5 T2 設定）",
            "UpdateDomainRecord": "UpdateDomainRecord（設定域名解析）",
            "UpdateNameServer": "UpdateNameServer（上層設定）",
            "UpdateOneToOneList": "UpdateOneToOneList（更新一對一IP清單）",
            "UpdateOneToOneSourceRecord": "UpdateOneToOneSourceRecord（來源域名解析設定）",
            "UpdateOneToOneTargetRecord": "UpdateOneToOneTargetRecord（目標域名解析設定）",
            "VerifyDomainPDNSTags": "VerifyDomainPDNSTags（驗證域名 PDNS Tag）",
            "VerifyTLD": "VerifyTLD（驗證頂級域名）"
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
                  curl -s -X GET "${BASE_URL}/workflow_api/adm/workflows/${workflowId}/jobs" \\
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
                      "title": "🚨 取得刪除域名項目資料 (Job狀態檢查 - 異常)",
                      "subtitle": "Workflow: ${workflowId}",
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
                            "text": "${allIssues.replace('"', '\\"')}"
                          }
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
                    "title": "⏰ 取得廳主買域名項目資料 (Job狀態檢查) 輪詢超時失敗",
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
      }
    }

    stage('Publish HTML Reports') {
      steps {
        publishHTML(target: [
          reportDir: "${HTML_REPORT_DIR}",
          reportFiles: '01_report.html', // 或其他主頁，依實際報告為主
          reportName: '申請廳主買域名 HTML Reports',
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
          results: [[path: 'allure-results']]
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

        // 對應中文狀態
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
