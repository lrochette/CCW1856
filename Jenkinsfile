import groovy.json.JsonSlurper

pipeline {
  agent any
  environment {
    TEST_INSTANCE = 'https://lrtest1.service-now.com'
    APP_VERSION = '1.0.1'
    APP_SYS_ID = 'b5a05473908010107f4468f7a3a96f5c'
  }
  node() {
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
              def app_progress_response = httpRequest authentication: "service_now_app", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/progress/"+app_progress

              def app_progress_json = (new JsonSlurper().parseText(app_progress_response.content))
              echo "${app_progress_json}"
              app_progress_status = "${app_progress_json.result.status}";
              println("Import App progress status is ${app_progress_status}");

              app_progress_json = null;
              app_progress_response = null;
              sleep 5 // sleep for 5 seconds
          }

          // get the final app import progress details

          def app_progress_response = httpRequest authentication: "service_now_app", acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'GET', url: "${TEST_INSTANCE}/api/sn_cicd/progress/"+app_progress

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
        }
      }
    }
  }
}
