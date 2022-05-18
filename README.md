# Jenkins pipeline for Openstack Deployment

Jenkins Pipeline (or simply "Pipeline") is a suite of plugins which supports implementing and integrating continuous delivery pipelines into Jenkins. A continuous delivery pipeline is an automated expression of your process for getting software from version control right through to your users and customers.
The steps of the Openstack_Deployment.docx are taken for the Jenkins pipeline. 

## Prerequisite:
	> Install OS  on all Hosts and do networking and check all servers are able to access from terminal. Ensure all Hosts should be able to ping IP from 
      each host to every other hosts. 
    > Add below parameter in all servers for add time & date in history for troubleshooting 
    > <Deployment node># echo "export HISTTIMEFORMAT='%F %T '" >> /etc/profile 
    > <Deployment node># echo "export HISTTIMEFORMAT='%F %T '" >> ~/.bash_profile 
    > Make all the servers passwordless from Deployment Node. 


## Stages:
Stage is block contains a series of steps in a pipeline.
In the Jenkins pipeline we have identified the following statges for openstack deployment:

1)	Parameters
2)	Update Yamls
3)	Colne the repository
4)	Anisble_bootstraping
5)	Copy etc directory
6)	Move Updated yamls to deployment node
7)	Execute OSA playbooks
8)	Copy Monitoring role


###### Stage 1: Parameters
This stage defines the variables which are required in Jenkins pipeline.We have analysed the document and came with many parameters. We have identified following parameters:
deployment_node_ip, OSA_repo, OSA_branch, container_cidr, storage_cidr, tunnel_cidr, used_ip_container, used_ip_storage, used_ip_tunnel, internal_lb_vip_address, external_lb_vip_address, name_infra_one, ip_infra_one, name_infra_two, ip_infra_two, name_infra_three, ip_infra_three, name_compute_one, ip_compute_one, name_compute_two, ip_compute_two, name_compute_three, ip_compute_three, proxy_env_url, lxc_hosts_container_image_url, security_ntp_servers, haproxy_use_keepalived, haproxy_ssl, openstack_service_publicuri_proto, keepalived_ping_address, mononeip, montwoip, monthreeip, keystone_web_server, entity_ids, sp_oidc_provider_metadata_url, sp_oidc_client_id, sp_oidc_client_secret, generate_fsid.


*The parameter can be defined in Jenkins pipeline in following way*
properties(
    [
        parameters(
            [string(description: 'Please enter the IP address of the host from where Openstack deployment will be executed', name: 'deployment_node_ip')
                          )
            ]
      ]       

 )


*The code snippet from the Jenkins file for parameter declaration*

