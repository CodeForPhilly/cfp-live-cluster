# Choose Native Plants Application (Live Cluster)

This README provides instructions for deploying and managing the Choose Native Plants application in the live Kubernetes cluster.

## Deployment Process

1. Update the `.holo/sources/choose-native-plants.toml` file to point to the new version tag:

```toml
[holosource]
url = "https://github.com/CodeForPhilly/pa-wildflower-selector"
ref = "refs/tags/vX.Y.Z"  # Replace with the new version tag
```

2. Update the `release-values.yaml` file with any necessary changes (e.g., new version number):

```yaml
app:
  image:
    tag: "X.Y.Z"  # Replace with the new version number (without the 'v' prefix)
```

Ensure that the version in both files match (with the toml file using the 'v' prefix and release-values.yaml without it).

3. Create or update any sealed secrets as needed:

```powershell
# Create a regular Kubernetes secret YAML file
# Example for PAC API credentials
kubectl create secret generic pac-api --from-literal=PAC_API_BASE_URL=https://app.plantagents.org --from-literal=PAC_API_KEY=your-api-key --dry-run=client -o yaml > pac-api-secret.yaml

# Seal the secret
kubeseal --cert https://sealed-secrets.live.k8s.phl.io/v1/cert.pem --namespace choose-native-plants -f pac-api-secret.yaml -w cfp-live-cluster/choose-native-plants.secrets/pac-api.yaml
```

4. Commit and push the changes to the repository:

```powershell
git add release-values.yaml choose-native-plants.secrets/
git commit -m "Update Choose Native Plants configuration"
git push origin main
```

4. The GitOps process will automatically deploy the changes to the live cluster.

## Important Notes

- Direct Helm commands are not used in the live environment. All changes are applied through the GitOps workflow using materialized manifests.
- The deployment may take some time to complete after pushing changes.
- Monitor the pod status to verify the deployment: `kubectl get pods -n choose-native-plants`
- The application is accessible at:
  - https://choose-native-plants.live.k8s.phl.io/
  - https://choosenativeplants.com/
  - https://www.choosenativeplants.com/

## Managing the Application

### Syncing Images

To sync images with the Linode Object Storage, use this command which automatically finds the current pod:

```powershell
$POD_NAME = kubectl get pods -n choose-native-plants -l app.kubernetes.io/name=choose-native-plants -o jsonpath='{.items[0].metadata.name}'
kubectl exec -it $POD_NAME -n choose-native-plants -c choose-native-plants-app -- /bin/sh /app/scripts/sync-images
```

### Downloading Data (Skip Images)

To download data without images, use this command:

```powershell
$POD_NAME = kubectl get pods -n choose-native-plants -l app.kubernetes.io/name=choose-native-plants -o jsonpath='{.items[0].metadata.name}'
kubectl exec -it $POD_NAME -n choose-native-plants -c choose-native-plants-app -- node download --skip-images
```

### Massaging Data

To process the data, use this command:

```powershell
$POD_NAME = kubectl get pods -n choose-native-plants -l app.kubernetes.io/name=choose-native-plants -o jsonpath='{.items[0].metadata.name}'
kubectl exec -it $POD_NAME -n choose-native-plants -c choose-native-plants-app -- node massage
```