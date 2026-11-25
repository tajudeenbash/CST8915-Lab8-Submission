# CST8915 Lab 8 — Algonquin Pet Store

GitHub: **tajudeenbash**  
Docker Hub: **tajubashiru**

## Demo video
https://youtu.be/8ISSmyCBu1I
**TODO: Add the YouTube demo link before submitting.** Maximum duration: 5 minutes.

## Task 1: Build, push, and deploy custom images

All eight application images were built for `linux/amd64`, pushed to Docker Hub, and deployed to AKS. MongoDB and RabbitMQ use the course-provided upstream images.

| Service | Forked source | Docker image |
| --- | --- | --- |
| store-front | [store-front-L8](https://github.com/tajudeenbash/store-front-L8) | `tajubashiru/store-front:lab8` |
| store-admin | [store-admin-L8](https://github.com/tajudeenbash/store-admin-L8) | `tajubashiru/store-admin:lab8` |
| order-service | [order-service-L8](https://github.com/tajudeenbash/order-service-L8) | `tajubashiru/order-service:lab8` |
| product-service | [product-service-L8](https://github.com/tajudeenbash/product-service-L8) | `tajubashiru/product-service:lab8` |
| makeline-service | [makeline-service-L8](https://github.com/tajudeenbash/makeline-service-L8) | `tajubashiru/makeline-service:lab8` |
| ai-service | [ai-service-L8](https://github.com/tajudeenbash/ai-service-L8) | `tajubashiru/ai-service:lab8` |
| virtual-customer | [virtual-customer-L8](https://github.com/tajudeenbash/virtual-customer-L8) | `tajubashiru/virtual-customer:lab8` |
| virtual-worker | [virtual-worker-L8](https://github.com/tajudeenbash/virtual-worker-L8) | `tajubashiru/virtual-worker:lab8` |

Example build and push, run from the corresponding service repository:

```powershell
docker build --platform linux/amd64 -t tajubashiru/order-service:lab8 .
docker push tajubashiru/order-service:lab8
```

### Deployment files

- [aps-all-in-one-Task1.yaml](aps-all-in-one-Task1.yaml): six application deployments, MongoDB, RabbitMQ, and services.
- [admin-tasks.yaml](admin-tasks.yaml): virtual customer and worker using custom images.
- [config-maps.yaml](config-maps.yaml): RabbitMQ plugin configuration.
- Populated secrets are deliberately excluded from this repository.

### Environment and deployment

The lab was deployed on the two-node AKS cluster `aks-lab8-belgium` in Belgium Central, in resource group `rg-cst8915-lab8`.

The Azure OpenAI resource `zorab-lab8-openai` is in France Central. Text generation uses deployment `lab8-text` with `gpt-4.1-mini`. The manifest contains public endpoint configuration, but API keys are obtained through Kubernetes Secret references.

From this repository directory, with Azure CLI authenticated:

```powershell
az aks get-credentials --resource-group rg-cst8915-lab8 --name aks-lab8-belgium
kubectl apply -f config-maps.yaml
kubectl apply -f "C:\Users\zorab\Downloads\lab8-secrets.yaml"
kubectl apply -f aps-all-in-one-Task1.yaml
kubectl apply -f admin-tasks.yaml
kubectl rollout status deployment --all --timeout=300s
kubectl scale deployment order-service --replicas=3
kubectl rollout status deployment/order-service --timeout=300s
kubectl get pods
kubectl get services
kubectl get deployments -o custom-columns=NAME:.metadata.name,READY:.status.readyReplicas,IMAGE:.spec.template.spec.containers[*].image
```

The secrets path is a private local file, not included here. Other users must supply their own credentials, resource endpoints, deployment names, and cluster details. Both Secrets must be in the same namespace as the application:

- `openai-gpt-api-secret`, key `OPENAI_GPT_API_KEY`
- `openai-dalle-api-secret`, key `OPENAI_DALLE_API_KEY`

The application manifest starts order-service at one replica; the explicit scaling command reproduces the three-replica exercise. Reapplying the manifest can restore one replica.

### Validation performed

After deploying the custom images:

- All eight deployments referenced `tajubashiru/*:lab8`.
- All 12 pods were Running and ready, with zero restarts, including three order-service replicas.
- Store Front loaded and a manual order could be placed.
- Store Admin displayed orders.
- Ask AI Assistant generated a product description.
- Virtual Customer logs showed HTTP `201 Created`.
- Virtual Worker logs showed completed orders.
- RabbitMQ management and MongoDB order records were inspected during the lab.
- `kubectl top pods` and `kubectl top nodes` returned resource metrics.

The virtual customer is configured for 200 orders/hour and the worker for 50/hour, so a growing queue is expected.

### Known limitation: image generation

**AI image generation is not complete.** The model catalog queries returned no GPT Image entries in the supported allowed regions checked; Belgium Central and Mexico Central returned unsupported-location errors for that model-list API.

The manifest's `lab8-image` deployment name is a placeholder and has not been deployed. Supplying the Azure resource key in the image Secret does not create or enable an image model. The instructor-provided credentials or approved access to a suitable image deployment are still needed. Text generation was tested successfully; image generation must not be presented as successful.

## Task 2: Azure managed-service alternatives

### MongoDB replacement: Azure DocumentDB with MongoDB compatibility

Azure DocumentDB, formerly Azure Cosmos DB for MongoDB (vCore), is a managed document database compatible with MongoDB clients and tools. It is a suitable candidate for storing the application's order documents while reducing the need to operate database infrastructure inside AKS.

This would remove responsibility for maintaining the MongoDB StatefulSet and its underlying database hosts. Migration would require checking supported MongoDB commands and driver behavior, moving existing data, configuring network access and authentication, and updating the application's connection settings. Compatibility should be tested rather than assumed.

Source: [Microsoft — Azure DocumentDB overview](https://learn.microsoft.com/en-us/azure/documentdb/overview).

### RabbitMQ replacement: Azure Service Bus

Azure Service Bus is a fully managed message broker with queues and publish-subscribe topics. A queue can decouple order submission from order processing, buffer demand, and distribute work among consumers. Durable messaging and dead-letter queues support handling failures, while Azure manages the broker infrastructure.

For this application, order-service would send messages and makeline-service would receive and settle them through a Service Bus SDK or compatible AMQP 1.0 client. This is not a hostname-only replacement for RabbitMQ AMQP 0.9.1 clients. Authentication, queue configuration, acknowledgements, retries, and duplicate handling need to be adapted and tested; alternatively, a RabbitMQ bridge can support a staged migration.

Sources: [Microsoft — Service Bus overview](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview), [Microsoft — RabbitMQ integration](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-integrate-with-rabbitmq).

These are researched alternatives, not services deployed as part of this submission.

## Submission checklist

- [x] Build and push eight application images.
- [x] Deploy custom images and verify application behavior.
- [x] Include updated application and simulator manifests.
- [x] Document managed-service research.
- [ ] Resolve image-generation access or obtain instructor guidance on the documented limitation.
- [ ] Record a demo of no more than five minutes and add its YouTube link above.
- [ ] Submit this public repository link in Brightspace.
- [ ] Clean up lab Azure resources after recording and when no longer required for assessment.