```shell
properties(
    [
        parameters(
            [string(description: 'Please enter the IP address of the host from where OSA deployment will be executed', name: 'deployment_node_ip'),
            string(description: 'OSA repository ', defaultValue: 'https://github.com/openstack/openstack-ansible.git', name: 'OSA_repo'),
            string(description: 'OSA branch. which defines the version of Openstack to be installed.', defaultValue: 'stable/train', name: 'OSA_branch'),
            string(description: 'Enter the CIDR for container  netwrok', name: 'container_cidr'),
            string(description: 'Enter the CIDR for storage  netwrok', name: 'storage_cidr'),
            string(description: 'Enter the CIDR for tunnel  netwrok', name: 'tunnel_cidr'),
            string(description: 'Enter the used IP range to avoid its assignment in container cidr for OS infra, Example: "192.168.0.10,192.168.0.30"',     name: 'used_ip_container'),
            string(description: 'Enter the used IP range to avoid its assignment in storage cidr for OS infra, Example: "192.168.0.10,192.168.0.30"', name: 'used_ip_storage'),
            string(description: 'Enter the used IP range to avoid its assignment in tunnel cidr for OS infra, Example: "192.168.0.10,192.168.0.30"', name: 'used_ip_tunnel'),
            string(description: 'internal_lb_vip_address', name: 'internal_lb_vip_address'),
            string(description: 'external_lb_vip_address', name: 'external_lb_vip_address'),
            string(description: 'Hostname of Infra host 1 ', name: 'name_infra_one'),
            string(description: 'IP address of Infra host 1', name: 'ip_infra_one'),
            string(description: 'Hostname of Infra host 2 ', name: 'name_infra_two'),
            string(description: 'IP address of Infra host 2 ', name: 'ip_infra_two'),
            string(description: 'Hostname of Infra host 3', name: 'name_infra_three'),
            string(description: 'IP address of Infra host 3 ', name: 'ip_infra_three'),
            string(description: 'Hostname of Compute host 1 ', name: 'name_compute_one'),
            string(description: 'IP address of Compute host 1', name: 'ip_compute_one'),
            string(description: 'Hostname of Compute host 2', name: 'name_compute_two'),
            string(description: 'IP address of Compute host 2', name: 'ip_compute_two'),
            string(description: 'Hostname of Compute host 3', name: 'name_compute_three'),
            string(description: 'IP address of Compute host 3', name: 'ip_compute_three'),
            string(description: 'proxy_env_url', defaultValue: 'http://10-246-183-0--25.maas-internal:8000/', name: 'proxy_env_url'),
            string(description: 'lxc_hosts_container_image_url', defaultValue: 'http://cdimage.ubuntu.com/ubuntu-base/releases/18.04/release/ubuntu-base-18.04.5-base-amd64.tar.gz', name: 'lxc_hosts_container_image_url'),
            string(description: 'security_ntp_servers', defaultValue: '164.100.255.1', name: 'security_ntp_servers'),
            string(description: 'Need to user haproxy_use_keepalived, mark it true or false', defaultValue: 'True', name: 'haproxy_use_keepalived'),
            string(description: 'Need to use SSL for haproxy_ssl, mark it True of false', defaultValue: 'False', name: 'haproxy_ssl'),
            string(description: 'openstack_service_publicuri_proto, http or https', defaultValue: 'http', name: 'openstack_service_publicuri_proto'),
            string(description: 'keepalived_ping_address' , name: 'keepalived_ping_address'),
            string(description: 'First ceph Mon IP' , name: 'mononeip'),
            string(description: 'Second Ceph Mon IP' , name: 'montwoip'),
            string(description: 'Third Ceph Mon IP ' , name: 'monthreeip'),
            string(description: 'keystone_web_server: apahce or nginx, when using OIDC it should be defaul apache', defaultValue: 'apache', name: 'keystone_web_server'),
            string(description: 'entity_ids link: Example: https://xyz.io/auth/realms/cloud', name: 'entity_ids'),
            string(description: 'Provide the correct URL as per environment', defaultValue: 'https://xxx/auth/realms/cloud/.well-known/openid-configuration', name: 'sp_oidc_provider_metadata_url'),
            string(description: 'Update the correct OIDC client ID, you can get this from IDP', name: 'sp_oidc_client_id'),
            password(description: 'Input the secret for user defined in sp_oidc_client_id', name: 'sp_oidc_client_secret')]
            )

    ]
    ) 
```
  
###### Stage 2: Update Yamls
The Stage 2 update the yaml files by taking the input value from the paramterers. All the values required are updated in the yaml file.

*The code snippet from the Jenkins file for update Yamls stage*
  
