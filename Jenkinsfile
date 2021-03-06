#!/usr/bin/env groovy

/* Only keep the 10 most recent builds. */
def projectProperties = [
    [$class: 'BuildDiscarderProperty',strategy: [$class: 'LogRotator', numToKeepStr: '10']],
]

properties(projectProperties)

def getRepoURL() {
  sh "git config --get remote.origin.url > .git/remote-url"
  return readFile(".git/remote-url").trim()
}

def getCommitSha() {
  sh "git rev-parse HEAD > .git/current-commit"
  return readFile(".git/current-commit").trim()
}

def getRepoName() {
  return sh(returnStdout: true, script: "git config --get remote.origin.url | sed 's%^.*/\\([^/]*\\)\\.git\$%\\1%g'").trim()
}

node("jessie-amd64") {
    stage('Checkout') {
        deleteDir()
        checkout scm

        // Parallel fetch all required Git source repositories
        parallel (
            "vyos-kernel": {
                dir('vyos-kernel') {
                    git branch: 'linux-vyos-4.19.y',
                        url: 'https://github.com/vyos/vyos-kernel.git'
                }
            },
            "vyos-wireguard": {
                dir('vyos-wireguard') {
                    git branch: 'current',
                        url: 'https://github.com/vyos/vyos-wireguard.git'
                }
            },
            "vyos-accell-ppp": {
                dir('vyos-accel-ppp') {
                    git branch: 'current',
                        url: 'https://github.com/vyos/vyos-accel-ppp.git'
                }
            }
        )
    }
    stage('Build Kernel') {
        dir ('vyos-kernel') {
            // compile the kernel, throw exception on failure
            // https://github.com/jenkinsci/pipeline-githubnotify-step-plugin/blob/master/README.md
            //githubNotify account: 'vyosbot',
            //    credentialsId: 'github-vyosbot',
            //    repo: getRepoName(),
            //    sha: getCommitSha(),
            //    targetUrl: '${env.BUILD_URL}',
            //    status: 'PENDING',
            //    description: 'This build is pending'
            try {
                sh "../build-kernel.sh"
            } catch(Exception e) {
                echo("error compiling kernel")
            }
        }
    }
    stage('Out of Tree Modules') {
        parallel (
            "wireguard": {
                dir('vyos-wireguard') {
                    sh '''
                        #!/bin/bash
                        KERNEL_VER=$(cat ../kernel-version)
                        DPKG_MOD_INST="debian/wireguard-modules.install"
                        echo "src/wireguard.ko /lib/modules/${KERNEL_VER}/extra" > ${DPKG_MOD_INST}
                        KERNELDIR=$(pwd)/../vyos-kernel dpkg-buildpackage -b -us -uc -tc
                    '''
                }
            },
            "accel-ppp": {
                dir('vyos-accel-ppp') {
                    sh '''
                        #!/bin/bash
                        KERNEL_VER=$(cat ../kernel-version)
                        DPKG_MOD_INST="debian/vyos-accel-ppp-ipoe-kmod.install"
                        echo "lib/modules/${KERNEL_VER}/extra/*.ko" > ${DPKG_MOD_INST}
                        sed -i "s#[0-9].[0-9][0-9].[0-9]*-amd64-vyos#"${KERNEL_VER}"#g" debian/rules
                        KERNELDIR=$(pwd)/../vyos-kernel dpkg-buildpackage -b -us -uc -tc
                    '''
                }
            },
            "intel-drivers": {
                sh './build-intel-drivers.sh'
            },
        )
    }

    stage('Deploy') {
        if ((currentBuild.result == null) || currentBuild.result == 'SUCCESS' ()) {
            sshagent(['0b3ab595-5e67-420a-9a44-5cb1d508bedf']) {
                sh """
                    #!/usr/bin/env bash
                    ./pkg-build.sh current "*.deb"
                """
            }
        }
    }
}
