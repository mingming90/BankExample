/*
 * Jenkins Pipeline:
 * GitHub -> Build with Visual Studio/MSBuild -> Parasoft dotTEST Static Analysis
 * -> Publish results to Parasoft DTP
 *
 * Recommended:
 *   - Put this file at the ROOT of the GitHub repository.
 *   - File name: Jenkinsfile   (no extension)
 *
 * Required Jenkins credential:
 *   ID: dottest-ci-settings
 *   Kind: Secret file
 *   File: a dotTEST .properties file containing DTP/license settings.
 */

pipeline {
    agent any

    options {
        // Jenkinsfile is loaded from SCM first. We perform the full source checkout
        // explicitly in the Checkout stage below.
        skipDefaultCheckout(true)

        // Avoid two builds/scans of this job running at the same time.
        disableConcurrentBuilds()

        timestamps()
    }

    environment {
        // ============================================================
        // EDIT THESE VALUES FOR YOUR PROJECT
        // ============================================================

        // Path to the solution relative to the Git repository root.
        // Examples:
        //   'ATM.sln'
        //   'src\\MyApplication.sln'
        SOLUTION = 'BankExample.slnx'

        BUILD_CONFIGURATION = 'Release'

        // Change this if dotTEST is installed elsewhere.
        //DOTTEST_CLI = 'D:\\Parasoft\\products\\dottest\\2026.1\\dottestcli.exe'
        DOTTEST_CLI = 'D:\\parasoft\\dottest\\2026.1\\dottestcli.exe'

        // Built-in Parasoft static analysis configuration.
        DOTTEST_CONFIG = 'builtin://Recommended Rules'

        // Local report directory inside Jenkins workspace.
        DOTTEST_REPORT_DIR = 'dottest-report'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code from GitHub...'

                // Uses the repository, branch and GitHub credentials configured
                // in the Jenkins job under "Pipeline script from SCM".
                checkout scm
            }
        }

        stage('Validate Environment') {
            steps {
                bat '''
                    @echo off
                    echo ============================================================
                    echo Validate Jenkins build environment
                    echo ============================================================

                    echo Workspace:
                    echo %WORKSPACE%
                    echo.

                    if not exist "%WORKSPACE%\\%SOLUTION%" (
                        echo ERROR: Solution file was not found:
                        echo %WORKSPACE%\\%SOLUTION%
                        exit /b 1
                    )

                    if not exist "%DOTTEST_CLI%" (
                        echo ERROR: dotTEST CLI was not found:
                        echo %DOTTEST_CLI%
                        exit /b 1
                    )

                    set "VSWHERE=%ProgramFiles(x86)%\\Microsoft Visual Studio\\Installer\\vswhere.exe"

                    if not exist "%VSWHERE%" (
                        echo ERROR: vswhere.exe was not found.
                        echo Visual Studio or Visual Studio Build Tools may not be installed correctly.
                        exit /b 1
                    )

                    echo Environment validation passed.
                '''
            }
        }

        stage('Build') {
            steps {
                bat '''
                    @echo off
                    setlocal

                    echo ============================================================
                    echo Locate MSBuild
                    echo ============================================================

                    set "VSWHERE=%ProgramFiles(x86)%\\Microsoft Visual Studio\\Installer\\vswhere.exe"
                    set "MSBUILD="

                    for /f "usebackq delims=" %%I in (`"%VSWHERE%" -latest -products * -requires Microsoft.Component.MSBuild -find MSBuild\\**\\Bin\\MSBuild.exe`) do (
                        if not defined MSBUILD set "MSBUILD=%%I"
                    )

                    if not defined MSBUILD (
                        echo ERROR: MSBuild.exe could not be located by vswhere.
                        exit /b 1
                    )

                    echo MSBuild:
                    echo %MSBUILD%
                    echo.

                    echo ============================================================
                    echo Restore and build solution
                    echo ============================================================

                    "%MSBUILD%" "%WORKSPACE%\\%SOLUTION%" ^
                        /restore ^
                        /m ^
                        /p:Configuration=%BUILD_CONFIGURATION%

                    if errorlevel 1 (
                        echo ERROR: Project build failed.
                        exit /b 1
                    )

                    echo.
                    echo Build completed successfully.
                    endlocal
                '''
            }
        }

        stage('dotTEST Static Analysis') {
            // steps {
            //     // Store DTP/license configuration as a Jenkins Secret File
            //     // instead of hard-coding credentials in this Jenkinsfile.
            //     withCredentials([
            //         file(
            //             credentialsId: 'dottest-ci-settings',
            //             variable: 'DOTTEST_SETTINGS'
            //         )
            //     ]) {
            //         bat '''
            //             @echo off

            //             echo ============================================================
            //             echo Parasoft dotTEST Static Analysis
            //             echo Configuration: %DOTTEST_CONFIG%
            //             echo ============================================================

            //             if exist "%WORKSPACE%\\%DOTTEST_REPORT_DIR%" (
            //                 rmdir /s /q "%WORKSPACE%\\%DOTTEST_REPORT_DIR%"
            //             )

            //             mkdir "%WORKSPACE%\\%DOTTEST_REPORT_DIR%"

            //             "%DOTTEST_CLI%" ^
            //                 -solution "%WORKSPACE%\\%SOLUTION%" ^
            //                 -solutionConfig "%BUILD_CONFIGURATION%" ^
            //                 -config "%DOTTEST_CONFIG%" ^
            //                 -nobuild ^
            //                 -settings "%DOTTEST_SETTINGS%" ^
            //                 -report "%WORKSPACE%\\%DOTTEST_REPORT_DIR%" ^
            //                 -property "build.id=%JOB_NAME%-%BUILD_NUMBER%" ^
            //                 -property "session.tag=Jenkins-StaticAnalysis" ^
            //                 -publish

            //             if errorlevel 1 (
            //                 echo ERROR: dotTEST execution failed.
            //                 exit /b 1
            //             )

            //             echo.
            //             echo dotTEST analysis completed and results were published to DTP.
            //         '''
            //     }
            // }
            steps {
                bat '''
                        @echo off

                        echo ============================================================
                        echo Parasoft dotTEST Static Analysis
                        echo Configuration: %DOTTEST_CONFIG%
                        echo ============================================================

                        if exist "%WORKSPACE%\\%DOTTEST_REPORT_DIR%" (
                            rmdir /s /q "%WORKSPACE%\\%DOTTEST_REPORT_DIR%"
                        )

                        mkdir "%WORKSPACE%\\%DOTTEST_REPORT_DIR%"

                        "%DOTTEST_CLI%" ^
                            -solution "%WORKSPACE%\\%SOLUTION%" ^
                            -solutionConfig "%BUILD_CONFIGURATION%" ^
                            -config "%DOTTEST_CONFIG%" ^
                            -nobuild ^
                            -settings "%DOTTEST_SETTINGS%" ^
                            -report "%WORKSPACE%\\%DOTTEST_REPORT_DIR%" ^
                            -property "build.id=%JOB_NAME%-%BUILD_NUMBER%" ^
                            -property "session.tag=Jenkins-StaticAnalysis" ^
                            -publish

                        if errorlevel 1 (
                            echo ERROR: dotTEST execution failed.
                            exit /b 1
                        )

                        echo.
                        echo dotTEST analysis completed and results were published to DTP.
                    '''
            }
        }
    }

    post {
        always {
            // Keep dotTEST local reports in Jenkins as build artifacts.
            archiveArtifacts(
                artifacts: 'dottest-report/**/*',
                allowEmptyArchive: true,
                fingerprint: false
            )
        }

        success {
            echo 'SUCCESS: Checkout -> Build -> dotTEST scan -> DTP publish completed.'
        }

        failure {
            echo 'FAILED: Open Jenkins Console Output and check the failed stage.'
        }
    }
}