```shell 
stage('Update Yamls') {
readdata = readYaml file: "/var/lib/jenkins/openstack_user_config.yml"

// modify
readdata.cidr_networks.container = "$container_cidr"
readdata.cidr_networks.storage = "$storage_cidr"
readdata.cidr_networks.tunnel = "$tunnel_cidr"
//readdata.used_ips[0] = "\"" + "$used_ip_container" + "\""
readdata.used_ips[0] = "$used_ip_container"
//readdata.used_ips[1] = "\"" + "$used_ip_storage" + "\""
readdata.used_ips[1] = "$used_ip_storage"
//readdata.used_ips[2] = "\"" + "$used_ip_tunnel" + "\""
readdata.used_ips[2] = "$used_ip_tunnel"
readdata.global_overrides.internal_lb_vip_address = "$internal_lb_vip_address"
readdata.global_overrides.external_lb_vip_address = "$external_lb_vip_address"

writeYaml file: "/var/lib/jenkins/openstack_user_config-${BUILD_NUMBER}.yml", data: readdata, overwrite: true

sh '''
   sed -i "s/^  infra1/  \${name_infra_one}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/infra1ip/\${ip_infra_one}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/^  infra2/  \${name_infra_two}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/infra2ip/\${ip_infra_two}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/^  infra3/  \${name_infra_three}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/infra3ip/\${ip_infra_three}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/^  compute1/  \${name_compute_one}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/compute1ip/\${ip_compute_one}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/^  compute2/  \${name_compute_two}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/compute2ip/\${ip_compute_two}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/^  compute3/  \${name_compute_three}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
   sed -i "s/compute3ip/\${ip_compute_three}/" /var/lib/jenkins/openstack_user_config-"${BUILD_NUMBER}".yml
'''

readdata = readYaml file: "/var/lib/jenkins/user_variable.yml"

readdata.proxy_env_url = "$proxy_env_url"
readdata.haproxy_use_keepalived = "$haproxy_use_keepalived"
readdata.openstack_service_publicuri_proto ="$openstack_service_publicuri_proto"
readdata.haproxy_ssl = "$haproxy_ssl"

writeYaml file: "/var/lib/jenkins/user_variable-${BUILD_NUMBER}.yml", data: readdata, overwrite: true

sh '''
    sed -i "s/ntpserver/\${security_ntp_servers}/" /var/lib/jenkins/user_variable-"${BUILD_NUMBER}".yml
    sed -i "s/mon1/\${mononeip}/" /var/lib/jenkins/user_variable-"${BUILD_NUMBER}".yml
    sed -i "s/mon2/\${montwoip}/" /var/lib/jenkins/user_variable-"${BUILD_NUMBER}".yml
    sed -i "s/mon3/\${monthreeip}/" /var/lib/jenkins/user_variable-"${BUILD_NUMBER}".yml
'''

sh '''
   cp /var/lib/jenkins/sso_variable.yml /var/lib/jenkins/sso_variable-${BUILD_NUMBER}.yml
   sed -i "s,metadata_url_,\${sp_oidc_provider_metadata_url}," /var/lib/jenkins/sso_variable-"${BUILD_NUMBER}".yml
   sed -i "s,client_id_,\${sp_oidc_client_id}," /var/lib/jenkins/sso_variable-"${BUILD_NUMBER}".yml
   sed -i "s,client_secret_,\${sp_oidc_client_secret}," /var/lib/jenkins/sso_variable-"${BUILD_NUMBER}".yml
   sed -i "s,entity_id_url,\${entity_ids}," /var/lib/jenkins/sso_variable-"${BUILD_NUMBER}".yml
   
''' 

}
```
  
###### Stage 3: Colne the repository

This stage is cloning the openstack-ansible in /opt directory in deployment node.

*The code snippet from the Jenkins file for Clone the repository stage*


```shell
stage('Colne the repository') {

    println "Passed deployment IP : ${deployment_node_ip}"
sh '''
    ssh -i /var/lib/jenkins/id_rsa root@"${deployment_node_ip}" "rm -rf /opt/openstack-ansible" 
    ssh -i /var/lib/jenkins/id_rsa root@"${deployment_node_ip}" "git clone -b ${OSA_branch} ${OSA_repo} /opt/openstack-ansible" 
      
'''
    
}
```
  
###### Stage 4: Anisble_bootstraping

This stage will run the bootstrap-ansible.sh script which is available in /opt/openstack-ansible/scripts folder.

*The code snippet from the Jenkins file for Anisble_bootstraping stage*

```shell
stage('Anisble_bootstraping') {
 
sh '''
    ssh -i /var/lib/jenkins/id_rsa  root@"${deployment_node_ip}" "cd /opt/openstack-ansible/scripts; ./bootstrap-ansible.sh" 
 ''' 
}
```
  
###### Stage 5: Copy etc directory

This stage copy etc directory from /opt/openstack-ansible/etc/openstack_deploy to /etc of the deployment host

*The code snippet from the Jenkins file for Copy etc directory stage*

```shell
stage('Copy etc directory') {
sh  '''
   ssh -i /var/lib/jenkins/id_rsa  root@"${deployment_node_ip}" "cp -pr /opt/openstack-ansible/etc/openstack_deploy /etc/ && cd /opt/openstack-ansible && ./scripts/pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml"
'''
}
```

###### Stage 6: Move Updated yamls to deployment node

This stage copies the yaml file updated from Jenkins server to deployment host.

*The code snippet from the Jenkins file for Move Updated yamls to deployment node stage*
  
