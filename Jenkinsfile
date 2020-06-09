import groovy.json.JsonSlurper

pipeline {
  agent any
  environment {
    TEST_INSTANCE    = 'https://lrtest1.service-now.com'
    PROD_INSTANCE    = 'https://lrochette1.service-now.com'
    APP_SYS_ID       = 'b5a05473908010107f4468f7a3a96f5c'
    ATF_SUITE_NAME   = 'CCW1856%20Suite'
    ATF_FOLDER       = "ATF-results"
    ATF_FILE_RESULT  = "${ATF_FOLDER}/results.xml"
  }

  stages {
    stage('build') {
      steps {
        snDevOpsStep()
        script {
          // import the app on to test instance

          def app_post_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', url: "${TEST_INSTANCE}/api/sn_cicd/sc/apply_changes?app_sys_id=${APP_SYS_ID}"
          def app_post_json = new JsonSlurper().parseText(app_post_response.content)

          echo "${app_post_json}"

          String app_progress = "${app_post_json.result.links.progress.id}";
          if(app_progress == null || app_progress == ""){
              currentBuild.description += "Stopping the build - Unable to track apply changes progress <br><br>"
              error ('Stopping the build import app progress is not found')
              return
          }

          String app_progress_status = "${app_post_json.result.status}";
          if(app_progress_status == "3"){
              currentBuild.description += "Stopping the build - Applying changes on the test instance is not successful <br><br>"
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
            currentBuild.description += "Stopping the build - Applying changes on the test instance is not successful because of ${app_progress_status_message} <br><br>"
             error ('Stopping the build import app is not successful')
            return
          }

          app_progress_json = null
          app_progress_response = null
          currentBuild.description += "Applied new commit changes successfully on the instance ${TEST_INSTANCE} <br><br>"
          snDevOpsArtifact(artifactsPayload: """{"artifacts": [{"name": "CCW1856.xml", "version": "1.0.$BUILD_NUMBER","semanticVersion": "1.0.$BUILD_NUMBER","repositoryName": "CCW"}]}""")
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
              currentBuild.description += "Stopping the build - Unable to track ATF suite execution progress <br><br>"

              error ('Stopping the build because of no atf suite prgress')
              return
          }
          String atf_progress_status = "${atf_post_json.result.status}";

          if (atf_progress_status == "3") {
              currentBuild.description += "Stopping the build - ATF suite start failed <br><br>"

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
              currentBuild.description += "Stopping the build - ATF suite result not found  because of ${progress_status_message}<br><br>"
              error('Stopping the build because of no atf suite result')
              return
          }

          // Now get the results:
          println("Getting final ATF results and saving them in JUnit format")
          def atf_result_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/testsuite/results/"+progress_result
          def atf_result_json = (new JsonSlurper().parseText(atf_result_response.content))

          String atf_result_status = "${atf_result_json.result.test_suite_status}";
          String atf_success_count = "${atf_result_json.result.rolledup_test_success_count}";
          String atf_failure_count = "${atf_result_json.result.rolledup_test_failure_count}";
          String atf_duration = "${atf_result_json.result.test_suite_duration}";

          if (atf_result_status != "success" && atf_result_status != "success_with_warnings") {
              currentBuild.description += "Stopping the build - ATF suite run is not successful <br><br>"
              error('Stopping the build because ATF suite run is not successful')
              return
          }

          // Save result as JUnit
          currentBuild.description += "ATF Tests ran successfully <br><br> Test Suite Name : ${atf_result_json.result.test_suite_name} <br>"
          currentBuild.description += "Test Suite result URL : ${atf_result_json.result.links.results.url} <br>"
          currentBuild.description += "Test Suite total run duration is : ${atf_result_json.result.test_suite_duration}) <br>"
          currentBuild.description += "Test Suite total success count is : ${atf_result_json.result.rolledup_test_success_count} <br>"

          echo "Creating ATF result folder ${ATF_FOLDER}"
          fileOperations([folderCreateOperation(${ATF_FOLDER})])

          echo "Saving Results into ${ATF_FILE_RESULT}"
          def xmlStr='<?xml version="1.0" encoding="UTF-8"?>\n'
          xmlStr += "<testsuites errors=\"${atf_failure_count}\" "
          xmlStr += "name=\"${atf_result_json.result.test_suite_name}\" "
          xmlStr += "tests=\"${atf_success_count}\" "
          xmlStr += "time=\"${atf_result_json.result.test_suite_duration}\" >\n"
          xmlStr +=  "</testsuites>\n"
          writeFile file: ${ATF_FILE_RESULT}, text: xmlStr

          atf_result_json = null;
          atf_result_response = null;

        }   // script in test

      }     // steps in test
/*      post {
          success {
              junit "${ATF_FILE_RESULT}"
          }
      }
*/
    }       // stage test

    stage('publish') {
      steps {
        snDevOpsStep()

        script {
          // publish the app to the repo
          currentBuild.description += "<br>publishing the app to the App-Store <br>"

          def app_publish_response = httpRequest authentication: "SN-lrtest1", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', url: "${TEST_INSTANCE}/api/sn_cicd/app_repo/publish?sys_id=${APP_SYS_ID}"
          def app_publish_json = (new JsonSlurper().parseText(app_publish_response.content))
          String app_publish_status = "${app_publish_json.result.status}";
          echo "${app_publish_json}"
          String app_publish_progress = "${app_publish_json.result.links.progress.id}";

          if(app_publish_progress == null || app_publish_progress == ""){
              currentBuild.description += "Stopping the build - Unable to track publish app progress <br><br>"
              error ('Stopping the build publish app prgress is not found')
              return
          }

          String app_publish_progress_status = "${app_publish_json.result.status}";
          if(app_publish_progress_status == "3"){
              currentBuild.description += "Stopping the build publish app is not successful<br><br>"
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
              currentBuild.description += "Stopping the build publish app is not successful because of ${app_publish_progress_status_message}<br><br>"
               error ('Stopping the build publish app is not successful')
              return
          }
          app_publish_progress_json = null
          app_publish_progress_response = null

          currentBuild.description += "App successfully published to the App-Store <br><br>"
        }   // script in publish
      }     // steps in publish
    }       // stage publish


    stage('prod') {
      steps {
        snDevOpsStep()
        snDevOpsPackage(name: "CCW1856 Scoped App", artifactsPayload: """{"artifacts": [{"name": "CCW1856.xml", "version": "1.0.$BUILD_NUMBER","repositoryName": "CCW"}]}""")
        snDevOpsChange()
        sh 'sleep 15'
        sh 'echo Installing to Prod'
      }

    }       // stage prod
  }         // stages
}           // pipeline
