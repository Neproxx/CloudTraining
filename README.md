# CloudTraining
Tutorial on training machine learning models on Azure spot instances as part of the KTH DevOps 2022 course

TODO: Give background and big picture

### Creating a training script with checkpointing

There are 4 main steps to the training script:

1.  Create a folder to save your model at some frequency (e.g after every epoch)
2.  Load your dataset of interest
3.  Specify the callback/checkpointing object (we used Keras callbacks)
4.  Check if model already exits in folder (that you save your model to) and simply resuming training if model exists

Firstly, we will create a folder called 'Saved_Model' in our working directory where we can store our saved models whenever training is interrupted. This can be made using:

```console
mkdir Saved_Model
```

For our training script example, we are going to use Tensorflow with Keras API to build a Convolutional Neural Network (CNN) on the following dataset (any other dataset of interest can be used): [horses_or_humans](https://www.tensorflow.org/datasets/catalog/horses_or_humans). 

We will also use the 'MobileNetV3Small' architecture as it has over a million parameters to learn but any other keras model instance can be used. You can find this architecture under:

```python
tf.keras.applications.MobileNetV3Small
```

Now we will load the dataset and split it into a training set and a validation set using:

```python
(train_ds, val_ds) = tfds.load(name='horses_or_humans', split=['train', 'test'],
                               as_supervised=True, batch_size=32)                           
```

The next part is to set the callback object in order to save the Keras model or model weights at some frequency. This can easily be done using [ModelCheckPoint](https://www.tensorflow.org/api_docs/python/tf/keras/callbacks/ModelCheckpoint) (check the link for a deeper description of the parameter settings):


```python
checkpoint = tf.keras.callbacks.ModelCheckpoint(
    filepath=os.path.join(os.getcwd(), 'Saved_Model', 'Models.{epoch}-{val_loss:.2f}.hdf5'),
    monitor='val_loss',
    verbose=0,
    save_best_only=False,
    save_weights_only=False,
    mode='min',
    save_freq='epoch',
    options=None,
    initial_value_threshold=None,
)                           
```

When specifying the name of the saved model, 'Models.{epoch}-{val_loss:.2f}' in our case, it is important to include the .{epoch} part as this will later on be used to inform us of the epoch number when our model gets terminated. We also saved the file using the '.hdf5' extension such that the whole model is contained in a single file. We also save the model at every epoch as can be set by the 'save_freq' setting.

Next is to check whether a model already exists in the 'Saved_Model' file and to simply resume training from there. We will also use a regular expression to extract the epoch number from the saved file name and then load the model to continue training from the last epoch before termination. 

This can be done in the following way:

```python
# If model already exists, continue training
if os.listdir(os.path.join(os.getcwd(), 'Saved_Model')):

    # Regular expression pattern to extract epoch number
    pattern = '[^0-9]+([0-9]+).+'
    filename = os.listdir(os.path.join(os.getcwd(), 'Saved_Model'))[-1]

    # Find epoch number
    last_epoch = int(re.findall(pattern=pattern, string=filename)[0])

    # Load model and continue training model from last epoch
    model = load_model(filepath=os.path.join(os.getcwd(), 'Saved_Model', filename))
    model.fit(x=train_ds, epochs=5, validation_data=val_ds, callbacks=[checkpoint], initial_epoch=last_epoch)                    
```

If no model exists already (i.e no training has been done yet), we simply define our model (MobileNetV3Small in our case) and compile then fit the model to the data for training.

Now whenever our training gets interuptted, the script will simply refer to the 'Saved_Model' file and just reload the model from where it left off.

Check out the [main.py](https://github.com/Neproxx/cloud-training/blob/main/main.py) in the repository to see the whole training script.


### Building a container

After creating your training script, the next step is to create what is called a 'container'. A container is basically a unit of software that packages up all the code and dependencies required to run your application (in this case the training script). To do this, we will use [Docker](https://www.docker.com/) which is a popular open platform for delivering software applications using containers.

Docker consists of two main components:

1. [Docker Engine](https://docs.docker.com/engine/) - This is the packaging tool used to build and run container images (a container is simply a running image)
2. [Docker Hub](https://docs.docker.com/docker-hub/) - This is a cloud service for sharing your applications (which we will use to download the container image on the VM)

To build the container image, we will have to create what is called a [Dockerfile](https://docs.docker.com/engine/reference/builder/). This is simply a file with a few lines of code that packages up all the required dependencies, libraries etc... in the container image. For our training script, the Dockerfile looks like this:


```dockerfile
FROM tensorflow/tensorflow

WORKDIR /app

COPY . .

# Install dependencies
RUN pip install tensorflow_datasets && \
    mkdir Saved_Model

CMD ["python", "main.py"]                    
```

1. Inherits the tensorflow image from Dockerhub as we require tensorflow in our application
2. Creates and sets the working directory inside the container to the folder '/app'
3. Copy the required files from our host machine into the '/app' folder (the current working directory) 
4. Install the 'tensorflow_datasets' python library as we use this to obtain the dataset for the training script. A 'Saved_Model' folder is also made where we would store our saved models in case training gets interrupted
5. Run our training script 'main.py'

Note that you MUST call your Dockerfile 'Dockerfile' and not any other name.

To finally build the container image, you must have Docker installed on your host computer. You can follow the steps provided [here](https://docs.docker.com/desktop/).
You can then build the container image from the Dockfile using your terminal by (where path refers to the directory containing the Dockerfile):

```console
docker build <path>
```
You can then view your created container images using:

```console
docker images
```

To push your image to Dockerhub, you must first have a [Dockerhub repository](https://docs.docker.com/docker-hub/repos/). You must name your local image using your Docker Hub username and the repository name that you created through Docker Hub on the web. Finally, you can now push the image to Dockerhub using:

```console
docker push <hub-user>/<repo-name>:<tag>
```


### Creating a VM
Setting up a VM programmatically is useful in production, however since this is a tutorial, it is easier and safer to use the UI in the webbrowser that informs you about prices, etc... Login to the [Azure portal](portal.azure.com) and start the process of creating a VM as shown below.

![Create VM](./images/01-Azure-tutorial-Create-VM.gif)

First of all, we need to create a resource group that groups all related components of an application together. This is very useful, as many resources will be created implicitly and you will only know about them and find them by investigating the resource group. A VM for example requires disk space and an IP address which both represent implicitly created resources. When you want to delete your application, you can find all of them inside your resource group. We create a resource group named "tutorial-resource-group". If you prefer to choose your own names in this and the following commands, you will have to replace them in the rest of the tutorial as well.

![Create resource group](./images/02-Azure-tutorial-Resource-Group-and-VM-name.gif)

Next up, we need a VM image to run on our new server. For us, it is important that it has Docker installed so that we can start the Docker container when the system boots up. The tensorflow image from Nvidia fits our purposes.

![Select correct image](./images/03-Azure-tutorial-Image-selection.gif)

We tick the "spot instance" box and notice that prices go down by 80% to 90%. We can select an eviction policy that is either solely based on capacity, meaning that Azure will terminate our instance when they require the hardware capacity, or also on price. Prices for cloud servers are dynamic and thus it is reasonable to evict a machine once it passes a price threshold. You may set it to for example 0.5$. 

![Configure spot instance](./images/04-Azure-tutorial-Spot-Instance-configuration.gif)

Most of our costs will be determined by the server size / hardware we select. In a real use case, we would select a large server size including GPUs in order to speed up model training. For educational purposes however, it is sufficient to select a small server that is covered by Azure's free plan.

![Select hardware](./images/05-Azure-tutorial-Select-hardware.gif)

Finally, enter a user name and a key pair name for ssh authentication. This will not be needed in the scope of the tutorial, but you could use these credentials to connect to the VM. Click on Review + create and see how your VM instance is created.


![Finish VM creation](./images/06-Azure-tutorial-click-on-create.gif)

### Setting up the VM
For managing the VM, we will use the Azure CLI. For this we either need to install it or start a docker container that already has it installed. We do the latter with the command below. 

```console
docker run -it mcr.microsoft.com/azure-cli bash
```

Note that "-it ... bash" tells docker to glue our terminal to the container and start bash inside of it so that we can interact with it. On the other hand "mcr.microsoft.com/azure-cli" is the image name that is provided by Microsoft and downloaded from [dockerhub](https://hub.docker.com/_/microsoft-azure-cli). Once the container is up and running, login by following the instructions inside the terminal.

```console
az login
```

Since we are using spot instances, our VM may be shut down or restarted at any point in time. Therefore, we have to make sure that the container always starts on bootup and continues training. We now create an executable script in the folder "/tutorial" which starts the container. Replace the image name in the docker container run line with your own image name if you built it yourself and pushed it to dockerhub.

```console
az vm run-command invoke -g tutorial-resource-group -n cloud-training --command-id RunShellScript --scripts \
"mkdir /tutorial
touch /tutorial/startup-script.sh
cat <<EOF > /tutorial/startup-script.sh
#\!/bin/bash
docker container run neprox/cloud-training-app 
EOF
chmod +x /tutorial/startup-script.sh"
```

The syntax cat "<<EOF > file_name ... EOF" below is commonly used to push a multiline string (indicated by "...") into a file. In addition, we need to explicitly tell the os with "chmod +x" that the file is allowed to be executed. Note also that we could login to the VM directly via ssh or install a software to login to the VM with a GUI, however for simplicity we stick to the Azure CLI.

After we have created a script that starts the container, we need to execute it on every bootup. The cronjobs package suits our purposes, as it executes jobs on a periodic schedule. In our case, we specify our schedule with the string "@reboot". Since the syntax can be hard to comprehend, the [crontab-generator] is very useful to generate the cronjob. The script that is executed below can be summarized by "Store all current cronjobs to cron_bkp, add a new cronjob, schedule all the cronjobs that are now in cron_bkp, delete the file cron_bkp".

```console
az vm run-command invoke -g tutorial-resource-group -n cloud-training --command-id RunShellScript --scripts \
"sudo crontab -l > cron_bkp
sudo echo '@reboot sudo /tutorial/startup-script.sh >/dev/null 2>&1' >> cron_bkp
sudo crontab cron_bkp
sudo rm cron_bkp"
```

### Model Training
It is time to train a model on our VM.

... TODO ...

Since it can be terminated / evicted at any moment, we will use the Azure CLI to simulate such an incident:  
```console
az vm simulate-eviction --name cloud-training --resource-group tutorial-resource-group
```

### Cleanup
... TODO ...
Explain how to delete the things again, give a hint that they can double check within the UI of the webbrowser.

```console
az group delete --name tutorial-resource-group
```

### Further Notes
In this tutorial, we stored the model on the VM directly. In reality you would prefer 
