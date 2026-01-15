pipeline {
    agent any
    stages{
        stage('Get Code'){
            steps{
                //Obtenemos el codigo
                git branch: 'feature_fix_coverage',
                url: 'https://github.com/NadianGG/practica_caso1.2.git'
            }
        }
        stage('Static'){
            steps{
                bat '''
                    python -m flake8 --exit-zero --format=pylint app >flake8.out
                '''
                recordIssues qualityGates: [[criticality: 'NOTE', integerThreshold: 8, threshold: 8.0, type: 'TOTAL'], [criticality: 'ERROR', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']], sourceCodeRetention: 'LAST_BUILD', tools: [flake8(pattern: 'flake8.out')]
            }
        }
        stage('Tests'){
            parallel{
                stage('Unit + Coverage') {
                    steps {
                        bat '''
                            set PYTHONPATH=%WORKSPACE%
                            python -m coverage run --branch --source=app --omit=app\\__init__.py,app\\api.py ^
                                -m pytest --junitxml=result-unit.xml test\\unit
                            python -m coverage xml
                        '''
                        junit 'result-unit.xml'
                        recordCoverage qualityGates: [
                            // LINE coverage
                            [criticality: 'NOTE',  metric: 'LINE',   threshold: 95.0],
                            [criticality: 'ERROR', metric: 'LINE',   threshold: 85.0],
                        
                            // BRANCH coverage
                            [criticality: 'NOTE',  metric: 'BRANCH', threshold: 90.0],
                            [criticality: 'ERROR', metric: 'BRANCH', threshold: 80.0]
                        ],
                        tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']]
                    }
                }
                stage('Service'){
                    steps{
                        bat '''
                            set FLASK_APP=app\\api.py
                            start python -m flask run
                            start java -jar C:\\Users\\Usuario\\Downloads\\wiremock-standalone-3.13.2.jar --port 9090 --root-dir test\\wiremock
                            set PYTHONPATH=%WORKSPACE%
                            ping localhost -n 3 > nul
                            python -m pytest --junitxml=result-rest.xml test\\rest
                        '''
                        junit 'result-rest.xml'
                    }
                }
            }
        }
        stage('Security'){
            steps{
                bat '''
                    python -m bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                '''
                recordIssues qualityGates: [[integerThreshold: 2, threshold: 2.0, type: 'TOTAL'], [criticality: 'FAILURE', integerThreshold: 4, threshold: 4.0, type: 'TOTAL']], sourceCodeRetention: 'LAST_BUILD', tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
            }
        }
        stage('Performance'){
            steps{
                bat '''
                    set FLASK_APP=app\\api.py
                    start python -m flask run
                    ping localhost -n 3 > nul
                    "C:\\Master DevOps\\JMeter\\apache-jmeter-5.6.3\\apache-jmeter-5.6.3\\bin\\jmeter" -n -t test\\jmeter\\flask.jmx -f -l flask.jtl
                '''
                perfReport sourceDataFiles: 'flask.jtl'
            }
        }
    }
    post {
        always {
            cleanWs(
                deleteDirs: true,
                disableDeferredWipeout: true,
                notFailBuild: true
            )
        }
    }
}