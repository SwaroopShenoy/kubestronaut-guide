# CRDs and Operators

Up: [CKA hub](../../CKA_2026_Complete_Crash_Course.md) · Domain 2 — Cluster Architecture (25%) · Prev: [etcd backup and restore](etcd-backup-restore.md) · Next: [Services and DNS](../03-services-networking/services-and-dns.md)

A CustomResourceDefinition (CRD) adds a new resource type to the API. An operator is a controller that watches those resources and acts on them.

## Create a CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames: [ct]
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              cronSpec:
                type: string
              image:
                type: string
              replicas:
                type: integer
```

Create an instance:

```yaml
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: my-cron
spec:
  cronSpec: "*/5 * * * *"
  image: my-cron-image
  replicas: 3
```

Check it:

```bash
kubectl apply -f crd.yaml
kubectl get crd | grep example.com
kubectl apply -f crontab.yaml
kubectl get crontabs            # or: kubectl get ct
kubectl explain crontab.spec
```

Validation rules can be added under a schema property with `x-kubernetes-validations` (CEL expressions). Use that for validation without a webhook.

## Find an operator's CRDs

```bash
kubectl get crd
kubectl api-resources | grep <group>
kubectl explain <resource>.spec
```

## Exam expectations

For CKA, apply an existing CRD, create instances, and list or describe them. Writing a controller is out of scope.

## Common mistakes

- Using the wrong `apiVersion` for the custom resource. It must be `<group>/<version>` from the CRD.
- Typing the plural name wrong. `kubectl get` uses the plural or the short name.
- Forgetting that deleting a CRD deletes all its instances.

## Quick reference

```bash
kubectl get crd
kubectl api-resources --api-group=<group>
kubectl explain <plural>.spec
```
