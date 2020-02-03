pipeline {
  agent {
    node {
      label 'Jenkins_slave_practice'
    }
  }
  environment { 
    def http_proxy="http://192.168.2.123:4411"
    def https_proxy="http://192.168.2.123:4411"
  }
  stages {
  stage('Init') {
		steps{
			script{
				println "welcome to Nick learn"
			}
		}
	}
	stage('hexo1') {
    		steps{
    			script{
    			container('hexo') {
    	          sh 'git clone https://github.com/wwb19/myblog.git'
    	          sh 'pwd'
    	          sh 'ls -lrt'
    	          sh 'echo ---------------------'
    			}
    		}
    	}	
    }
    
    stage('hexo2') {
		steps{
			script{
    			container('hexo') {
    			  dir('/home/jenkins/agent/workspace/test/myblog'){
    	          sh 'pwd'
    	          sh 'ls -lrt'
    	          sh 'rm -rf .deploy_git'
    	          sh 'hexo generate'
    			  }
    			}
		    }
	    }
    }
  }
}
