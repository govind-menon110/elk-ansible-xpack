[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

# Ansible playbook for Xpack enabled ELK

Special thanks to fedelemantuano for the [kibana role](https://github.com/fedelemantuano/ansible-kibana) and the [ansible elasticsearch role](https://github.com/elastic/ansible-elasticsearch) which has been used extensively. Enables out of the box installation of a fully functional secure cluster (TLS and basic auth) with a basic elastic license courtesy of [elastic](https://elastic.co)

## Please modify the scripts to meet your needs especially the `group_vars/all.yml` and `disk.yml` file!

# Preparation:
1. Install ansible first: 
    ``` sh
    $ sudo python -m pip install ansible
    ```
    [Must be installed on machine having connectivity to all other machines where one wants to install ES, Kibana, etc through SSH]
2. You may need to keep the private key for ssh handy in case it is used to access Machines (eg: In the case of AWS).  Let us name the key `your_key.pem`
3. Have IPs of all the systems handy and note down which systems will be your Master node, Data node and Coordinating Node.
4. This script uses pre-built ES roles. Run the following only after ansible has been installed:
    ``` sh
    $ ansible-galaxy install elastic.elasticsearch,7.9.1
    ```
    ``` sh
    $ ansible-galaxy install fedelemantuano.kibana
    ```

# Changing Script values:
The following need to be checked and changed:
Remember the IP list you made earlier? It will help you now.

1. Input all IPs in the [inventory.ini](./inventory.ini) file in required positions (decide which machines would be your master node, data node, etc and put them under corresponding blocks). For example if 1.1.1.1 is master, 1.2.2.1 is coordinating, 1.3.3.1 is data and 1.2.2.1 is kibana, my inventory.ini file will look like this:
    ``` yaml
     [hosts]
      1.1.1.1 ansible_ssh_common_args='-o StrictHostKeyChecking=no'
      1.2.2.1 ansible_ssh_common_args='-o StrictHostKeyChecking=no'
      1.3.3.1 ansible_ssh_common_args='-o StrictHostKeyChecking=no'
     [data]
      1.3.3.1
     [masters]
      1.1.1.1
     [coordinating]
      1.2.2.1
     [kibana]
      1.2.2.1
     ```
     Though not recommended, by default host key check during ssh is disabled by default for usability.
2.  Install elasticsearch locally on your system (from where ansible is being orchestrated). Run the following in sequence:
    ```sh
    $ /usr/share/elasticsearch/bin/elasticsearch-certutil cert --keep-ca-key --pass "" --out "<full_path_to_current_folder>/certs.zip" --silent --ca-pass ""
    ```

    OR

    ```sh
    /<elasticsearch-downloaded-folder>/bin/elasticsearch-certutil cert --keep-ca-key --pass "" --out "<full_path_to_current_folder>/certs.zip" --silent --ca-pass ""
    ```

    THEN

    ```sh
    $ unzip certs.zip
    ```

    ```sh
    $ sudo openssl pkcs12 -in ./ca/ca.p12 -clcerts -nokeys -chain -out ./ca/ca.pem
    ```
2. Continuing this example, the [all.yml](./group_vars/all.yml) file in the group_vars folder must be modified as follows. 
    ``` yaml
    kibana_encryption_key: ""<32_character_alphanumeric>""
    kibana_es_user: "kibana_system"
    kibana_password: "<your_kibana_password>"
    kibana_version: "7.10.1"

    elastic_bootstrappass: "changeme"
    elastic_superuser: "elastic"
    elastic_superuserpass: "<your_elastic_password>"

    logstash_userpass: "<your_logstash_password>"

    apm_userpass: "<your_apm_password>"

    monitoring_userpass: "<your_monitoring_password>"

    beats_userpass: "<your_beats_password>"

    es_data_dir: "/es-data"

    es_master_heap_size: "2g"
    es_data_heap_size: "2g"
    es_coord_heap_size: "2g"
    es_cluster_name: "<cluster_name>"

    es_common_p12_path: "<complete_path_to_unzipped_certs_folder>/instance/instance.p12"
    kibana_ca_cert: "<complete_path_to_unzipped_certs_folder>/ca/ca.pem" 
     ```
4. Next you need to make sure [disk.yaml](./disk.yml) is changed to match the disk names that is used by your machines. `nvme1n1`, `xvdb` and `xdf` are most commonly used for AWS EC2 instances and have been included in the script. If yours is different, change the disk type.

# Running the script:
Just run the following from the machine connected to all other machines through ssh

``` sh
$ ansible-playbook main.yml -i inventory.ini --user ec2-user --key-file your_key.pem
```
**(Use ctrl + c to exit ansible if it gets stuck at checking if kibana restarted)**

**Make sure the cluster is up (kibana would not be up yet).**

Once confirmed that the cluster is formed, run:

``` sh
$ ansible-playbook certs.yml -i inventory.ini --user ec2-user --key-file your_key.pem
```

Now kibana should work!

