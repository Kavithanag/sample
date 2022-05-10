# Jenkins pipeline for CEPH Deployment

Jenkins Pipeline (or simply "Pipeline") is a suite of plugins which supports implementing and integrating continuous delivery pipelines into Jenkins. A continuous delivery pipeline is an automated expression of your process for getting software from version control right through to your users and customers.

## Creating a Simple Pipeline Steps
Initial pipeline usage typically involves the following tasks:
1.	Downloading and installing the Pipeline plugin (Unless it is already part of your Jenkins installation)
2.	Creating a Pipeline of a specific type
3.	Configuring your Pipeline
4.	Controlling Flow through your Pipeline
5.	Scaling your Pipeline

###### To create a simple pipeline from the Jenkins interface, perform the following steps:
1.	Click **New Item** on your Jenkins home page, enter a name for your (pipeline) job, select **Pipeline**, and click **OK**.
2.	In the Script text area of the configuration screen, enter your pipeline syntax. If you are new to pipeline creation, you might want to start by opening Snippet Generator and selecting the “Hello Word” snippet. **Note:** Pipelines are written as Groovy scripts that tell Jenkins what to do when they are run, but because relevant bits of syntax are introduced as needed, you do not need deep expertise in Groovy to create them, although basic understanding of Groovy is helpful.
3.	Click **Save**.
4.	Click Build **Now** to create the pipeline.
5.	Click **▾** and select **Console Output** to see the output.

The steps of the CEPH_Deployment_ver1 are taken for the Jenkins pipeline. Step no refer to the step no of CEPH_Deployment_ver1 document.

Jekins pipeline automated the manual steps in CEPH_Deployment_ver1.

## Prerequisite:
  Deployment Node: User should decide the deployment node. In this case deployment node is sp-dev-infra1 
  node ( Can be any Infra node)
  Copy the MAAS Private key into Infra 1 for keylessssh.
  Deployment version required in SP-dev Lab is Octopus version of CEPH

## Stages:
Stage is block contains a series of steps in a pipeline.
In the Jenkins pipeline we have identified the following statges:

1.	Parameters
2.	Pre-check
3.	Update variables
4.	Clone repo
5.	Create inventory
6.	copy updated files
7.	Execute lvcreate
8.	Create lv files perhost
9.	Execute
10.	Post Check

###### Stage 1: Parameters
this stage defines the variables which are required in Jenkins pipeline.We have analysed the document and came with many parameters. We have identified following parameters:
deployment_node_ip, enable_ceph_dashboard, generate_fsid, fsid, monitor_address_block, public_network, cluster_network, monitor_interface, inventory,  disks

*The parameter can be defined in Jenkins pipeline in following way*

```shell
properties(
    [
        parameters(
            [string(description: 'Please enter the IP address of the host from where Ceph deployment will be executed', name: 'deployment_node_ip')
                          )
            ]
      ]       
 )
```

The code snippet from the Jenkins file for parameter declaration

