# is

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 7.0.0](https://img.shields.io/badge/AppVersion-7.0.0-informational?style=flat-square)

A Helm chart for WSO2 Identity server

## Deployment architecture

## Required permission
User or service principle who installs the Helm chart, needs to possess actions `"create", "get", "list", "update", "delete"` on following K8s kinds,

| Kind                    | API Version                      |
|-------------------------|----------------------------------|
| ConfigMap               | v1                               |
| Deployment              | apps/v1                          |
| HorizontalPodAutoscaler | autoscaling/v1                   |
| Ingress                 | networking.k8s.io/v1             |
| PersistentVolume        | v1                               |
| PersistentVolumeClaim   | v1                               |
| PodDisruptionBudget     | policy/v1                        |
| Role                    | rbac.authorization.k8s.io/v1     |
| RoleBinding             | rbac.authorization.k8s.io/v1     |
| SecretProviderClass     | secrets-store.csi.x-k8s.io/v1    |
| Service                 | v1                               |
| ServiceAccount          | v1                               |

## Prerequisites 
* Kubernetes ingress controller. Default integration is [Kubernetes Nginx ingress controller](https://github.com/kubernetes/ingress-nginx). 
* [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/introduction) and [secrets-store-csi-driver-provider-azure](https://github.com/Azure/secrets-store-csi-driver-provider-azure) 
* [Azure Storage account](https://learn.microsoft.com/en-us/azure/aks/azure-csi-files-storage-provision) to cater persistence volume type `ReadWriteMany` [access mode](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) and a share `is` under the Azure storage account. 
* [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/overview) to persists secrets 
* Pre-configured RDBMS. Please refer to [documentation](https://is.docs.wso2.com/en/latest/deploy/set-up-separate-databases-for-clustering/#!) on setting up databases. 

## Install Helm chart on Azure Kubernetes service(AKS)

Please replace the places holders `<NAMESPACE>` with Namespace where the WSO2 Identity server Helm chart is installed,

```shell
export NAMESPACE=<NAMESPACE>
```

1. Create Kubernetes namespace

    ```shell
    kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}
    ```
2.Configure rate limiting configuration 
   Need to configure rate limiting zone configuration under [http-snippet](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#http-snippet) in [Kubernetes Nginx ingress controller](https://github.com/kubernetes/ingress-nginx)

   ```yaml
   http-snippet: |-
     limit_req_zone rate_limit_zone_is zone=is:10m rate=150r/s;
   ```
   
3. Create a Kubernetes TLS secret for SSL termination at ingress controller. For this you need possess the SSL certificate and the key,
   ```shell
   kubectl create secret tls is-tls \
   --cert=path/to/cert/file \
   --key=path/to/key/file \
   -n ${NAMESPACE}
   ```
   
3. Create a Kubernetes secret for keystore files. It is required to have four Java keystore files for the deployment. Please refer to the [documentation](https://is.docs.wso2.com/en/latest/deploy/security/configure-keystores-in-wso2-products/#configure-keystores) for more details and how to create key stores.

* Internal keystore(internal.jks): The key store which is used for encrypting/decrypting internal data
* Primary keystore(primary.jks): Certificates used for signing messages that are communicated with external parties(such SAML, OIDC id_token signing)
* TLS keystore(tls.jks): The key store which is used for tls communication.
* Client truststore(client-truststore.jks): Certificates of trusted third parties

   ```shell
   kubectl create secret generic keystores \
   --from-file=internal.jks \
   --from-file=primary.jks \
   --from-file=tls.jks \
   --from-file=client-truststore.jks \
   -n ${NAMESPACE}
   ```
  
4. Create [Azure storage account secret](https://learn.microsoft.com/en-us/azure/aks/azure-csi-files-storage-provision#create-a-kubernetes-secret) for persistence volume.
   Replace `<AZURE_STORAGE_NAME>` with Azure storage account name and `<AZURE_STORAGE_KEY>` with Azure storage account access key.
  ```shell
   export AZURE_STORAGE_NAME='<AZURE_STORAGE_NAME>'
   export AZURE_STORAGE_KEY='<AZURE_STORAGE_KEY>'
   kubectl create secret generic azure-storage-csi \
   --from-literal=azurestorageaccountname="${AZURE_STORAGE_NAME}" \
   --from-literal=azurestorageaccountkey="${AZURE_STORAGE_KEY}" \
   -n ${NAMESPACE}
   ```

6. Configure Azure key vault 
 - Add `internal.jks` keystore password as the secret with the name `INTERNAL-KEYSTORE-PASSWORD-DECRYPTED`. Replace `<AZURE_KEY_VAULT_NAME>` with Azure Key vault name, `<AZURE_SUBSCRIPTION_ID>` with Azure subscription ID and `<INTERNAL_KEYSTORE_PASSWORD_DECRYPTED>` with internal keystore(`internal.jks`) password.

    ```shell
    export AZURE_KEY_VAULT_NAME='<AZURE_KEY_VAULT_NAME>'
    export AZURE_SUBSCRIPTION_ID='<AZURE_SUBSCRIPTION_ID>'
    export INTERNAL_KEYSTORE_PASSWORD_DECRYPTED='<INTERNAL_KEYSTORE_PASSWORD_DECRYPTED>'
    
    az login
    az account set -s "${AZURE_SUBSCRIPTION_ID}"
    az keyvault secret set --vault-name "${AZURE_KEY_VAULT_NAME}" --name "INTERNAL-KEYSTORE-PASSWORD-DECRYPTED" --value "${INTERNAL_KEYSTORE_PASSWORD_DECRYPTED}"
    ```

- Create a Kubernetes secret to hold service principal credentials to access keyvault for [secrets-store-csi-driver-provider-azure](https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/configurations/identity-access-modes/service-principal-mode/).
  Replace `<AZURE_KEY_VAULT_SP_APP_ID>` with Azure active directory service principle application ID and `<AZURE_KEY_VAULT_SP_APP_SECRET>` with Azure active directory service principle application secret
     
   ```shell

    export AZURE_KEY_VAULT_SP_APP_ID='<AZURE_KEY_VAULT_SP_APP_ID>'
    export AZURE_KEY_VAULT_SP_APP_SECRET='<AZURE_KEY_VAULT_SP_APP_SECRET>'
       
    kubectl create secret generic azure-kv-secret-store-sp \
    --from-literal=clientid="${AZURE_KEY_VAULT_SP_APP_ID}" \
    --from-literal=clientsecret="${AZURE_KEY_VAULT_SP_APP_SECRET}" \
    -n ${NAMESPACE}
    ```

7. Encrypt secrets using WSO2 secure vault encryption

Following set of secure vault encrypted secrets are required for the deployment, please follow the [guideline](https://is.docs.wso2.com/en/latest/deploy/security/encrypt-passwords-with-cipher-tool/) to encrypt secrets using WSO2 secure vault encryption. Make sure to use previously created `internal.jks` keystore for the WSO2 secure vault encryption.

```shell
export DATABASE_IDENTITY_ENCRYPTED_PASSWORD='<Identity database encrypted user password >'
export DATABASE_SHARED_ENCRYPTED_PASSWORD='<Shared database encrypted user password>'
export DATABASE_USER_ENCRYPTED_PASSWORD='<User database encrypted user password>'
export DATABASE_CONSENT_ENCRYPTED_PASSWORD='<Consent database encrypted user password>'
export DATABASE_BPS_ENCRYPTED_PASSWORD='<BPS database encrypted user password>'
export KEYSTORE_INTERNAL_ENCRYPTED_PASSWORD='<Internal key store encrypted password>'
export KEYSTORE_INTERNAL_ENCRYPTED_KEY_PASSWORD='<Internal key store key encrypted password>'
export KEYSTORE_PRIMARY_ENCRYPTED_PASSWORD='<Primary key store encrypted password>'
export KEYSTORE_PRIMARY_ENCRYPTED_KEY_PASSWORD='<Primary key store key encrypted password>'
export KEYSTORE_TLS_ENCRYPTED_PASSWORD='<TLS key store encrypted password>'
export KEYSTORE_TLS_ENCRYPTED_KEY_PASSWORD='<TLS key store key encrypted password>'
export SUPER_ADMIN_ENCRYPTED_PASSWORD='<Super admin user encrypted password>'
export TRUSTSTORE_ENCRYPTED_PASSWORD='<Client truststore encrypted password>'
export IDENTITY_AUTH_FRAMEWORK_ENDPOINT_ENCRYPTED_APP_PASSWORD='<Encrypted app password>'
```

8. Install Helm chart
   Replace `<>` places holders with values as below,
    * **<IMAGE_REGISTRY_HOSTNAME>**: Azure container register(ACR) hostname
    * **<IMAGE_REPOSITORY_NAME>**: Azure container register(ACR) identity server image repository name
    * **<IMAGE_DIGEST>**: Azure container register(ACR) identity server image digest
    * **<AZURE_TENANT_ID>**: Azure tenant ID of Azure Key vault
    * **<AZURE_KEY_VAULT_NAME>**: Azure Key vault name
    * **<AZURE_KEY_VAULT_RG>**: Azure resource group name of Key vault
    * **<DATABASE_IDENTITY_URL>**: Identity database JDBC URL
    * **<DATABASE_IDENTITY_USER>**: Identity database username
    * **<DATABASE_SHARED_URL>**: Shared database JDBC URL
    * **<DATABASE_SHARED_USER>**: Shared database username
    * **<DATABASE_USER_URL>**: User database JDBC URL
    * **<DATABASE_USER_USER>**: User database username
    * **<DATABASE_CONSENT_URL>**: Consent database JDBC URL
    * **<DATABASE_CONSENT_USER>**: Consent database username
    * **<DATABASE_BPS_URL>**: BPS database JDBC URL
    * **<DATABASE_BPS_USER>**: BPS database username
    * **<IS_HOSTNAME>**: Identity server public hostname
    * **<SUPER_ADMIN_USERNAME>**: Identity server super admin username
   
    ```shell
    export IMAGE_REGISTRY_HOSTNAME='<IMAGE_REGISTRY_HOSTNAME>'
    export IMAGE_REPOSITORY_NAME='<IMAGE_REPOSITORY_NAME>'
    export IMAGE_DIGEST='<IMAGE_DIGEST>'
    export AZURE_TENANT_ID='<AZURE_TENANT_ID>'
    export AZURE_KEY_VAULT_NAME='<AZURE_KEY_VAULT_NAME>'
    export AZURE_KEY_VAULT_RG='<AZURE_KEY_VAULT_RG>'
    export DATABASE_IDENTITY_URL='<DATABASE_IDENTITY_URL>'
    export DATABASE_IDENTITY_USER='<DATABASE_IDENTITY_USER>'
    export DATABASE_SHARED_URL='<DATABASE_SHARED_URL>'
    export DATABASE_SHARED_USER='<DATABASE_SHARED_USER>'
    export DATABASE_USER_URL='<DATABASE_USER_URL>'
    export DATABASE_USER_USER='<DATABASE_USER_USER>'
    export DATABASE_CONSENT_URL='<DATABASE_CONSENT_URL>'
    export DATABASE_CONSENT_USER='<DATABASE_CONSENT_USER>'
    export DATABASE_BPS_URL='<DATABASE_CONSENT_URL>'
    export DATABASE_BPS_USER='<DATABASE_BPS_USER>'
    export IS_HOSTNAME='<IS_HOSTNAME>'
    export SUPER_ADMIN_USERNAME='<SUPER_ADMIN_USERNAME>'
    
    helm template is-test . -n "${NAMESPACE}" \
    --set deployment.image.registry="${IMAGE_REGISTRY_HOSTNAME}" \
    --set deployment.image.repository="${IMAGE_REPOSITORY_NAME}" \
    --set deployment.image.digest="${IMAGE_DIGEST}" \
    --set deployment.ingress.hostName="${IS_HOSTNAME}" \
    --set deploymentToml.database.identity.url="${DATABASE_IDENTITY_URL}" \
    --set deploymentToml.database.identity.username="${DATABASE_IDENTITY_USER}" \
    --set deploymentToml.database.identity.encryptedPassword="${DATABASE_IDENTITY_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.database.shared.url="${DATABASE_SHARED_URL}" \
    --set deploymentToml.database.shared.username="${DATABASE_SHARED_USER}" \
    --set deploymentToml.database.shared.encryptedPassword="${DATABASE_SHARED_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.database.user.url="${DATABASE_USER_URL}" \
    --set deploymentToml.database.user.username="${DATABASE_USER_USER}" \
    --set deploymentToml.database.user.encryptedPassword="${DATABASE_USER_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.database.consent.url="${DATABASE_CONSENT_URL}" \
    --set deploymentToml.database.consent.username="${DATABASE_CONSENT_USER}" \
    --set deploymentToml.database.consent.encryptedPassword="${DATABASE_CONSENT_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.database.bps.url="${DATABASE_BPS_URL}" \
    --set deploymentToml.database.bps.username="${DATABASE_BPS_USER}" \
    --set deploymentToml.database.bps.encryptedPassword="${DATABASE_BPS_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.keystore.internal.encryptedPassword="${KEYSTORE_INTERNAL_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.keystore.internal.encryptedKeyPassword="${KEYSTORE_INTERNAL_ENCRYPTED_KEY_PASSWORD}" \
    --set deploymentToml.keystore.primary.encryptedPassword="${KEYSTORE_PRIMARY_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.keystore.primary.encryptedKeyPassword="${KEYSTORE_PRIMARY_ENCRYPTED_KEY_PASSWORD}" \
    --set deploymentToml.keystore.tls.encryptedPassword="${KEYSTORE_TLS_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.keystore.tls.encryptedKeyPassword="${KEYSTORE_TLS_ENCRYPTED_KEY_PASSWORD}" \
    --set deploymentToml.superAdmin.encryptedPassword="${SUPER_ADMIN_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.superAdmin.username="${SUPER_ADMIN_USERNAME}" \
    --set deploymentToml.truststore.encryptedPassword="${TRUSTSTORE_ENCRYPTED_PASSWORD}" \
    --set deploymentToml.identity.authFramework.endpoint.encryptedAppPassword="${IDENTITY_AUTH_FRAMEWORK_ENDPOINT_ENCRYPTED_APP_PASSWORD}" \
    --set deployment.secretStore.azure.keyVault.name="${AZURE_KEY_VAULT_NAME}" \
    --set deployment.secretStore.azure.keyVault.resourceGroup="${AZURE_KEY_VAULT_RG}" \
    --set deployment.secretStore.azure.keyVault.resourceGroup="${AZURE_KEY_VAULT_RG}" \
    --set deployment.secretStore.azure.keyVault.servicePrincipalAppID="${AZURE_KEY_VAULT_SP_APP_ID}" \
    --set deployment.secretStore.azure.keyVault.subscriptionId="${AZURE_SUBSCRIPTION_ID}" \
    --set deployment.secretStore.azure.keyVault.tenantId="${AZURE_TENANT_ID}" 
    ```

# Compatibility

| Kubernetes Version | Helm Version | Secrets Store CSI Driver Version | Compatibility Notes                  |
|--------------------|--------------|----------------------------------|--------------------------------------|
| v1.26.x            | v3.xx        | v1.3.0                           | Fully compatible.                    |

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| deployment.JKSSecretName | string | `"keystores"` | K8s secret name which contains JKS files |
| deployment.apparmor.profile | string | `"runtime/default"` |  |
| deployment.buildVersion | string | `"7.0.0-alpha2"` | Product version |
| deployment.extraVolumeMounts | list | `[]` | Additional volumeMounts to the pods. All the configuration mounts should be done under the path "/home/wso2carbon/wso2-config-volume/" |
| deployment.extraVolumes | list | `[]` | Additional volumes to the pod. |
| deployment.hpa.averageUtilizationCPU | int | `65` | Average CPU utilization for HPA |
| deployment.hpa.averageUtilizationMemory | int | `75` | averageUtilizationMemory parameter should be greater than 75 if not un expected scaling will happen during rolling update. |
| deployment.hpa.enabled | bool | `false` | Enable HPA for the deployment |
| deployment.hpa.maxReplicas | int | `2` | Max replica count for HPA(https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) |
| deployment.image.digest | string | `""` | Container image digest |
| deployment.image.pullPolicy | string | `"Always"` | Refer to the Kubernetes documentation on updating images (https://kubernetes.io/docs/concepts/containers/images/#updating-images) |
| deployment.image.registry | string | `""` | Container image registry host name |
| deployment.image.repository | string | `""` | Container image repository name |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/affinity" | string | `"cookie"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/backend-protocol" | string | `"HTTPS"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/configuration-snippet" | string | `"limit_req zone=is burst=150 nodelay;                                                   \nlimit_req_status 429;\n"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/force-ssl-redirect" | string | `"true"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/proxy-buffer-size" | string | `"64k"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/proxy-buffering" | string | `"on"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/session-cookie-conditional-samesite-none" | string | `"true"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/session-cookie-hash" | string | `"sha1"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/session-cookie-name" | string | `"paf"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/session-cookie-path" | string | `"/"` |  |
| deployment.ingress.annotations."nginx.ingress.kubernetes.io/session-cookie-samesite" | string | `"None"` |  |
| deployment.ingress.hostName | string | `"wso2is.com"` | Host name of the Identity server as Key Manager |
| deployment.ingress.ingressClassName | string | `"nginx"` | Ingress class name |
| deployment.ingress.tlsSecretsName | string | `"is-tls"` | K8s TLS secret for configured hostname |
| deployment.livenessProbe | object | `{"periodSeconds":10}` | Indicates whether the container is running |
| deployment.livenessProbe.periodSeconds | int | `10` | How often (in seconds) to perform the probe |
| deployment.persistence.azure.enabled | bool | `true` | Enable persistence for artifact sharing using Azure file share |
| deployment.persistence.azure.fileShare | string | `"is"` | Names of Azure File shares for persisted data |
| deployment.persistence.azure.secretName | string | `"azure-storage-csi"` | K8s secret name for the Azure file share CI driver  |
| deployment.persistence.capacity | string | `"100Gi"` | Define capacity for persistent runtime artifacts which are shared between instances of the Identity Server profile |
| deployment.persistence.enabled | bool | `false` | Enable persistence for artifact sharing |
| deployment.persistence.subPaths.tenants | string | `"is/tenants"` | Azure storage account tenants file share path |
| deployment.persistence.subPaths.userstores | string | `"is/userstores"` | Azure storage account userstores file share path |
| deployment.preStopHookWaitSeconds | int | `10` | preStopHookWaitInSeconds waits before calling server stop in the pre stop hook. |
| deployment.productPackName | string | `"wso2is"` | Product pack name |
| deployment.progressDeadlineSeconds | int | `600` | Progress deadline seconds where the Deployment controller waits before indicating (in the Deployment status) that the Deployment progress has stalled. |
| deployment.readinessProbe | object | `{"initialDelaySeconds":60,"periodSeconds":10}` | Indicates whether the container is ready to service requests |
| deployment.readinessProbe.initialDelaySeconds | int | `60` | Number of seconds after the container has started before readiness probes are initiated |
| deployment.readinessProbe.periodSeconds | int | `10` | How often (in seconds) to perform the probe |
| deployment.replicas | int | `2` | Number of deployment replicas |
| deployment.resources.jvm.javaOpts | string | `"-Djdk.tls.ephemeralDHKeySize=2048 -Djdk.tls.rejectClientInitiatedRenegotiation=true -Dhttpclient.hostnameVerifier=Strict -Djdk.tls.client.protocols=TLSv1.2 -Djava.util.prefs.systemRoot=/home/wso2carbon/.java -Djava.util.prefs.userRoot=/home/wso2carbon/.java/.userPrefs"` | JVM parameters |
| deployment.resources.jvm.memOpts | string | `"-Xms2048m -Xmx2048m"` | JVM memory options |
| deployment.resources.limits.cpu | string | `"3"` | The maximum amount of CPU that should be allocated for a Pod |
| deployment.resources.limits.memory | string | `"4Gi"` | The maximum amount of memory that should be allocated for a Pod |
| deployment.resources.requests | object | `{"cpu":"2","memory":"2Gi"}` | as per official documentation (https://is.docs.wso2.com/en/latest/setup/installation-prerequisites/) |
| deployment.resources.requests.cpu | string | `"2"` | The minimum amount of CPU that should be allocated for a Pod |
| deployment.resources.requests.memory | string | `"2Gi"` | The minimum amount of memory that should be allocated for a Pod |
| deployment.secretStore.azure.enabled | bool | `true` | Enable Azure Key Vault integration. |
| deployment.secretStore.azure.keyVault.name | string | `""` | Name of the target Azure Key Vault instance |
| deployment.secretStore.azure.keyVault.resourceGroup | string | `""` | Name of the Azure Resource Group to which the target Azure Key Vault belongs |
| deployment.secretStore.azure.keyVault.servicePrincipalAppID | string | `""` | Service Principal created for transacting with the target Azure Key Vault For advanced details refer to official documentation (https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/service-principal-mode.md) |
| deployment.secretStore.azure.keyVault.subscriptionId | string | `""` | Subscription ID of the target Azure Key Vault |
| deployment.secretStore.azure.keyVault.tenantId | string | `""` | Azure Active Directory tenant ID of the target Key Vault |
| deployment.secretStore.azure.nodePublishSecretRef | string | `"azure-kv-secret-store-sp"` | The name of the Kubernetes secret that contains the service principal credentials to access Azure Key Vault. https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/configurations/identity-access-modes/service-principal-mode/#configure-service-principal-to-access-keyvault |
| deployment.securityContext.runAsUser | int | `10001` | Run as user ID |
| deployment.securityContext.seccompProfile.type | string | `"RuntimeDefault"` | Seccomp profile type |
| deployment.startupProbe | object | `{"failureThreshold":30,"initialDelaySeconds":60,"periodSeconds":5}` | Startup probe executed prior to Liveness Probe taking over |
| deployment.startupProbe.failureThreshold | int | `30` | Number of attempts |
| deployment.startupProbe.initialDelaySeconds | int | `60` | Number of seconds after the container has started before startup probes are initiated |
| deployment.startupProbe.periodSeconds | int | `5` | How often (in seconds) to perform the probe |
| deployment.strategy.rollingUpdate.maxSurge | int | `1` | The maximum number of pods that can be scheduled above the desired number of pods |
| deployment.strategy.rollingUpdate.maxUnavailable | int | `0` | The maximum number of pods that can be unavailable during the update |
| deployment.terminationGracePeriodSeconds | int | `40` | Pod termination grace period. K8s API server waits this period after pre stop hook and sending TERM signal |
| deploymentToml.account.recovery.endpoint.auth.hash | string | `""` |  |
| deploymentToml.clustering.domain | string | `"wso2.is.domain"` |  |
| deploymentToml.clustering.enabled | bool | `true` |  |
| deploymentToml.clustering.localMemberPort | string | `"4001"` | This defines local member port |
| deploymentToml.clustering.membershipScheme | string | `"kubernetes"` | This defines membership schema type |
| deploymentToml.database.bps.driver | string | `"com.microsoft.sqlserver.jdbc.SQLServerDriver"` | The database JDBC driver |
| deploymentToml.database.bps.encryptedPassword | string | `""` | The secure vault encrypted database password |
| deploymentToml.database.bps.poolOptions | string | `"maxActive = \"50\" maxWait = \"60000\" minIdle = \"5\" testOnBorrow = true validationInterval = \"30000\" validationQuery = \"SELECT 1; COMMIT\""` | The database pool options |
| deploymentToml.database.bps.type | string | `"mssql"` | The SQL server type(ex: mysql, mssql) |
| deploymentToml.database.bps.url | string | `""` | The database JDBC URL |
| deploymentToml.database.bps.username | string | `""` | The database username |
| deploymentToml.database.consent.driver | string | `"com.microsoft.sqlserver.jdbc.SQLServerDriver"` | The database JDBC driver |
| deploymentToml.database.consent.encryptedPassword | string | `""` | The secure vault encrypted database password |
| deploymentToml.database.consent.poolOptions | string | `"maxActive = \"80\" maxWait = \"60000\" minIdle = \"5\" testOnBorrow = true validationQuery=\"SELECT 1\" validationInterval=\"30000\" defaultAutoCommit=false"` | The database pool options |
| deploymentToml.database.consent.type | string | `"mssql"` | The SQL server type(ex: mysql, mssql) |
| deploymentToml.database.consent.url | string | `""` | The database JDBC URL |
| deploymentToml.database.consent.username | string | `""` | The database username |
| deploymentToml.database.identity.driver | string | `"com.microsoft.sqlserver.jdbc.SQLServerDriver"` | The database JDBC driver |
| deploymentToml.database.identity.encryptedPassword | string | `""` | The secure vault encrypted database password |
| deploymentToml.database.identity.poolOptions | string | `nil` | The database pool options |
| deploymentToml.database.identity.type | string | `"mssql"` | The SQL server type(ex: mysql, mssql) |
| deploymentToml.database.identity.url | string | `""` | The database JDBC URL |
| deploymentToml.database.identity.username | string | `""` | The database username |
| deploymentToml.database.shared.driver | string | `"com.microsoft.sqlserver.jdbc.SQLServerDriver"` | The database JDBC driver |
| deploymentToml.database.shared.encryptedPassword | string | `""` | The secure vault encrypted database password |
| deploymentToml.database.shared.poolOptions | string | `nil` | The database pool options |
| deploymentToml.database.shared.type | string | `"mssql"` | The SQL server type(ex: mysql, mssql) |
| deploymentToml.database.shared.url | string | `""` | The database JDBC URL |
| deploymentToml.database.shared.username | string | `""` | The database username |
| deploymentToml.database.user.driver | string | `"com.microsoft.sqlserver.jdbc.SQLServerDriver"` | The database JDBC driver |
| deploymentToml.database.user.encryptedPassword | string | `""` | The secure vault encrypted database password |
| deploymentToml.database.user.poolOptions | string | `nil` | The database pool options |
| deploymentToml.database.user.type | string | `"mssql"` | The SQL server type(ex: mysql, mssql) |
| deploymentToml.database.user.url | string | `""` | The database JDBC URL |
| deploymentToml.database.user.username | string | `""` | The database username |
| deploymentToml.identity.authFramework.endpoint.encryptedAppPassword | string | `""` |  |
| deploymentToml.keystore.internal.alias | string | `"wso2carbon"` |  |
| deploymentToml.keystore.internal.encryptedKeyPassword | string | `""` |  |
| deploymentToml.keystore.internal.encryptedPassword | string | `""` |  |
| deploymentToml.keystore.internal.fileName | string | `"internal.jks"` |  |
| deploymentToml.keystore.internal.type | string | `"JKS"` |  |
| deploymentToml.keystore.primary.alias | string | `"wso2carbon"` |  |
| deploymentToml.keystore.primary.encryptedKeyPassword | string | `""` |  |
| deploymentToml.keystore.primary.encryptedPassword | string | `""` |  |
| deploymentToml.keystore.primary.fileName | string | `"primary.jks"` |  |
| deploymentToml.keystore.primary.type | string | `"JKS"` |  |
| deploymentToml.keystore.tls.alias | string | `"wso2carbon"` |  |
| deploymentToml.keystore.tls.encryptedKeyPassword | string | `""` |  |
| deploymentToml.keystore.tls.encryptedPassword | string | `""` |  |
| deploymentToml.keystore.tls.fileName | string | `"tls.jks"` |  |
| deploymentToml.keystore.tls.type | string | `"JKS"` |  |
| deploymentToml.outputAdapter.email.enabled | bool | `false` |  |
| deploymentToml.outputAdapter.email.encryptedPassword | string | `""` |  |
| deploymentToml.outputAdapter.email.fromAddress | string | `""` |  |
| deploymentToml.outputAdapter.email.hostname | string | `""` |  |
| deploymentToml.outputAdapter.email.port | int | `587` |  |
| deploymentToml.outputAdapter.email.username | string | `""` |  |
| deploymentToml.recaptcha.apiUrl | string | `""` |  |
| deploymentToml.recaptcha.enabled | bool | `false` |  |
| deploymentToml.recaptcha.encryptedSecretKey | string | `""` |  |
| deploymentToml.recaptcha.encryptedSiteKey | string | `""` |  |
| deploymentToml.recaptcha.verifyUrl | string | `""` |  |
| deploymentToml.server.offset | string | `"0"` | Change default ports(https://is.docs.wso2.com/en/latest/references/default-ports-of-wso2-products/#:~:text=For%20each%20additional%20WSO2%20product,to%20the%20server%20during%20startup.) |
| deploymentToml.superAdmin.createAdminAccount | bool | `true` |  |
| deploymentToml.superAdmin.encryptedPassword | string | `""` |  |
| deploymentToml.superAdmin.username | string | `""` |  |
| deploymentToml.truststore.encryptedPassword | string | `""` |  |
| deploymentToml.truststore.fileName | string | `"client-truststore.jks"` |  |
| deploymentToml.truststore.type | string | `"JKS"` |  |
| deploymentToml.userStore.type | string | `"database_unique_id"` |  |
| k8sKindAPIVersions | object | `{"configMap":"v1","deployment":"apps/v1","horizontalPodAutoscaler":"autoscaling/v1","ingress":"networking.k8s.io/v1","persistentVolume":"v1","persistentVolumeClaim":"v1","podDisruptionBudget":"policy/v1","role":"rbac.authorization.k8s.io/v1","roleBinding":"rbac.authorization.k8s.io/v1","secret":"v1","secretProviderClass":"secrets-store.csi.x-k8s.io/v1","service":"v1","serviceAccount":"v1"}` | K8s API versions for K8s kinds |
| k8sKindAPIVersions.configMap | string | `"v1"` | K8s API version for kind ConfigMap |
| k8sKindAPIVersions.deployment | string | `"apps/v1"` | K8s API version for kind Deployment |
| k8sKindAPIVersions.horizontalPodAutoscaler | string | `"autoscaling/v1"` | K8s API version for kind HorizontalPodAutoscaler |
| k8sKindAPIVersions.ingress | string | `"networking.k8s.io/v1"` | K8s API version for kind Ingress |
| k8sKindAPIVersions.persistentVolume | string | `"v1"` | K8s API version for kind PersistentVolume |
| k8sKindAPIVersions.persistentVolumeClaim | string | `"v1"` | K8s API version for kind PersistentVolumeClaim |
| k8sKindAPIVersions.podDisruptionBudget | string | `"policy/v1"` | K8s API version for kind PodDisruptionBudget |
| k8sKindAPIVersions.role | string | `"rbac.authorization.k8s.io/v1"` | K8s API version for kind Role |
| k8sKindAPIVersions.roleBinding | string | `"rbac.authorization.k8s.io/v1"` | K8s API version for kind RoleBinding |
| k8sKindAPIVersions.secret | string | `"v1"` | K8s API version for kind Secret |
| k8sKindAPIVersions.secretProviderClass | string | `"secrets-store.csi.x-k8s.io/v1"` | K8s API version for kind SecretProviderClass |
| k8sKindAPIVersions.service | string | `"v1"` | K8s API version for kind Service |
| k8sKindAPIVersions.serviceAccount | string | `"v1"` | K8s API version for kind ServiceAccount |
