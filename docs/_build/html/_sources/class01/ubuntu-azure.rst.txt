Deploy App Servers
==================

#. Go to f5student#-rg resource group
#. Deploy App1

   - Click "Add"
   - Select Ubuntu Server 18.04 LTS

#. Complete the virtual machine template with the following values **(don’t follow the screen shot)**

   +------------------------+--------------------------+
   | Resource Group         | f5student#-rg            |
   +------------------------+--------------------------+
   | Virtual machine name   | f5student#-app1          |
   +------------------------+--------------------------+
   | Admin type             | Password                 |
   +------------------------+--------------------------+
   | Username               | azureuser                |
   +------------------------+--------------------------+
   | Password               | ChangeMeNow123!          |
   +------------------------+--------------------------+
   | Public inbound ports   | none                     |
   +------------------------+--------------------------+

   .. image:: ./images/arm3nic.png

#. Click the “Next : Disks”
#. Click “Create” after confirming entries
#. This will take appoximately 10 minutes

   - You can monitor deployment on the azure dashboard by opening the Notifications in the azure portal

#. Continue with the Lab. The deployment will complete by the time the BIG-IP configuration is required


