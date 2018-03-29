def baseArtifactNames = ['TestMacApp': 'TestMacApp']
def xcodeSchemes = ['TestMacApp']
def repoURL = 'https://github.com/emlynmu/testmacapp.git'
def defaultCheckout = 'master'
def exportOptions = ''
def projectDirectory = ''
def xcodeChoices = ['9.2', '9.1', '9.GM.seed', '8.3.3', '8.3.2', '8.2.1', '8.1', '8.0', '7.3', '7.3.1', '9.2b2', '9.1b2', '9.0b6', '9.0b5', '9.0b4', '9.0b3', '9.0b2', '9.0b1']
def repoCredentialsId = '4d3f6157-10a1-4e6a-86d2-e2004d649cb0' // nedpixel (id_github_ned_feb_2016)
def jenkinsCredentialId = 'c8ae583e-d409-4ff8-b325-862619d99bc8'
def iTunesConnectCredentialsId = 'fbee02f9-869a-4585-9345-e124a4da3a76'
def buildDirectory = 'build'
def xcodePath = "/Applications/Xcode-${env.XCODE_VERSION}.app"
def alToolPath = "${xcodePath}/Contents/Applications/Application Loader.app/Contents/Frameworks/ITunesSoftwareService.framework/Versions/A/Support/altool"

properties([
    parameters([
        choice(choices: xcodeSchemes.join('\n'), description: 'Xcode scheme to build.', name: 'XCODE_SCHEME'),
        choice(choices: xcodeChoices.join('\n'), description: 'Xcode version to utilize.', name: 'XCODE_VERSION'),
        string(name: 'GIT_CHECKOUT', defaultValue: defaultCheckout, description: 'The branch / tag / commit to check out.'),
        booleanParam(name: 'UPLOAD_TO_TESTFLIGHT', defaultValue: false, description: 'Whether or not to upload to TestFlight.')
        ])
    ])

withEnv(["DEVELOPER_DIR=${xcodePath}/Contents/Developer"]) {
    node('xcode-9.1') {
        def workspace = pwd()
        def versionedArtifact =  "${baseArtifactNames.get(XCODE_SCHEME)}-${BUILD_NUMBER}"

        stage('Checkout') {
            deleteDir() // Start with a clean workspace

            checkout([$class: 'GitSCM',
                branches: [[name: env.GIT_CHECKOUT]],
                doGenerateSubmoduleConfigurations: false,
                extensions: [[$class: 'SubmoduleOption', parentCredentials: true]],
                submoduleCfg: [],
                userRemoteConfigs: [[
                credentialsId: repoCredentialsId,
                url: repoURL
                ]]
                ])
        }

        stage('Set Version') {
            def infoPlistFile = sh(returnStdout: true, script: "cd './${projectDirectory}' && xcodebuild -scheme '${XCODE_SCHEME}' -showBuildSettings | sed '/ *INFOPLIST_FILE *= */!d;s/ *INFOPLIST_FILE *= *//'").trim()
            sh "cd './${projectDirectory}' && defaults write '${infoPlistFile}' CFBundleVersion ${BUILD_NUMBER}"
        }

        stage('Unlock Keychain') {
            withCredentials([usernamePassword(credentialsId: jenkinsCredentialId, passwordVariable: 'JENKINS_PASSWORD', usernameVariable: 'JENKINS_USERNAME')]) {
                sh "security unlock-keychain -p '$JENKINS_PASSWORD'"
            }
        }

        stage('Build') {
            sh "cd './${projectDirectory}' && xcodebuild clean archive -allowProvisioningUpdates -scheme '${XCODE_SCHEME}' -archivePath '${buildDirectory}/${versionedArtifact}.xcarchive'"
        }

        stage('Export') {
            sh "cd './${projectDirectory}' && xcodebuild -exportArchive -allowProvisioningUpdates -exportPath '${buildDirectory}' -archivePath '${buildDirectory}/${versionedArtifact}.xcarchive' -exportOptionsPlist 'exportOptions/${XCODE_SCHEME}.plist'"
        }

        stage('Archive') {
            sh "cd './${projectDirectory}/${buildDirectory}' && zip -r '${versionedArtifact}.xcarchive.zip' '${versionedArtifact}.xcarchive'"
            sh "cd './${projectDirectory}/${buildDirectory}' && mv '${XCODE_SCHEME}.ipa' '${versionedArtifact}.ipa' "

            def relativeArchivePath = "${projectDirectory}/${buildDirectory}/${versionedArtifact}.xcarchive.zip".replaceFirst(/^\.?\//, "")
            def relativeIPAPath = "${projectDirectory}/${buildDirectory}/${versionedArtifact}.ipa".replace("//", "/").replaceFirst(/^\.?\//, "")

            archiveArtifacts artifacts: "${relativeArchivePath},${relativeIPAPath}", onlyIfSuccessful: true
        }

        stage('TestFlight') {
            if (env.UPLOAD_TO_TESTFLIGHT == "true") {
                withCredentials([usernamePassword(credentialsId: iTunesConnectCredentialsId, passwordVariable: 'ITUNES_CONNECT_PASSWORD', usernameVariable: 'ITUNES_CONNECT_USERNAME')]) {
                    sh "'${alToolPath}' --upload-app -f '${projectDirectory}/${buildDirectory}/${versionedArtifact}.ipa' -t ios -u '${env.ITUNES_CONNECT_USERNAME}' -p @env:ITUNES_CONNECT_PASSWORD"
                }
            }
            else {
                echo 'Skipped TestFlight upload.'
            }
        }
    }
}
