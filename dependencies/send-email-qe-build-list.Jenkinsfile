#!/usr/bin/env groovy

// PARAMETERS for this pipeline:
// MIDSTM_BRANCH="crw-2.4-rhel-8"
// mailSubject  - subject to put on the email, eg., CRW 2.3.0.RC-mm-yy ready for QE
// errataURL - URL for the errata, eg., https://errata.devel.redhat.com/errata/container/56923
// unresolvedCriticalsBlockersURL - URL for unresolved blockers/criticals, eg., Unresolved criticals/blockers:
//   https://issues.redhat.com/browse/CRW-883?jql=fixversion%20%3D%202.4.0.GA%20AND%20project%20%3D%20CRW%20AND%20priority%20%3E%20Major%20AND%20resolution%20is%20null
// additionalNotes - footer for the email
// doSendEmail - boolean: if checked, send mail; if not, draft email but do not send
// doOSBS - boolean: if checked, include OSBS images in email
// doStage - boolean: if checked, include RHCC stage images in email
// recipientOverride - if set, send mail to recipient(s) listed rather than default mailing lists

import hudson.FilePath
import groovy.transform.Field
@Field String CRW_VERSION_F = ""
def String getCrwVersion(String MIDSTM_BRANCH) {
  if (CRW_VERSION_F.equals("")) {
    CRW_VERSION_F = sh(script: '''#!/bin/bash -xe
    curl -sSLo- https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/''' + MIDSTM_BRANCH + '''/dependencies/VERSION''', returnStdout: true).trim()
  }
  return CRW_VERSION_F
}

def installSkopeo(String skopeo_version)
{
sh '''#!/bin/bash -xe
pushd /tmp >/dev/null
  rm -f /tmp/skopeo*.rpm /tmp/containers-common*.rpm
  skopeo_URL=https://rpmfind.net/linux/centos/8-stream/AppStream/x86_64/os/Packages
  curl -sSLO ${skopeo_URL}/containers-common-''' + skopeo_version + '''.rpm
  curl -sSLO ${skopeo_URL}/skopeo-''' + skopeo_version + '''.rpm
  sudo yum remove -y skopeo containers-common || true
  sudo yum install -y /tmp/skopeo*.rpm /tmp/containers-common*.rpm || true
  rm -f /tmp/skopeo*.rpm /tmp/containers-common*.rpm
popd >/dev/null
skopeo --version
'''
}

def sendMail(mailSubject,mailBody) { // NEW_OSBS
    // # TOrecipients - comma and space separated list of recipient email addresses
    // # CCrecipients - comma and space separated list of recipient email addresses
    // # mailBodyFile - file to use as email input

    def sender="nboldt@redhat.com" // # use a bot instead?
    def TOrecipients = "codeready-workspaces-qa@redhat.com"
    def CCrecipients = "che-prod@redhat.com"
    if (!recipientOverride.equals("")) {
        TOrecipients="${recipientOverride}"
        CCrecipients="${recipientOverride}"
    }
    writeFile(file: 'mailbody.tmp', text: mailBody)
    // # use mailx -r or sendmail -f, depending on what's available on the server
    sh '''#!/bin/bash -xe
if [[ -x /bin/mailx ]]; then 
    /bin/mailx -s "''' + mailSubject + '''" -c "''' + CCrecipients + '''" -r "''' + sender + '''" "''' + TOrecipients + '''" < mailbody.tmp
else 
    /bin/mail -s "''' + mailSubject + '''" "''' + TOrecipients + '''" -c"''' + CCrecipients + '''" -- -f"''' + sender + '''" < mailbody.tmp
fi
rm -f mailbody.tmp
'''   
}

def checkFailure(arrayLines,serverName,errorOccurred)
{
    arrayLines.each  { 
        line -> if (line?.toString().indexOf("No tags matching")>-1 || line?.toString().indexOf("ERROR")>-1) { 
            errorOccurred = errorOccurred + line + '\n'; 
            currentBuild.result = 'FAILURE'
        }
    }
    return errorOccurred
}

