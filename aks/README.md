### Prequisites

1. An Azure subscription. If you don't have an Azure subscription, you can create a [free account](https://azure.microsoft.com/free).
1. [Azure CLI installed](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).
1. Install kubectl 1.18 or later
   ```bash
   sudo az aks install-cli

   # ensure you got a version 1.18 or greater
   kubectl version --client
   ```
1. [Register the AAD-V2 feature for AKS-managed Azure AD](https://docs.microsoft.com/en-us/azure/aks/managed-aad#before-you-begin)
1. Provision [a regional hub and spoke virtual networks](./secure-baseline/networking/network-deploy.azcli)
   > Note: execute this step from VSCode for a better experience
1. Generate a CA self-signed cert

   > :warning: WARNING
   > Do not use the certificates created by these scripts for production. The certificates are provided for demonstration purposes only. For your production cluster, use your security best practices for digital certificates creation and lifetime management.
   > Self-signed certificates are not trusted by default and they can be difficult to maintain. Also, they may use outdated hash and cipher suites that may not be strong. For better security, purchase a certificate signed by a well-known certificate authority.

   Cluster Certificate - AKS Internal Load Balancer

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
         -out traefik-ingress-internal-aks-ingress-contoso-com-tls.crt \
         -keyout traefik-ingress-internal-aks-ingress-contoso-com-tls.key \
         -subj "/CN=*.aks-ingress.contoso.com/O=Contoso Aks Ingress" && \
   rootCertWilcardIngressController=$(cat traefik-ingress-internal-bicycle-contoso-com-tls.crt | base64 -w 0)
   ```

   App Gateway Certificate

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
          -out appgw.crt \
          -keyout appgw.key \
          -subj "/CN=bicycle.contoso.com/O=Contoso Bicycle" && \
   openssl pkcs12 -export -out appgw.pfx -in appgw.crt -inkey appgw.key -passout pass: && \
   appGatewayListernerCertificate=$(cat appgw.pfx | base64 -w 0)
   ```
1. create [the BU 0001's app team secure AKS cluster (ID: A0008)](./secure-baseline/cluster-deploy.azcli)
   > Note: execute this step from VSCode for a better experience
1. Get the AKS Ingress Controller User Assigned Identity details
   ```bash
   export TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.aksIngressControllerUserManageIdentityResourceId.value -o tsv)
   export TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.aksIngressControllerUserManageIdentityClientId.value -o tsv)
   ```
1. Get Azure KeyVault name
   ```bash
   export KEYVAULT_NAME=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.keyVaultName.value -o tsv)
   ```
1. Get the Azure Tenant Id
   ```bash
   export TENANT_ID=$(az account show --query tenantId --output tsv)
   ```

### Manually deploy the Ingress Controller and a basic workload

The following example creates the Ingress Controller (Traefik),
the ASPNET Core Docker sample web app and an Ingress object to route to its service.

```bash
# Import the *.aks-ingress.contoso.com to AzureKeyVault
az keyvault set-policy --certificate-permissions import -n $KEYVAULT_NAME --upn $(az account show --query user.name -o tsv) && \
cat traefik-ingress-internal-aks-ingress-contoso-com-tls.crt traefik-ingress-internal-aks-ingress-contoso-com-tls.key > traefik-ingress-internal-aks-ingress-contoso-com-tls.pem && \
az keyvault certificate import --vault-name $KEYVAULT_NAME -f traefik-ingress-internal-aks-ingress-contoso-com-tls.pem -n traefik-ingress-internal-aks-ingress-contoso-com-tls

# Ensure Flux has created the following namespace and then press Ctrl-C
kubectl get ns a0008 -w

# Create the traefik Azure Identity and the Azure Identity Binding to let
# Azure Active Directory Pod Identity to get tokens on behalf of the Traefik's User Assigned
# Identity and later on assign them to the Traefik's pod
cat <<EOF | kubectl apply -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: aksic-to-keyvault-identity
  namespace: a0008
spec:
  type: 0
  resourceID: $TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID
  clientID: $TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID
---
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: aksic-to-keyvault-identity-binding
  namespace: a0008
spec:
  azureIdentity: aksic-to-keyvault-identity
  selector: traefik-ingress-controller
EOF

# Create a `SecretProviderClasses` resource with with your Azure KeyVault parameters
# for the Secrets Store CSI driver.
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: aks-ingress-contoso-com-tls-secret-csi-akv
  namespace: a0008
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"
    keyvaultName: "${KEYVAULT_NAME}"
    objects:  |
      array:
        - |
          objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
          objectAlias: tls.crt
          objectType: cert
        - |
          objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
          objectAlias: tls.key
          objectType: secret
    tenantId: "${TENANT_ID}"
EOF

# Install Traefik ingress controller with Azure CSI Provider to obtain
# the TLS certificates from Azure KeyVault.

  > :warning: IMPORTANT
  > Azure Pod Identity, there can be some amount of delay in obtaining the certificates from Azure KeyVault.
  > During the Traefik's pod creation time, aad-pod-identity will need to create the AzureAssignedIdentity for the pod based on the AzureIdentity
  > and AzureIdentityBinding, retrieve token for Azure KeyVault. This process can take time to complete and it's possible
  > for the pod volume mount to fail during this time. When the volume mount fails, kubelet will keep retrying until it succeeds.
  > So the volume mount will eventually succeed after the whole process for retrieving the token is complete.
  > For more informaton, please refer to https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/pod-identity-mode.md

kubectl apply -f https://raw.githubusercontent.com/mspnp/reference-architectures/master/aks/workload/traefik.yaml

# Wait for Traefik to be ready
kubectl wait --namespace a0008 \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=traefik-ingress-ilb \
  --timeout=90s

# Check Traefik is handling HTTPS
kubectl -n a0008 run -i --rm --generator=run-pod/v1 --tty alpine --image=alpine -- sh
echo '10.240.4.4 hello.aks-ingress.contoso.com' | tee -a /etc/hosts
apk add openssl
echo | openssl s_client -showcerts -servername hello.aks-ingress.contoso.com -connect hello.aks-ingress.contoso.com:443 2>/dev/null | openssl x509 -inform pem -noout -text
exit 0

# Install the ASPNET core sample web app
kubectl apply -f https://raw.githubusercontent.com/mspnp/reference-architectures/master/aks/workload/aspnetapp.yaml

# the ASPNET Core webapp sample is all setup. Wait until is ready to process requests running:
kubectl wait --namespace a0008 \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=aspnetapp \
  --timeout=90s

# In this momment your Ingress Controller (Traefik) is reading your Ingress
# resource object configuration, updating its status and creating a router to
# fulfill the new exposed workloads route.
# Please take a look at this and notice that the Address is set with the Internal Load Balancer Ip from
# the configured subnet

kubectl get ingress aspnetapp-ingress -n a0008

# Validate the router to the workload is configured, SSL offloading and allowing only known Ips
# Please notice only the Azure Application Gateway is whitelisted as known client for
# the workload's router. Therefore, please expect a Http 403 response
# as a way to probe the router has been properly configured

kubectl -n a0008 run -i --rm --generator=run-pod/v1 --tty curl --image=curlimages/curl -- sh
curl --insecure -k -I --resolve bu0001a0008-00.aks-ingress.contoso.com:443:10.240.4.4 https://bu0001a0008-00.aks-ingress.contoso.com
exit 0
```

### Test the web app

```bash
# query the BU 0001's Azure Application Gateway Public Ip

export APPGW_PUBLIC_IP=$(az deployment group show --resource-group rg-enterprise-networking-spokes -n spoke-BU0001A0008 --query properties.outputs.appGwPublicIpAddress.value -o tsv)
```

1. Map the Azure Application Gateway public ip address to the application domain names. To do that, please open your hosts file (C:\windows\system32\drivers\etc\hosts or /etc/hosts) and add the following record in local host file:
   \${APPGW_PUBLIC_IP} bicycle.contoso.com

1. In your browser navigate the site anyway (A warning will be present)
   https://bicycle.contoso.com/
