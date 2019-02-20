Ansible Role: iptables-rules
=======

This role:
* Installs the `iptables` and `iptables-services` packages
* Disables firewalld
* Enables the `iptables` service
* Creates a `/etc/sysconfig/iptables` file containing iptables rules based on YAML structured data you provide. It also validates its syntax. And it creates a backup.
* Loads the new rules file by restarting the iptables service.

Requirements
------------
A RHEL or CentOS system, version 7 (v6 is coming).


Role Variables
--------------

For each system, you can set these two variables:
* `input_rules`: a list of rules to be activated in the INPUT chain
* `output_rules`: a list of rules to be activated in the OUTPUT chain

For each chain to be managed, you can set its default policy:
* `input_rule_default_target`: The default policy for the INPUT chain, default is ACCEPT
* `output_rule_default_target`: The default policy for the OUTPUT chain, default is ACCEPT

Each rule is made of:
* `comment`: (string) A brief description of what this rule does. It will be used to populate the comments in the iptables rule.
* `proto`: (string or a list of) The protocol(s), i.e.: tcp, udp, icmp.
* `dport`: (string or a list of) The ports(s) for this rule. Do not set this as an integer, use a string, i.e.: `"22"`
* `source`: (string or a list of) The source IP address(es) for this rule (INPUT chain only).
* `destination`: (string or a list of) The destination IP address(es) for this rule (OUTPUT chain only).
* `target`: (string) The target for this rule, i.e.: ACCEPT, DROP, REJECT
* `raw`: (string) The complete rule to be written as-is, i.e.: `-i lo -j ACCEPT`. 

You can use the predefined rules you find in the [defaults/main.yaml](defaults/main.yaml) file and also create your own. See Example Playbook below.

**Note**: This role will automatically add the minimum set of rules needed if you set the default policy other than ACCEPT:
* Allow loopback traffic
* Allow all ICMP
* Allow established and related traffic

Be sure to allow SSH connection by using the template rule `input_rule_ssh` or to create your own.

Example Playbook
----------------

The variables for your server:
```yaml
# host_vars/srv1.example.com.yaml

# Deny (drop) all traffic except the rules below
input_rule_default_target: "DROP"

input_rules:
# Only allow ssh traffic by using one of the templates
- "{{ input_rule_ssh }}"
# and allow a certain LAN to make www traffic, with a custom rule
- comment: "Allow Web traffic from my LAN"
  proto: "tcp"
  dport:
  - "80"
  - "443"
  target: "ACCEPT"
  source: "192.168.1.0/24"
```

The main playbook:
```yaml
# playbook.yaml
- hosts: all
  gather_facts: false
  become: true
  tasks:
  - tags:
    include_role:
      name: fcarrus.iptables-rules
```

The above would produce the following `/etc/sysconfig/iptables` file:
```
#
# Ansible managed
#
*filter
:INPUT DROP [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

# INPUT Chain

# INPUT: -m comment --comment "Loopback traffic"
-A INPUT -i lo -j ACCEPT

# INPUT: -m comment --comment "ICMP traffic"
-A INPUT -p icmp  -m comment --comment "ICMP traffic" -j ACCEPT

# INPUT: -m comment --comment "Return traffic"
-A INPUT -m state --state ESTABLISHED,RELATED -m comment --comment "Return traffic" -j ACCEPT

# INPUT: -m comment --comment "Allow SSH"
-A INPUT -p tcp -m state --state NEW -m comment --comment "Allow SSH" -j ACCEPT

# INPUT: -m comment --comment "Allow Web traffic from my LAN"
-A INPUT -s 192.168.1.0/24 -p tcp -m state --state NEW --dport 80 -m comment --comment "Allow Web traffic from my LAN" -j ACCEPT
-A INPUT -s 192.168.1.0/24 -p tcp -m state --state NEW --dport 443 -m comment --comment "Allow Web traffic from my LAN" -j ACCEPT

COMMIT
```

See the [defaults/main.yaml](defaults/main.yaml) file for more examples.


TODOs
-----
* Implement RHEL-6 and CentOS-6 compatible versions.
* Implement `forward_rules`.
* Implement `destination` for INPUT chain and `source` for OUTPUT chain.
* Implement `interface` specification.
* 