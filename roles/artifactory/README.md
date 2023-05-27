artifactory
=========

The artifactory role installs the Artifactory Pro software onto the host. Per the Vars below, it will configure a node as primary or secondary. This role uses secondary roles artifactory_nginx to install nginx.

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

[To be Updated later]

Role Variables
--------------

* server_name: mandatory This is the server name. eg. "artifactory.54.175.51.178.xip.io"
* artifactory_upgrade_only: Perform an software upgrade only. Default is false.

Additional variables can be found in defaults/main.yml.


Example Playbook
----------------

Create a new playbook in the static-assignment directory and update it with the below code:

    - hosts: host_name
      roles:
         - artifactory_role_name

Update site.yml update "playbooks" directory with the below details:

```YAML
- hosts: host_name
- name: artifactory assignment
  ansible.builtin.import_playbook: ../static-assignments/artifactory.yml
```


License
-------

BSD
