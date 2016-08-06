node {
    
    // Mark the code checkout 'Checkout'....
    stage 'Checkout'

    // Get the Terraform tool.
    wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm']) {
        dir(project) {

            // Mark the code build 'plan'....
            stage name: 'Plan', concurrency: 1

            // Output Terraform version
            sh "terraform --version"

            //Remove the terraform state file so we always start from a clean state
            if (fileExists(".terraform/terraform.tfstate")) {
                sh "rm -rf .terraform/terraform.tfstate"
            }
            if (fileExists("status")) {
                sh "rm status"
            }

            sh "./init"
            sh "terraform get"
            sh "terraform plan -out=plan.out -detailed-exitcode; echo \$? > status"
            def exitCode = readFile('status').trim()
            def apply = false
            echo "Terraform Plan Exit Code: ${exitCode}"
            if (exitCode == "0") {
                currentBuild.result = 'SUCCESS'
            }
            if (exitCode == "1") {
                slackSend channel: '#ci', color: '#0080ff', message: "Plan Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER} (<${env.JOB_URL}|Open>)"
                currentBuild.result = 'FAILURE'
            }
            if (exitCode == "2") {
                stash name: "plan", includes: "plan.out"
                slackSend channel: '#ci', color: 'good', message: "Plan Awaiting Approval: ${env.JOB_NAME} - ${env.BUILD_NUMBER} (<${env.JOB_URL}|Open>)"
                try {
                    input message: 'Apply Plan?', ok: 'Apply'
                    apply = true
                } catch (err) {
                    slackSend channel: '#ci', color: 'warning', message: "Plan Discarded: ${env.JOB_NAME} - ${env.BUILD_NUMBER} (<${env.JOB_URL}|Open>)"
                    apply = false
                    currentBuild.result = 'UNSTABLE'
                }
            }

            if (apply) {
                stage name: 'Apply', concurrency: 1
                unstash 'plan'
                if (fileExists("status.apply")) {
                    sh "rm status.apply"
                }
                sh 'terraform apply plan.out; echo \$? > status.apply'
                def applyExitCode = readFile('status.apply').trim()
                if (applyExitCode == "0") {
                    slackSend channel: '#ci', color: 'good', message: "Changes Applied ${env.JOB_NAME} - ${env.BUILD_NUMBER} (<${env.JOB_URL}|Open>)"    
                } else {
                    slackSend channel: '#ci', color: 'danger', message: "Apply Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER} (<${env.JOB_URL}|Open>)"
                    currentBuild.result = 'FAILURE'
                }
            }

        }
    }
}