```shell
properties(
    [
        parameters(
            [string(description: 'Please enter the IP address of the host from where Ceph deployment will be executed', name: 'deployment_node_ip'),
            choice(name: 'enable_ceph_dashboard', choices: ['false', 'true'], description: 'Ceph Dashboard'),
            choice(name: 'generate_fsid', choices: ['false', 'true'], description: 'If new fsid need to generated select true'),
            string(description: 'Enter the cluster fsid, create using python -c "import uuid; print(str(uuid.uuid4()))"', name: 'fsid', defaultValue: '6fad73d1-9cf6-47c0-8ac1-227a79a8d276'),
            string(description: 'Enter the CIDR for monitor  netwrok', name: 'monitor_address_block', defaultValue: '10.246.184.0/24'),
            string(description: 'Enter the CIDR for public  netwrok', name: 'public_network', defaultValue: '10.246.184.0/24'),
            string(description: 'Enter the CIDR for cluster network', name: 'cluster_network', defaultValue: '10.246.185.0/24'),
            string(description: 'monitor interface name', name: 'monitor_interface',  defaultValue:'br-mgmt'),
            text(description: 'Paste the Ansible inventory ', name: 'inventory', defaultValue: '[all]\nsp-dev-infra[1:3]\nsp-dev-compute[1:3]\nsp-dev-storage[1:8]\n[all:vars]\nansible_python_interpreter=/usr/bin/python3\n[mons]\nsp-dev-infra[1:3]\n[mons:vars]\nansible_python_interpreter=/usr/bin/python3\n[osds]\nsp-dev-storage[1:8]\n[osds-dell]\nsp-dev-storage[1:5]\n[osds-hp]\nsp-dev-storage[6:8]\n[mgrs]\nsp-dev-infra[1:3]\n[mgrs:vars]\nansible_python_interpreter=/usr/bin/python3\n[osds:vars]\nansible_python_interpreter=/usr/bin/python3\n[osds-dell:vars]\nansible_python_interpreter=/usr/bin/python3\n[osds-hp:vars]\nansible_python_interpreter=/usr/bin/python3'),
            text(description: 'In Each line enter the osd group name and its disk specific details seprated by comma \n Example:-  \n osds-dell,journal_size,logfile,nvvme_device,hdd_device1,hdd_device2..,hdd_device \n osds-dell,journal_size,logfile,/dev/sdw,/dev/sda,/dev/sdb,/dev/sdc,/dev/sde \n osd-hp,/dev/sdw,/dev/sda,/dev/sdb,/dev/sdc...', name: 'disks', defaultValue: 'osds-dell,18000,./lv-create-dell.log,/dev/sdw,/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde,/dev/sdf,/dev/sdg,/dev/sdh,/dev/sdi,/dev/sdj,/dev/sdk,/dev/sdl,/dev/sdm,/dev/sdn,/dev/sdo,/dev/sdp,/dev/sdq,/dev/sdr,/dev/sds,/dev/sdt,/dev/sdu,/dev/sdv,/dev/sdx\nosds-hp,5500,./lv-create-hp.log,/dev/sdv,/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde,/dev/sdf,/dev/sdg,/dev/sdh,/dev/sdi,/dev/sdj,/dev/sdk,/dev/sdl,/dev/sdm,/dev/sdn,/dev/sdo,/dev/sdp,/dev/sdq,/dev/sdr,/dev/sds,/dev/sdt,/dev/sdu')
            
            ]
            
            )
    ]
    )
```
###### Stage 2: Pre-check

The Stage 2 lists information about all available or the specified block devices for all the servers in the group osds

*The code snippet from the Jenkins file for pre-check stage*

```shell
stage ('Pre-check') {
    sh """
    ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@"${deployment_node_ip}" "ansible osds -m shell -a 'lsblk'"
    
    """
    
}
```

###### Stage 3: Update variables
Stage 3 is creating host_vars directory,lv.vars.yaml from disks variable, all.yaml is updated with the parameter’s value which was declared in the first stage.

*The code snippet from the Jenkins file for Update variables stage*

```shell
stage('Update variables'){
    sh 'mkdir -p  "/var/lib/jenkins/ceph/${BUILD_NUMBER}/host_vars"'
    vardata = readYaml file: "/var/lib/jenkins/ceph/lv-vars.yaml"
    playdata = readYaml file: "/var/lib/jenkins/ceph/lv-create.yaml"
    // modify
   
    String[] str;
      line = disks.split('\n');
      
    for( String values : line ){
       items = values.split(',')
       file_name = items[0]
       vardata.journal_size = items[1]
       vardata.logfile_path = items[2]
       vardata.nvme_device = items[3]
       playdata[0].hosts = items[0]
       playdata[0].tasks[0].include_vars.file = "lv-vars-${file_name}.yaml"
       
       items.eachWithIndex { item, index ->
           println "$line" "$item"
           if (index>3){
           vardata.hdd_devices[index - 4] = "$item"
           }
       }
    writeYaml file: "/var/lib/jenkins/ceph/${BUILD_NUMBER}/lv-vars-${file_name}.yaml", data: vardata, overwrite: true
    writeYaml file: "/var/lib/jenkins/ceph/${BUILD_NUMBER}/lv-create-${file_name}.yaml", data: playdata, overwrite: true
   }
    allvar = readYaml file: "/var/lib/jenkins/ceph/all.yaml"
    allvar.dashboard_enabled = "$enable_ceph_dashboard"
    allvar.generate_fsid = "$generate_fsid"
    allvar.fsid = "$fsid"
    allvar.monitor_address_block = "$monitor_address_block"
    allvar.public_network = "$public_network"
    allvar.cluster_network = "$cluster_network"
    allvar.monitor_interface = "$monitor_interface"
    
    writeYaml file: "/var/lib/jenkins/ceph/${BUILD_NUMBER}/all.yaml", data: allvar, overwrite: true
}
```

