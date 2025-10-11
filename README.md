# hadoop-cluster-ansible

Hadoop Cluster Ansible Playbook
This Ansible playbook automates the deployment of a Hadoop cluster. It sets up a single master node and multiple worker nodes with HDFS and YARN components.

### Requirements
x86-64 Platform (arm64 does not work!)
Ansible 2.19.2 or higher
Ubuntu/Debian target machines
SSH access to all nodes
Python 3.x on target machines

### Quick Start
1. Check connectivity to all nodes
```
ansible all -i inventory.yml -m ping -v
```
2. Validate playbook syntax
```
ansible-playbook -i inventory.yml site.yml --syntax-check
```
3. Dry run (тут будет failed из-за несуществующей директории /opt/hadoop, все нормально)
```
ansible-playbook -i inventory.yml site.yml --check -v
```
4. Deploy cluster
```
# Deploy master node first
ansible-playbook -i inventory.yml site.yml --limit hadoop_master -v

# Then deploy worker nodes
ansible-playbook -i inventory.yml site.yml --limit hadoop_worker -v
```
### Verification
Check Hadoop processes on master:
```
ssh alice3e@192.168.56.2 "jps"
# Should show: NameNode, ResourceManager
```

Check Hadoop processes on workers:
```
ssh alice3e@192.168.56.4 "jps"
# Should show: DataNode, NodeManager
```
View cluster status:
```
# On master node
hdfs dfsadmin -report
yarn node -list
```