```shell
stage ('Move Updated yamls to deployment node') {

sh  """
   scp -i /var/lib/jenkins/id_rsa /var/lib/jenkins/openstack_user_config-${BUILD_NUMBER}.yml root@"${deployment_node_ip}":/etc/openstack_deploy/openstack_user_config.yml
   scp -i /var/lib/jenkins/id_rsa /var/lib/jenkins/user_variable-${BUILD_NUMBER}.yml root@"${deployment_node_ip}":/etc/openstack_deploy/user_variable.yml
   scp -i /var/lib/jenkins/id_rsa /var/lib/jenkins/sso_variable-${BUILD_NUMBER}.yml root@"${deployment_node_ip}":/etc/openstack_deploy/sso_variable.yml
   scp -i /var/lib/jenkins/id_rsa /var/lib/jenkins/grafana.yaml root@"${deployment_node_ip}":/etc/openstack_deploy/env.d/grafana.yaml
   scp -i /var/lib/jenkins/id_rsa /var/lib/jenkins/masakari.yaml root@"${deployment_node_ip}":/etc/openstack_deploy/env.d/masakari.yaml
   scp -i /var/lib/jenkins/id_rsa /var/lib/jenkins/prometheus.yaml root@"${deployment_node_ip}":/etc/openstack_deploy/env.d/prometheus.yaml
   scp -i /var/lib/jenkins/id_rsa /var/lib/jenkins/ops-exporter.yaml root@"${d eployment_node_ip}":/etc/openstack_deploy/env.d/ops-exporter.yaml
"""

}
```
  
###### Stage 7: Execute OSA playbooks

In this stage first it is  moving into /opt/openstack-ansible/playbooks and from there it is executing the playbook everything.yaml for actual deployment of openstack.

*The code snippet from the Jenkins file for Execute OSA playbooks stage*

```shell
stage ('Execute OSA playbooks') {

sh """
    ssh -i /var/lib/jenkins/id_rsa  root@"${deployment_node_ip}" "cd /opt/openstack-ansible/playbooks/ ; openstack-ansible setup-everything.yml "
"""   
}
```
  
###### Stage 8: Copy Monitoring role

This stage is copying the monitoring.tar to deployment host and then executing the monitoring_start.yml.

*The code snippet from the Jenkins file for Copy Monitoring role stage*

```shell
stage ('Copy Monitoring role'){
sh """
    scp -i /var/lib/jenkins/id_rsa /var/lib/jenkins/monitoring.tar root@"${deployment_node_ip}":/etc/ansible/role/
    ssh -i /var/lib/jenkins/id_rsa  root@"${deployment_node_ip}" "cd /etc/ansible/role ; tar -xvf monitoring.tar; cp -r monitoring/playbook/ /opt/openstack-ansible/playbooks/; cd /opt/openstack-ansible/playbooks/; openstack-ansible monitoring_install.yml && openstack-ansible monitoring_start.yml"
    
"""
    
}
```
## Jenkins Pipeline Execution Output
we have run this Jenkins pipeline script in the NIC lab environment. Pipeline execution of Openstack deployment is successful.

The parameters description :

**deployment_node_ip:**
this vaiable expects hostname or IP of the deployment node. This the node where ansible is installed for ceph deployment and other prerequisites like passwordless connectivity from deployment node other nodes is achieved.

**OSA_repo:**
This is OSA Repository link which is https://github.com/openstack/openstack-ansible.git.

**OSA_branch:**
OSA branch. which defines the version of Openstack to be installed.

**container_cidr:**
This parameter defines CIDR for container network.

**storage_cid:**
This parameter defines CIDR for storage network.

**tunnel_cidr:**
This parameter defines CIDR for tunnel network.

**used_ip_containe:**
This parameter defines used IP range to avoid its assignment in container cidr for OS infra, Example: "192.168.0.10,192.168.0.30".

**used_ip_storage:**
This parameter defines used IP range to avoid its assignment in storage cidr for OS infra, Example: "192.168.0.10,192.168.0.30".

*used_ip_tunne:*
This parameter defines used IP range to avoid its assignment in tunnel cidr for OS infra, Example: "192.168.0.10,192.168.0.30".

*internal_lb_vip_address:*
This parameter defines the internal_lb_vip address.

*external_lb_vip_address*
This parameter defines the external_lb_vip address.

*name_infra_one:*
This parameter defines host name of infra1.