###### Stage 4: Clone repo
This stage is cloning the ceph ansible repo by executing the deployment.sh in the deployment node.

*The code snippet from the Jenkins file for Clone repo stage*

```shell
stage('Clone repo') {
    sh """
       sleep 3600
       scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no  "/var/lib/jenkins/ceph/deployment.sh" root@"${deployment_node_ip}":/tmp/deployment.sh
       ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@"${deployment_node_ip}" 'chmod +x /tmp/deployment.sh;/tmp/deployment.sh'
    """
}
```

###### Stage 5: Create inventory
This stage creates the inventory file in the Jenkins server by using ‘inventory’ variable which was declared in parameter section.

*The code snippet from the Jenkins file for Create inventory stage*

```shell
stage('Create inventory'){
    sh 'echo "${inventory}" > "/var/lib/jenkins/ceph/${BUILD_NUMBER}/hosts-${BUILD_NUMBER}"'
}
```

###### Stage 6: copy updated files
This stage copies all the updated files from /var/lib/jenkins/ceph/ directory to deployment host in the respective location for deployment.

*The code snippet from the Jenkins file for copy updated stage*

```shell
stage ('copy updated files'){
    sh """
    scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no "/var/lib/jenkins/ceph/${BUILD_NUMBER}/hosts-${BUILD_NUMBER}" root@"${deployment_node_ip}":/etc/ansible/hosts
    scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no  /var/lib/jenkins/ceph/${BUILD_NUMBER}/lv-vars-*.yaml root@"${deployment_node_ip}":/usr/share/ceph-ansible/infrastructure-playbooks/vars/
    scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no  /var/lib/jenkins/ceph/${BUILD_NUMBER}/lv-create-*.yaml root@"${deployment_node_ip}":/usr/share/ceph-ansible/infrastructure-playbooks/
    scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no  '/var/lib/jenkins/ceph/${BUILD_NUMBER}/all.yaml' root@"${deployment_node_ip}":/etc/ansible/group_vars/all.yml
    """
}
```

###### Stage 7: Execute lvcreate
In this stage  lvcreate is achieved in Jenkins pipeline by splitting the disks variable and assigned to item variable and lvcreate is executed for both the server groups (HP and Dell).

The code snippet from the Jenkins file for execute lvcreate stage

```shell
stage('Execute lvcreate'){
        String[] str;
      line = disks.split('\n');
      
    for( String values : line ){
       items = values.split(',')
       file_name = items[0]
       sh """
       ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@"${deployment_node_ip}" "cd /usr/share/ceph-ansible/infrastructure-playbooks/;echo "Executing lvcreate for Hostgroup ${file_name}" ;ansible-playbook lv-create-${file_name}.yaml"
       """
}
}
```

###### Stage 8: Create lv files perhost
lv files for each host: the disk variable is created with the default value and then disk variable is split based on ‘\n’ character to divide the server groups. Then each divide group is assigned to line variable to traverse. Item is assigned with line and item[0] is assigned to filename and item[2] is assigned to logfile_path.  Using these variables lv files created for each host.


*The code snippet from the Jenkins file for execute lvcreate stage*

