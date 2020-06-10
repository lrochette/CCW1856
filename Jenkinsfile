import groovy.json.JsonSlurper

pipeline {
  agent any
  environment {
    TEST_INSTANCE    = 'https://lrtest1.service-now.com'
    PROD_INSTANCE    = 'https://lruat1.service-now.com'
    APP_SYS_ID       = 'b5a05473908010107f4468f7a3a96f5c'
    ATF_SUITE_NAME   = 'CCW1856%20Suite'
    ATF_FOLDER       = "ATF-results"
    ATF_FILE_RESULT  = "${ATF_FOLDER}/results.xml"
    VERSION          = "1.0.$BUILD_NUMBER"
  }

  stages {
    stage('build') {
      steps {
        snDevOpsStep()
        script {
          // import the app on to test instance

          currentBuild.description = ""
          def app_post_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', url: "${TEST_INSTANCE}/api/sn_cicd/sc/apply_changes?app_sys_id=${APP_SYS_ID}"
          def app_post_json = new JsonSlurper().parseText(app_post_response.content)

          echo "${app_post_json}"

          String app_progress = "${app_post_json.result.links.progress.id}";
          if(app_progress == null || app_progress == ""){
              currentBuild.description += "Stopping the build - Unable to track apply changes progress \n\n"
              error ('Stopping the build import app progress is not found')
              return
          }

          String app_progress_status = "${app_post_json.result.status}";
          if(app_progress_status == "3"){
              currentBuild.description += "Stopping the build - Applying changes on the test instance is not successful \n\n"
              error ('Stopping the build - apply changes for the app on the test instance failed')
              return
          }
          app_post_json = null;
          app_post_response = null;

          while (app_progress_status != "2" && app_progress_status != "3" && app_progress_status != "4"){
              def app_progress_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/progress/"+app_progress
              def app_progress_json = (new JsonSlurper().parseText(app_progress_response.content))
              echo "${app_progress_json}"
              app_progress_status = "${app_progress_json.result.status}";
              println("Import App progress status is ${app_progress_status}");

              app_progress_json = null;
              app_progress_response = null;
              sleep 5 // sleep for 5 seconds
          }

          // get the final app import progress details

          def app_progress_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/progress/"+app_progress
          def app_progress_json = (new JsonSlurper().parseText(app_progress_response.content))

          app_progress_status = "${app_progress_json.result.status}";
          String app_progress_status_label = "${app_progress_json.result.status_label}";
          String app_progress_status_message = "${app_progress_json.result.status_message}";
          String app_progress_status_detail = "${app_progress_json.result.status_detail}";
          String app_progress_precent_complete = "${app_progress_json.result.percent_complete}";
          println("Import App Progress status is ${app_progress_status}")
          println("Import App Progress status label is ${app_progress_status_label}")
          println("Import App Progress status message is ${app_progress_status_message}")
          println("Import App Progress status detail is ${app_progress_status_detail}")
          println("Import App Progress percent complete is ${app_progress_precent_complete}")

          if(app_progress_status != "2"){
            currentBuild.description += "Stopping the build - Applying changes on the test instance is not successful because of ${app_progress_status_message} \n\n"
             error ('Stopping the build import app is not successful')
            return
          }

          app_progress_json = null
          app_progress_response = null
          currentBuild.description += "Applied new commit changes successfully on the instance ${TEST_INSTANCE} \n\n"
          snDevOpsArtifact(artifactsPayload: """{"artifacts": [{"name": "CCW1856.xml", "version": "$VERSION","semanticVersion": "$VERSION","repositoryName": "CCW"}]}""")
        }
      }
    }

    stage ('test') {
      steps {
        snDevOpsStep()
        script {
          currentBuild.description += "Executing ATF Test Suites on ${TEST_INSTANCE}\n\n"

          def atf_post_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', url: "${TEST_INSTANCE}/api/sn_cicd/testsuite/run?test_suite_name=${ATF_SUITE_NAME}"
          def atf_post_json = new JsonSlurper().parseText(atf_post_response.content)

          echo "${atf_post_json}"

          // take progress id from the atf suite response and track it

          String atf_progress = "${atf_post_json.result.links.progress.id}";

          if (atf_progress == null || atf_progress == "") {
              currentBuild.description += "Stopping the build - Unable to track ATF suite execution progress \n\n"

              error ('Stopping the build because of no atf suite prgress')
              return
          }
          String atf_progress_status = "${atf_post_json.result.status}";

          if (atf_progress_status == "3") {
              currentBuild.description += "Stopping the build - ATF suite start failed \n\n"

              error ('Stopping the build ATF Suite start failed')
              return
          }
          atf_post_json = null;

          while (atf_progress_status != "2" && atf_progress_status != "3" && atf_progress_status != "4") {
              def atf_progress_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/progress/"+atf_progress

              def atf_progress_json = (new JsonSlurper().parseText(atf_progress_response.content))
              echo "${atf_progress_json}"
              atf_progress_status = "${atf_progress_json.result.status}";
              println("ATF Suite progress status is ${atf_progress_status}");

              atf_progress_json = null;
              atf_progress_response = null;
              sleep 3 // sleep for 5 seconds
          }

          // get the final progress details

          def atf_progress_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/progress/"+atf_progress
          def atf_progress_json = (new JsonSlurper().parseText(atf_progress_response.content))

          atf_progress_status = "${atf_progress_json.result.status}";
          String progress_status_label = "${atf_progress_json.result.status_label}";
          String progress_status_message = "${atf_progress_json.result.status_message}";
          String progress_result = "${atf_progress_json.result.links.results.id}";
          String progress_result_url = "${atf_progress_json.result.links.results.url}";
          String progress_precent_complete = "${atf_progress_json.result.percent_complete}";

          println("Progress status is ${atf_progress_status}")
          println("Progress status label is ${progress_status_label}")
          println("Progress status message is ${progress_status_message}")
          println("Result ID is ${progress_result}")
          println("Result url is ${progress_result_url}")
          println("Progress percent complete is ${progress_precent_complete}")

          atf_progress_json = null;
          atf_progress_response = null;

          if (progress_result == null || progress_result == "") {
              currentBuild.description += "Stopping the build - ATF suite result not found  because of ${progress_status_message}\n\n"
              error('Stopping the build because of no atf suite result')
              return
          }

          // Now get the results:
          println("Getting final ATF results and saving them in JUnit format")
          def atf_result_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/testsuite/results/"+progress_result
          def atf_result_json = (new JsonSlurper().parseText(atf_result_response.content))

          String atf_result_status = "${atf_result_json.result.test_suite_status}";
          def atf_success_count = "${atf_result_json.result.rolledup_test_success_count}";
          def atf_failure_count = "${atf_result_json.result.rolledup_test_failure_count}";
          def atf_total_count = atf_success_count + atf_failure_count;
          def strArray="${atf_result_json.result.test_suite_duration}".split();
          String atf_duration = strArray[0];

          println("Getting detailled individuals test results")
          def detailled_results_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/now/table/sys_atf_test_result?parent="+progress_result
          def detailled_results_json = (new JsonSlurper().parseText(detailled_results_response)

          echo "Creating ATF result folder ${ATF_FOLDER}"
          fileOperations([folderCreateOperation("${ATF_FOLDER}")])
          echo "Saving Results into ${ATF_FILE_RESULT}"
          def runtimeStr=tc.run_time
          def runtime=runTimeStr.split()[0]
          def xmlStr='<?xml version="1.0" encoding="UTF-8"?>\n'
          xmlStr += """<testsuite name="${atf_result_json.result.test_suite_name}"
    failures="${atf_failure_count} tests="${atf_total_count} time="${atf_duration}" >\n"""

          // loop on each test case
          detailled_results_json.result.each { tc->
            println("  parsing ${tc.test_name}")
            xmlStr += """  <testcase name="${tc.test_name}" classname="${tc.test_name}" status="${tc.status}" time="${atf_duration}">
  </testcase>\n"""
          }
          xmlStr += "</testsuite>\n"

          writeFile file: ATF_FILE_RESULT, text: xmlStr
          println ("Final XML:\n $xmlStr\n")
          
          if (atf_result_status != "success" && atf_result_status != "success_with_warnings") {
              currentBuild.description += "Stopping the build - ATF suite run is not successful \n\n"
              error('Stopping the build because ATF suite run is not successful')
              return
          }

          // Save result as JUnit
          currentBuild.description += "ATF Tests ran successfully \n\n Test Suite Name : ${atf_result_json.result.test_suite_name} \n"
          currentBuild.description += "Test Suite result URL : ${atf_result_json.result.links.results.url} \n"
          currentBuild.description += "Test Suite total run duration is : $atf_duration\n"
          currentBuild.description += "Test Suite total success count is : ${atf_result_json.result.rolledup_test_success_count} \n"



          atf_result_json = null;
          atf_result_response = null;

        }   // script in test

      }     // steps in test
      post {
          success {
              junit "${ATF_FILE_RESULT}"
          }
      }
    }       // stage test

    stage('publish') {
      steps {
        snDevOpsStep()

        script {
          // publish the app to the repo
          currentBuild.description += "\npublishing the app to the App-Store \n"

          def app_publish_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', url: "${TEST_INSTANCE}/api/sn_cicd/app_repo/publish?sys_id=${APP_SYS_ID}&version=${VERSION}"
          def app_publish_json = (new JsonSlurper().parseText(app_publish_response.content))
          String app_publish_status = "${app_publish_json.result.status}";
          echo "${app_publish_json}"
          String app_publish_progress = "${app_publish_json.result.links.progress.id}";

          if(app_publish_progress == null || app_publish_progress == ""){
              currentBuild.description += "Stopping the build - Unable to track publish app progress \n\n"
              error ('Stopping the build publish app progress is not found')
              return
          }

          String app_publish_progress_status = "${app_publish_json.result.status}";
          if(app_publish_progress_status == "3"){
              currentBuild.description += "Stopping the build publish app is not successful\n\n"
              error ('Stopping the build APP publish failed')
              return
          }
          app_publish_json = null;
          app_publish_response = null;

          while (app_publish_progress_status != "2" && app_publish_progress_status != "3" && app_publish_progress_status != "4"){
              def app_publish_progress_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/progress/"+app_publish_progress
              def app_publish_progress_json = (new JsonSlurper().parseText(app_publish_progress_response.content))
              echo "${app_publish_progress_json}"
              app_publish_progress_status = "${app_publish_progress_json.result.status}";
              println("Publish App progress status is ${app_publish_progress_status}");

              app_publish_progress_json = null;
              app_publish_progress_response = null;
              sleep 3 // sleep for 5 seconds
          }

          // get the final app publish progress details
          def app_publish_progress_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/progress/"+app_publish_progress
          def app_publish_progress_json = (new JsonSlurper().parseText(app_publish_progress_response.content))
          app_publish_progress_status = "${app_publish_progress_json.result.status}";
          String app_publish_progress_status_label = "${app_publish_progress_json.result.status_label}";
          String app_publish_progress_status_message = "${app_publish_progress_json.result.status_message}";
          String app_publish_progress_status_detail = "${app_publish_progress_json.result.status_detail}";
          String app_publish_progress_precent_complete = "${app_publish_progress_json.result.percent_complete}";
          println("App publish Progress status is ${app_publish_progress_status}")
          println("App publish Progress status label is ${app_publish_progress_status_label}")
          println("App publish Progress status message is ${app_publish_progress_status_message}")
          println("App publish Progress status detail is ${app_publish_progress_status_detail}")
          println("App publish Progress percent complete is ${app_publish_progress_precent_complete}")

          if(app_publish_progress_status != "2"){
              currentBuild.description += "Stopping the build publish app is not successful because of ${app_publish_progress_status_message}\n\n"
               error ('Stopping the build publish app is not successful')
              return
          }
          app_publish_progress_json = null
          app_publish_progress_response = null

          currentBuild.description += "App successfully published to the App-Store \n\n"
        }   // script in publish
      }     // steps in publish
    }       // stage publish


    stage('prod') {
      steps {
        snDevOpsStep()
        snDevOpsPackage(name: "CCW1856 Scoped App", artifactsPayload: """{"artifacts": [{"name": "CCW1856.xml", "version": "$VERSION","repositoryName": "CCW"}]}""")
        snDevOpsChange()

        script {
          def app_install_response = httpRequest authentication: "SN-lruat1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', url: "${PROD_INSTANCE}/api/sn_cicd/app_repo/install?sys_id=${APP_SYS_ID}&version=${VERSION}"
          def app_install_json = (new JsonSlurper().parseText(app_install_response.content))

          String app_install_status = "${app_install_json.result.status}";
          echo "${app_install_json}"
          String app_install_progress = "${app_install_json.result.links.progress.id}";

          if(app_install_progress == null || app_install_progress == ""){
             error ('Stopping the build install app prgress is not found')
             return
          }

          String app_install_progress_status = "${app_install_json.result.status}";

          if(app_install_progress_status == "3"){
             currentBuild.description += "Stopping the build install app is not succesful on ${PROD_INSTANCE} - unable to track progress<br><br>"
             error ('Stopping the build APP Install failed')
             return
          }
          app_install_json = null;
          app_install_response = null;

          while (app_install_progress_status != "2" && app_install_progress_status != "3" && app_install_progress_status != "4"){
             def app_install_progress_response = httpRequest authentication: "SN-lruat1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${PROD_INSTANCE}/api/sn_cicd/progress/"+app_install_progress

             def app_install_progress_json = (new JsonSlurper().parseText(app_install_progress_response.content))
             echo "${app_install_progress_json}"
             app_install_progress_status = "${app_install_progress_json.result.status}";
             println("Install App progress status is ${app_install_progress_status}");

             app_install_progress_json = null;
             app_install_progress_response = null;
             sleep 3 // sleep for 5 seconds
          }

          // get the final app install progress details
          def app_install_progress_response = httpRequest authentication: "SN-lruat1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${PROD_INSTANCE}/api/sn_cicd/progress/"+app_install_progress
          def app_install_progress_json = (new JsonSlurper().parseText(app_install_progress_response.content))

          app_install_progress_status = "${app_install_progress_json.result.status}";
          String app_install_progress_status_label = "${app_install_progress_json.result.status_label}";
          String app_install_progress_status_message = "${app_install_progress_json.result.status_message}";
          String app_install_progress_status_detail = "${app_install_progress_json.result.status_detail}";
          String app_install_progress_precent_complete = "${app_install_progress_json.result.percent_complete}";
          println("App install Progress status is ${app_install_progress_status}")
          println("App install Progress status label is ${app_install_progress_status_label}")
          println("App install Progress status message is ${app_install_progress_status_message}")
          println("App install Progress status detail is ${app_install_progress_status_detail}")
          println("App install Progress percent complete is ${app_install_progress_precent_complete}")

          if(app_install_progress_status != "2"){
             currentBuild.description += "Stopping the build install app is not succesful on ${PROD_INSTANCE} because of ${app_install_progress_status_message} <br><br>"

              error ('Stopping the build install app is not succesful')
             return
          }

          app_install_progress_json = null
          app_install_progress_response = null

          currentBuild.description += "App successfully installed on the client instance ${PROD_INSTANCE} <br><br>"
        }
      }

    }       // stage prod
  }         // stages
}           // pipeline
