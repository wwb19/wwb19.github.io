pipeline {
  agent {
    node {
      label 'Jenkins_slave_practice'
    }
  }
  environment { 
    def HTTP_PROXY="http://192.168.2.123:4411"
    def HTTPS_PROXY="http://192.168.2.123:4411"
    def all_proxy="socks5://192.168.2.123:4411"
	  
    GITHUB_TOKEN=credentials('98f28b74-a03f-432e-a2a9-18cf52150e4a')
    
  }
  stages {
  stage('Init') {
		steps{
			scmSkip(deleteBuild: true, skipPattern:'.*\\[ci skip\\].*')
			script{
				println "welcome to Nick learn"
				sh 'printenv'
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
    	          sh 'sed -i "s/__GITHUB_TOKEN__/${GITHUB_TOKEN}/" _config.yml'
    	          sh 'hexo generate'
    	          sh 'hexo deploy'
    			}
    			}
		    }
	    }
    }
  }
}