*ip_infra_one:*
This parameter defines IP of infra1.

*name_infra_two:*
This parameter defines host name of infra2.

*ip_infra_two:*
This parameter defines IP of infra2.

*name_infra_three:*
This parameter defines host name of infra3.

*ip_infra_three:*
This parameter defines IP of infra3.

*name_compute_one:*
This parameter defines host name of compute1.

*ip_compute_one:*
This parameter defines IP of compute1.

*name_compute_two:*
This parameter defines host name of compute2.

*ip_compute_two:*
This parameter defines IP of compute2.

*name_compute_three:*
This parameter defines host name of compute3.

*ip_compute_three:*
This parameter defines IP of compute3.

*proxy_env_url:*
This parameter defines the proxy_env_url.

*lxc_hosts_container_image_url:*
This parameter defines lxc_host_container_image_url.

*security_ntp_servers:*
This parameter defines security_ntp_servers.

*haproxy_use_keepalived:*
Need to user haproxy_use_keepalived, mark it true or false.

*haproxy_ssl:*
Need to use SSL for haproxy_ssl, mark it True of false.

*openstack_service_publicuri_proto:*
openstack_service_publicuri_proto, http or https

*keepalived_ping_address:*
This parameter defines keepalived_ping_address.
 
*mononeip:*
This parameter defines monone IP.

*montwoip:*
This parameter defines montwo IP.

*monthreeip:*
This parameter defines monthree IP.

*keystone_web_server:*
This parameter defines keystone_web_server: apahce or nginx, when using OIDC it should be defaul apache.

*entity_ids:*
This parameter defines entity_ids of OIDC link: Example: https://xyz.io/auth/realms/cloud.

*sp_oidc_provider_metadata_url:*
This parameter defines sp_oidc_provider_metadata_url. Provide the correct URL as per environment.

*sp_oidc_client_id:*
Update the correct OIDC client ID, you can get this from IDP

*sp_oidc_client_secret:*
Input the secret for user defined in sp_oidc_client_id

*generate_fs:*
Default setup-everything.yml.

## We can find the console output for each stages as below:

###### stage ‘update yamls’

```shell
[Pipeline] { (Update Yamls)
[Pipeline] readYaml
[Pipeline] writeYaml
[Pipeline] sh
+ sed -i s/^  infra1/  sp-dev-infra1/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/infra1ip/10.246.184.12/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/^  infra2/  sp-dev-infra2/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/infra2ip/10.246.184.13/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/^  infra3/  sp-dev-infra3/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/infra3ip/10.246.184.14/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/^  compute1/  sp-dev-compute1/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/compute1ip/10.246.184.15/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/^  compute2/  sp-dev-compute2/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/compute2ip/10.246.184.16/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/^  compute3/  sp-dev-compute3/ /var/lib/jenkins/openstack_user_config-20.yml
+ sed -i s/compute3ip/10.246.184.17/ /var/lib/jenkins/openstack_user_config-20.yml
[Pipeline] readYaml
[Pipeline] writeYaml
[Pipeline] sh
+ sed -i s/ntpserver/164.100.255.1/ /var/lib/jenkins/user_variable-20.yml
+ sed -i s/mon1/10.246.184.12/ /var/lib/jenkins/user_variable-20.yml
+ sed -i s/mon2/10.246.184.13/ /var/lib/jenkins/user_variable-20.yml
+ sed -i s/mon3/10.246.184.14/ /var/lib/jenkins/user_variable-20.yml
[Pipeline] sh
+ cp /var/lib/jenkins/sso_variable.yml /var/lib/jenkins/sso_variable-20.yml
+ sed -i s,metadata_url_,https://sso.cloud.gov.in/auth/realms/test-v1/.well-known/openid-configuration, /var/lib/jenkins/sso_variable-20.yml
+ sed -i s,client_id_,os_cloud_1, /var/lib/jenkins/sso_variable-20.yml
+ sed -i s,client_secret_,C8yCHEXYNauWA49DMp01pWKnASnp27FW, /var/lib/jenkins/sso_variable-20.yml
+ sed -i s,entity_id_url,https://sso.cloud.gov.in/auth/realms/test-v1/, /var/lib/jenkins/sso_variable-20.yml
[Pipeline] }
```

###### Stage 'clone the repository'

