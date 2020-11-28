# Setting up a machine learning cloud environment from scratch

In the last post I began to build a tensorflow classifier model. At the training stage we began to run into some hardware deficiencies. In this post I will go over how to set up an Ubuntu cloud instance on google compute engine and set up everything we need to build and train our model.

There are several machine learning pre-built images in the google compute engine Marketplace which could also be used, however I ran into issues trying to deploy some of these images due to there not being enough GPUs available in the default region or some reason or another. The pre-trained ELMo model we are using from tensorflow hub also seems to not work well with tensorflow 2 so I decided to just go from scratch.

### Generate an SSH key pair
If you don't already have an ssh key pair the first step is to generate one.
We will generate an SSH key pair using RSA and 4096 bits.
```console
ssh-keygen -t rsa -b 4096 -C "username"
```
Replace username with whatever you like and when prompted choose a name for your key pair.
You should now have have two file my\_key and my\_key.pub.

### Creating the instance
In the VM instances page in the Google cloud platform console click "create".  
I then selected us-west1-b as the region/zone.  
For the machine type I selected n1-standard-8, 8 vCPU cores and 30GB of RAM.  
Now click on the drop down menu "CPU platform and GPU" and add a GPU, I just went with the default of a Tesla K80.  
For the boot disk I went with Ubuntu 16.04 LTS and 20GB disk space.  
Now click the "Management, security, disks, networking, sole tenancy" menu.  
Go across to the Security tab and paste in the contents of your public key file (my\key.pub) 
You should also go across to the Disks tab and ensure "Delete boot disk when instance is deleted" is checked.  
Now click create. If everything went right you should have a new instance running. 

### Install the basics 
Copy the external IP of the new instance e.g. 123.456.789.123  
Log into the new instance using your ssh key, chosen username and IP of the instance:
```console
ssh -i my_key username@123.456.789.123
```
You should now have a terminal prompt to your Ubuntu instance.  
Now we want to install python3 pip and set up a virtual env, basically following the steps here: [https://www.tensorflow.org/install/pip](https://www.tensorflow.org/install/pip)  
```console
sudo apt update
sudo apt install python3-dev python3-pip python3-venv
```
Now create a virtual environment:
```console
python3 -m venv --system-site-packages ./venv
```
For some reason I was getting an error: "The virtual environment was not created successfully because ensurepip is not
available."  
After finding this thread [https://stackoverflow.com/questions/39539110/pyvenv-not-working-because-ensurepip-is-not-available](https://stackoverflow.com/questions/39539110/pyvenv-not-working-because-ensurepip-is-not-available) the first solution worked.
I ran:
```Console
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"
sudo dpkg-reconfigure locales
```
and was then able to create the virtual env without error.
Now activate the virtual environment and check you have the correct versions of python and pip running:
```console
source ./venv/bin/activate
pyhton --version
pip --version
```
On my installation it was not quite right, I had to run:
```console
python3 -m pip install --upgrade pip
```
Now I get the following output when checking the python and pip versions while the venv is activated:
```console
(venv) user@instance-1:~$ pip --version
pip 20.1.1 from /home/user/venv/lib/python3.5/site-packages/pip (python 3.5)
(venv) user@instance-1:~$ python --version
Python 3.5.2
(venv) user@instance-1:~$ 

```
This shows we are running Python 3 and pip is now pointing to the correct location within the virtual environment directory.  
Next install Tensorflow GPU:
```console
pip install tensorflow-gpu==1.15
```
Next we need to install CUDA, we will follow the instructions for Ubuntu 16.04 [https://www.tensorflow.org/install/gpu?hl=tr](https://www.tensorflow.org/install/gpu?hl=tr)  
```console
sudo apt-get install gnupg-curl
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_10.1.243-1_amd64.deb
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub
sudo dpkg -i cuda-repo-ubuntu1604_10.1.243-1_amd64.deb
sudo apt-get update
wget http://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1604/x86_64/nvidia-machine-learning-repo-ubuntu1604_1.0.0-1_amd64.deb
sudo apt install ./nvidia-machine-learning-repo-ubuntu1604_1.0.0-1_amd64.deb
sudo apt-get update
```
Now reboot:
```console
sudo reboot
```
And then (after waiting a minute and then SSHing back into the instance) run the rest of the CUDA install commands:
```console
sudo apt-get install --no-install-recommends \
    cuda-10-1 \
    libcudnn7=7.6.4.38-1+cuda10.1  \
    libcudnn7-dev=7.6.4.38-1+cuda10.1


# Install TensorRT. Requires that libcudnn7 is installed above.
sudo apt-get install -y --no-install-recommends \
    libnvinfer6=6.0.1-1+cuda10.1 \
    libnvinfer-dev=6.0.1-1+cuda10.1 \
    libnvinfer-plugin6=6.0.1-1+cuda10.1

```
You can run `nvidia-smi` to check the install went ok.  
Now install various python libraries:
```console
pip install pandas matplotlib jupyterlab scipy scikit-learn tensorflow_hub keras
```
### Create an SSH port forwarding and run Jupyter Notebook
By default when you run Jupyter Notebook it runs locally on port 8888. As we are running it on a remote server we need to be able to access it from out local machine. Rather than exposing the port publicly we will just create a tunnel via SSH to forward the port.
```console
ssh -i my_key -L 8000:localhost:8888 username@123.456.789.123
```
Then run Jupyter Notebook (remember to run this command after activating the venv)
```console
jupyter notebook
```
And that's it. You should now be able to access Jupyter Notebook running on the GPU enabled cloud instance by going to localhost:8000.

Don't forget to delete the instance when you are finished, they are quite expensive.