```shell
stage('Create lv files perhost'){
    
    String[] str;
      line = disks.split('\n');
      
    for( String values : line ){
       items = values.split(',')
       file_name = items[0]
       logfile_path = items[2]
       sh """
    mkdir -p  "/var/lib/jenkins/ceph/${BUILD_NUMBER}/host_vars"
    ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@"${deployment_node_ip}"  "ansible "${file_name}" --list-hosts   | tail -n +2 |sed 's/^ *//g'" > /var/lib/jenkins/ceph/"${BUILD_NUMBER}"/"${file_name}"_hosts
    scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no -pr root@${deployment_node_ip}:/usr/share/ceph-ansible/infrastructure-playbooks/${logfile_path} /var/lib/jenkins/ceph/${BUILD_NUMBER}/
    for value in `cat /var/lib/jenkins/ceph/${BUILD_NUMBER}/"${file_name}_hosts"`;
        do 
        cp /var/lib/jenkins/ceph/osds.yaml  /var/lib/jenkins/ceph/${BUILD_NUMBER}/host_vars/\$value
        cat /var/lib/jenkins/ceph/${BUILD_NUMBER}/${logfile_path} |tail +3 >> /var/lib/jenkins/ceph/${BUILD_NUMBER}/host_vars/\$value
        done
    
    scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no -pr /var/lib/jenkins/ceph/${BUILD_NUMBER}/host_vars root@${deployment_node_ip}:/etc/ansible/
    """
 
    }
```

###### Stage 9: Execute
In this stage site.yaml  sample file is copied to site.yaml file and then copied and excecuted in deployment node

*The code snippet from the Jenkins file for Execute stage*

```shell
stage('Execute ')
    sh """
    ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@"${deployment_node_ip}" "cd /usr/share/ceph-ansible/; cp site.yml.sample site.yml; ansible-playbook site.yml"
    """
    
}
```

###### Stage 10: Post Check
In this stage after the playbook execution is completed ‘post check’ is done in Jenkins pipeline by running the command “ceph -s” in deployment node.

*The code snippet from the Jenkins file for Post Check stage*

```shell
stage('Post Check'){
    ssh """
    ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@"${deployment_node_ip}" "ceph -s"
    """

}
```

**Jenkins Pipeline Execution Output**

we have run this Jenkins pipeline script in the NIC lab environment. Pipeline execution of ceph deployment is successful.

We can find the console output for each stages as below:

###### stage 'Pre-check'