```shell
[Pipeline] stage
[Pipeline] { (Colne the repository)
[Pipeline] echo
Passed deployment IP : sp-dev-infra2
[Pipeline] sh
+ ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa root@sp-dev-infra2 rm -rf /opt/openstack-ansible
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
+ ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa root@sp-dev-infra2 git clone -b stable/train https://github.com/openstack/openstack-ansible.git /opt/openstack-ansible
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
Cloning into '/opt/openstack-ansible'...
[Pipeline] }
```


###### Stage 'Ansible_bootstraping'

```shell
[Pipeline] { (Anisble_bootstraping)
[Pipeline] sh
+ ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa root@sp-dev-infra2 cd /opt/openstack-ansible/scripts; ./bootstrap-ansible.sh
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
+ export HTTP_PROXY=
+ HTTP_PROXY=
+ export HTTPS_PROXY=
+ HTTPS_PROXY=
+ export ANSIBLE_PACKAGE=ansible==2.8.13
+ ANSIBLE_PACKAGE=ansible==2.8.13
+ export ANSIBLE_ROLE_FILE=ansible-role-requirements.yml
+ ANSIBLE_ROLE_FILE=ansible-role-requirements.yml
+ export USER_ROLE_FILE=user-role-requirements.yml
+ USER_ROLE_FILE=user-role-requirements.yml
+ export SSH_DIR=/root/.ssh
+ SSH_DIR=/root/.ssh
+ export DEBIAN_FRONTEND=noninteractive
+ DEBIAN_FRONTEND=noninteractive
+ export SETUP_ARA=false
+ SETUP_ARA=false
+ export PIP_OPTS=
+ PIP_OPTS=
+ export OSA_WRAPPER_BIN=scripts/openstack-ansible.sh
+ OSA_WRAPPER_BIN=scripts/openstack-ansible.sh
++ dirname ./bootstrap-ansible.sh
+ cd ./..
+ info_block 'Checking for required libraries.'
+ source scripts/scripts-library.sh
++ LINE=----------------------------------------------------------------------
++ ANSIBLE_PARAMETERS=
+++ date +%s
++ STARTTIME=1652601567
++ COMMAND_LOGS=/openstack/log/ansible_cmd_logs
++ PIP_COMMAND=/opt/ansible-runtime/bin/pip
++ ZUUL_PROJECT=
++ GATE_EXIT_LOG_COPY=false
++ GATE_EXIT_LOG_GZIP=true
++ GATE_EXIT_RUN_ARA=true
++ GATE_EXIT_RUN_DSTAT=true
++ [[ -n '' ]]
++ '[' -z '' ']'
+++ grep -c '^processor' /proc/cpuinfo
++ CPU_NUM=80
++ '[' 80 -lt 10 ']'
++ ANSIBLE_FORKS=10
++ trap 'exit_fail 398 0 '\''Received STOP Signal'\''' SIGHUP SIGINT SIGTERM
++ trap 'exit_fail 399 0' ERR
+++ id -u
++ '[' 0 '!=' 0 ']'
++ '[' '!' -d etc -a '!' -d scripts -a '!' -d playbooks ']'
```

###### Stage ' Copy etc directory'

```shell
[Pipeline] stage
[Pipeline] { (Copy etc directory)
[Pipeline] sh
+ ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa root@sp-dev-infra2 cp -pr /opt/openstack-ansible/etc/openstack_deploy /etc/ && cd /opt/openstack-ansible && ./scripts/pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
Creating backup file [ /etc/openstack_deploy/user_secrets.yml.tar ]
Operation Complete, [ /etc/openstack_deploy/user_secrets.yml ] is ready
[Pipeline] }
```

###### stage 'Move updated yaml to deployment node'


