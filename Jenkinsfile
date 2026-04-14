pipeline {

    agent any

    tools {
        maven 'Maven3'
        jdk   'JDK21'
    }

    parameters {
        // ❌ Supprimer le paramètre BRANCH — géré automatiquement
        choice(
            name:    'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Environnement de déploiement cible'
        )
        booleanParam(
            name:         'SKIP_TESTS',
            defaultValue: false,
            description:  'Ignorer les tests (urgence uniquement !)'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch  : ${env.BRANCH_NAME}"   // ✅ variable automatique
                echo "Commit  : ${env.GIT_COMMIT}"
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile -B'
            }
        }

        stage('Tests unitaires') {
            when {
                not { expression { return params.SKIP_TESTS } }
            }
            steps {
                sh 'mvn test -B'
            }
            post {
                always { junit '**/target/surefire-reports/*.xml' }
                failure { echo 'Tests unitaires en ECHEC — vérifier les logs' }
            }
        }

        stage('Tests intégration') {
            when {
                not { expression { return params.SKIP_TESTS } }
            }
            steps {
                sh 'mvn verify -Dsurefire.skip=true -B'
            }
            post {
                always { junit '**/target/failsafe-reports/*.xml' }
            }
        }

        stage('Couverture JaCoCo') {
            steps {
                sh 'mvn jacoco:report -B'
            }
            post {
                always {
                    jacoco(
                        execPattern:         '**/target/jacoco.exec',
                        classPattern:        '**/target/classes',
                        sourcePattern:       '**/src/main/java',
                        minimumLineCoverage: '70'
                    )
                }
            }
        }

        stage('Qualité') {
            steps {
                sh '''
                    mvn checkstyle:checkstyle \
                        pmd:pmd \
                        pmd:cpd \
                        spotbugs:spotbugs \
                        -B
                '''
            }
            post {
                always {
                    recordIssues(
                        enabledForFailure: true,
                        tools: [
                            checkStyle(pattern: '**/checkstyle-result.xml'),
                            pmdParser(pattern:  '**/pmd.xml'),
                            cpd(pattern:        '**/cpd.xml'),
                            spotBugs(pattern:   '**/spotbugsXml.xml')
                        ],
                        qualityGates: [[
                            threshold: 10,
                            type:      'TOTAL',
                            unstable:  true
                        ]]
                    )
                }
            }
        }

        // ✅ Deploy uniquement sur main ou staging — pas sur les branches de dev
        stage('Archive') {
            when {
                anyOf {
                    branch 'main'
                    branch 'staging'
                }
            }
            steps {
                archiveArtifacts(
                    artifacts:         '**/target/*.jar',
                    fingerprint:       true,
                    allowEmptyArchive: false
                )
                echo "Artefact archivé — branche : ${env.BRANCH_NAME}"
            }
        }
    }

    post {
        always {
            echo "Pipeline terminée — branche : ${env.BRANCH_NAME} — statut : ${currentBuild.currentResult}"
        }

        failure {
            emailext(
                subject: "❌ FAILED [${env.BRANCH_NAME}]: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Le build a échoué.

Projet  : ${env.JOB_NAME}
Branche : ${env.BRANCH_NAME}
Build   : #${env.BUILD_NUMBER}
Commit  : ${env.GIT_COMMIT}

URL     : ${env.BUILD_URL}
Logs    : ${env.BUILD_URL}console
                """,
                to: 'stephanedoungue@gmail.com',
                attachLog: true
            )
        }

        unstable {
            emailext(
                subject: "⚠️ UNSTABLE [${env.BRANCH_NAME}]: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Le build est instable.

Projet  : ${env.JOB_NAME}
Branche : ${env.BRANCH_NAME}
Build   : #${env.BUILD_NUMBER}

URL : ${env.BUILD_URL}
                """,
                to: 'stephanedoungue@gmail.com'
            )
        }

        fixed {
            emailext(
                subject: "✅ FIXED [${env.BRANCH_NAME}]: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Le build est redevenu stable.

Projet  : ${env.JOB_NAME}
Branche : ${env.BRANCH_NAME}
Build   : #${env.BUILD_NUMBER}

URL : ${env.BUILD_URL}
                """,
                to: 'stephanedoungue@gmail.com'
            )
        }
    }
}