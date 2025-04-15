# Multi Instance GPU with Red Hat OpenShift AI

## Using MIG in Red Hat OpenShift AI
Once MIG is enabled (refer to these [instructions](https://github.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/blob/main/README.md) to enable MIG), you can now go ahead and test it using Red Hat OpenShift AI (RHOAI) as explained below.
For using MIG profiles in Red Hat OpenShift AI we will need to:

 - Configure Accelerator Profiles in RHOAI Dashboard
 - Deploy a model server and the model with the associated MIG Profile
 - Scale the model to provide Load Balancing for the model server

#### Configure Accelerator Profiles in RHOAI Dashboard
An accelerator profile is a Custom Resource Definition (CRD) that specifies the configuration for a particular accelerator. It can be managed directly through the OpenShift AI Dashboard under **Settings → Accelerator Profiles**. Here we will need to create all the needed profiles corresponding to our MIG configuration **```NVIDIA mig-1g-6gb```** and **```NVIDIA mig-2g-12gb```** in this case.

 1. Login to RHOAI dashboard
 
![Login to RHOAI](https://raw.githubusercontent.com/rohitralhan/MIG-with-RHOAI/refs/heads/main/images/RHOAILoginOut.gif)

 2. Navigate to **`Settings → Accelerator Profiles`** click **`Create accelerated profiles`** button and follow the on-screen instructions to create two accelerator profiles **```NVIDIA mig-1g-6gb```** and **```NVIDIA mig-2g-12gb```**.

![Accelerator Profiles](https://raw.githubusercontent.com/rohitralhan/MIG-with-RHOAI/refs/heads/main/images/CreateAccProfileOut.gif)

---
#### Deploy a model server and the model with the associated MIG Profile
Once the Accelerator Profiles are created, navigate to the data science project, create a model server, and deploy the model. 
For this demonstration, we are using an iris model available [here](https://github.com/rohitralhan/MIG-with-RHOAI/raw/refs/heads/main/models/rf_iris.onnx). 

 1. Download the [rf_iris.onnx](https://github.com/rohitralhan/MIG-with-RHOAI/raw/refs/heads/main/models/rf_iris.onnx) model.
 2. Upload the model to a S3 or S3 compatible bucket. You can use Minio (S3 compatible object store) on OpenShift, for instructions refer to [minio install](https://ai-on-openshift.io/tools-and-applications/minio/minio/). DO NOT upload the model to the bucket root, create a folder in the bucket and then upload the model.
 3. Login to the Red Hat OpenShift AI console 
 4. Navigate to **`Data Science Projects --> Create Project`**, follow the onscreen instructions to create the project
 5. Next create a data connection for retrieving the **`rf_iris.onnx`** models.
	 1. Navigate to the **`Connections`** tab and click **`Create Connection`** button
	 2. On the **`Add conncetion`** screen fill the form with the following values and click create. (for instructions on installing Minio an S3 compaitable object store, refer to [minio install](https://ai-on-openshift.io/tools-and-applications/minio/minio/)). If you are NOT using Minio on OpenShift, replace the S3 connection details as appropriate to your environment.
		- Connection type: **`S3 compatible object storage`**
		- Connection name: **`iris-data`**
		- Access key: **`<<minio login user name>>`**
		- Secret key: **`<<minio login user password>>`**
		- Endpoint: **`<<minio (service) url>>`**
		- Region: **`us-east-1`**
		- Bucket: **`<<minio bucket name>>`**
 6. Next we will create a model server 
 7. Navigate to the **`Models`** tab and click **`Select multi-model`** and then click **`Add model server`** button. On the **`Add model server`** screen fill the form with the following values:
	- Model server name: infer-model-server
	- Serving runtime: **`OpenVINO Model Server`**
	- Model server replicas: **`1`**
	- Model server size: **`Small`** (this can be adjusted according to the model needs)
	- Accelerator: **`NVIDIA mig-2g-12gb`** (the reference to the MIG partition created)
	- Number of accelerators: **`1`**
	- **`Enable`** the **`Make deployed models available through an external route`** option
	- **`Enable`** the **`Require token authentication`** option
 	- Click **```Add```** to add a model server.
 8. Once the model server is added click **`Deploy model`** on the right of the model server. Fill out the form with the following values:
	- Model name:  **`iris-model`**
	- Model framework:  **`onnx - 1`**
	- Select  Existing data connection, created in step 5 above
	- Select the  **`iris-data`**  data connection
	- Path:  **`<<path inside the bucket>>`**, path to the model in the S3 buket
9. Click **`Deploy`** to deploy the model

![Model Deployment](https://raw.githubusercontent.com/rohitralhan/MIG-with-RHOAI/refs/heads/main/images/DeployOut.gif)

Once the model is successfully deployed, navigate to  **```Workloads --> Pods --> nvidia-driver-daemonset-***** pod --> Terminal```** in the **```nvidia-gpu-operator```** project ,  type **```nvidia-smi```** it will show an output similar to the one shown in the image below. As you can see in our case the model server is using the MIG profile with GIID as 1 which is our 12 GB MIG instance as selected above while creating the model server.

![NVIDIA SMI Output](https://raw.githubusercontent.com/rohitralhan/MIG-with-RHOAI/refs/heads/main/images/NvidiaMig-2g.png)

#### Test the model

To test the model set the below `environment variables` and run the `curl` command as shown below

- Retrieve the `model url` from `Data Science Project --> Project --> Models --> Deployed models --> Inference endpoint --> Click Internal and external endpoint details - Copy External url`
- Retrieve the `model token` from `Data Science Project --> Project --> Models --> Tokens --> Token secret --> Click Copy icon`

```
MODEL_URL="Retrieve from the deployed model" 
TOKEN="Retrieve from the deployed model"
HEADER="Authorization: Bearer ${TOKEN}"
PAYLOAD='{"inputs": [{ "name": "X", "shape": [1, 4], "datatype": "FP32", "data": [3, 4, 3, 2] } ]}'
```

```curl -X POST $MODEL_URL -H $HEADER -d $PAYLOAD  -k```


![Model Details](https://raw.githubusercontent.com/rohitralhan/MIG-with-RHOAI/refs/heads/main/images/model-details.gif)

---
#### Scale model server
Scaling a model server in **OpenShift** for **load balancing**, using **Multi-Instance GPU (MIG)** is a great way to optimize GPU resource utilization, especially on **NVIDIA  A or H series** or similar GPUs. Next we will look at how we can scale the model server in RHOAI:

Follow the steps below to scale the model server:<BR>
&nbsp;&nbsp;i. Login to Red Hat OpenShift AI<BR>
&nbsp;&nbsp;ii. Navigate to the project you created above **```your-project-name --> Models tab```**<BR>
&nbsp;&nbsp;iii. Click the three dots next to **```Deploy Model```** button<BR>
&nbsp;&nbsp;iv. Click **```Edit model server```**<BR>
&nbsp;&nbsp;v. Increase the **```Number of model server replicas to deploy```** to 3<BR>
&nbsp;&nbsp;vi. Change the **```Accelerator```** to **```NVIDIA mig-1g-6gb```**<BR>
&nbsp;&nbsp;vii. Click the **```Update```** button to update/redeploy the model<BR>

For this test we have a cluster with two nodes and one A30 GPU each, which in all-balanced mode, partitions the GPU in 3 slices, one 12G slice and two 6G slices on each node and hence we need to update the accelerator profile to `NVIDIA mig-1g-6gb` which will allow us to scale to 3 replicas and utilize three of the `NVIDIA mig-1g-6gb` slices across two nodes.

The animation below shows multiple mig-1g-6gb partitions being utilized across two worker nodes having an NVIDIA A30 GPU each and two mig-1g-6gb partitions on each GPU.

![Scale model server](https://raw.githubusercontent.com/rohitralhan/MIG-with-RHOAI/refs/heads/main/images/ScaleDeployOut.gif)