```shell
[Pipeline] stage
[Pipeline] { (Move Updated yamls to deployment node)
[Pipeline] sh
+ scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa /var/lib/jenkins/openstack_user_config-20.yml root@sp-dev-infra2:/etc/openstack_deploy/openstack_user_config.yml
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
+ scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa /var/lib/jenkins/user_variable-20.yml root@sp-dev-infra2:/etc/openstack_deploy/user_variable.yml
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
+ scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa /var/lib/jenkins/sso_variable-20.yml root@sp-dev-infra2:/etc/openstack_deploy/sso_variable.yml
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
+ scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa /var/lib/jenkins/grafana.yaml root@sp-dev-infra2:/etc/openstack_deploy/env.d/grafana.yaml
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
+ scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa /var/lib/jenkins/masakari.yaml root@sp-dev-infra2:/etc/openstack_deploy/env.d/masakari.yaml
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
+ scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa /var/lib/jenkins/prometheus.yaml root@sp-dev-infra2:/etc/openstack_deploy/env.d/prometheus.yaml
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
+ scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa /var/lib/jenkins/ops-exporter.yaml root@sp-dev-infra2:/etc/openstack_deploy/env.d/ops-exporter.yaml
Failed to add the host to the list of known hosts (/var/lib/jenkins/.ssh/known_hosts).
Warning: the ECDSA host key for 'sp-dev-infra2' differs from the key for the IP address '10.246.184.13'
Offending key for IP in /var/lib/jenkins/.ssh/known_hosts:2
[Pipeline] }
```

###### Stage 'Execute OSA playbooks'

```shell
[Pipeline] stage
[Pipeline] { (Execute OSA playbooks)
[Pipeline] sh
+ echo Befor Execution of host playbooks +++++++++++++++++++++++++++++++++++++++++++++++
Befor Execution of host playbooks +++++++++++++++++++++++++++++++++++++++++++++++
+ echo Befor Execution of host openstack playbook and after infrastructure playbook +++++++++++++++++++++++++++++++++++++++++++++++
Befor Execution of host openstack playbook and after infrastructure playbook +++++++++++++++++++++++++++++++++++++++++++++++
+ ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa root@sp-dev-infra2 cd /opt/openstack-ansible/playbooks/;ansible sp-dev-infra[1..3]-host_containers -m shell -a "ping -c1 10.246.184.1"
------------------------------------------------------------------------------
* WARNING                                                                    *
* You are accessing a secured system and your actions will be logged along   *
* with identifying information. Disconnect immediately if you are not an     *
* authorized user of this system.                                            *
------------------------------------------------------------------------------
[0;35mVariable files: "-e @/etc/openstack_deploy/user_secrets.yml -e @/etc/openstack_deploy/user_variable.yml -e @/etc/openstack_deploy/user_variables.yml "[0m
[WARNING]: Invalid characters were found in group names but not replaced, use
-vvvv to see details
[WARNING]: Unable to parse /etc/openstack_deploy/inventory.ini as an inventory
source
sp-dev-infra1_galera_container-1084b8a8 | CHANGED | rc=0 >>
PING 10.246.184.1 (10.246.184.1) 56(84) bytes of data.
64 bytes from 10.246.184.1: icmp_seq=1 ttl=64 time=0.623 ms

--- 10.246.184.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.623/0.623/0.623/0.000 ms
sp-dev-infra1_rsyslog_container-815b399b | CHANGED | rc=0 >>
PING 10.246.184.1 (10.246.184.1) 56(84) bytes of data.
64 bytes from 10.246.184.1: icmp_seq=1 ttl=64 time=0.484 ms

--- 10.246.184.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.484/0.484/0.484/0.000 ms
sp-dev-infra1_horizon_container-f2d50e48 | CHANGED | rc=0 >>
PING 10.246.184.1 (10.246.184.1) 56(84) bytes of data.
64 bytes from 10.246.184.1: icmp_seq=1 ttl=64 time=0.521 ms

--- 10.246.184.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.521/0.521/0.521/0.000 ms
sp-dev-infra1_heat_api_container-35aabe28 | CHANGED | rc=0 >>
PING 10.246.184.1 (10.246.184.1) 56(84) bytes of data.
64 bytes from 10.246.184.1: icmp_seq=1 ttl=64 time=0.555 ms

--- 10.246.184.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.555/0.555/0.555/0.000 ms
sp-dev-infra1_utility_container-6dd1678d | CHANGED | rc=0 >>
PING 10.246.184.1 (10.246.184.1) 56(84) bytes of data.
64 bytes from 10.246.184.1: icmp_seq=1 ttl=64 time=0.543 ms

--- 10.246.184.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.543/0.543/0.543/0.000 ms
sp-dev-infra1_memcached_container-72f36e12 | CHANGED | rc=0 >>
PING 10.246.184.1 (10.246.184.1) 56(84) bytes of data.
64 bytes from 10.246.184.1: icmp_seq=1 ttl=64 time=0.529 ms

--- 10.246.184.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.529/0.529/0.529/0.000 ms
sp-dev-infra1_neutron_server_container-6b9a5c80 | CHANGED | rc=0 >>
PING 10.246.184.1 (10.246.184.1) 56(84) bytes of data.
64 bytes from 10.246.184.1: icmp_seq=1 ttl=64 time=0.544 ms

--- 10.246.184.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.544/0.544/0.544/0.000 ms
sp-dev-infra1_placement_container-51c60932 | CHANGED | rc=0 >>
PING 10.246.184.1 (10.246.184.1) 56(84) bytes of data.
64 bytes from 10.246.184.1: icmp_seq=1 ttl=64 time=0.774 ms

--- 10.246.184.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.774/0.774/0.774/0.000 ms
sp-dev-infra1_cinder_api_container-92d07d92 | CHANGED | rc=0 >>
PING 10.246.184.1 (10.246.184.1) 56(84) bytes of data.
64 bytes from 10.246.184.1: icmp_seq=1 ttl=64 time=0.725 ms
```


