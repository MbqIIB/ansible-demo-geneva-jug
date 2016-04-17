# Ansible demo @ Geneva JUG

http://genevajug.ch/events/2016-02-23-ansible.html

### Prerequisite:

* Linux machine running on EC2 with Ansible
* SSH key named "ansible_ec2_key.pem"
* copy the SSH key to "/etc/ansible/ssh/ansible_ec2_key.pem"


### Demo - EC2 provisioning

In this demo, we have a look at Amazon EC2 configuration to create Security Group and EC2 instances thanks to Ansible.

```
# Presentation of the host file with some groups but no host
nano hosts
ansible --list-hosts all

# Provision 2 instances
ansible-playbook playbooks/ec2_provision.yml
```
  

### Demo - Ansible inventory & ad-hoc commands

In this demo, we walk through basic concepts of Ansible and core Configuration principles:

- **Static inventory** to [declare hosts](http://docs.ansible.com/ansible/intro_inventory.html) of the demo

- **Patterns** to [select hosts](http://docs.ansible.com/ansible/intro_patterns.html) with `ansible --list-hosts`
  - host name or group name
  - mix of groups (union, intersection)
  - wildcard
  - regex

- Ops Hello World with `ansible -m ping`

- Gather **Ansible facts** (data from remote machines) with `ansible -m setup`

- Test basic [ad-hoc commands](http://docs.ansible.com/ansible/intro_adhoc.html) with `ansible -m shell`

- **Idempotence** principle by executing the same command twice

- **Configuration Drift** leading to [Snowflake servers](http://martinfowler.com/bliki/SnowflakeServer.html) by uninstalling a package rather than reprovisioning from scrath with [Phoenix servers](http://martinfowler.com/bliki/PhoenixServer.html)

```
# navigate through the inventory
nano hosts
ansible --list-hosts all
ansible --list-hosts ec2_hosts
ansible --list-hosts 'all:!local'
ansible --list-hosts 'ec2_hosts:other_hosts'

# check connectivity at host level
ping <host>

# check connectivity at ansible level
ansible -m ping ec2_hosts

ansible -m shell -a "uptime" ec2_hosts

ansible -m setup <host>
ansible -m setup -a "filter=ansible_eth[0-2]" ec2_hosts


# install ntp package with basic configuration
ansible -m apt -a "name=ntp state=latest force=yes update_cache=yes" --sudo <host>
ssh -i ./ssh/ansible_ec2_key.pem  ubuntu@<host>
nano /etc/ntp.conf
exit

# same command executed a second time, nothing changes
ansible -m apt -a "name=ntp state=latest force=yes update_cache=yes" --sudo <host>

# remove the ntp package, but the "libopts" remains on the machine
ansible -m apt -a "name=ntp state=absent" --sudo <host>
```

### Demo - Ansible playbooks & variables

In this demo, we walk through automation concepts of Ansible:

- **Playbooks** to express the sequence of tasks to obtain the "desired state"

- **Variables** associated to hosts and groups

- **Jinja2 templates** to [generate the configuration](http://docs.ansible.com/ansible/playbooks_filters.html)

- **host-contextual variables** with `ansible -m debug "var=hostvars['<host>']" localhost`

- playbook **tags** to [execute it partially](http://docs.ansible.com/ansible/playbooks_tags.html)

```
# script to install the NTP package and configure it thanks to a Jinja2 template
nano playbooks/ntp_template.yml

# NTP configuration template, which is a Ansible-managed file
nano playbooks/templates/ntp.conf.j2

# re-open the playbook to move the variables in the "group_vars/all.yml" (much better !)
nano playbooks/ntp_template.yml
###
	ntp_hosts:
		- 0.ubuntu.pool.ntp.org
		- 1.ubuntu.pool.ntp.org
		- 2.ubuntu.pool.ntp.org
		- 3.ubuntu.pool.ntp.org
###

nano group_vars/all.yml
###
ntp_hosts:
  - 0.ubuntu.pool.ntp.org
  - 1.ubuntu.pool.ntp.org
  - 2.ubuntu.pool.ntp.org
  - 3.ubuntu.pool.ntp.org
###

# look at variables depending on the host - same config
ansible -m debug -a "var=hostvars['localhost']" localhost
ansible -m debug -a "var=hostvars['<host>']" localhost

# changes variables for the host group "ec2_hosts"
nano group_vars/ec2_hosts.yml
ntp_hosts:
  - 0.ch.pool.ntp.org
  - 1.ch.pool.ntp.org
  - 2.ch.pool.ntp.org
  - 3.ch.pool.ntp.org

# look at variables depending on the host - different configs
ansible -m debug -a "var=hostvars['localhost']" localhost
ansible -m debug -a "var=hostvars['<host>']" localhost

# execute the NTP playbook
ansible-playbook playbooks/ntp_template.yml --extra-vars "target=ec2_hosts"

# connect remotely and check the deployed configuration
ssh -i ./ssh/ansible_ec2_key.pem  ubuntu@<host>
nano /etc/ntp.conf
exit

# uncomment the "uninstall" task
nano playbooks/ntp_template.yml

# use tags to execute the uninstallation, with the same playbook used for installation
ansible-playbook playbooks/ntp_template.yml --extra-vars "target=ec2_hosts" --tags "uninstall"

# terminate EC2 instances
ansible-playbook -i hosts playbooks/ec2_terminate.yml --extra-vars "target=ec2_hosts"

# check that hosts are removed from the host file
nano hosts
```

### Demo - Deploy a website


### Demo - Upgrade a website using rolling-upgrade
