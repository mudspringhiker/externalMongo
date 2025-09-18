# Installing MAS with External MongoDB

The following are the steps in installing MAS using an external mongodb instance deployed in a separate OpenShift cluster (where another MAS instance is also deployed and using that mongodb instance as a dependency), following this [document](https://www.ibm.com/support/pages/installing-mas-using-ibm-mas-cli-utility). It assumes that the [IBM MAS CLI](https://ibm-mas.github.io/cli/) container has been created. In the test, MAS CLI v15.6.0, was used.

1. Prepare the requirements for the installation.
- The MAS license file should be the same as the one from the external mongodb.
```
[ibmmas/cli:15.6.0]mascli$ ls /mnt/home/licenses/license.dat
/mnt/home/licenses/license.dat
```
- In this example, the configuration file for the external mongodb will be created during the installation. A copy of the mongodb certificate should be in hand and accessible in the MAS CLI container.

```
[ibmmas/cli:15.6.0]mascli$ ls /mnt/home/externalmongodemo
ca.crt
```
2. Log in to the OCP cluster and run `mas install`.


