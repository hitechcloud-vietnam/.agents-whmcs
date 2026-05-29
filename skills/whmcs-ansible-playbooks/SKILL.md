---
name: whmcs-ansible-playbooks
description: Ansible automation for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Ansible Automation Skill

## Overview
This skill provides patterns and implementations for managing Ansible playbooks and automation in WHMCS for configuration management and deployment.

## Implementation Patterns

### Ansible Playbook Manager
```php
<?php
/**
 * WHMCS Ansible Playbook Management
 * Handles Ansible automation
 */

namespace WHMCS\Module\DevOps\Ansible;

class AnsiblePlaybookManager {
    private $db;
    private $ansibleRunner;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->ansibleRunner = new AnsibleRunner();
    }

    /**
     * Create Ansible playbook
     */
    public function createPlaybook(array $params): array {
        $playbookId = 'apb_' . bin2hex(random_bytes(12));

        $playbook = [
            'id' => $playbookId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'description' => $params['description'] ?? '',
            'playbook_content' => $params['playbook'],
            'inventory' => $params['inventory'] ?? 'inventory.yml',
            'tags' => json_encode($params['tags'] ?? []),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_ansible_playbooks', $playbook);

        return [
            'success' => true,
            'playbook_id' => $playbookId
        ];
    }

    /**
     * Execute playbook
     */
    public function executePlaybook(string $playbookId, array $options = []): array {
        $playbook = $this->getPlaybook($playbookId);

        if (!$playbook) {
            throw new \Exception("Playbook not found");
        }

        $executionId = 'exe_' . bin2hex(random_bytes(8));
        $startTime = microtime(true);

        $result = $this->ansibleRunner->execute($playbook, [
            'tags' => $options['tags'] ?? [],
            'limit' => $options['limit'] ?? null,
            'check' => $options['check'] ?? false,
            'diff' => $options['diff'] ?? false
        ]);

        $duration = microtime(true) - $startTime;

        // Log execution
        $this->logExecution($executionId, $playbookId, $result, $duration);

        return [
            'success' => $result['success'],
            'execution_id' => $executionId,
            'changed' => $result['changed'] ?? 0,
            'failed' => $result['failed'] ?? 0,
            'duration_seconds' => round($duration, 2),
            'output' => $result['output']
        ];
    }

    /**
     * Generate standard playbooks
     */
    public function generateStandardPlaybooks(int $serviceId): array {
        $playbooks = [];

        // Web server setup
        $playbooks[] = $this->createPlaybook([
            'service_id' => $serviceId,
            'name' => 'Setup Web Server',
            'playbook' => $this->getWebServerPlaybook()
        ]);

        // Database setup
        $playbooks[] = $this->createPlaybook([
            'service_id' => $serviceId,
            'name' => 'Setup Database',
            'playbook' => $this->getDatabasePlaybook()
        ]);

        // Security hardening
        $playbooks[] = $this->createPlaybook([
            'service_id' => $serviceId,
            'name' => 'Security Hardening',
            'playbook' => $this->getSecurityPlaybook()
        ]);

        return [
            'success' => true,
            'playbooks_created' => count($playbooks)
        ];
    }

    /**
     * Get playbook templates
     */
    public function getPlaybookTemplates(): array {
        return [
            'web_server' => [
                'name' => 'Web Server Setup',
                'playbook' => $this->getWebServerPlaybook()
            ],
            'database' => [
                'name' => 'Database Setup',
                'playbook' => $this->getDatabasePlaybook()
            ],
            'security' => [
                'name' => 'Security Hardening',
                'playbook' => $this->getSecurityPlaybook()
            ],
            'monitoring' => [
                'name' => 'Monitoring Setup',
                'playbook' => $this->getMonitoringPlaybook()
            ]
        ];
    }

    // Private helper methods

    private function getWebServerPlaybook(): string {
        return <<<PLAYBOOK
---
- name: Setup Web Server
  hosts: webservers
  become: yes
  vars:
    nginx_version: "1.24"
    php_version: "8.2"

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Configure Nginx
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Reload Nginx

    - name: Install PHP
      apt:
        name:
          - php{{ php_version }}-fpm
          - php{{ php_version }}-mysql
        state: present

  handlers:
    - name: Reload Nginx
      service:
        name: nginx
        state: reloaded
PLAYBOOK;
    }

    private function getDatabasePlaybook(): string {
        return <<<PLAYBOOK
---
- name: Setup Database Server
  hosts: databases
  become: yes

  tasks:
    - name: Install MariaDB
      apt:
        name:
          - mariadb-server
          - mariadb-client
        state: present

    - name: Configure MariaDB
      template:
        src: my.cnf.j2
        dest: /etc/mysql/my.cnf
      notify: Restart MariaDB

    - name: Create WHMCS database
      mysql_db:
        name: whmcs
        state: present

    - name: Create WHMCS user
      mysql_user:
        name: whmcs_user
        password: "{{ mysql_password }}"
        priv: 'whmcs.*:ALL'
        host: '%'
        state: present

  handlers:
    - name: Restart MariaDB
      service:
        name: mariadb
        state: restarted
PLAYBOOK;
    }

    private function getSecurityPlaybook(): string {
        return <<<PLAYBOOK
---
- name: Security Hardening
  hosts: all
  become: yes

  tasks:
    - name: Configure SSH
      template:
        src: sshd_config.j2
        dest: /etc/ssh/sshd_config
      notify: Restart SSH

    - name: Setup UFW
      ufw:
        state: enabled
        policy: deny

    - name: Allow SSH
      ufw:
        rule: allow
        port: '22'
        proto: tcp

    - name: Allow HTTPS
      ufw:
        rule: allow
        port: '443'
        proto: tcp

    - name: Install Fail2Ban
      apt:
        name: fail2ban
        state: present

  handlers:
    - name: Restart SSH
      service:
        name: sshd
        state: restarted
PLAYBOOK;
    }

    private function getMonitoringPlaybook(): string {
        return <<<PLAYBOOK
---
- name: Setup Monitoring
  hosts: all
  become: yes

  tasks:
    - name: Install Prometheus node exporter
      apt:
        name: prometheus-node-exporter
        state: present

    - name: Enable node exporter
      service:
        name: prometheus-node-exporter
        enabled: yes
        state: started
PLAYBOOK;
    }
}

/**
 * Ansible Runner
 */
class AnsibleRunner {
    private $ansiblePath = '/usr/bin/ansible-playbook';
    private $inventoryPath = '/etc/ansible/hosts';
    private $playbookPath = '/var/lib/whmcs/ansible/playbooks';

    public function execute(array $playbook, array $options): array {
        $playbookFile = $this->playbookPath . '/' . $playbook['id'] . '.yml';
        file_put_contents($playbookFile, $playbook['playbook_content']);

        $command = $this->buildCommand($playbookFile, $options);

        $output = shell_exec($command . ' 2>&1');

        return [
            'success' => strpos($output, 'failed=0') !== false,
            'changed' => $this->extractChanged($output),
            'failed' => $this->extractFailed($output),
            'output' => $output
        ];
    }

    private function buildCommand(string $playbookFile, array $options): string {
        $cmd = $this->ansiblePath . ' ' . $playbookFile;
        $cmd .= ' -i ' . $this->inventoryPath;

        if (!empty($options['tags'])) {
            $cmd .= ' --tags ' . implode(',', $options['tags']);
        }

        if ($options['check'] ?? false) {
            $cmd .= ' --check';
        }

        if ($options['diff'] ?? false) {
            $cmd .= ' --diff';
        }

        return $cmd;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_ansible_playbooks` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `description' TEXT,
  `playbook_content' TEXT NOT NULL,
  `inventory' TEXT,
  `tags' TEXT,
  `created_at' DATETIME NOT NULL
);

CREATE TABLE `mod_ansible_executions` (
  `id` VARCHAR(50) PRIMARY KEY,
  `playbook_id' VARCHAR(50) NOT NULL,
  `status' ENUM('running', 'success', 'failed') NOT NULL,
  'changed_count' INT DEFAULT 0,
  'failed_count' INT DEFAULT 0,
  'output' TEXT,
  'duration_seconds' INT,
  'created_at' DATETIME NOT NULL,
  FOREIGN KEY (`playbook_id`) REFERENCES `mod_ansible_playbooks`(`id`)
);
```

## Best Practices

1. **Idempotent Playbooks**: Design for repeatable execution
2. **Use Roles**: Organize playbooks into reusable roles
3. **Tag Tasks**: Use tags for selective execution
4. **Check Mode**: Test with --check before applying
5. **Vault Encryption**: Protect sensitive data with Ansible Vault

## Related Skills

- whmcs-terraform-modules
- whmcs-config-management
- whmcs-gitops-workflow
- whmcs-chef-cookbooks