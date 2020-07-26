# kubernetes-automation
The complete solution to build a k8s environment in an automated fashion on a CentOS/RedHat based operating systems. 


### How will your main playbook looks.

```---
- hosts: all
  roles: 
  - k8s-prereq

- hosts: masters[0]
  roles:
  - k8s

- hosts: all
  gather_facts: false
  roles:
  - k8s-node-joining
```

### Lets get a sample inventory you can make use of

```---
all:
  children:
    masters:
        hosts:
          kube-master-1:
            ansible_host: 104.211.8.100
    workers:
        hosts:
          kube-worker-1:
            ansible_host: 104.211.8.100
          kube-worker-2:
            ansible_host: 104.211.8.100
  vars:
    ansible_ssh_user: username
    ansible_become: yes
```

### How do we run the playbook

```
ansible-playbook -i inventory.yaml k8s.yaml
```

### TO DOs

```Want to contribute or find a Bug? Great!

Feel free to create a PR.
```

### Author
```Naveen Ramasamy, An Infra Automation Engineer.
```