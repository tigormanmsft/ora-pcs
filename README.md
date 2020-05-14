# ora-pcs
Azure CLI bash script to automatically configure a Pacemaker/Corosync (PCS) cluster for an Oracle Standard Edition database

## Description
This Azure CLI bash script fully automates the creation an Oracle database in an HA-LVM cluster on two Azure VMs, using Azure shared disk as the database storage.  Linux HA-LVM on Oracle Linux and Red Hat uses open-source Pacemaker and Corosync to manage the HA-LVM cluster.  The cluster is set up so that only one VM is active with full access to the Oracle database and listener.  All Oracle services can be failed over to the second VM using the HA-LVM cluster.

The "cr_orapcs.sh" bash script automates the following steps...

 1. verify that subscription and resource group exist and are accessible
 2. set defaults for resource group and location
 3. create storage account
 4. create vnet, subnet, network security group with rules
 5. create proximity placement group (PPG) for the two VMs
 6. create Azure public IP objects for the two VMs
 7. create the NIC, VM, and shared disk attached for the first VM
 8. create the NIC, VM, and shared disk attached for the second VM
 9. display the public IP addresses for both VMs for later use
10. On both VMs, do the following steps...
    a. copy Oracle "oraInst.loc" file from inventory location to "/etc" directory
    b. use "yum" to install the LVM2 package
    c. create directory mountpoint "/u02"
11. On the first VM only, do the following tasks...
    a. partition the shared disk
    b. make a physical volume from the partition
    c. create a volume group from the physical partition
    d. create logical volume within the volume group
    e. create an EXT4 filesystem within the logical volume
    f. mount the filesystem on "/u02"
    g. create subdirectories within "/u02" for Oracle database/configuration files
    h. use the Oracle Database Creation Assistant to create a database and a TNS listener
    i. create a service account for PCS within the database
    j. shutdown the Oracle database and stop the TNS Listener
    k. copy the Oracle PWDFILE and SPFILE to the shared disk and create symlinks in their place
    l. edit the Oracle TNS sqlnet.ora, listener.ora, and tnsnames.ora configuration files
       to replace the IP hostname of the first VM with the virtual IP address
    m. copy the Oracle TNS configuration files to the shared disk, and create symlinks in their place
    n. unmount the filesystem on "/u02"
12. On the second VM only, do the following tasks...
    a. create an entry in the "/etc/oratab" configuration file
    b. create adump and dpdump subsdirectories
13. On both VMs, do the following steps...
    a. use "yum" to install the PCS package and start/enable the PCSD daemon
    b. set a password for the PCS account used for remote access
14. On the first VM only, do the following steps...
    a. create and start the PCS cluster, then enable it for automatic restart on node reboot
    b. set cluster properties to disable STONITH and disable QUORUM for 2-node HA operation
15. On both VMs, disable LVMETAD daemon and reboot the VM
16. On the first VM only, create the following PCS resources within a resource group...
    a. virtual IP
    b. volume group
    c. filesystem
    d. database
    e. listener

## Diagnostics and output
The "orapcs_output.txt" file contains an example of the output generated by the bash script.  Additionally, the script also saves stdout and stderr output from all commands to a ".log" file in the present working directory, for diagnostic purposes.  If anything fails, it is wise to look in the ".log" file for more information.

## Call syntax
Command-line Parameters:

Usage: ./cr_orapcs.sh -G val -N -M -O val -P val -S val -V val -d val -i val -p val -r val -s val -u val -v

	-G resource=group-name  name of the Azure resource group (default: ${_azureOwner}-${_azureProject}-rg)
	-N                      skip steps to create vnet/subnet, public-IP, NSG, rules, and PPG
	-M                      skip steps to create VMs and storage
	-O owner-tag            name of the owner to use in Azure tags (no default)
	-P project-tag          name of the project to use in Azure tags (no default)
	-S subscription         name of the Azure subscription (no default)
	-V vip-IPaddr		IP address for the virtual IP (VIP) (default: 10.0.0.10)
	-d domain-name          IP domain name (default: internal.cloudapp.net)
	-i instance-type        name of the Azure VM instance type for database nodes (default: Standard_DS11-1_v2)
	-p Oracle-port          port number of the Oracle TNS Listener (default: 1521)
	-r region               name of Azure region (default: westus2)
	-s ORACLE_SID           Oracle System ID (SID) value (default: oradb01)
	-u urn                  Azure URN for the VM from the marketplace (default: Oracle:Oracle-Database-Ee:12.2.0.1:12.2.20180725)
	-v                      set verbose output is true (default: false)
  
Notes on call syntax:
The "-N" and "-M" switches were mainly used for initial debugging, and might well be removed in more mature versions of the script.  They intended to skip over some steps if something failed later on.

## Testing steps

Use SSH to access each of the two VMs.  From the Azure administrative account on the VM, you can use the "sudo" utility to execute all of the PCS commands as root, or just enter "sudo su -" to open a shell as root.

The command "pcs status" (or "sudo pcs status") displays the overall status of the PCS cluster, the nodes, and each of the cluster resources.

The command "pcs cluster status" (or "sudo pcs cluster status") displays additional status information about the PCS cluster.

### Forcing failover of Oracle services, VIP, and shared disk

Using the fully-qualified IP hostname of the VM in place of the label "<vm>"...
  
     $ sudo pcs node standby <vm>

...will put the indicated "<vm>" into PCS "standby" mode, which means that the VM cannot host services.  This will force all services to failover to the remaining node.  Issue the command above, and then monitor the progress of failover using the "sudo pcs status" command.
  
To take the "<vm>" out of PCS standby mode and allow it to host services again, issue the following command...
  
     $ sudo pcs node unstandby <vm>

...and then follow up with the "sudo pcs status" command to view any changes to status, which should not happen.

To failback the Oracle services to the original VM, make the other VM (on which the services currently reside) in standby mode, and be sure to "unstandby" that node after the Oracle services have been successfully forced off it.

Of course, the purpose of the HA-LVM cluster is high-availability in the event of failure, so killing some of the services directly is another way to test, bearing in mind that the PCS cluster polls periodically for failover.  In other words, failover will not occur instantly after failure, but 30 seconds or 60 seconds later when the polling discovers the failure and then retries to verify that the failure has indeed happened.
