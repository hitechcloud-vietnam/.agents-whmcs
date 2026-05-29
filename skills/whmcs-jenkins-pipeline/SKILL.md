---
name: whmcs-jenkins-pipeline
description: Jenkins integration for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Jenkins Pipeline Skill

## Overview
This skill provides patterns for Jenkins pipeline configuration for WHMCS.

## Implementation Patterns

### Jenkins Pipeline Manager
```php
<?php
/**
 * WHMCS Jenkins Pipeline
 * Manages Jenkins pipelines
 */

namespace WHMCS\Module\DevOps\Jenkins;

class JenkinsPipelineManager {
    /**
     * Generate Jenkinsfile
     */
    public function generateJenkinsfile(): string {
        return <<<JENKINS
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'composer install'
            }
        }

        stage('Test') {
            steps {
                sh 'vendor/bin/phpunit'
            }
        }

        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }

    post {
        always {
            junit '**/test-results/*.xml'
        }
    }
}
JENKINS;
    }

    /**
     * Trigger build
     */
    public function triggerBuild(string $jobName, array $params = []): array {
        $jenkinsUrl = getenv('JENKINS_URL');

        $ch = curl_init("{$jenkinsUrl}/job/{$jobName}/build");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($params),
            CURLOPT_RETURNTRANSFER => true
        ]);

        $result = curl_exec($ch);
        curl_close($ch);

        return ['success' => true, 'job' => $jobName];
    }
}
```

## Best Practices

1. **Pipeline as Code**: Use declarative pipelines
2. **Shared Libraries**: Create shared pipeline libraries
3. **Agent Labels**: Use agent labels for targeting
4. **Credentials**: Use Jenkins credentials
5. **Notifications**: Set up notifications

## Related Skills

- whmcs-github-actions
- whmcs-gitlab-ci
- whmcs-cicd-integration
- whmcs-gitops-workflow