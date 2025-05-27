# model-as-a-service
This repository provides a step by step guide to building a 'Model as a Service' based on Red Hat products, *3scale* and *Single Sign-On*. It is an extension of the work presented in [model-aas](https://github.com/rh-aiservices-bu/models-aas) with more detailed instructions.

At the end, offer your users a portal through which they can register and get access keys to the models' endpoints.

Although not a reference architecture (there are many ways to implement this type of solution), this can serve as a starting point to create such a service in your environment.

Further implementation could feature quotas, rate limits, different plans, billing,...

**What This Project Is About**  
This project is like a **DIY guide** for building a secure, manageable **Machine Learning Model as a Service (MaaS)**,  essentially a web API that delivers predictions, using **Red Hat tools**.  
It walks you through deploying a model that users can access via a web portal, where they can sign up, get API keys, and interact with your model.  
This Guide is an enhanced version of an earlier project (model-aas), with more detailed steps for easier understanding — especially if you’re learning or building a prototype.

**Technologies Used**:  
  * **OpenShift** – to deploy your model
  * **3scale API Management** – to control, track, and limit access
  * **Single Sign-On (SSO)** – to manage secure user logins
  

**What You’ll Build**  
By following this guide, you’ll have:
  * A **Developer Portal** (a web interface for users)
  * A system where users can:
    - Sign up and log in securely
    - Receive their own **API keys**
    - Use your model safely via those keys 
 
**Optional Advanced Features You Can Add Later**  
  * **Quotas** – limit how much each user can access
  * **Rate limits** – control how frequently requests are made
  * **Access plans** – like free vs. paid tiers
  * **Billing systems** – charge based on usage  
Similar to how commercial APIs like OpenAI or Google Cloud operate.
  
**How It Works: The Flow**
  1. A user visits the Developer Portal
  2. They log in via SSO (Single Sign-On)
  3. They receive an API key
  4. They send requests to your model using the key
  5. 3scale checks if they’re allowed and monitors usage
  6. If permitted, the request reaches the model
  7. The model returns a prediction/result
 
 **Core Components**  
 This project is made up of *four* main components that work together to deliver your Machine Learning Model as a Service.
1. **Developer Portal**  
  - Web interface for users to:
    - Sign up / Log in
    - Discover available APIs
    - Get access credentials (API keys)
  -  Think of it as a dashboard for developers.
2. **API Management with 3scale**
  - Controls who can access your model
  - Tracks usage (analytics)
  - Enforces quotas, rate limits, and subscription plans
  - Acts as the gatekeeper for your API
3. **Single Sign-On (SSO)**
  - Provides secure login via one authentication system
  - Enables seamless access across services
  - Tools like Keycloak or Red Hat SSO are commonly used
4. **Machine Learning Model as a Service**
  - Your model is deployed in the cloud (e.g., on OpenShift)
  - It receives input via API calls and returns predictions
  - Examples: Predict house prices, detect fraud, classify images, etc.


## Architecture Overview

![architecture](img/architecture.drawio.svg)

**How the Components Work Together**  
The diagram above illustrates how all the parts of the MaaS architecture interact, from the user sending a request to the model delivering a response, using *3scale* and *OpenShift AI*.

 **What’s Happening in the Diagram?**  
At a high level, this setup connects a client application to a machine learning model via a secure, managed API pathway. Here’s how each part contributes:
 1. **Application (Client Side)**  
This could be any software — like a web or mobile app — that wants to use your model’s prediction service.  
  - It sends a request to your API (e.g. with input data)
  - The request travels first to the 3scale API Gateway
 2. **3scale API Gateway (Traffic Controller)**  
 This component ensures only valid and authorized requests are allowed through.
  - Verifies API keys and access permissions
  - Reports traffic and usage data to the API Manager
  - Forwards approved requests to the ML model hosted on OpenShift AI

3. **3scale API Manager (Control Center)**  
 Acts as the decision-maker and bridge between other parts.
  - Manages:
    - Access control and usage policies
    - Communication with the API Gateway for real-time verification
    - Synchronization with the Developer Portal to publish API specs
    - Backend configurations via the Admin Portal

4. **3scale Developer Portal (User Interface for Developers)**  
 A self-service portal for external users who want to access your model.
  - Lets users:
    - Register for an account
    - Explore your API documentation
    - Retrieve their personal API keys

5. **3scale Admin Portal (Your Control Panel)**  
 Designed for the API owner or system admin.
  - Allows you to:
    - Define access plans (e.g., free vs premium)
    - Monitor API usage analytics
    - Manage users, API keys, and billing

6. **OpenShift AI (Model Hosting Environment)**  
 This is where your deployed machine learning model lives.
  - Receives requests forwarded by the API Gateway
  - Processes input data and returns predictions
  - Supports scaling, versioning, and lifecycle management for your model

 **Summary: Request Flow**
  1. A client application sends an API request.
  2. The request goes to the API Gateway, which checks with the API Manager.
  3. If the request is valid, it’s forwarded to the OpenShift AI model.
  4. The model generates and returns a prediction.
  5. Usage data is logged and sent to the API Manager.
  6. Developers manage their access through the Developer Portal.
  7. Admins manage the system via the Admin Portal.



## Screenshots

Portal:

![portal.png](img/portal.png)

Services:

![services.png](img/services.png)

Service detail:

![service_detail.png](img/service_detail.png)

Statistics:

![traffic.png](img/traffic.png)


## Deployment - Red Hat 3Scale
These deployments are working on an OpenShift AI instance on the Red Hat demo platform.

### Requirements
Red Hat 3scale uses a *system storage* with an RWX volume, which can be provided by *OpenShift Data Foundation*.

### Instructions
- Create a project (e.g. ***3scale***) on openshift
- Clone this repository: [model-as-a-service](https://github.com/MohammadB88/model-as-a-service.git) onto the cluster using a *web terminal* or the *bastian server*.
- Create a secret using files in "[deployment/3scale/llm_metrics_policy](./deployment/3scale/llm_metrics_policy/)"

```sh
oc create secret generic llm-metrics \
    -n 3scale \
    --from-file=./apicast-policy.json \
    --from-file=./custom_metrics.lua \
    --from-file=./init.lua \
    --from-file=./llm.lua \
    --from-file=./portal_client.lua \
    --from-file=./response.lua \
    && oc label secret llm-metrics apimanager.apps.3scale.net/watched-by=apimanager
```

Hinweis: These files are copied from [APIcast LLM Metrics Policy](https://github.com/rh-aiservices-bu/models-aas/tree/main/deployment/3scale/llm_metrics_policy#apicast-llm-metrics-policy)

- Deploy the Red Hat Integration-3scale operator in the ***3scale*** namespace only, as shown below:
![ocp_3scale_operator_1.png](img/ocp_3scale_operator_1.png)
![ocp_3scale_operator_2.png](img/ocp_3scale_operator_2.png)

- Go to the installed operator page and create a Custom Policy Definition instance using [deployment/3scale/llm-metrics-policy.yaml](./deployment/3scale/llm-metrics-policy.yaml). 
    - **Attention:** Namespace should be the same as the one where the 3scale operator is installed.
    
- In the same page, go to the tab ***"API Manager"*** and create an instance using [deployment/3scale/apimanager.yaml](./deployment/3scale/apimanager.yaml). 
    - **Attention 1:** Namespace should be the same as the one where the 3scale operator is installed.
    - **Attention 2:** The attribute ***wildcardDomain*** contains the main domain for the cluster, where *3scale* is deployed. 
    For example, if I deploy my *3scale* instance in a cluster with *URL: https://console-openshift-console.apps.cluster-abc.abc.sandbox123.opentlc.com*, then the variable should be set as *wildcardDomain: apps.cluster-abc.abc.sandbox123.opentlc.com*

- It takes some minutes till all the Deployments (15) are finished successfully. 

- Go to the *routes* on your cluster and find the one that starts with *https://maas-admin ...*.

- Open the link and log in with credentials stored in in the secret *system-seed* (ADMIN_USER and ADMIN_PASSWORD). 

- Close the widget page and go to the *"Account Settings"* to set the *"Organization Name"* and your *"Timezone"*. 
For example, I set the name to *"MaaS on RHOAI"* and time to *"Berlin"*. 

![account_setting_1.png](img/account_setting_1.png)
![account_setting_2.png](img/account_setting_2.png)


*"Organization Name"* appears at the top left corner of the main page.
![portal_orga_name.png](img/portal_orga_name.png)

#### Backends - Provided Model Enpoints 
- We now add a backend with the corresponding *"API-Endpoint"* provided by the model server:
![backend_1.png](img/backend_1.png)
![backend_2.png](img/backend_2.png)

Set the *Endpoint-URL*, a display and a system name for your model, as well as a description about the model. 
![backend_3.png](img/backend_3.png)

When created, you will see that the backend is now added to the list.
![backend_4.png](img/backend_4.png)

***If you have more model deployed, add them here using the same process.***


#### Products - APIs for Customers
- Go to the *"Products"* section and click on *"create a Product"*:
![products_1.png](img/products_1.png)
![products_2.png](img/products_2.png)

Give the product a display and a system name, as well as a description about the product.
![products_3.png](img/products_3.png)

The created product is now added to the list.
![products_4.png](img/products_4.png)

##### --->>> Products - Backends <<<---

- Click on the product's name and go to the *"integration->Backends"* from the left menu
  Add a backend by selecting the recently created backend and add it to the product:
  ![products_backends.png](img/products_backends.png)

##### --->>> Products - Methods & Mapping Rules <<<---

- Under the *"integration->Methods and Metrics"*, we add a method (e.g. *"List Models"*):
  ![products_methods.png](img/products_methodes.png)

- Then we go to *"integration/Mapping Rules"*, and define available api-endpoints (*"Patterns"*) for that model:
  ![products_mappingrules.png](img/products_mappingrules.png)
  
- We follow the same steps to add 3 more *"Methods"* and *"Mapping Rules"*:
  ![products_methods_list.png](img/products_methodes_list.png)
  
  ![products_mappingrules_list.png](img/products_mappingrules_list.png)

##### --->>> Products - Authentications <<<---

- At this step we change the authentication method for using the products. Go to *"integration->Settings"*. Change the *"Auth user key"* field content to **Authorization** and the *"Credentials location"* field to **As HTTP Basic Authentication**. When finished, click on Update product at the bottom of the page to save the changes:
  ![products_authentication.png](img/products_authentication.png)
  
##### --->>> Products - Policies <<<---

- We add two more policies to each product under the *"integration->Policies"* section:
  - Add the Policies in this order:
    1. CORS Request Handling:
       1. ALLOW_HEADERS: `Authorization`, `Content-type`, `Accept`.
       2. allow_origine: *
       3. allow_credentials: checked
       ![products_policies_CORS.png](img/products_policies_CORS.png)
    2. Optionally LLM Monitor for OpenAI-Compatible token usage. See [Readme](./deployment/3scale/llm_metrics_policy/README.md) for information and configuration.
    ![products_policies_LLMMonitor.png](img/products_policies_LLMMonitor.png)
    3. 3scale APIcast
  - After all the configurations, **DO NOT FORGET** to *"Update Policy Chain"*. Otherwise, all your changes will be lost.
  ![products_policies_list.png](img/products_policies_list.png)

##### --->>> Products - Configuration & staging <<<---

- From the *"Integration->Configuration"* menu, promote the configuration to *"staging"* then *"production"*.
  ![products_config_staging_1.png](img/products_config_staging_1.png)
  ![products_config_staging_2.png](img/products_config_staging_2.png)


##### --->>> Products - Applications & Application Plans <<<---

- For each Product, from the *"Applications->Application Plans"* menu, create a new Application Plan. At the moment and for this plan, we are not forcing the approval for the applications.
  ![products_application_plans.png](img/products_application_plans.png)

- In the page, where application plans are listed, leave the Default plan to *"No plan selected"* so that users can choose their services for their applications. **Note that the plan is hidden and hence we should publish it**:
  ![products_application_plans_1.png](img/products_application_plans_1.png)

- In *"Applications->Settings->Usage"* Rules, set the Default Plan to Default. This will allow the users to see the different available Products:
  ![products_application_plans_2.png](img/products_application_plans_2.png)

**Attention:** If I want to create a new application from the portal, the anly available plan is a default *"Basic"* plan:
![portal_applicaion_basic.png](img/portal_applicaion_basic.png)

Therefore, we create the first application directly from the ***3scale admin portal***, by going to the *"Applications->Listing"*:
![products_application_creation.png](img/products_application_creation.png)

So that, we can now select the desired product (in our case *"granite-7b-instruct"*), when creating applications from the portal:
![portal_applicaion_select_product.png](img/portal_applicaion_select_product.png)
![portal_applicaion_standardplan.png](img/portal_applicaion_standardplan.png)

##### --->>> Products - Active Docs <<<---

**Attention:** Without *"Active Docs"*, the *"Enpoint URL"* will not be shown in the application page on the portal:
![portal_applicaion_endpointURL.png](img/portal_applicaion_endpointURL.png)

Hence, we add a *"spec"* under *"Active Docs"* for the product. In order to do that, copy the content of the file [deployment/3scale/active_docs.json](./deployment/3scale/active_docs.json) in the corresponding sectio of the spec, set a proper name and system name, check the box to publish the docs and click on *"Create spec"*.
![products_active_docs.png](img/products_active_docs.png)


#### Audience - Portal Content 
- Under section **Audience**, go to the *"Developer Portal->Content"* and start editing the corresponding pages and files:
![audience_1.png](img/audience_1.png)
![audience_portal_content_1.png](img/audience_portal_content_1.png)

**Attention:** These portal configurations are based on these files [models-aas repo - Portal Configurations](https://github.com/rh-aiservices-bu/models-aas/tree/main/deployment/3scale/portal), **BUT** some of the files are updated as the ones on the repo did not create the pages as they should and some files are not working.

- In this directory, go to this path: [deployment/3scale/portal](deployment/3scale/portal).
- From the *"deployment/3scale/portal"* folder, apply all the modifications to the different pages and Publish them.
- The content of this folder is arranged following the same organization of the site.
- New Pages may have to be created with the type depending of the type of content (html, javascript, css), some others have only to be modified.
- **DO NOT FORGET** to save and publish both the *Draft* and *Published*
- Files that should be ***edited*** are as follows:
  - **Layouts**
    - **Main Layout** 
  - **Root**
    - **Homepage**
    - **Docmentation**
    - **Applications->Show**
    - **Applications->Choose Service**
    - **Applications->Index**
    - **css->default.css**
    - **Login->New**
  - **Partials**
    - **submenu**
  
- Files that should be ***created*** are as follows:
- **Examples**
- 
  ![audience_portal_content_examples.png](img/audience_portal_content_examples.png)

- **css->rhoai_custom.css**
- 
  ![audience_portal_content_rhoai_custom.png](img/audience_portal_content_rhoai_custom.png)

- **javascripts->secret_keys.js**
- 
  ![audience_portal_content_secret_keys.png](img/audience_portal_content_secret_keys.png)

- **images->rhoai.png**
- 
  ![audience_portal_content_images.png](img/audience_portal_content_images.png)