```shell
[Pipeline] properties
[Pipeline] stage
[Pipeline] { (Pre-check)
[Pipeline] sh
+ ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@sp-dev-infra1 ansible osds -m shell -a 'lsblk'
[WARNING]: Invalid characters were found in group names but not replaced, use
-vvvv to see details
sp-dev-storage1 | CHANGED | rc=0 >>
NAME              MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0               7:0    0  67.8M  1 loop /snap/lxd/22753
loop1               7:1    0  61.9M  1 loop /snap/core20/1405
loop2               7:2    0  43.6M  1 loop /snap/snapd/15177
sda                 8:0    0   1.7T  0 disk 
├─sda1              8:1    0   512M  0 part /boot/efi
└─sda2              8:2    0   1.7T  0 part 
  └─vgroot-lvroot 253:0    0   1.7T  0 lvm  /
sdb                 8:16   0   1.7T  0 disk 
sdc                 8:32   0   1.7T  0 disk 
sdd                 8:48   0   1.7T  0 disk 
sde                 8:64   0   1.7T  0 disk 
sdf                 8:80   0   1.7T  0 disk 
sdg                 8:96   0   1.7T  0 disk 
sdh                 8:112  0   1.7T  0 disk 
sdi                 8:128  0   1.7T  0 disk 
sdj                 8:144  0   1.7T  0 disk 
sdk                 8:160  0   1.7T  0 disk 
sdl                 8:176  0   1.7T  0 disk 
sdm                 8:192  0   1.7T  0 disk 
sdn                 8:208  0   1.7T  0 disk 
sdo                 8:224  0   1.7T  0 disk 
sdp                 8:240  0   1.7T  0 disk 
sdq                65:0    0   1.7T  0 disk 
sdr                65:16   0   1.7T  0 disk 
sds                65:32   0   1.7T  0 disk 
sdt                65:48   0   1.7T  0 disk 
sdu                65:64   0   1.7T  0 disk 
sdv                65:80   0   1.7T  0 disk 
sdw                65:96   0 447.1G  0 disk 
sdx                65:112  0   1.7T  0 disk 
sp-dev-storage5 | CHANGED | rc=0 >>
NAME              MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0               7:0    0  43.6M  1 loop /snap/snapd/15177
loop1               7:1    0  67.8M  1 loop /snap/lxd/22753
loop2               7:2    0  61.9M  1 loop /snap/core20/1405
sda                 8:0    0   1.7T  0 disk 
├─sda1              8:1    0   512M  0 part /boot/efi
└─sda2              8:2    0   1.7T  0 part 
  └─vgroot-lvroot 253:0    0   1.7T  0 lvm  /
sdb                 8:16   0   1.7T  0 disk 
sdc                 8:32   0   1.7T  0 disk 
sdd                 8:48   0   1.7T  0 disk 
sde                 8:64   0   1.7T  0 disk 
sdf                 8:80   0   1.7T  0 disk 
sdg                 8:96   0   1.7T  0 disk 
sdh                 8:112  0   1.7T  0 disk 
sdi                 8:128  0   1.7T  0 disk 
sdj                 8:144  0   1.7T  0 disk 
sdk                 8:160  0   1.7T  0 disk 
sdl                 8:176  0   1.7T  0 disk 
sdm                 8:192  0   1.7T  0 disk 
sdn                 8:208  0   1.7T  0 disk 
sdo                 8:224  0   1.7T  0 disk 
sdp                 8:240  0   1.7T  0 disk 
sdq                65:0    0   1.7T  0 disk 
sdr                65:16   0   1.7T  0 disk 
sds                65:32   0   1.7T  0 disk 
sdt                65:48   0   1.7T  0 disk 
sdu                65:64   0   1.7T  0 disk 
sdv                65:80   0   1.7T  0 disk 
sdw                65:96   0 447.1G  0 disk 
sdx                65:112  0   1.7T  0 disk 
```

###### Stage 'Update variables'

```shell
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Update variables)
[Pipeline] sh
+ mkdir -p /var/lib/jenkins/ceph/33/host_vars
[Pipeline] readYaml
[Pipeline] readYaml
[Pipeline] writeYaml
[Pipeline] writeYaml
[Pipeline] writeYaml
[Pipeline] writeYaml
[Pipeline] readYaml 
[Pipeline] writeYaml
[Pipeline] }

###### Stage 'Clone repo'

```shell
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Clone repo)
[Pipeline] sh
+ scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no /var/lib/jenkins/ceph/deployment.sh root@sp-dev-infra1:/tmp/deployment.sh
+ ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@sp-dev-infra1 chmod +x /tmp/deployment.sh;/tmp/deployment.sh

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Reading package lists...
Building dependency tree...
Reading state information...
python is already the newest version (2.7.15~rc1-1).
python-dev is already the newest version (2.7.15~rc1-1).
git is already the newest version (1:2.17.1-1ubuntu0.11).
The following packages were automatically installed and are no longer required:
  bridge-utils cephadm containerd docker.io fonts-lyx ibverbs-providers
  javascript-common libaio1 libbabeltrace1 libblas3 libdw1 libgfortran4
  libgoogle-perftools4 libibverbs1 libjbig0 libjpeg-turbo8 libjpeg8
  libjs-jquery libjs-jquery-ui liblapack3 liblcms2-2 libleveldb1v5
  liblttng-ust-ctl4 liblttng-ust0 libnl-route-3-200 liboath0 librabbitmq4
  librdkafka1 librdmacm1 libsnappy1v5 libtcmalloc-minimal4 libtiff5 liburcu6
  libwebp6 libwebpdemux2 libwebpmux3 nvme-cli pigz python-attr python-funcsigs
  python-matplotlib-data python-pastedeploy-tpl python-pluggy python-py
  python-pytest python3-bcrypt python3-bs4 python3-cherrypy3 python3-cycler
  python3-dateutil python3-decorator python3-fasteners python3-html5lib
  python3-joblib python3-kubernetes python3-logutils python3-lxml python3-mako
  python3-matplotlib python3-monotonic python3-nose python3-numpy
  python3-oauth2client python3-olefile python3-paste python3-pastedeploy
  python3-pastescript python3-pecan python3-pil python3-pluggy
  python3-prettytable python3-py python3-pyinotify python3-pyparsing
  python3-pytest python3-repoze.lru python3-routes python3-rsa python3-scipy
  python3-simplegeneric python3-simplejson python3-singledispatch
  python3-sklearn python3-sklearn-lib python3-sqlalchemy python3-tempita
  python3-tz python3-uritemplate python3-waitress python3-webencodings
  python3-webob python3-websocket python3-webtest python3-werkzeug runc
  smartmontools ubuntu-fan
