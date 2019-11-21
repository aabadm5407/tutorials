---
subcollection: solution-tutorials
copyright:
  years: 2018, 2019
lastupdated: "2019-11-05"
lasttested: "2019-07-31"

---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:screen: .screen}
{:tip: .tip}
{:pre: .pre}

# Apply end to end security to a cloud application - In Progress
{: #cloud-e2e-security-inprogress}

No application architecture is complete without a clear understanding of potential security risks and how to protect against such threats. Application data is a critical resource which can not be lost, compromised or stolen. Additionally, data should be protected at rest and in transit through encryption techniques. Encrypting data at rest protects information from disclosure even when it is lost or stolen. Encrypting data in transit (e.g. over the Internet) through methods such as HTTPS, SSL, and TLS prevents eavesdropping and so called man-in-the-middle attacks.

Authenticating and authorizing users' access to specific resources is another common requirement for many applications. Different authentication schemes may need to be supported: customers and suppliers using social identities, partners from cloud-hosted directories, and employees from an organization’s identity provider.

This tutorial walks you through key security services available in the {{site.data.keyword.cloud}} catalog and how to use them together. An application that provides file sharing will put security concepts into practice.
{:shortdesc}

## Objectives
{: #objectives}

* Encrypt content in storage buckets with your own encryption keys
* Require users to authenticate before accessing an application
* Monitor and audit security-related API calls and other actions across cloud services

## Services used
{: #services}

This tutorial uses the following runtimes and services:
* [{{site.data.keyword.containershort_notm}}](https://{DomainName}/kubernetes/catalog/cluster)
* [{{site.data.keyword.registryshort_notm}}](https://{DomainName}/kubernetes/registry/main/start)
* [{{site.data.keyword.appid_short}}](https://{DomainName}/catalog/services/AppID)
* [{{site.data.keyword.cloudant_short_notm}}](https://{DomainName}/catalog/services/cloudantNoSQLDB)
* [{{site.data.keyword.cos_short}}](https://{DomainName}/catalog/services/cloud-object-storage)
* [{{site.data.keyword.at_short}}](https://{DomainName}/observe/activitytracker/create)
* [{{site.data.keyword.keymanagementserviceshort}}](https://{DomainName}/catalog/services/key-protect)
* Optional: [{{site.data.keyword.cloudcerts_short}}](https://{DomainName}/catalog/services/certificate-manager)

This tutorial requires a [non-Lite account](https://{DomainName}/docs/account?topic=account-accounts#accounts) and may incur costs. Use the [Pricing Calculator](https://{DomainName}/estimator/review) to generate a cost estimate based on your projected usage.

## Architecture
{: #architecture}

The tutorial features a sample application that enables groups of users to upload files to a common storage pool and to provides access to those files via shareable links. The application is written in Node.js and deployed as a Docker container to the {{site.data.keyword.containershort_notm}}. It leverages several security-related services and features to improve the application's security posture.

<p style="text-align: center;">

  ![Architecture](images/solution34-cloud-e2e-security/Architecture.png)
</p>

1. User connects to the application.
2. If using a custom domain and a TLS certificate, the certificate is managed by and deployed from the {{site.data.keyword.cloudcerts_short}}.
3. {{site.data.keyword.appid_short}} secures the application and redirects the user to the authentication page. Users can also sign up.
4. The application runs in a Kubernetes cluster from an image stored in the {{site.data.keyword.registryshort_notm}}. This image is automatically scanned for vulnerabilities.
5. Uploaded files are stored in {{site.data.keyword.cos_short}} with accompanying metadata stored in {{site.data.keyword.cloudant_short_notm}}.
6. File storage buckets leverage a user-provided key to encrypt data.
7. Application management activities are logged by {{site.data.keyword.at_full_notm}}.

## Before you begin
{: #prereqs}

1. Install all the necessary command line (CLI) tools by [following these steps](https://{DomainName}/docs/cli?topic=cloud-cli-getting-started). The following plugins are required:
  * container-service/kubernetes-service
  * activity-tracker
2. Ensure you have the latest version of plugins used in this tutorial; use `ibmcloud plugin update --all` to upgrade.

## Create services
{: #setup}

### Decide where to deploy the application

1. Identify the **location** and **resource group** where you will deploy the application and its resources.

### Capture user and application activities
{: #activity-tracker }

The {{site.data.keyword.at_full_notm}} service records user-initiated activities that change the state of a service in {{site.data.keyword.Bluemix_notm}}. At the end of this tutorial, you will review the events that were generated by completing the tutorial's steps.

1. Access the {{site.data.keyword.cloud_notm}} catalog and create an instance of [{{site.data.keyword.at_full_notm}}](https://{DomainName}/observe/activitytracker/create). Note that there can only be one instance of {{site.data.keyword.at_short}} per region. Set the **Service name** to **secure-file-storage-activity-tracker**.
1. Ensure you have the right permissions assigned to manage the service instance by following [these instructions](/docs/services/Activity-Tracker-with-LogDNA?topic=logdnaat-iam_manage_events#admin_account_opt1).

### Create a cluster for the application

{{site.data.keyword.containershort_notm}} provides an environment to deploy highly available apps in Docker containers that run in Kubernetes clusters.

Skip this section if you have an existing cluster you want to reuse with this tutorial.
{: tip}

| Kubernetes on Classic |
|:-----------------|
| Access the [cluster creation page](https://{DomainName}/kubernetes/catalog/cluster/create). | 
| 1. Set the **Location** to the one used in previous steps. |
| 1. Set **Cluster type** to **Standard**. |
| 1. Set **Availability** to **Single Zone**. |
| 1. Select a **Master Zone**. |
| 1. Keep the default **Kubernetes version** and **Hardware isolation**. |
{: class="simple-tab-table"}
{: caption="Table 1. IAM roles" caption-side="top"}
{: #simpletabtable1}
{: tab-title="Kubernetes on Classic"}
{: tab-group="K8s-simple"}
{: class="simple-tab-table"}

| Kubernetes on VPC |
|:-----------------|
| 1. Access the [cluster creation page](https://{DomainName}/kubernetes/catalog/cluster/create). | 
| 2. Set the **Location** to the one used in previous steps. |
| 3. Set **Cluster type** to **Standard**. |
| 4. Set **Availability** to **Single Zone**. |
| 5. Select a **Master Zone**. |
| 6. Keep the default **Kubernetes version** and **Hardware isolation**. |
{: caption="Table 2. IAM roles - Access" caption-side="top"}
{: #simpletabtable2}
{: tab-title="Kubernetes on VPC"}
{: tab-group="K8s-simple"}
{: class="simple-tab-table"}


1. Access the [cluster creation page](https://{DomainName}/kubernetes/catalog/cluster/create).
   1. Set the **Location** to the one used in previous steps.
   2. Set **Cluster type** to **Standard**.
   3. Set **Availability** to **Single Zone**.
   4. Select a **Master Zone**.
2. Keep the default **Kubernetes version** and **Hardware isolation**.
3. If you plan to deploy only this tutorial on this cluster, set **Worker nodes** to **1**.
4. Set the **Cluster name** to **secure-file-storage-cluster**.
5. Click the **Create Cluster** button.

While the cluster is being provisioned, you will create the other services required by the tutorial.

### Use your own encryption keys

{{site.data.keyword.keymanagementserviceshort}} helps you provision encrypted keys for apps across {{site.data.keyword.Bluemix_notm}} services. {{site.data.keyword.keymanagementserviceshort}} and {{site.data.keyword.cos_full_notm}} [work together to protect your data at rest](https://{DomainName}/docs/services/key-protect/integrations?topic=key-protect-integrate-cos#integrate-cos). In this section, you will create one root key for the storage bucket.

1. Create an instance of [{{site.data.keyword.keymanagementserviceshort}}](https://{DomainName}/catalog/services/kms).
   * Set the name to **secure-file-storage-kp**.
   * Select the resource group where to create the service instance.
2. Under **Manage**, click the **Add Key** button to create a new root key. It will be used to encrypt the storage bucket content.
   * Set the name to **secure-file-storage-root-enckey**.
   * Set the key type to **Root key**.
   * Then **Generate key**.

Bring your own key (BYOK) by [importing an existing root key](https://{DomainName}/docs/services/key-protect?topic=key-protect-import-root-keys#import-root-keys).
{: tip}

### Setup storage for user files

The file sharing application saves files to a {{site.data.keyword.cos_short}} bucket. The relationship between files and users is stored as metadata in a {{site.data.keyword.cloudant_short_notm}} database. In this section, you'll create and configure these services.

#### A bucket for the content

1. Create an instance of [{{site.data.keyword.cos_short}}](https://{DomainName}/catalog/services/cloud-object-storage).
   * Set the **name** to **secure-file-storage-cos**.
   * Use the same **resource group** as for the previous services.
2. Under **Service credentials**, create a *New credential*.
   * Set the **name** to **secure-file-storage-cos-acckey**.
   * Set **Role** to **Writer**.
   * Do not specify a **Service ID**.
   * Set **Inline Configuration Parameters** to **{"HMAC":true}**. This is required to generate pre-signed URLs.
   * Click **Add**.
   * Make note of the credentials by clicking **View credentials**. You will need them in a later step.
3. Click **Endpoint** from the menu: set **Resiliency** to **Regional** and set the **Location** to the target location. Copy the **Private** service endpoint. It will be used later in the configuration of the application.

Before creating the bucket, you will grant **secure-file-storage-cos** access to the root key stored in **secure-file-storage-kp**.

1. Go to [Identity & Access > Authorizations](https://{DomainName}/iam/#/authorizations) in the {{site.data.keyword.cloud_notm}} console.
2. Click the **Create** button.
3. In the **Source service** menu, select **Cloud Object Storage**.
4. In the **Source service instance** menu, select the **secure-file-storage-cos** service previously created.
5. In the **Target service** menu, select **Key Protect**.
6. In the **Target service instance** menu, select the **secure-file-storage-kp** service to authorize.
7. Enable the **Reader** role.
8. Click the **Authorize** button.

Finally create the bucket.

1. Access the **secure-file-storage-cos** service instance from the [Resource List](https://{DomainName}/resources).
2. Click **Create bucket**.
   1. Set the **name** to a unique value, such as **&lt;your-initials&gt;-secure-file-upload**.
   2. Set **Resiliency** to **Regional**.
   3. Set **Location** to the same location where you created the **secure-file-storage-kp** service.
   4. Set **Storage class** to **Standard**
3. Select the checkbox to **Add Key Protect Keys**.
   1. Select the **secure-file-storage-kp** service.
   2. Select **secure-file-storage-root-enckey** as the key.
4. Click **Create bucket**.

#### A database map relationships between users and their files

The {{site.data.keyword.cloudant_short_notm}} database will contain metadata for all files uploaded from the application.

1. Create an instance of [{{site.data.keyword.cloudant_short_notm}}](https://{DomainName}/catalog/services/cloudantNoSQLDB).
   * Set the **name** to **secure-file-storage-cloudant**.
   * Set the location.
   * Use the same **resource group** as for the previous services.
   * Set **Available authentication methods** to **Use only IAM**.
2. Under **Service credentials**, create *New credential*.
   * Set the **name** to **secure-file-storage-cloudant-acckey**.
   * Set **Role** to **Manager**
   * Keep the default values for the *Optional* fields
   * **Add**.
3. Make note of the credentials by clicking **View credentials**. You will need them in a later step.
4. Under **Manage**, launch the Cloudant dashboard.
5. Click **Create Database** to create a database named **secure-file-storage-metadata**.

{{site.data.keyword.cloudant_short_notm}} instances on dedicated hardware allow private endpoints. Instances with dedicated service plans allow IP whitelisting. See [{{site.data.keyword.cloudant_short_notm}} Secure access control](https://{DomainName}/docs/services/Cloudant?topic=cloudant-security#secure-access-control) for details.
{: tip}

### Authenticate users

With {{site.data.keyword.appid_short}}, you can secure resources and add authentication to your applications. {{site.data.keyword.appid_short}} [integrates](https://{DomainName}/docs/containers?topic=containers-ingress_annotation#appid-auth) with {{site.data.keyword.containershort_notm}} to authenticate users accessing applications deployed in the cluster.

1. Create an instance of [{{site.data.keyword.appid_short}}](https://{DomainName}/catalog/services/AppID).
   * Set the **Service name** to **secure-file-storage-appid**.
   * Use the same **location** and **resource group** as for the previous services.
2. Under **Manage Authentication**, in the **Authentication Settings** tab, add a **web redirect URL** pointing to the domain you will use for the application. For example, if your cluster Ingress subdomain is
`<cluster-name>.us-south.containers.appdomain.cloud`, the redirect URL will be `https://secure-file-storage.<cluster-name>.us-south.containers.appdomain.cloud/appid_callback`. {{site.data.keyword.appid_short}} requires the web redirect URL to be **https**. You can view your Ingress subdomain in the cluster dashboard or with `ibmcloud ks cluster-get <cluster-name>`.

You should customize the identity providers used as well as the login and user management experience in the {{site.data.keyword.appid_short}} dashboard. This tutorial uses the defaults for simplicity. For a production environment, consider to use Multi-Factor Authentication (MFA) and advanced password rules.
{: tip}

## Deploy the app

All services have been configured. In this section you will deploy the tutorial application to the cluster.

### Get the code

1. Get the application's code:
   ```sh
   git clone https://github.com/IBM-Cloud/secure-file-storage
   ```
   {: codeblock}
2. Go to the **secure-file-storage** directory:
   ```sh
   cd secure-file-storage
   ```
   {: codeblock}

### Build the Docker image

1. [Build the Docker image](https://{DomainName}/docs/services/Registry?topic=registry-registry_images_#registry_images_creating) in {{site.data.keyword.registryshort_notm}}.
   - Find the registry endpoint with `ibmcloud cr info`, such as **us**.icr.io or **uk**.icr.io.
   - Create a namespace to store the container image.
      ```sh
      ibmcloud cr namespace-add secure-file-storage-namespace
      ```
      {: codeblock}
   - Use **secure-file-storage** as the image name.

      ```sh
      ibmcloud cr build -t <location>.icr.io/secure-file-storage-namespace/secure-file-storage:latest .
      ```
      {: codeblock}

### Fill in credentials and configuration settings

1. Copy `credentials.template.env` to `credentials.env`:
   ```sh
   cp credentials.template.env credentials.env
   ```
   {: codeblock}
2. Edit `credentials.env` and fill in the blanks with these values:
   * the {{site.data.keyword.cos_short}} service regional endpoint, the bucket name, the credentials created for **secure-file-storage-cos**,
   * and the credentials for **secure-file-storage-cloudant**.
3. Copy `secure-file-storage.template.yaml` to `secure-file-storage.yaml`:
   ```sh
   cp secure-file-storage.template.yaml secure-file-storage.yaml
   ```
   {: codeblock}
4. Edit `secure-file-storage.yaml` and replace the placeholders (`$IMAGE_PULL_SECRET`, `$REGISTRY_URL`, `$REGISTRY_NAMESPACE`, `$IMAGE_NAME`, `$TARGET_NAMESPACE`, `$INGRESS_SUBDOMAIN`, `$INGRESS_SECRET`) with the correct values. As example, assuming the application is deployed to the _default_ Kubernetes namespace:

| Variable | Value | Description |
| -------- | ----- | ----------- |
| `$IMAGE_PULL_SECRET` | Keep the lines commented in the .yaml | A secret to access the registry.  |
| `$REGISTRY_URL` | *us.icr.io* | The registry where the image was built in the previous section. |
| `$REGISTRY_NAMESPACE` | *secure-file-storage-namespace* | The registry namespace where the image was built in the previous section. |
| `$IMAGE_NAME` | *secure-file-storage* | The name of the Docker image. |
| `$TARGET_NAMESPACE` | *default* | the Kubernetes namespace where the app will be pushed. |
| `$INGRESS_SUBDOMAIN` | *secure-file-stora-123456.us-south.containers.appdomain.cloud* | Retrieve from the cluster overview page or with `ibmcloud ks cluster-get secure-file-storage-cluster`. |
| `$INGRESS_SECRET` | *secure-file-stora-123456* | Retrieve from the cluster overview page or with `ibmcloud ks cluster-get secure-file-storage-cluster`. |

`$IMAGE_PULL_SECRET` is only needed if you want to use another Kubernetes namespace than the default one. This requires additional Kubernetes configuration (e.g. [creating a Docker registry secret in the new namespace](https://{DomainName}/docs/containers?topic=containers-images#other)).
{: tip}

### Deploy to the cluster

1. Retrieve the cluster configuration and set the KUBECONFIG environment variable.
   ```sh
   $(ibmcloud ks cluster-config --export secure-file-storage-cluster)
   ```
   {: codeblock}
2. Create the secret used by the application to obtain service credentials:
   ```sh
   kubectl create secret generic secure-file-storage-credentials --from-env-file=credentials.env
   ```
   {: codeblock}
3. Bind the {{site.data.keyword.appid_short_notm}} service instance to the cluster.
   ```sh
   ibmcloud ks cluster-service-bind --cluster secure-file-storage-cluster --namespace default --service secure-file-storage-appid
   ```
   {: codeblock}
   If you have several services with the same name the command will fail. You should pass the service GUID instead of its name. To find the GUID of a service, use `ibmcloud resource service-instance secure-file-storage-appid`.
   {: tip}
4. Deploy the app.
   ```sh
   kubectl apply -f secure-file-storage.yaml
   ```
   {: codeblock}

## Test the application

The application can be accessed at `https://secure-file-storage.<cluster-name>.<location>.containers.appdomain.cloud/`.

1. Go to the application's home page. You will be redirected to the {{site.data.keyword.appid_short_notm}} default login page.
2. Sign up for a new account with a valid email address.
3. Wait for the email in your inbox to verify the account.
4. Login.
5. Choose a file to upload. Click **Upload**.
6. Use the **Share** action on a file to generate a pre-signed URL that can be shared with others to access the file. The link is set to expire after 5 minutes.

Authenticated users have their own spaces to store files. While they can not see each other files, they can generate pre-signed URLs to grant temporary access to a specific file.

You can find more details about the application in the [source code repository](https://github.com/IBM-Cloud/secure-file-storage).

## Review Security Events

Now that the application and its services have been successfully deployed, you can review the security events generated by that process. All the events are centrally available in {{site.data.keyword.at_short}} instance.

1. From the [**Observability**](https://{DomainName}/observe/activitytracker) dashboard, locate the {{site.data.keyword.at_short}} instance **secure-file-storage-activity-tracker** and click **View LogDNA**.
4. Review all logs sent to the service as you were provisioning and interacting with resources.

## Optional: Use a custom domain and encrypt network traffic
By default, the application is accessible on a generic hostname at a subdomain of `containers.appdomain.cloud`. However, it is also possible to use a custom domain with the deployed app. For continued support of **https**, access with encrypted network traffic, either a certificate for the desired hostname or a wildcard certificate needs to be provided. In the following section, you will either upload an existing certificate or order a new certificate in the {{site.data.keyword.cloudcerts_short}} and deploy it to the cluster. You will also update the app configuration to use the custom domain.

For secured connection, you can either obtain a certificate from [Let's Encrypt](https://letsencrypt.org/) as described in the following [{{site.data.keyword.cloud}} blog](https://www.ibm.com/cloud/blog/secure-apps-on-ibm-cloud-with-wildcard-certificates) or through [{{site.data.keyword.cloudcerts_long}}](https://{DomainName}/docs/services/certificate-manager?topic=certificate-manager-ordering-certificates).
{: tip}

1. Create an instance of [{{site.data.keyword.cloudcerts_short}}](https://{DomainName}/catalog/services/certificate-manager)
   * Set the name to **secure-file-storage-certmgr**.
   * Use the same **location** and **resource group** as for the other services.
2. Click on **Import Certificate** to import your existing certificate.
   * Set name to **SecFileStorage** and description to **Certificate for e2e security tutorial**.
   * Upload the certificate file using the **Browse** button.
   * Click **Import** to complete the import process.
3. Locate the entry for the imported certificate and expand it
   * Verify the certificate entry, e.g., that the domain name matches your custom domain. If you uploaded a wildcard certificate, an asterisk is included in the domain name.
   * Click the **copy** symbol next to the certificate's **crn**.
4. Switch to the command line to deploy the certificate information as a secret to the cluster. Execute the following command after copying in the crn from the previous step.
   ```sh
   ibmcloud ks alb-cert-deploy --secret-name secure-file-storage-certificate --cluster secure-file-storage-cluster --cert-crn <the copied crn from previous step>
   ```
   {: codeblock}
   Verify that the cluster knows about the certificate by executing the following command.
   ```sh
   ibmcloud ks alb-certs --cluster secure-file-storage-cluster
   ```
   {: codeblock}
5. Edit the file `secure-file-storage.yaml`.
   * Find the section for **Ingress**.
   * Uncomment and edit the lines covering custom domains and fill in your domain and host name.
   The CNAME entry for your custom domain needs to point to the cluster. Check this [guide on mapping custom domains](https://{DomainName}/docs/containers?topic=containers-ingress#private_3) in the documentation for details.
   {: tip}
6. Apply the configuration changes to the deployed:
   ```sh
   kubectl apply -f secure-file-storage.yaml
   ```
   {: codeblock}
7. Switch back to the browser. In the [{{site.data.keyword.Bluemix_notm}} Resource List](https://{DomainName}/resources) locate the previously created and configured {{site.data.keyword.appid_short}} service and launch its management dashboard.
   * Go to **Manage** under the **Identity Providers**, then to **Settings**.
   * In the **Add web redirect URLs** form add `https://secure-file-storage.<your custom domain>/appid_callback` as another URL.
8. Everything should be in place now. Test the app by accessing it at your configured custom domain `https://secure-file-storage.<your custom domain>`.

## Security: Rotate service credentials
To maintain security, service credentials, passwords and other keys should be replaced (rotated) a regular basis. Many security policies have a requirement to change passwords and credentials every 90 days or with similar frequency. Moreover, in the case an employee leaves the team or in (suspected) security incidents, access privileges should be changed immediately.

In this tutorial, services are utilized for different purposes, from storing files and metadata over securing application access to managing Docker images. Rotating the service credentials typically involves
- renaming the existing service keys,
- creating a new set of credentials with the previously used name,
- replacing the access data in existing Kubernetes secrets and applying the changes,
- and, after verification, deactivating the old credentials by deleting the old service keys.

The [GitHub repository](https://github.com/IBM-Cloud/secure-file-storage) for this tutorial includes scripts to automate the steps, either by invoking them on the command line or as part of a continuous delivery pipeline.

## Expand the tutorial

Security is never done. Try the below suggestions to enhance the security of your application.

* Use [{{site.data.keyword.DRA_short}}](https://{DomainName}/catalog/services/devops-insights) to perform static and dynamic code scans
* Ensure only quality code is released by using policies and rules with [{{site.data.keyword.DRA_short}}](https://{DomainName}/catalog/services/devops-insights)
* Replace {{site.data.keyword.keymanagementservicelong_notm}} by {{site.data.keyword.hscrypto}} for even greater security and control over encryption keys.

## Remove resources
{:removeresources}

To remove the resource, delete the deployed container and then the provisioned services.

1. Delete the deployed container:
   ```sh
   kubectl delete -f secure-file-storage.yaml
   ```
   {: codeblock}
2. Delete the secrets for the deployment:
   ```sh
   kubectl delete secret secure-file-storage-credentials
   ```
   {: codeblock}
3. Remove the Docker image from the container registry:
   ```sh
   ibmcloud cr image-rm <location>.icr.io/secure-file-storage-namespace/secure-file-storage:latest
   ```
   {: codeblock}
4. In the [{{site.data.keyword.Bluemix_notm}} Resource List](https://{DomainName}/resources) locate the resources that were created for this tutorial. Use the search box and **secure-file-storage** as pattern. Delete each of the services by clicking on the context menu next to each service and choosing **Delete Service**. Note that the {{site.data.keyword.keymanagementserviceshort}} service can only be removed after the key has been deleted. Click on the service instance to get to the related dashboard and to delete the key.

If you share an account with other users, always make sure to delete only your own resources.
{: tip}

## Related content
{:related}

* [{{site.data.keyword.security-advisor_short}} documentation](https://{DomainName}/docs/services/security-advisor?topic=security-advisor-about#about)
* [Security to safeguard and monitor your cloud apps](https://www.ibm.com/cloud/garage/architectures/securityArchitecture)
* [{{site.data.keyword.Bluemix_notm}} Platform security](https://{DomainName}/docs/overview?topic=overview-security#security)
* [Security in the IBM Cloud](https://www.ibm.com/cloud/security)
* Tutorial: [Best practices for organizing users, teams, applications](https://{DomainName}/docs/tutorials?topic=solution-tutorials-users-teams-applications#users-teams-applications)
* Blog: [Secure Apps on IBM Cloud with Wildcard Certificates](https://www.ibm.com/cloud/blog/secure-apps-on-ibm-cloud-with-wildcard-certificates)
* Blog: [Cloud Offboarding: How to Remove a User and Maintain Security](https://www.ibm.com/cloud/blog/cloud-offboarding-how-to-remove-a-user-and-maintain-security)
* Blog: [Going Passwordless on IBM Cloud Thanks to FIDO2](https://www.ibm.com/cloud/blog/going-passwordless-on-ibm-cloud-thanks-to-fido2)