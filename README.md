# Cloud-Native App for Best Buy

## Demo Video

https://youtu.be/j8H9qqsTv5w

## **Application Overview**

The **Best Buy App** includes the following components:

| Service              | Description                                                  | Notes                                    |
| -------------------- | ------------------------------------------------------------ | ---------------------------------------- |
| **Store-Front**      | Customer-facing app for browsing and placing orders.         |                                          |
| **Store-Admin**      | Employee-facing app for managing products and viewing orders . |                                          |
| **Order-Service**    | Handles order creation and sends data to the managed order queue. | Replace RabbitMQ with a managed service. |
| **Product-Service**  | Handles CRUD operations for product data.                    |                                          |
| **Makeline-Service** | Processes and completes orders by reading from the order queue. |                                          |
| **AI-Service**       | Generates product descriptions and images.                   | Use GPT-4 and DALL-E models.             |
| **Database**         | MongoDB for persisting order and product data.               |                                          |



## Application Architecture

![](./assets/diagram.png)

## Application and Architecture Explanation

Customers visit the store-front to view products and make purchases, store front fetch product information from product-service and present to customers. After customer confirm their order, store-front will send the order information to order-service, and order-service will encrypt it and pass it to Azure Service Bus. Then makeline-service will listen to the messages in Azure Service Bus and put the latest message into mongodb. Store-admin service grabs order data via makeline-service from mongodb, and it also leverages ai-service to generate new product's description (gpt-4) and product images (dall-e-3).

## Deployment Instructions

### Step 1: Clone the BestBuy Demo Repository

