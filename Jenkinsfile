pipeline {
    agent any

    tools {
        maven 'M3'
    }

    environment {
        IMAGE_NAME = 'imagen-vehiculos-s8'
        CONTAINER_NAME = 'contenedor-vehiculos-s8'
        APP_PORT = '9090'
        CONTAINER_PORT = '8080'
        APP_CONTEXT = 'vehiculosBuild'
    }

    stages {
        stage('Obtener codigo desde GitHub') {
            steps {
                checkout scm
            }
        }

        stage('Compilar WAR con Maven') {
            steps {
                dir('Sucursal_vehiculos') {
                    sh '''
                        mvn clean package -DskipTests
                        ls -lh target/vehiculosBuild.war
                    '''
                }
            }
        }

        stage('Construir imagen Docker') {
            steps {
                dir('Sucursal_vehiculos') {
                    sh '''
                        docker build -t "$IMAGE_NAME" .
                    '''
                }
            }
        }

        stage('Desplegar contenedor Tomcat') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 's8-rds-jdbc-url',
                        variable: 'SPRING_DATASOURCE_URL'
                    ),
                    usernamePassword(
                        credentialsId: 's8-rds-credentials',
                        usernameVariable: 'SPRING_DATASOURCE_USERNAME',
                        passwordVariable: 'SPRING_DATASOURCE_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x

                        docker rm -f "$CONTAINER_NAME" || true

                        docker run -d \
                          --name "$CONTAINER_NAME" \
                          -p "$APP_PORT:$CONTAINER_PORT" \
                          -e SPRING_DATASOURCE_URL="$SPRING_DATASOURCE_URL" \
                          -e SPRING_DATASOURCE_USERNAME="$SPRING_DATASOURCE_USERNAME" \
                          -e SPRING_DATASOURCE_PASSWORD="$SPRING_DATASOURCE_PASSWORD" \
                          "$IMAGE_NAME"
                    '''
                }
            }
        }

        stage('Verificar contenedor') {
            steps {
                sh '''
                    sleep 30
                    docker ps --filter "name=$CONTAINER_NAME"
                    docker logs --tail=80 "$CONTAINER_NAME"
                '''
            }
        }

        stage('Validar aplicacion y Swagger') {
            steps {
                sh '''
                    curl -f "http://localhost:$APP_PORT/$APP_CONTEXT/"
                    curl -f "http://localhost:$APP_PORT/$APP_CONTEXT/api-docs"
                    curl -f "http://localhost:$APP_PORT/$APP_CONTEXT/vehiculos"
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline CI/CD finalizado correctamente.'
        }
        failure {
            sh 'docker logs --tail=100 "$CONTAINER_NAME" || true'
            echo 'El pipeline presentó un error. Revisar la consola.'
        }
    }
}