```

###### Stage 'Create inventory'

```shell
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Create inventory)
[Pipeline] sh
+ echo [all]
sp-dev-infra[1:3]
sp-dev-compute[1:3]
sp-dev-storage[1:8]
[all:vars]
ansible_python_interpreter=/usr/bin/python3
[mons]
sp-dev-infra[1:3]
[mons:vars]
ansible_python_interpreter=/usr/bin/python3
[osds]
sp-dev-storage[1:8]
[osds-dell]
sp-dev-storage[1:5]
[osds-hp]
sp-dev-storage[6:8]
[mgrs]
sp-dev-infra[1:3]
[mgrs:vars]
ansible_python_interpreter=/usr/bin/python3
[osds:vars]
ansible_python_interpreter=/usr/bin/python3
[osds-dell:vars]
ansible_python_interpreter=/usr/bin/python3
[osds-hp:vars]
ansible_python_interpreter=/usr/bin/python3
[Pipeline] }
```

###### stage 'copy updated files'

```shell
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (copy updated files)
[Pipeline] sh
+ scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no /var/lib/jenkins/ceph/33/hosts-33 root@sp-dev-infra1:/etc/ansible/hosts
+ scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no /var/lib/jenkins/ceph/33/lv-vars-osds-dell.yaml /var/lib/jenkins/ceph/33/lv-vars-osds-hp.yaml root@sp-dev-infra1:/usr/share/ceph-ansible/infrastructure-playbooks/vars/
+ scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no /var/lib/jenkins/ceph/33/lv-create-osds-dell.yaml /var/lib/jenkins/ceph/33/lv-create-osds-hp.yaml root@sp-dev-infra1:/usr/share/ceph-ansible/infrastructure-playbooks/
+ scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no /var/lib/jenkins/ceph/33/all.yaml root@sp-dev-infra1:/etc/ansible/group_vars/all.yml
[Pipeline] }
```

###### Stage 'Execute lvcreate'

```shell
[Pipeline] stage
[Pipeline] { (Execute lvcreate)
[Pipeline] sh
+ ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@sp-dev-infra1 cd /usr/share/ceph-ansible/infrastructure-playbooks/;echo Executing lvcreate for Hostgroup osds-dell ;ansible-playbook lv-create-osds-dell.yaml
Executing lvcreate for Hostgroup osds-dell
[WARNING]: Invalid characters were found in group names but not replaced, use
-vvvv to see details

PLAY [creates logical volumes for the bucket index or fs journals on a single device.] ***

TASK [Gathering Facts] *********************************************************
ok: [sp-dev-storage3]
ok: [sp-dev-storage2]
ok: [sp-dev-storage4]
ok: [sp-dev-storage5]
ok: [sp-dev-storage1]

TASK [include vars of lv_vars.yaml] ********************************************
ok: [sp-dev-storage1]
ok: [sp-dev-storage2]
ok: [sp-dev-storage3]
ok: [sp-dev-storage4]
ok: [sp-dev-storage5]

TASK [fail if nvme_device is not defined] **************************************
skipping: [sp-dev-storage1]
skipping: [sp-dev-storage2]
skipping: [sp-dev-storage3]
skipping: [sp-dev-storage4]
skipping: [sp-dev-storage5]

TASK [install lvm2] ************************************************************
ok: [sp-dev-storage5]
ok: [sp-dev-storage4]
ok: [sp-dev-storage1]
ok: [sp-dev-storage3]
ok: [sp-dev-storage2]
```

###### Stage 'Create lv files perhost'

```shell
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Create lv files perhost)
[Pipeline] sh
+ mkdir -p /var/lib/jenkins/ceph/33/host_vars
+ ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@sp-dev-infra1 ansible osds-dell --list-hosts   | tail -n +2 |sed 's/^ *//g'
[WARNING]: Invalid characters were found in group names but not replaced, use
-vvvv to see details
+ scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no -pr root@sp-dev-infra1:/usr/share/ceph-ansible/infrastructure-playbooks/./lv-create-dell.log /var/lib/jenkins/ceph/33/
+ cat /var/lib/jenkins/ceph/33/osds-dell_hosts
+ cp /var/lib/jenkins/ceph/osds.yaml /var/lib/jenkins/ceph/33/host_vars/sp-dev-storage1
+ cat /var/lib/jenkins/ceph/33/./lv-create-dell.log
+ tail +3
+ cp /var/lib/jenkins/ceph/osds.yaml /var/lib/jenkins/ceph/33/host_vars/sp-dev-storage2
+ cat /var/lib/jenkins/ceph/33/./lv-create-dell.log
+ tail +3
+ cp /var/lib/jenkins/ceph/osds.yaml /var/lib/jenkins/ceph/33/host_vars/sp-dev-storage3
+ cat /var/lib/jenkins/ceph/33/./lv-create-dell.log
+ tail +3
+ cp /var/lib/jenkins/ceph/osds.yaml /var/lib/jenkins/ceph/33/host_vars/sp-dev-storage4
+ cat /var/lib/jenkins/ceph/33/./lv-create-dell.log
+ tail +3
+ cp /var/lib/jenkins/ceph/osds.yaml /var/lib/jenkins/ceph/33/host_vars/sp-dev-storage5
+ cat /var/lib/jenkins/ceph/33/./lv-create-dell.log
+ tail +3
+ scp -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no -pr /var/lib/jenkins/ceph/33/host_vars root@sp-dev-infra1:/etc/ansible/
```

###### Stage ' Execute '

```shell
[Pipeline] stage
[Pipeline] { (Execute )
[Pipeline] sh
+ ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@sp-dev-infra1 cd /usr/share/ceph-ansible/; cp site.yml.sample site.yml; ansible-playbook site.yml
[WARNING]: Could not match supplied host pattern, ignoring: mdss
[WARNING]: Could not match supplied host pattern, ignoring: rgws
[WARNING]: Could not match supplied host pattern, ignoring: nfss
[WARNING]: Could not match supplied host pattern, ignoring: rbdmirrors
[WARNING]: Could not match supplied host pattern, ignoring: clients
[WARNING]: Could not match supplied host pattern, ignoring: iscsigws
[WARNING]: Could not match supplied host pattern, ignoring: iscsi-gws
[WARNING]: Could not match supplied host pattern, ignoring: grafana-server
[WARNING]: Could not match supplied host pattern, ignoring: rgwloadbalancers
```

###### Stage 'Post Check'

```shell
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Post Check)
[Pipeline] sh
+ ssh -i /var/lib/jenkins/id_rsa -o StrictHostKeyChecking=no root@sp-dev-infra1 ceph -s
  cluster:
    id:     6fad73d1-9cf6-47c0-8ac1-227a79a8d276
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            clock skew detected on mon.sp-dev-infra2, mon.sp-dev-infra3
            1 pgs not deep-scrubbed in time
            1 pgs not scrubbed in time
 
  services:
    mon: 3 daemons, quorum sp-dev-infra1,sp-dev-infra2,sp-dev-infra3 (age 7m)
    mgr: sp-dev-infra2(active, since 6m), standbys: sp-dev-infra3, sp-dev-infra1
    osd: 178 osds: 178 up (since 67s), 178 in (since 67s)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   183 GiB used, 249 TiB / 249 TiB avail
    pgs:     1 active+clean
 
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```










