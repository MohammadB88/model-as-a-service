# model-as-a-service
This repository provides a step by step guide to building a 'Model as a Service' based on Red Hat products, *3scale* and *Single Sign-On*. It is an extension of the work presented in [model-aas](https://github.com/rh-aiservices-bu/models-aas) with more detailed instructions.

At the end, offer your users a portal through which they can register and get access keys to the models' endpoints.

Although not a reference architecture (there are many ways to implement this type of solution), this can serve as a starting point to create such a service in your environment.

Further implementation could feature quotas, rate limits, different plans, billing, ...

**What This Project Is About**  
This project is like a **DIY guide** for building a secure, manageable **Machine Learning Model as a Service (MaaS)**,  essentially a web API that delivers predictions, using **Red Hat tools**.  
It walks you through deploying a model that users can access via a web portal, where they can sign up, get API keys, and interact with your model.  
This Guide is an enhanced version of an earlier project (model-aas), with more detailed steps for easier understanding - especially if you’re learning or building a prototype.

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
This could be any software -like a web or mobile app - that wants to use your model’s prediction service.  
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
Red Hat 3scale uses a *system storage* (*persistent system storage*) with an RWX volume, which can be provided by *OpenShift Data Foundation (ODF)*.

### Instructions
- **Create a project** (e.g. ***3scale***) on openshift. (*in your openshift cluster*)
- **Clone this repository:** [model-as-a-service](https://github.com/MohammadB88/model-as-a-service.git) onto the cluster using a *web terminal* or the *bastion server*.
- **Create a secret using files** in "[deployment/3scale/llm_metrics_policy](./deployment/3scale/llm_metrics_policy/)"

```sh
oc create secret generic llm-metrics \
    -n 3scale \
    --from-file=./apicast-policy.json \
    --from-file=./custom_metrics.lua \
    --from-file=./init.lua \
    --from-file=./llm.lua \
    --from-file=./portal_client.lua \
    --from-file=./response.lua \
    && oc label secret llm-metrics apimanager.apps.3scale.net/watched-by=apimanager -n 3scale
```

Hinweis: These files are copied from [APIcast LLM Metrics Policy](https://github.com/rh-aiservices-bu/models-aas/tree/main/deployment/3scale/llm_metrics_policy#apicast-llm-metrics-policy)

- Deploy the Red Hat Integration-3scale operator into the ***3scale*** namespace only, as shown below:

![ocp_3scale_operator_1.png](img/ocp_3scale_operator_1.png)
![ocp_3scale_operator_2.png](img/ocp_3scale_operator_2.png)

- Go to the installed Operator page and create a Custom Policy Definition instance using [deployment/3scale/llm-metrics-policy.yaml](./deployment/3scale/llm-metrics-policy.yaml). 
    - **Attention:** The namespace should be the same as the one where the 3scale operator is installed.
    
![CPD_operator_1.png](img/CPD_operator_1.png)
![CPD_operator_2.png](img/CPD_operator_2.png)

⚠️ **Note:** You may see a *'failed'* status for the policy definition at this stage. This can be ignored - continue with the next steps.

- On the same page, go to the ***"API Manager"*** tab and create an instance using [deployment/3scale/apimanager.yaml](./deployment/3scale/apimanager.yaml). 
    - **Attention 1:** The namespace should be the same as the one where the 3scale operator is installed.
    - **Attention 2:** The ***wildcardDomain*** attribute contains(*defines*) the main domain for the cluster, where *3scale* is deployed. 
    For example, if you deploy your *3scale* instance in a cluster with *URL: https://console-openshift-console.apps.cluster-abc.abc.sandbox123.opentlc.com*, then the variable should be set as: *wildcardDomain: apps.cluster-abc.abc.sandbox123.opentlc.com*

    - **Attention 3:** After creating your *APIManager* instance (e.g., apimanager-sample), the status may show **Preflights**. 
This means scale is performing initial checks before full deployment. It's normal and may take a few minutes.

      **What to do:** Just wait. The status will change to **Available, Running**, or disappear once the checks are complete. Refresh the page occasionally to monitor progress.
 

- It takes some minutes for all (15) Deployments to complete successfully. 

- Go to the *Routes* section of your cluster and find the one that starts with *https://maas-admin ...*.

- Open the link and log in using the credentials stored in the *system-seed* secret (ADMIN_USER and ADMIN_PASSWORD). 

- After logging in, close the widget page and go to the **"Account Settings"** to set your *Organization Name* and your *Timezone*. 
For example, you might set the name to *"MaaS on RHOAI"* and time zone to *"Berlin"*. 

![account_setting_1.png](img/account_setting_1.png)
![account_setting_2.png](img/account_setting_2.png)


The *"Organization Name"* appears in the top-left corner of the main page.

![portal_orga_name.png](img/portal_orga_name.png)

#### Backends - Provided Model Enpoints 
- Now We will add a backend using the corresponding *"API-Endpoint"* provided by the model server: 

![backend_1.png](img/backend_1.png)
![backend_2.png](img/backend_2.png)

Set the *Endpoint-URL*, a display name, a system name for your model, and a description of the model.  

![backend_3.png](img/backend_3.png)

When created, you will see that the backend has been added to the list. 

![backend_4.png](img/backend_4.png)

⚠️ **Note:** *If you have more models deployed, add them here using the same process.*  
(💡 **Tip:** To add additional models, repeat this process for each one deployed.) 


#### Products - APIs for Customers
- Go to the ***"Products"*** section and click *"create a Product"*:

![products_1.png](img/products_1.png)
![products_2.png](img/products_2.png)

Give the product a display, a system name and a description of the product.

![products_3.png](img/products_3.png)

The product you created is now added to the list.

![products_4.png](img/products_4.png)

##### --->>> Products - Backends <<<---

- Click on the product's name and go to the *"integration → Backends"* from the left menu.
  Add a backend by selecting the recently created backend and attaching it to the product:

  ![products_backends.png](img/products_backends.png)

##### --->>> Products - Methods & Mapping Rules <<<---

- Under the *"integration → Methods and Metrics"*, add a method (e.g. *'List Models'*):

  ![products_methods.png](img/products_methodes.png)

- Then, go to *"integration → Mapping Rules"*, and define available Api-Endpoints (*"Patterns"*) for that model (*method*):

  ![products_mappingrules.png](img/products_mappingrules.png)
  
- Repeat these steps to add 3 more *"Methods"* and corresponding *"Mapping Rules"*:

  ![products_methods_list.png](img/products_methodes_list.png)
  
  ![products_mappingrules_list.png](img/products_mappingrules_list.png)

##### --->>> Products - Authentications <<<---

At this step we change the authentication method used for the products. 
- Go to *"integration → Settings"* 
- Change the *"Auth user key"* field content to 'Authorization'
- Set the *"Credentials location"* field to 'As HTTP Basic Authentication' 
- Scroll down and click *"Update Product*" to save changes

![products_authentication.png](img/products_authentication.png)
  
##### --->>> Products - Policies <<<---

We add two more policies to each product under the *"integration → Policies"* section.
  - Apply the policies in the following order:
    1. **CORS Request Handling:**
       - ALLOW_HEADERS: `Authorization`, `Content-type`, `Accept`. 
       - allow_origin: `*`
       - allow_credentials: checked ✔️

    ![products_policies_CORS.png](img/products_policies_CORS.png)

    2.*(Optional)* **LLM Monitor** - for OpenAI-Compatible token usage. See [Readme](./deployment/3scale/llm_metrics_policy/README.md) for more information and configuration.

    ![products_policies_LLMMonitor.png](img/products_policies_LLMMonitor.png)

    3. **3scale APIcast**

  💡**Important:** After completing the configuration, **DO NOT FORGET to click "Update Policy Chain".**  
    If you skip this step, all your changes will be lost.

  ![products_policies_list.png](img/products_policies_list.png)

##### --->>> Products - Configuration & Staging <<<---

- From the *"Integration → Configuration"* menu, promote the configuration to *"Staging"*, then to *"Production"*.
  ![products_config_staging_1.png](img/products_config_staging_1.png)
  ![products_config_staging_2.png](img/products_config_staging_2.png)


##### --->>> Products - Applications & Application Plans <<<---

- For each Product, go to *"Applications → Application Plans"*, and create a new Application Plan. At this stage, we are not forcing (*requiring*) approval for the applications under this plan.

  ![products_application_plans.png](img/products_application_plans.png)

- On the page, where application plans are listed, leave the *Default plan* set to *"No plan selected"* so users can choose their preferrend services for their (*when creating*) applications. **Note that the plan is hidden by difoult and hence we should publish it**:

  ![products_application_plans_1.png](img/products_application_plans_1.png)

- In *"Applications → Settings → Usage Rules"*, set the *Default plan* to 'Default'. This will allow the users to see (*This ensures users will see*) the different available Products when browsing the portal:

  ![products_application_plans_2.png](img/products_application_plans_2.png)

**Attention:** When creating a new application from the (*developer*) portal, the only available plan is a default *"Basic"* plan:

![portal_applicaion_basic.png](img/portal_applicaion_basic.png)

Therefore, we first create an application directly from the ***3scale admin portal***, by going to the *"Applications → Listing"*:

![products_application_creation.png](img/products_application_creation.png)

This way, we can now select the desired product (in our case e.g., *"granite-7b-instruct"*), when creating applications from the portal:

![portal_applicaion_select_product.png](img/portal_applicaion_select_product.png)
![portal_applicaion_standardplan.png](img/portal_applicaion_standardplan.png)

##### --->>> Products - Active Docs <<<---

**Attention:** Without *"Active Docs"*, the *"Enpoint URL"* will not be displayed on the application page in the (*developer*) portal:

![portal_applicaion_endpointURL.png](img/portal_applicaion_endpointURL.png)

To fix this, we add a *"spec"* under *"Active Docs"* for the product. In order to do that,
  - Copy the content of the file: [deployment/3scale/active_docs.json](./deployment/3scale/active_docs.json)
  - Paste it into the corresponding section of the *Active Docs > spec* form.
  - Set a proper (*meaning*) *"name"* and *"system name"*.
  - Check the box to *"publish the docs"* and click on *"Create spec"*.

![products_active_docs.png](img/products_active_docs.png)


#### Audience - Portal Content 
Under the **Audience** section, go to the *"Developer Portal → Content"* and begin editing the corresponding pages and files:

![audience_1.png](img/audience_1.png)
![audience_portal_content_1.png](img/audience_portal_content_1.png)

**Attention:** These portal configurations are based on these files [models-aas repo - Portal Configurations](https://github.com/rh-aiservices-bu/models-aas/tree/main/deployment/3scale/portal). **Hoever** some files in the Repository have been updated, as the original versions on the repo did not create the pages correctly and a few are non-functional.

- In this directory, go to this path (*Navigate to the following directory in your cloned repo*): [deployment/3scale/portal](deployment/3scale/portal).
- From the *"deployment/3scale/portal"* folder, apply all necessary modifications to the relevant pages. Then make sure to *"Save and Publish"*, both the *Draft* and *Published* versions.
- **Structure:** The content of this folder are arranged following the same organization of the site. (*The content of this folder are organized to match the layout of the Developer Portal site.*)

⚠️ **Note:** New Pages may have to be created with the type depending of the type of content (html, javascript, css), some others have only to be modified.
(*Some pages may need to be created. Choose the type based on the content (HTML, JavaScript, CSS). Others may already exist and only need to be updated.*)

⚠️ **Note:** DO NOT FORGET to save and publish both the *Draft* and *Published*.

- Files that should be ***edited*** are as follows:
  - **Layouts**
    - **Main Layout** 
  - **Root**
    - **Homepage**
    - **Docmentation**
    - **Applications → Show**
    - **Applications → Choose Service**
    - **Applications → Index**
    - **css → default.css**
    - **Login → New**
  - **Partials**
    - **submenu**
  
- Files that should be ***created*** in the Developer Portal content section are as follows:
(*In this step, we will add new pages and files to the Developer Portal, based on the specifications listed below.*)
  - **Examples:** Create a 'New Page' named *'Examples'*.

  ![audience_portal_content_examples.png](img/audience_portal_content_examples.png)

- **css → rhoai_custom.css:** Create a 'New Page' named *'rhoai_custom.css'* under the **css** section.

  ![audience_portal_content_rhoai_custom.png](img/audience_portal_content_rhoai_custom.png)

- **javascripts → secret_keys.js:** Create a 'New Page' named *'secret_keys.js'* under the **javascripts** section.


  ![audience_portal_content_secret_keys.png](img/audience_portal_content_secret_keys.png)

- **images → rhoai.png:** Create a 'New File' named *'rhoai.png'* under the **images** section.
 
  ![audience_portal_content_images.png](img/audience_portal_content_images.png)
