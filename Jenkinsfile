pipeline {
    agent any

    environment {
        // Injected from Jenkins credentials
        MONGO_URI = credentials('MONGO_URI')
        SECRET_KEY = credentials('SECRET_KEY')
        VENV = "venv"
    }

    options {
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Cloning repository..."
                git branch: 'main', url: 'https://github.com/hariprn/flask_practice.git'
            }
        }

        stage('Setup Python Environment') {
            steps {
                echo "Setting up virtual environment..."
                sh '''
                python3 -m venv $VENV
                . $VENV/bin/activate
                pip install --upgrade pip
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                echo "Installing dependencies..."
                sh '''
                . $VENV/bin/activate
                pip install -r requirements.txt
                pip install pytest python-dotenv
                '''
            }
        }

        stage('Run Tests') {
            steps {
                echo "Running tests..."
                sh '''
                . $VENV/bin/activate
                pytest
                '''
            }
        }
	stage('Deploy to Staging') {
	   when { 
	       expression { 
		   currentBuild.currentResult == 'SUCCESS' 
	       } 
	   } 
	   steps { 
	       sh ''' 
	       . $VENV/bin/activate 
	       echo "Deploying to staging..." 
               pkill -f app.py || true 
	       nohup python app.py > app.log 2>&1 & 
	       ''' 
	   } 
	}
    }

    post {
    always {
        echo 'Pipeline completed'
    }
    success {
        mail to: 'your-email@gmail.com',
             subject: "SUCCESS: Job ${env.JOB_NAME} Build ${env.BUILD_NUMBER}",
             body: "Build succeeded. Check details at ${env.BUILD_URL}"
    }
    failure {
        mail to: 'your-email@gmail.com',
             subject: "FAILURE: Job ${env.JOB_NAME} Build ${env.BUILD_NUMBER}",
             body: "Build failed. Check logs at ${env.BUILD_URL}"
    }
}
}
