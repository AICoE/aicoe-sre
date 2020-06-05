# Set up Monitoring

## Installation

Once the cluster admins have installed the Prometheus operator and provided the requisite permissions to the admins, you can run the following commands.

```bash
oc create serviceaccount aicoe-prometheus
oc create -f psi-ocp.yaml
```

## Adding New Namespaces for Prometheus to Monitor

**NOTE:** Steps 1 and 2 require cluster admin permissions.

1. Make sure that MultiNamespace install is set to true

    ```yaml
    installModes:
        - supported: true
          type: OwnNamespace
        - supported: true
          type: SingleNamespace
        - supported: false
          type: MultiNamespace
        - supported: false
          type: AllNamespaces
    ```

2. Make sure that `targetNamespaces` in the Prometheus Operator Group has your namespace

    ```yaml
    spec:
      targetNamespaces:
        - dh-psi-monitoring
        - aicoe-argocd
        - <new_target_namespace>
    ```

3. Give the prometheus service account access to your namespace

    ```bash
    oc project <target_namespace>
    oc policy add-role-to-user view system:serviceaccount:dh-psi-monitoring:aicoe-prometheus
    ```

4. Make sure that the Prometheus CR has a label that matches a label for the target namespace:
This namespace selector can be found at [psi-ocp.yaml](https://github.com/AICoE/aicoe-sre/blob/master/monitoring/prometheus/psi-ocp.yaml#L16)

    ```yaml
    serviceMonitorNamespaceSelector:
      matchLabels:
        <target_namespace_label_key>: <target_namespace_label_value>
    ```

    This label needs to be part of the namespace spec. You can find the namespace labels by running the following command

    ```bash
    oc get project <target_namespace> -o yaml
    ```

## Common issues

### Prometheus unable to detect pod/service monitors

Make sure that the monitor has the `prometheus: aicoe-sre` label on it.