###### Stage ‘Copy Monitoring role’


```shell
[Pipeline] stage
[Pipeline] { (Copy Monitoring role)
[Pipeline] sh
+ scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa /var/lib/jenkins/monitoring.tar root@sp-dev-infra2:/etc/ansible/roles/
------------------------------------------------------------------------------
* WARNING                                                                    *
* You are accessing a secured system and your actions will be logged along   *
* with identifying information. Disconnect immediately if you are not an     *
* authorized user of this system.                                            *
------------------------------------------------------------------------------
+ ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/id_rsa root@sp-dev-infra2 cd /etc/ansible/roles ; tar -xvf monitoring.tar; cp -r monitoring/playbooks/* /opt/openstack-ansible/playbooks/
------------------------------------------------------------------------------
* WARNING                                                                    *
* You are accessing a secured system and your actions will be logged along   *
* with identifying information. Disconnect immediately if you are not an     *
* authorized user of this system.                                            *
------------------------------------------------------------------------------
monitoring/
monitoring/.gitignore
monitoring/tasks/
monitoring/tasks/install-prometheus.yml
monitoring/tasks/main.yml
monitoring/tasks/install-node-exporter.yml
monitoring/tasks/install-libvirt-exporter.yml
monitoring/tasks/grafana-imports.yml
monitoring/tasks/install-grafana.yml
monitoring/tasks/install-openstack-exporter.yml
monitoring/defaults/
monitoring/defaults/main.yml
monitoring/.travis.yml
monitoring/files/
monitoring/files/dashboard/
monitoring/files/dashboard/node-exporter-for-prometheus-dashboard-en-v20201010_rev9.json
monitoring/files/dashboard/openstack-dashboard.json
monitoring/files/dashboard/node-exporter.json
monitoring/files/dashboard/openstack-vm-stats.json
monitoring/files/rules.yml
monitoring/files/node_exporter-1.0.1.linux-amd64.tar.gz
monitoring/files/openstack-exporter.tar.gz
monitoring/files/prometheus-2.23.0.linux-amd64.tar.gz
monitoring/files/libvirt-exporter.tar.gz
monitoring/templates/
monitoring/templates/openstack_exporter.service.j2
monitoring/templates/prometheus.service.j2
monitoring/templates/grafana.ini.j2
monitoring/templates/clouds.yaml.j2
monitoring/templates/dashboards/
monitoring/templates/dashboards/nodes.json.j2
monitoring/templates/dashboards/cluster.json.j2
monitoring/templates/clouds.yml.j2
monitoring/templates/node_exporter.service.j2
monitoring/templates/libvirt_exporter.service.j2
monitoring/templates/prometheus.yml.j2
monitoring/templates/datasource.json.j2
monitoring/README.md
monitoring/vars/
monitoring/vars/main.yml
monitoring/playbooks/
monitoring/playbooks/inventory
monitoring/playbooks/monitoring_stop.yml
monitoring/playbooks/runtests.sh
monitoring/playbooks/monitoring_start.yml
monitoring/playbooks/testsuite.sh
monitoring/playbooks/monitoring_install.yml
monitoring/handlers/
monitoring/handlers/main.yml
monitoring/meta/
monitoring/meta/main.yml
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```







