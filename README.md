# API Gateway Private Integration with VPC Link to Internal NLB using NGINX Ingress Controller on AWS EKS

## Overview
This guide explains how to configure **AWS API Gateway** with a **VPC Link** to an **Internal Network Load Balancer (NLB)** using **NGINX Ingress Controller** on **AWS EKS**. The goal is to keep all backend services private while exposing them securely via API Gateway.

## Prerequisites
- AWS CLI installed and configured
- Helm installed
- An existing **Amazon EKS Cluster**
- AWS API Gateway (REST API - Regional)
- Backend microservices deployed in EKS

## Steps to Configure

### 1. Create API Gateway (Regional) and Resources
- Navigate to **API Gateway** in AWS Console.
- Create a **REST API (Regional)**.
- Define resources and methods but **do not configure integration yet**.

### 2. Deploy NGINX Ingress Controller with Internal NLB
Run the following Helm command to install the **NGINX Ingress Controller**:

```sh
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-internal"="true"
```

After running this command successfully, verify in AWS Console that the **Internal NLB** has been created.

### 3. Deploy Backend Services in EKS
- Ensure backend services are deployed with **Service Type: ClusterIP**.

### 4. Configure Ingress Resources for Backend Microservices
Create an **Ingress** resource for each backend microservice using the following template:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-name
  namespace: namespace-name
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /nameoftheapi(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: service-name
                port:
                  number: 80
```
- Replace `nameoftheapi`, `namespace-name`, and `service-name` with actual values.
- Repeat this step for each backend microservice.

### 5. Test Internal NLB Routing
To confirm that requests reach the backend services, run the following from a pod inside the same namespace:

```sh
curl http://NLB-DNS/nameoftheapi
```

Expected response: `Welcome to express/node/nameoftheservice`.

Repeat for all backend microservices by modifying the path accordingly:

```yaml
- path: /nameoftheAPI1(/|$)(.*)
- path: /nameoftheAPI2(/|$)(.*)
- path: /nameoftheAPI3(/|$)(.*)
```

### 6. Create VPC Link for Internal NLB
- Navigate to **VPC Links** in API Gateway.
- Create a **VPC Link**, selecting the **Internal NLB** created in Step 2.

### 7. Configure API Gateway Integration Request
- In API Gateway, go to your **resource method**.
- Click on **Integration Request**.
- Select **VPC Link** and choose the **VPC Link** created in Step 6.
- Set **Method Type** (GET, POST, etc.).
- Use the following URL format:
  ```sh
  http://NLB-DNS/route-of-your-api/path-of-resource
  ```
- Save and test the integration.

### 8. Enable CORS and Deploy API Gateway
- Enable **CORS** for each resource.
- Deploy the **API Gateway**.

### 9. Update Frontend Environment and Redeploy
- Wait **2-3 minutes** for changes to propagate.
- Add the **API Gateway Invoke URL** to the **frontend service ENV file**.
- Redeploy the frontend application to apply changes.

## Conclusion
By following this guide, you have successfully set up **API Gateway** with a **VPC Link** to an **Internal NLB**, keeping all backend services private. Using **NGINX Ingress Controller**, multiple backend microservices can be efficiently routed within the **EKS cluster**. If only a single backend service is needed, a **LoadBalancer** type Kubernetes service can be used instead of the **Ingress Controller**.



