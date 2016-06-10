# CoreOS Linux on Azure: White-Glove Lab
 
## Intro

You will need to do three things to prepare for and provision this lab:

* Set up a discovery URL for the etcd cluster
* Create an SSH keypair
* Download and deploy the Azure Resource Manager deployment template

### Set up a discovery URL for the etcd cluster

When you deploy them template, you will have the option to choose a 3-node, 5-node or 7-node etcd cluster.  What size you choose is up to you, but the size you use to generate the discovery URL and the size of the cluster you specify when deploying the template must match or the deployment will not behave correctly.

For your convenience, to generate a discovery URL, just click on the appropriate link, then copy and save the URL that is returned in your browser window:

* [3-node cluster](https://discovery.etcd.io/new?size=3)
* [5-node cluster](https://discovery.etcd.io/new?size=5)
* [7-node cluster](https://discovery.etcd.io/new?size=7)

### Create an SSH keypair

(Note: if you already have a keypair and would like to use it for this deployment, you can skip this step)

#### Linux and MacOS X

For Linux and MacOS X machines, creating the keypair should be as simple as this:

`$ ssh-keygen -f ~/.ssh/id_rsa_coreos_whiteglove`

You will be prompted for the passphrase and the keypair will be created, with the public key in id_rsa_coreos_whiteglove.pub.  When SSH'ing into your VMs after the deployment, use the -i flag to specify the private key file:

`$ ssh -i ~/.ssh/id_rsa_coreos_whiteglove core@[public IP address]`

#### Windows

For Windows there are no SSH tools built in -- if you already have SSH tools set up on Windows, you can skip this step, but if not, follow these instructions to download and use PuTTY.

First download [PuTTY](https://the.earth.li/~sgtatham/putty/latest/x86/putty.exe), [PuTTYgen](https://the.earth.li/~sgtatham/putty/latest/x86/puttygen.exe) and [Pageant](https://the.earth.li/~sgtatham/putty/latest/x86/pageant.exe) to a folder somewhere on your computer.  Then open a command prompt and change to that directory (in this example I downloaded all of them to my Downloads folder:

`C:\Users\kensey>cd Downloads`

Now start puttygen:

`C:\Users\kensey\Downloads>puttygen.exe`

A window will open; click the "Generate" button and follow the instructions.  After the keypair is generated, click "Save private key" and save it to a file in the same folder you saved PuTTY in.  Copy and save the text from the public key pane at the top -- this will be used during the ARM template deployment.

To make your life easier, you may wish to load Pageant so you don't have to specify the key file or enter the key passphrase with PuTTY:

`C:\Users\kensey\Downloads>pageant.exe`

Once Pageant starts up it will create a system tray icon of a computer with a hat.  Double-click the icon to open the Pageant GUI, then click "Add key" and browse to the private key file (which will have a .ppk extension) that you saved earlier.  You will be prompted for the passphrase (if any) and the key will be added, after which PuTTY will use Pageant for the key.

### Download and deploy the Azure Resource Manager deployment template

The Azure Resource Manager deployment template for this lab is located in this GitHub repository, under the filename [coreos-whiteglove.json](https://raw.githubusercontent.com/omkensey/coreos-whiteglove-lab/master/coreos-whiteglove.json).

When you deploy the template, the required options with no defaults are the storage account name, SSH public key (in OpenSSH format, e.g. "ssh-rsa AAAA.....") and discovery URL that you created earlier.  There is also an option for the etcd cluster size, which should match the size you specified when creating the discovery URL.

After the notice that deployment was successful, to SSH into your instances:

`C:\Users\kensey\Downloads>putty.exe core@[public IP]`

If you're able to do that, you're ready to continue with the lab.
