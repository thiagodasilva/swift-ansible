Swift-ansible is a simple way to deploy swift. I personal use it to manage a
test cluster, and so isn't full featured, I add features as I require them.
for example, there is no EC policy support yet.

The idea, of the deployment is your entire swift cluster is described in an easy
to read and use yaml file::
```
  swift_setup.yml
```
Which means, this file is an up to date representation of your cluster.
The scripts/ directory is non-ansible and contains important scripts to use
with the deployment::
```
  scripts/
    swift_inventory.py
    swift_rings.py
```
The swift_inventory.py script generates an ansible inventory hosts file, this
will one day will just be used as a dynamic inventory, but I haven't gotten
around to that yet. Swift_rings.py, creates/updates builder files and creates
rings and takes in the swift_setup.yml file.
To see how this works see the playbooks/build_swift_rings.yml playbook.

Extra configuration of the swift cluster that isn't managed by the yml file,
pipelines for example, and simply managed by editing the swift config files
located into the specific swift roles. The swift roles are::
```
  roles/
    swift_common
    swift_account
    swift_container
    swift_object
    swift_proxy
```
swift_common, sets up everything common, swift dependencies, checkout the
source, installs the source, sets up syslog, swift users, etc.
swift_{proxy, account, container, object}, sets up the specific daemons used by
swift. So you need to modify the proxy pipeline, you edit::
```
  roles/swift_proxy/templates/proxy-server.conf.j2
```

If testing code and you've patched something you can simply deploy the code
throughout the cluster with::
```
   ansible-playbook -i /etc/ansible/hosts /etc/ansible/playbooks/setup_swift.yml -t deploy_code
   ansible-playbook -i /etc/ansible/hosts /etc/ansible/playbooks/restart_swift.yml
```

Need to run a command and a set of servers, for example I have backed up the
swift storage nodes at a certain point for testing, which means I can restore
this point by writing a bash script that simply runs::
```
  ansible container -i /etc/ansible/hosts -m shell -a "swift-init stop all"
  ansible container -i /etc/ansible/hosts -m shell -a "rm -Rf /mnt/sda/*"
  ansible container -i /etc/ansible/hosts -m shell -a "rm -Rf /mnt/sdb/*"
  ansible container -i /etc/ansible/hosts -m shell -a "rm -Rf /mnt/sdc/*"
  ansible container -i /etc/ansible/hosts -m shell -a "rm -Rf /mnt/sdd/*"
  ansible container -i /etc/ansible/hosts -m shell -a "tar -xvf /root/cont_300k.tar -C /mnt/"
  ansible-playbook -i /etc/ansible/hosts /etc/ansible/playbooks/restart_swift.yml
```
NOTE: in my test cluster my storage nodes are members of the container, account
and object groups, so by using the container group (ansible container) above
will hit all my storage nodes.


Swift-Ansible can also be installed on VMs created by Vagrant, currently this
will require the following manual steps::
  * `vagrant up`  <= This will create 6 VMs by default (see Vagrantfile)
  * `cp swift_setup.yml.example swift_setup.yml`
  * `python scripts/swift_inventory.py -s swift_setup.yml`
  * `ssh-copy-id root@192.168.9.10` <= repeat for each newly created VM
  * `ansible-playbook -i hosts configure_swift.yml`
  * `mkdir rings && cd rings`
  * `python ../scripts/swift_rings.py -s ../swift_setup.yml`
