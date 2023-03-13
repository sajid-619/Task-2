# Task-2

# Vagrant up
Place those files in a folder on your machine, with Vagrant installed and execute vagrant up. You will end up with 3 nodes in your VirtualBox.

# Installing the required software
This task is performed by ansible. Note before executing it, you need to setup ssh to be able to access the nodes without a password. You can do it easily with Vagrant with vagrant ssh-config >~/.ssh/config and then add to your hosts file the line:

```sh
  127 .0 .0 .1 master node1 node2
  ```
You should be able then to access the servers with ssh vagrant@master, ssh vagrant@node1

The first snippet is about installing the required software:
```sh
    - shell: "apt-get -y update && apt-get install -y apt-transport-https"
- apt_repository:
    repo: 'deb http://apt.kubernetes.io/ kubernetes-xenial main'
    state: present
- name: install docker and kubernetes
  apt: name={{ item }} state=present allow_unauthenticated=yes
  with_items:
    - docker.io
    - kubelet
    - kubeadm
    - kubectl
    - ntp
   ```

Once you have installed packages, there are a few mandatory configurations. This is the ansible script performing them:
```sh
command: modprobe {{item}}
  with_items:    
  - ip_vs
  - ip_vs_rr
  - ip_vs_wrr
  - ip_vs_sh
  - nf_conntrack_ipv4
- lineinfile: path=/etc/modules line='{{item}}' create=yes state=present
  with_items: 
  - ip_vs
  - ip_vs_rr
  - ip_vs_wrr
  - ip_vs_sh
  - nf_conntrack_ipv4
  - sysctl: name=net.ipv4.ip_forward value=1 state=present reload=yes sysctl_set=yes
  - service: name=docker state=started enabled=yes
  - service: name=ntp state=started enabled=yes
  - service: name=kubelet state=started enabled=yes
```
# Installing a Network plugin
```sh
- sysctl: name=net.bridge.bridge-nf-call-ip6tables value=1 state=present reload=yes sysctl_set=yes
- sysctl: name=net.bridge.bridge-nf-call-iptables value=1 state=present reload=yes sysctl_set=yes
- name: install weave net
  shell: >
    export KUBECONFIG=/etc/kubernetes/admin.conf ;
    export kubever=$(sudo kubectl version | base64 | tr -d '\n') ;
    curl --location "https://cloud.weave.works/k8s/net?k8s-version=$kubever" >/etc/kubectl/weave.yml ;
    kubectl apply -f /etc/kubectl/weave.yml
- shell: >
    export KUBECONFIG=/etc/kubernetes/admin.conf ;
    kubectl get pods -n kube-system -l name=weave-net
  register: result
  until: result.stdout.find("Running") != -1
  retries: 100
  delay: 10
  ```
# Configuring nodes
This is a task performed by this ansible script.
```sh
- sysctl: name=net.bridge.bridge-nf-call-ip6tables value=1 state=present reload=yes sysctl_set=yes
- sysctl: name=net.bridge.bridge-nf-call-iptables value=1 state=present reload=yes sysctl_set=yes
- shell: "cat /etc/kubeadm-join.sh"
  register: cat_kubeadm_join
  when: inventory_hostname == master_hostname
- set_fact:
    kubeadm_join: "{{cat_kubeadm_join.stdout}}"
  when: inventory_hostname == master_hostname
- name: join nodes
  shell: >
     systemctl stop kubelet ; kubeadm reset ; 
     echo "{{hostvars[master_hostname].kubeadm_join}}" >/etc/kubeadm-join.sh ; 
     bash /etc/kubeadm-join.sh
  args:
    creates: /etc/kubeadm-join.sh
  when: inventory_hostname != master_hostname
- name: checking all nodes up
  shell: >
      export KUBECONFIG=/etc/kubernetes/admin.conf ;
      kubectl get nodes {{item}}
  register: result
  until: result.stdout.find("Ready") != -1
  retries: 100
  delay: 10
  with_items: "{{ groups['nodes'] }}"
  when: inventory_hostname == master_hostname
  ```