def errorOccurred = ""
timeout(120) {
    node("rhel7-releng"){ 
        try { 
            stage "Fetch latest image tags and send email"
            cleanWs()
            installSkopeo("1.1.0-1.module_el8.3.0+432+2e9cbcd8.x86_64")
            if (mailSubject.contains("CRW 2.y.0.tt-mm-yy ready for QE") || mailSubject.equals(""))
            {
                doSendEmail="false"
                errorOccurred = errorOccurred + 'Error: need to set an actual email subject. Failure!\n'
                currentBuild.description="Invalid email subject!"
                currentBuild.result = 'FAILURE'
            } else {
                currentBuild.description=mailSubject
                sh (
                    script: 'curl -sSLO https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/'+MIDSTM_BRANCH+'/product/getLatestImageTags.sh && chmod +x getLatestImageTags.sh',
                    returnStdout: true).trim().split( '\n' )
                sh (
                    script: 'curl -sSLO https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/'+MIDSTM_BRANCH+'/product/getTagForImage.sh && chmod +x getTagForImage.sh',
                    returnStdout: true).trim().split( '\n' )
                def NEW_QUAY = ""
                def NEW_OSBS = ""
                def NEW_STG = ""
                def NEW_NVR = ""
                parallel quay_check: {
                    NEW_QUAY = sh (
                        script: "./getLatestImageTags.sh --${MIDSTM_BRANCH} --quay | tee ${WORKSPACE}/LATEST_IMAGES.quay",
                        returnStdout: true).trim().split( '\n' )
                        errorOccurred = checkFailure(NEW_QUAY, "Quay", errorOccurred)
                }, 
                osbs_check: {
                    if (doOSBS.equals("true")) {
                        NEW_OSBS = sh (
                        script: "./getLatestImageTags.sh --${MIDSTM_BRANCH} --osbs | tee ${WORKSPACE}/LATEST_IMAGES.osbs",
                        returnStdout: true).trim().split( '\n' )
                        errorOccurred = checkFailure(NEW_OSBS, "OSBS", errorOccurred)
                    }
                }, 
                stg_check: {
                    if (doStage.equals("true")) {
                        NEW_STG = sh (
                        script: "./getLatestImageTags.sh --${MIDSTM_BRANCH} --stage | tee ${WORKSPACE}/LATEST_IMAGES.stage",
                        returnStdout: true).trim().split( '\n' )
                        errorOccurred = checkFailure(NEW_STG, "Stage", errorOccurred)
                    }
                }, 
                nvr_check: {
                    NEW_NVR = sh (
                        script: "./getLatestImageTags.sh --${MIDSTM_BRANCH} --nvr | tee ${WORKSPACE}/LATEST_IMAGES.nvr",
                        returnStdout: true).trim().split( '\n' )
                }

                // diff quay tag list vs. nvr tag list
                sh(script: '''#!/bin/bash -xe
      ${WORKSPACE}/getTagForImage.sh $(cat ${WORKSPACE}/LATEST_IMAGES.quay) > ${WORKSPACE}/LATEST_IMAGES.quay.tagsonly
      ${WORKSPACE}/getTagForImage.sh $(cat ${WORKSPACE}/LATEST_IMAGES.nvr)  > ${WORKSPACE}/LATEST_IMAGES.nvr.tagsonly
      ''', returnStdout: true)
                def DIFF_LATEST_IMAGES_QUAY_V_NVR = sh (
                    script: 'diff -u0 ${WORKSPACE}/LATEST_IMAGES.{quay,nvr}.tagsonly | grep -v "@@" | grep -v "LATEST_IMAGES" || true',
                    returnStdout: true
                ).trim()

                def DIFF_LATEST_IMAGES_QUAY_V_OSBS = ""
                def DIFF_LATEST_IMAGES_QUAY_V_STG = ""

                if (doOSBS.equals("true")) {
                    // diff quay tag list vs. OSBS tag list
                    sh(script: '''#!/bin/bash -xe
        ${WORKSPACE}/getTagForImage.sh $(cat ${WORKSPACE}/LATEST_IMAGES.osbs)  > ${WORKSPACE}/LATEST_IMAGES.osbs.tagsonly
        ''', returnStdout: true)
                    DIFF_LATEST_IMAGES_QUAY_V_OSBS = sh (
                        script: 'diff -u0 ${WORKSPACE}/LATEST_IMAGES.{quay,osbs}.tagsonly | grep -v "@@" | grep -v "LATEST_IMAGES" || true',
                        returnStdout: true
                    ).trim()
                }
                if (doStage.equals("true")) {
                    // diff quay tag list vs. stage tag list
                    sh(script: '''#!/bin/bash -xe
        ${WORKSPACE}/getTagForImage.sh $(cat ${WORKSPACE}/LATEST_IMAGES.stage)  > ${WORKSPACE}/LATEST_IMAGES.stage.tagsonly
        ''', returnStdout: true)
                    DIFF_LATEST_IMAGES_QUAY_V_STG = sh (
                        script: 'diff -u0 ${WORKSPACE}/LATEST_IMAGES.{quay,stage}.tagsonly | grep -v "@@" | grep -v "LATEST_IMAGES" || true',
                        returnStdout: true
                    ).trim()
                }

                archiveArtifacts fingerprint: false, artifacts:"LATEST_IMAGES*"
                if (!DIFF_LATEST_IMAGES_QUAY_V_NVR.equals("") || !DIFF_LATEST_IMAGES_QUAY_V_OSBS.equals("") || !DIFF_LATEST_IMAGES_QUAY_V_STG.equals("")) {
                    // error! quay and nvr versions do not match
                    errorOccurred = errorOccurred + 'Error: Quay & Brew image versions not aligned:\n' + 
                    "=================== QUAY v NVR ===================\n" + 
                    DIFF_LATEST_IMAGES_QUAY_V_NVR + '\n' + 
                    "=================== QUAY v OSBS ===================\n" + 
                    DIFF_LATEST_IMAGES_QUAY_V_OSBS + '\n' + 
                    "=================== QUAY v STG ===================\n" + 
                    DIFF_LATEST_IMAGES_QUAY_V_STG + '\n' + 
                    ' Failure!\n'
                    currentBuild.description="Quay/Brew version mismatch!"
                    currentBuild.result = 'FAILURE'

                    // trigger a push of latest images in Brew to Quay
                    build job: "push-latest-containers-to-quay_${MIDSTM_BRANCH}", 
                        parameters: [[$class: 'StringParameterValue', name: 'MIDSTM_BRANCH', value: "${MIDSTM_BRANCH}"]],
                        propagate: false,
                        wait: true

                    // trigger an update of metadata and registries
                    build job: "update-digests-in-registries-and-metadata_${MIDSTM_BRANCH}",
                        parameters: [[$class: 'StringParameterValue', name: 'MIDSTM_BRANCH', value: "${MIDSTM_BRANCH}"]],
                        propagate: false,
                        wait: true
                }

                def NEW_QUAY_L=""; NEW_QUAY.each { line -> if (line?.trim()) { NEW_QUAY_L=NEW_QUAY_L+"- ${line}\n" } }
                def NEW_OSBS_L=""; NEW_OSBS.each { line -> if (line?.trim()) { NEW_OSBS_L=NEW_OSBS_L+"= ${line}\n" } }
                def NEW_STG_L="";  NEW_STG.each  { line -> if (line?.trim()) { NEW_STG_L=NEW_STG_L + "* ${line}\n" } }
                def NEW_NVR_L="";  NEW_NVR.each  { line -> if (line?.trim()) { NEW_NVR_L=NEW_NVR_L + "  ${line}\n" } } 

                CRW_VERSION = getCrwVersion(MIDSTM_BRANCH)
                println "CRW_VERSION = '" + CRW_VERSION + "'"

                def mailBody = mailSubject + '''

Latest crwctl builds:

https://codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com/job/crwctl_''' + CRW_VERSION + '''/lastSuccessfulBuild/artifact/codeready-workspaces-chectl/dist/channels/
 - or -
https://github.com/redhat-developer/codeready-workspaces-chectl/releases

Quay Images:
''' + NEW_QUAY_L

if (doOSBS.equals("true")) { 
    mailBody = mailBody + '''
OSBS Images:
''' + NEW_OSBS_L
}

if (doStage.equals("true")) {
    mailBody = mailBody + '''
Stage Images:
''' + NEW_STG_L
}

mailBody = mailBody + '''
Brew NVRs (for use in ''' + errataURL + '''):
''' + NEW_NVR_L

mailBody = mailBody + '''
Unresolved blockers + criticals:
''' + unresolvedCriticalsBlockersURL

if (!additionalNotes.equals("")) {
mailBody = mailBody + '''
---------------

''' + additionalNotes
}

mailBody = mailBody + '''

---------------
Generated by https://codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com/job/send-email-qe-build-list/
'''

                echo "Subject: " + mailSubject + "\n\n" +
"Body: \n" +  
"============================================================\n" + 
mailBody + 
"\n============================================================\n"

                if (doSendEmail.equals("true") && errorOccurred.equals(""))
                {
                    sendMail(mailSubject,mailBody) // NEW_OSBS
                }
            }
            if (!errorOccurred.equals("")) {
                echo errorOccurred
            }
        } catch (e) {
            if (!errorOccurred.equals("")) {
                echo errorOccurred
                return
            }
            throw e
        }
    }
}