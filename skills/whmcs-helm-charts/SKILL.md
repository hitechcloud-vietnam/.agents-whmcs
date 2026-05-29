---
name: whmcs-helm-charts
description: Helm chart packaging for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Helm Charts Skill

## Overview
This skill provides patterns and implementations for creating and managing Helm charts for WHMCS applications.

## Implementation Patterns

### Helm Chart Manager
```php
<?php
/**
 * WHMCS Helm Chart Management
 * Creates and manages Helm charts
 */

namespace WHMCS\Module\DevOps\Helm;

class HelmChartManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Create Helm chart
     */
    public function createChart(array $params): array {
        $chartId = 'chart_' . bin2hex(random_bytes(8));

        $chartDir = "/var/lib/whmcs/helm/charts/{$chartId}";

        // Create chart structure
        $this->createChartStructure($chartDir, $params);

        // Create Chart.yaml
        $chartYaml = $this->generateChartYaml($params);
        file_put_contents($chartDir . '/Chart.yaml', $chartYaml);

        // Create values.yaml
        $valuesYaml = $this->generateValuesYaml($params);
        file_put_contents($chartDir . '/values.yaml', $valuesYaml);

        // Create templates
        $this->createTemplates($chartDir, $params);

        $chart = [
            'id' => $chartId,
            'name' => $params['name'],
            'version' => $params['version'] ?? '1.0.0',
            'path' => $chartDir,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_helm_charts', $chart);

        return [
            'success' => true,
            'chart_id' => $chartId,
            'chart_path' => $chartDir
        ];
    }

    /**
     * Package Helm chart
     */
    public function packageChart(string $chartId): array {
        $chart = $this->getChart($chartId);

        if (!$chart) {
            throw new \Exception("Chart not found");
        }

        $outputDir = "/var/lib/whmcs/helm/packages";
        $command = "helm package {$chart['path']} -d {$outputDir}";

        exec($command, $output, $return);

        if ($return !== 0) {
            throw new \Exception("Failed to package chart");
        }

        $packageFile = "{$outputDir}/{$chart['name']}-{$chart['version']}.tgz";

        $this->db->update('mod_helm_charts', [
            'package_path' => $packageFile,
            'packaged_at' => date('Y-m-d H:i:s')
        ], ['id' => $chartId]);

        return [
            'success' => true,
            'chart_id' => $chartId,
            'package' => $packageFile
        ];
    }

    /**
     * Generate WHMCS Helm chart template
     */
    public function generateWHMCSChart(array $params): array {
        $chartDir = "/tmp/whmcs-chart";

        // Chart.yaml
        $chartYaml = <<<YAML
apiVersion: v2
name: whmcs
description: A Helm chart for WHMCS
type: application
version: 1.0.0
appVersion: "8.0"
keywords:
  - whmcs
  - billing
maintainers:
  - name: WHMCS
    email: support@whmcs.com
YAML;

        // values.yaml
        $valuesYaml = <<<YAML
# Default values for WHMCS
replicaCount: 1

image:
  repository: whmcs/whmcs
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80
  httpsPort: 443

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: whmcs.local
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: whmcs-tls
      hosts:
        - whmcs.local

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 100m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

database:
  host: whmcs-db
  port: 3306
  name: whmcs
  user: whmcs

redis:
  enabled: true
  host: whmcs-redis
  port: 6379

persistence:
  enabled: true
  storageClass: standard
  size: 10Gi
YAML;

        return [
            'chart_yaml' => $chartYaml,
            'values_yaml' => $valuesYaml
        ];
    }

    // Private helper methods

    private function createChartStructure(string $dir, array $params): void {
        $dirs = [
            $dir,
            $dir . '/templates',
            $dir . '/charts'
        ];

        foreach ($dirs as $d) {
            if (!is_dir($d)) {
                mkdir($d, 0755, true);
            }
        }
    }

    private function generateChartYaml(array $params): string {
        return <<<YAML
apiVersion: v2
name: {$params['name']}
description: {$params['description'] ?? 'A Helm chart'}
type: application
version: {$params['version'] ?? '1.0.0'}
appVersion: "{$params['app_version'] ?? '1.0'}"
YAML;
    }

    private function generateValuesYaml(array $params): string {
        return <<<YAML
# Default values
replicaCount: 1

image:
  repository: {$params['image_repo'] ?? 'nginx'}
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
YAML;
    }

    private function createTemplates(string $chartDir, array $params): void {
        // deployment.yaml
        $deployment = <<<DEPLOY
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - name: http
              containerPort: 80
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
DEPLOY;
        file_put_contents($chartDir . '/templates/deployment.yaml', $deployment);

        // service.yaml
        $service = <<<SERVICE
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Chart.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
SERVICE;
        file_put_contents($chartDir . '/templates/service.yaml', $service);
    }
}
```

## Chart Structure
```
whmcs-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration
├── templates/          # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl    # Template helpers
└── charts/             # Sub-charts (optional)
```

## Best Practices

1. **Versioning**: Follow semantic versioning for charts
2. **Documentation**: Include README and NOTES.txt
3. **Template Testing**: Use helm lint and helm unittest
4. **Values Schema**: Define values schema for validation
5. **Dependencies**: Manage dependencies properly

## Related Skills

- whmcs-kubernetes-modules
- whmcs-docker-compose
- whmcs-canary-releases
- whmcs-blue-green-deploy