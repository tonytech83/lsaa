## PREPARATION

1. Install Python 3.9 on OpenSUSE VMs required from Ansible

2. Create `inventory.ini` with both VMs.

```ini
[vms:children]
router
client

[router]
router ansible_host=192.168.88.110

[client]
client ansible_host=192.168.99.102

[vms:vars]
ansible_connection=ssh
ansible_user=vagrant
ansible_ssh_pass=vagrant
ansible_python_interpreter=/usr/bin/python3.9
```

3. Create `main.yaml`

```yaml
- name: Configure Two-Node NAT task
  hosts: all
  become: true
  vars:
    vm1_external: "eth1" # External NIC on VM1 (Router)
    vm1_internal: "eth2" # Internal NIC on VM1 (Router)
    vm2_external: "eth1" # External NIC on VM2 (Client)
    vm2_vbox: "eth0" # VirtualBox NIC on VM2 (Client)
    gateway_ip: "192.168.99.101" # Internal IP of VM1
    external_dns: "8.8.8.8" # DNS to test connectivity

  tasks:
    - name: Ensure firewalld is installed
      zypper:
        name: firewalld
        state: present

    - name: Enable and start firewalld
      service:
        name: firewalld
        state: started
        enabled: true

    - name: Configure NAT on VM1
      when: inventory_hostname == "router"
      block:
        - name: Enable IP Forwarding
          sysctl:
            name: net.ipv4.ip_forward
            value: "1"
            state: present
            reload: true

        - name: Add external interface to public zone
          command: sudo firewall-cmd --permanent --zone=public --change-interface="{{ vm1_external }}"

        - name: Add internal interface to internal zone
          command: sudo firewall-cmd --permanent --zone=internal --change-interface="{{ vm1_internal }}"

        - name: Enable masquerading in public zone
          command: sudo firewall-cmd --zone=public --add-masquerade --permanent

        - name: Allow all traffic from internal (vm1_internal) to external (vm1_external)
          command: sudo firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i "{{ vm1_internal }}" -o "{{ vm1_external }}" -j ACCEPT

        - name: Allow return traffic for established connections from external (vm1_external) to internal (vm1_internal)
          command: sudo firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i "{{ vm1_external }}" -o "{{ vm1_internal }}" -m state --state RELATED,ESTABLISHED -j ACCEPT

        - name: Reload firewalld to apply changes
          command: sudo firewall-cmd --reload

    - name: Configure VM2 default gateway and disable external and vbox NICs
      when: inventory_hostname == "client"
      block:
        - name: Disable external NIC
          command: wicked ifdown "{{ vm2_external }}"

        - name: Disable vbox NIC
          command: wicked ifdown "{{ vm2_vbox }}"

        - name: Remove existing default gateway (if any)
          command: ip route del default
          ignore_errors: true

        - name: Set default gateway via VM1
          command: ip route add default via "{{ gateway_ip }}"

    - name: Test connectivity from VM2 to the internet
      when: inventory_hostname == "client"
      command: ping -c 5 "{{ external_dns }}"
      register: ping_test

    - name: Debug ping results
      debug:
        msg: "Ping test result: {{ ping_test.stdout_lines }}"
      when:
        - inventory_hostname == "client"
        - ping_test is defined
```

4. Execute ansible playbook

```sh
$ ansible-playbook -i inventory.ini main.yaml
[WARNING]: Found both group and host with same name: router
[WARNING]: Found both group and host with same name: client

PLAY [Configure Two-Node NAT task] **********************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************
ok: [router]
ok: [client]

TASK [Ensure firewalld is installed] ********************************************************************************************************
ok: [router]
ok: [client]

TASK [Enable and start firewalld] ***********************************************************************************************************
ok: [client]
ok: [router]

TASK [Enable IP Forwarding] *****************************************************************************************************************
skipping: [client]
ok: [router]

TASK [Add external interface to public zone] ************************************************************************************************
skipping: [client]
changed: [router]

TASK [Add internal interface to internal zone] **********************************************************************************************
skipping: [client]
changed: [router]

TASK [Enable masquerading in public zone] ***************************************************************************************************
skipping: [client]
changed: [router]

TASK [Allow all traffic from internal (vm1_internal) to external (vm1_external)] ************************************************************
skipping: [client]
changed: [router]

TASK [Allow return traffic for established connections from external (vm1_external) to internal (vm1_internal)] *****************************
skipping: [client]
changed: [router]

TASK [Reload firewalld to apply changes] ****************************************************************************************************
skipping: [client]
changed: [router]

TASK [Disable external NIC] *****************************************************************************************************************
skipping: [router]
changed: [client]

TASK [Disable vbox NIC] *********************************************************************************************************************
skipping: [router]
changed: [client]

TASK [Remove existing default gateway (if any)] *********************************************************************************************
skipping: [router]
changed: [client]

TASK [Set default gateway via VM1] **********************************************************************************************************
skipping: [router]
changed: [client]

TASK [Test connectivity from VM2 to the internet] *******************************************************************************************
skipping: [router]
changed: [client]

TASK [Debug ping results] *******************************************************************************************************************
skipping: [router]
ok: [client] => {
    "msg": "Ping test result: ['PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.', '64 bytes from 8.8.8.8: icmp_seq=1 ttl=254 time=3.77 ms', '64 bytes from 8.8.8.8: icmp_seq=2 ttl=254 time=4.17 ms', '64 bytes from 8.8.8.8: icmp_seq=3 ttl=254 time=5.38 ms', '64 bytes from 8.8.8.8: icmp_seq=4 ttl=254 time=6.23 ms', '64 bytes from 8.8.8.8: icmp_seq=5 ttl=254 time=5.38 ms', '', '--- 8.8.8.8 ping statistics ---', '5 packets transmitted, 5 received, 0% packet loss, time 4011ms', 'rtt min/avg/max/mdev = 3.769/4.984/6.230/0.894 ms']"
}

PLAY RECAP **********************************************************************************************************************************
client                     : ok=9    changed=5    unreachable=0    failed=0    skipped=7    rescued=0    ignored=0
router                     : ok=10   changed=6    unreachable=0    failed=0    skipped=6    rescued=0    ignored=0
```