To begin, clone the [CST8915-final](https://github.com/DytAC-study/CST8915-final.git) repository, which contains all necessary deployment files.

**Review the Deployment Files**:

- Navigate to the `Deployment Files` folder
- This folder contains YAML files for deploying all necessary Kubernetes resources, including services, deployments, StatefulSets, ConfigMaps, and Secrets.
- `bb-all-in-one-servicebus.yaml` is for the application version with Azure Service Bus
- `bb-all-in-one-rabbitmq.yaml` is for the application version with Rabbitmq
- `secrets-openai.yaml` stores the Azure Openai Service key
- `config-maps.yaml` stores the configuration for Rabbitmq plugins.

## Step 2: Create an Azure Kubernetes Cluster (AKS)

1. **Log in to Azure Portal:**

   - Go to [https://portal.azure.com](https://portal.azure.com/) and log in with your Azure account.

2. **Create a Resource Group:**

   - In the Azure Portal, search for **Resource Groups** in the search bar.
   - Create a resource group for this deployment.

3. **Create an AKS Cluster:**

   - In the search bar, type **Kubernetes services** and click on it.
   - Click **Create** and select **Kubernetes cluster**
   - In the `Basics` tap fill in the following details:
     - **Subscription**: Select your subscription.
     - **Resource group**: Choose the one just created.
     - **Cluster preset configuration**: Choose `Dev/Test`.
     - **Region**: Same as your resource group (e.g., `Canada`).
     - **Availability zones**: `None`.
     - **AKS pricing tier**: `Free`.
     - **Kubernetes version**: `Default`.
     - **Automatic upgrade**: `Disabled`.
     - **Automatic upgrade scheduler**: `No schedule`.
     - **Node security channel type**: `None`.
     - **Security channel scheduler**: `No schedule`.
     - **Authentication and Authorization**: `Local accounts with Kubernetes RBAC`.
   - In the Node pools, create a masterpool and some worker pools.
     - Set **node size** to `D2as_v4`.

4. **Connect to the AKS Cluster:**

   - Once the AKS cluster is deployed, navigate to the cluster in the Azure Portal.

   - In the overview page, click on **Connect**.

   - Select **Azure CLI** tap. You will need Azure CLI. If you don't have it: [**Install Azure CLI**](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

   - Login to your azure account using the following command:

     ```
     az login
     ```

   - Set the cluster subscription using the command shown in the portal (it will look something like this):

     ```
     az account set --subscription 'subscribtion-id'
     ```

   - Copy the command shown in the portal for configuring `kubectl` (it will look something like this):

     ```
     az aks get-credentials --resource-group <your resource group> --name <your cluster name>
     ```

   - Verify Cluster Access:

     - Test your connection to the AKS cluster by listing all nodes:

       ```
       kubectl get nodes
       ```

       You should see details of the nodes in your AKS cluster if the connection is successful.

## Step 3: Set Up the AI Backing Services

To enable AI-generated product descriptions and image generation features, you will deploy the required **Azure OpenAI Services** for GPT-4 (text generation) and DALL-E 3 (image generation). This step is essential to configure the **AI Service** component in the Bestbuy Demo application.

### 1: Create an Azure OpenAI Service Instance

1. **Navigate to Azure Portal**:
   - Go to the [Azure Portal](https://portal.azure.com/).
2. **Create a Resource**:
   - Select **Create a Resource** from the Azure portal dashboard.
   - Search for **Azure OpenAI** in the marketplace.
3. **Set Up the Azure OpenAI Resource**:
   - Choose the **East US** region for deployment to ensure capacity for GPT-4 and DALL-E 3 models.
   - Fill in the required details:
     - Resource group: Use an existing one or create a new group.
     - Pricing tier: Select `Standard`.
4. **Deploy the Resource**:
   - Click **Review + Create** and then **Create** to deploy the Azure OpenAI service.

------

### 2: Deploy the GPT-4 and DALL-E 3 Models

1. **Access the Azure OpenAI Resource**:
   - Navigate to the Azure OpenAI resource you just created.
2. **Deploy GPT-4**:
   - Go to the **Model Deployments** section and click **Add Deployment**.
   - Choose **GPT-4** as the model and provide a deployment name (e.g., `gpt-4-deployment`).
   - Set the deployment configuration as required and deploy the model.
3. **Deploy DALL-E 3**:
   - Repeat the same process to deploy **DALL-E 3**.
   - Use a descriptive deployment name (e.g., `dalle-3-deployment`).
4. **Note Configuration Details**:
   - Once deployed, note down the following details for each model:
     - Deployment Name
     - Endpoint URL

------

### 3: Retrieve and Configure API Keys

1. **Get API Keys**:

   - Go to the **Keys and Endpoints** section of your Azure OpenAI resource.
   - Copy the **API Key (API key 1)** and **Endpoint URL**.

2. **Base64 Encode the API Key**:

   - Use the following command to Base64 encode your API key:

     ```
     echo -n "<your-api-key>" | base64
     ```

   - Replace `<your-api-key>` with your actual API key.

------

### Task 4: Update AI Service Deployment Configuration in the `Deployment Files` folder.

1. **Modify Secretes YAML**:

   - Edit the `secrets-openai.yaml` file.
   - Replace `OPENAI_API_KEY` placeholder with the Base64-encoded value of the `API_KEY`.

2. **Modify Deployment YAML**:

   - Edit the `bb-all-in-one-servicebus.yaml` file.
   - Replace the placeholders with the configurations you retrieved:
     - `AZURE_OPENAI_DEPLOYMENT_NAME`: Enter the deployment name for GPT-4.
     - `AZURE_OPENAI_ENDPOINT`: Enter the endpoint URL for the GPT-4 deployment.
     - `AZURE_OPENAI_DALLE_ENDPOINT`: Enter the endpoint URL for the DALL-E 3 deployment.
     - `AZURE_OPENAI_DALLE_DEPLOYMENT_NAME`: Enter the deployment name for DALL-E 3.

   Example configuration in the YAML file:

   ```
   - name: AZURE_OPENAI_API_VERSION
     value: "2024-07-01-preview"
   - name: AZURE_OPENAI_DEPLOYMENT_NAME
     value: "gpt-4-deployment"
   - name: AZURE_OPENAI_ENDPOINT
     value: "https://<your-openai-resource-name>.openai.azure.com/"
   - name: AZURE_OPENAI_DALLE_ENDPOINT
     value: "https://<your-openai-resource-name>.openai.azure.com/"
   - name: AZURE_OPENAI_DALLE_DEPLOYMENT_NAME
     value: "dalle-3-deployment"
   ```

## Step 4: Deploy the Secrets

- Create and Deploy the Secret for OpenAI API:

  - Make sure that you have replaced Base64-encoded-API-KEY in secrets.yaml with your Base64-encoded OpenAI API key.

  ```
  kubectl apply -f secrets-openai.yaml
  ```

- Verify:

  ```
  kubectl get configmaps
  kubectl get secrets
  ```

## Step 4: Deploy the Bestbuy Demo Application

1. **Apply the YAML file to the AKS cluster:**

   - In this step, use the K8s deployment YAML file provided: `bb-all-in-one-servicebus.yaml`.

   - Open the terminal and navigate to the file directory.

   - Run the following command to apply the YAML configuration and deploy the application to AKS:

     ```
     kubectl apply -f bb-all-in-one-servicebus.yaml
     ```

2. **Verify the deployment:**

   - After the command executes, verify that the pods are running by using the following command:

     ```
     kubectl get pods
     ```

3. **Check services:**

   - Confirm that all services are up and running:

     ```
     kubectl get services
     ```

   - using `EXTERNAL-IP`:80  of store-front and store-admin services to visit those pages.

## Docker Images

| **Service**        | **Docker Image Link**                                        |
| ------------------ | ------------------------------------------------------------ |
| `store-front`      | [store-front-bb-v1](https://hub.docker.com/repository/docker/dytac/store-front-bb-v1/general) |
| `store-admin`      | [store-admin-bb-v1](https://hub.docker.com/repository/docker/dytac/store-admin-bb-v1/general) |
| `order-service`    | [order-service-bb-v1](https://hub.docker.com/repository/docker/dytac/order-service-bb-v1/general) |
| `product-service`  | [product-service-bb-v1](https://hub.docker.com/repository/docker/dytac/product-service-bb-v1/general) |
| `makeline-service` | [makeline-service-bb-v1](https://hub.docker.com/repository/docker/dytac/makeline-service-bb-v1/general) |
| `ai-service`       | [ai-service-bb-v1](https://hub.docker.com/repository/docker/dytac/ai-service-bb-v1/general) |
| `virtual-customer` | [virtual-customer-bb-v1](https://hub.docker.com/repository/docker/dytac/virtual-customer-bb-v1/general) |
| `virtual-worker`   | [virtual-worker-bb-v1](https://hub.docker.com/repository/docker/dytac/virtual-worker-bb-v1/general) |

## Microservice Git Repositories

| Service            | Description                              | Github Repo                                                  |
| ------------------ | ---------------------------------------- | ------------------------------------------------------------ |
| `store-front`      | Web app for customers to place orders    | [store-front-bb-v1](https://github.com/DytAC-study/store-front-bb-v1.git) |
| `store-admin`      | Web app for store employees              | [store-admin-bb-v1](https://github.com/DytAC-study/store-admin-bb-v1.git) |
| `order-service`    | Handles order placement                  | [order-service-bb-v1](https://github.com/DytAC-study/order-service-bb-v1.git) |
| `product-service`  | Handles CRUD operations on products      | [product-service-bb-v1](https://github.com/DytAC-study/product-service-bb-v1.git) |
| `makeline-service` | Processes and completes orders           | [makeline-service-bb-v1](https://github.com/DytAC-study/makeline-service-bb-v1.git) |
| `ai-service`       | AI-based product descriptions and images | [ai-service-bb-v1](https://github.com/DytAC-study/ai-service-bb-v1.git) |
| `virtual-customer` | Simulates customer order creation        | [virtual-customer-bb-v1](https://github.com/DytAC-study/virtual-customer-bb-v1.git) |
| `virtual-worker`   | Simulates order completion               | [virtual-worker-bb-v1](https://github.com/DytAC-study/virtual-worker-bb-v1.git) |

## Problem faced: Ai services not function correctly

## 1. Dall-e-3 reached quota limit

I couldn't create the dall-e-3 service as it reached quota limit. I explained it in the video.

## 2. gpt-4 service not responding

I configured everything in the yaml file, the ai service is running normally in k8s, but when I try to generate description of my product, it just send the request to gpt-4 service with no response.

I check the log of the ai-service pod, and it shows it was calling openai with no error, but also no response. I also mentioned this in my video.

```
Generative AI capabilities:  description, image
INFO:     10.224.0.5:49810 - "GET /health HTTP/1.1" 200 OK
Generative AI capabilities:  description, image
INFO:     10.224.0.5:46014 - "GET /health HTTP/1.1" 200 OK
Generative AI capabilities:  description, image
INFO:     10.224.0.5:46016 - "GET /health HTTP/1.1" 200 OK
Calling OpenAI
Generative AI capabilities:  description, image
INFO:     10.224.0.5:57230 - "GET /health HTTP/1.1" 200 OK
Generative AI capabilities:  description, image
INFO:     10.224.0.5:57232 - "GET /health HTTP/1.1" 200 OK
Generative AI capabilities:  description, image
INFO:     10.224.0.5:38886 - "GET /health HTTP/1.1" 200 OK
Generative AI capabilities:  description, image
INFO:     10.224.0.5:38884 - "GET /health HTTP/1.1" 200 OK
```

