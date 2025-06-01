# model-as-a-service
This repository provides a step by step guide to building a 'Model as a Service' based on Red Hat products, *Openshift AI*, *3scale* and *Single Sign-On*. It is an extension of the work presented in [model-aas](https://github.com/rh-aiservices-bu/models-aas) with more detailed instructions.

**Purpose of the Project**  
This project is a walk-through for users to access deployed models on for example *Openshift AI* via a web portal, where they can sign up, get API keys, and interact with the provided models as backend products. It works very similar to how commercial APIs like OpenAI or Google Cloud operate.

A user logs into the Developer Portal via SSO and creates an application, which automatically generates a model endpoint and API key. They use this key to send requests to the model, with 3scale validating access and monitoring usage. The model then returns predictions or results based on the requests.

**Technologies Used**:  
  * **OpenShift AI** – to deploy your model
  * **3scale API Management** – to control, track, and limit access
  * **Single Sign-On (SSO)** – to manage secure user logins
 
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


## Architecture Overview

![architecture](img/architecture.drawio.svg)

The diagram above illustrates how all the parts of the MaaS architecture interact, from the user sending a request to the model delivering a response, using *3scale* and *OpenShift AI*. At a high level, this setup connects a client application to a machine learning model via a secure, managed API pathway. 

Here’s how each part contributes:

The client-side application (e.g., a web or mobile app) sends a request with input data to your API, which first passes through the 3scale API Gateway.

**3scale API Gateway**
The 3scale API Gateway acts as a traffic controller, verifying API keys, enforcing access permissions, reporting usage, and forwarding authorized requests to the ML model on OpenShift AI. The 3scale API Manager serves as the control center, managing access policies, syncing with the Developer Portal, configuring backends, and coordinating with the API Gateway for real-time request validation.

**3scale Developer and Admin Portal**
The 3scale Developer Portal is a self-service interface where external users can register, access API documentation, and retrieve their personal API keys. Meanwhile, the 3scale Admin Portal is the API owner's control panel, enabling them to define access plans, manage users and billing, and monitor API usage and analytics.

**OpenShift AI - Model Hosting**  
OpenShift AI is the hosting environment for your deployed machine learning model, where it receives requests forwarded by the API Gateway. It processes input data to generate predictions and supports model scaling, versioning, and lifecycle management.

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
Red Hat 3scale uses a *persistent system storage* with an RWX volume, which can be provided by *OpenShift Data Foundation (ODF)*.

### Instructions
- **Create a project** (e.g. ***3scale***) in your openshift cluster
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

⚠️ **Note:** These files are copied from [APIcast LLM Metrics Policy](https://github.com/rh-aiservices-bu/models-aas/tree/main/deployment/3scale/llm_metrics_policy#apicast-llm-metrics-policy)

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

- Open the link and log in using the credentials stored in the secret *'system-seed'* (ADMIN_USER and ADMIN_PASSWORD). 

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

💡 **Tip:** To add additional models, repeat this process for each one of them


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

    2. *(Optional)* **LLM Monitor** - for OpenAI-Compatible token usage. See [Readme](./deployment/3scale/llm_metrics_policy/README.md) for more information and configuration.

    ![products_policies_LLMMonitor.png](img/products_policies_LLMMonitor.png)

    3. **3scale APIcast**

  💡**Important:** After completing the configuration, **DO NOT FORGET to click "Update Policy Chain".**  
    Otherwise, all your changes will be lost.

  ![products_policies_list.png](img/products_policies_list.png)

##### --->>> Products - Configuration & Staging <<<---

- From the *"Integration → Configuration"* menu, promote the configuration to *"Staging"*, then to *"Production"*.
  ![products_config_staging_1.png](img/products_config_staging_1.png)
  ![products_config_staging_2.png](img/products_config_staging_2.png)


##### --->>> Products - Applications & Application Plans <<<---

- For each Product, go to *"Applications → Application Plans"*, and create a new Application Plan. At this stage, we are not forcing (*requiring*) approval for the applications under this plan.

  ![products_application_plans.png](img/products_application_plans.png)

- On the page, where application plans are listed, leave the *Default plan* set to *"No plan selected"* so users can choose their preferrend services when creating applications. **Note that the plan is hidden by default and hence we should publish it**:

  ![products_application_plans_1.png](img/products_application_plans_1.png)

- In *"Applications → Settings → Usage Rules"*, set the *Default plan* to 'Default'. This will allow the users to see (*This ensures users will see*) the different available Products when browsing the portal:

  ![products_application_plans_2.png](img/products_application_plans_2.png)

**Attention:** When creating a new application from the developer portal, the only available plan is a default *"Basic"* plan:

![portal_applicaion_basic.png](img/portal_applicaion_basic.png)

Therefore, we first create an application directly from the ***3scale admin portal***, by going to the *"Applications → Listing"*:

![products_application_creation.png](img/products_application_creation.png)

This way, we can now select the desired product (in our case e.g., *"granite-7b-instruct"*), when creating applications from the portal:

![portal_applicaion_select_product.png](img/portal_applicaion_select_product.png)
![portal_applicaion_standardplan.png](img/portal_applicaion_standardplan.png)

##### --->>> Products - Active Docs <<<---

**Attention:** Without *"Active Docs"*, the *"Enpoint URL"* will not be displayed on the application page in the developer portal:

![portal_applicaion_endpointURL.png](img/portal_applicaion_endpointURL.png)

To fix this, we add a *"spec"* under *"Active Docs"* for the product. In order to do that,
  - Copy the content of the file: [deployment/3scale/active_docs.json](./deployment/3scale/active_docs.json)
  - Paste it into the corresponding section of the *Active Docs > spec* form.
  - Set a proper (or meaningful) *"name"* and *"system name"*.
  - Check the box to *"publish the docs"* and click on *"Create spec"*.

![products_active_docs.png](img/products_active_docs.png)


#### Audience - Portal Content 
Under the **Audience** section, go to the *"Developer Portal → Content"* and begin editing the corresponding pages and files:

![audience_1.png](img/audience_1.png)
![audience_portal_content_1.png](img/audience_portal_content_1.png)

⚠️ **Note:** These portal configurations are based on these files [models-aas repo - Portal Configurations](https://github.com/rh-aiservices-bu/models-aas/tree/main/deployment/3scale/portal). **However** some files in the Repository have been updated, as the original versions on the repo did not create the pages correctly and a few are non-functional.

- In this directory, navigate to the following directory in your cloned repo: [deployment/3scale/portal](deployment/3scale/portal).
- From the *"deployment/3scale/portal"* folder, apply all necessary modifications to the relevant pages. Then make sure to *"Save and Publish"*, both the *Draft* and *Published* versions.
- **Structure:** The content of this folder are organized to match the layout of the Developer Portal site.

⚠️ **Note:** Some pages may need to be created. Choose the type based on the content (HTML, JavaScript, CSS). Others may already exist and only need to be updated.

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
  
- In this step, we will add new pages and files to the default **Developer Portal**, based on the specifications listed below:
  - **Examples:** Create a 'New Page' named *'Examples'*.

  ![audience_portal_content_examples.png](img/audience_portal_content_examples.png)

- **css → rhoai_custom.css:** Create a 'New Page' named *'rhoai_custom.css'* under the **css** section.

  ![audience_portal_content_rhoai_custom.png](img/audience_portal_content_rhoai_custom.png)

- **javascripts → secret_keys.js:** Create a 'New Page' named *'secret_keys.js'* under the **javascripts** section.


  ![audience_portal_content_secret_keys.png](img/audience_portal_content_secret_keys.png)

- **images → rhoai.png:** Create a 'New File' named *'rhoai.png'* under the **images** section.
 
  ![audience_portal_content_images.png](img/audience_portal_content_images.png)
