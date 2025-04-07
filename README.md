# Multi Instance GPU with Red Hat OpenShift AI

## Using MIG in Red Hat OpenShift AI
MIG is now enabled, you can now go ahead and test it using Red Hat OpenShift AI (RHOAI) as explained below.
For using MIG profiles in Red Hat OpenShift AI we will need to:

 - Configure Accelerator Profiles in RHOAI Dashboard
 - Deploy a model server and the model with the associated MIG Profile
 - Scale the model to provide Load Balancing for the model server

#### Configure Accelerator Profiles in RHOAI Dashboard
An accelerator profile is a Custom Resource Definition (CRD) that specifies the configuration for a particular accelerator. It can be managed directly through the OpenShift AI Dashboard under **Settings → Accelerator Profiles**. Here we will need to create all the needed profiles corresponding to our MIG configuration **```mig-1g-6gb```** and **```mig-2g-12gb```** in this case.

 1. Login to RHOAI dashboard
 
![enter image description here](https://raw.githubusercontent.com/rohitralhan/GPUSharingMIG/refs/heads/main/images/RHOAILoginOut.gif)

 2. Navigate to **`Settings → Accelerator Profiles`** click **`Create accelerated profiles`** button and follow the on-screen instructions to create two accelerator profiles **```mig-1g-6gb```** and **```mig-2g-12gb```**.

![enter image description here](https://raw.githubusercontent.com/rohitralhan/GPUSharingMIG/refs/heads/main/images/CreateAccProfileOut.gif)

---
#### Deploy a model server and the model with the associated MIG Profile
Once the Accelerator Profiles are created, navigate to the data science project, create a model server, and deploy the model. 
For this demonstration, we are using an iris model available [here](https://raw.githubusercontent.com/rohitralhan/GPUSharingMIG/refs/heads/main/models/rf_iris.onnx). 

 1. Download the [rf_iris.onnx](https://raw.githubusercontent.com/rohitralhan/GPUSharingMIG/refs/heads/main/models/rf_iris.onnx) model.
 2. Upload the model to a S3/S3 compatible bucket 
 3. Login to the Red Hat OpenShift AI console 
 4. Navigate to **```Data Science Projects --> Create Project```**, follow the onscreen instructions to create the project
 5. Next create a data connection for saving the **```rf_iris.onnx```** models.
	 1. Navigate to the **```Connections```** tab and click **```Add Connection```** button
	 2. Follow the onscreen instructions for the **```S3 compatible object storage```** and click create   
 6. Now we will create a model server 
 7. Navigate to the **```Models```** tab and click **```Add model server```** button. Follow the onscreen. Fill out the form with the following values:  
	 i. Model server name: infer-model-server  
	 ii. Serving runtime: **`OpenVINO Model Server`**  
	 iii. Model server replicas: **`1`**
	 iv. Model server size: **`Small`** (this can be adjusted according to the model needs)
	 v. Accelerator: **`NVIDIA mig-2g-12gb`** (the reference to the MIG partition created)
	 vi. Number of accelerators: **`1`**
	 vii. **`Enable`** the Make deployed models available through an external route option  
	 viii. **`Enable`** the Require token authentication option
 8. Click **```Add```** to add a model server.
 9. Once the model server is added click **```Deploy model```** on the right of the model server. Fill out the form with the following values:
	 i. Model name:  **```iris-model```**
	 ii. Model framework:  **`onnx - 1`**
	 iii. Select  Existing data connection
	 iv. Select the  **`iris-data-connection`**  data connection
	 v. Path:  **`iris`** or path to your model in the S3 buket
 10. Click **```Deploy```** to deploy the model

![enter image description here](https://raw.githubusercontent.com/rohitralhan/GPUSharingMIG/refs/heads/main/images/DeployOut.gif)

If you navigate to  **```Workloads --> Pods --> nvidia-driver-daemonset-***** pod --> Terminal```** in the **```nvidia-gpu-operator```** project ,  type **```nvidia-smi```** it will how a similar output shown in the image below. As you can see in our case the model server is using the MIG profile with GIID as 1 which is our 12 GB MIG instance as selected above while creating the model server.

![enter image description here](https://raw.githubusercontent.com/rohitralhan/GPUSharingMIG/refs/heads/main/images/NvidiaMig-2g.png)

---
#### Scale model server for Load Balancing
Scaling a model server in **OpenShift** for **load balancing**, using **Multi-Instance GPU (MIG)** is a great way to optimize GPU resource utilization, especially on **NVIDIA  A or H series** or similar GPUs. Next we will look at how we can scale the model server in RHOAI:

Follow the steps below to scale the model server:<BR>
&nbsp;&nbsp;i. Login to Red Hat OpenShift AI<BR>
&nbsp;&nbsp;ii. Navigate to the project **```sample-project --> Models tab```**<BR>
&nbsp;&nbsp;iii. Click the three dots next to **```Deploy Model```** button<BR>
&nbsp;&nbsp;iv. Click **```Edit model server```**<BR>
&nbsp;&nbsp;v. Increase the **```Number of model server replicas to deploy```** to 3<BR>
&nbsp;&nbsp;vi. Change the **```Accelerator```** to **```NVIDIA mig-1g-6gb```**<BR>
&nbsp;&nbsp;vii. Click the **```Update```** button to update/redeploy the model<BR>

The animation below shows multiple mig-1g-6gb partitions being utilized across two worker nodes having an NVIDIA A30 GPU each and two mig-1g-6gb partitions on each GPU.

![enter image description here](https://raw.githubusercontent.com/rohitralhan/GPUSharingMIG/refs/heads/main/images/ScaleDeployOut.gif)
