---
icon: material/ansible
---

#### Vars Precedence
Ansible does apply variable precedence, and you might have a use for it. Here is the order of precedence from least to greatest (the last listed variables override all other variables):

1. command line values (for example, -u my_user, these are not variables)
2. role defaults (defined in role/defaults/main.yml) 1
3. inventory file or script group vars 2
4. inventory group_vars/all 3
5. playbook group_vars/all 3
6. inventory group_vars/* 3
7. playbook group_vars/* 3
8. inventory file or script host vars 2
9. inventory host_vars/* 3
10. playbook host_vars/* 3
11. host facts / cached set_facts 4
12. play vars
13. play vars_prompt
14. play vars_files
15. role vars (defined in role/vars/main.yml)
16. block vars (only for tasks in block)
17. task vars (only for the task)
18. include_vars
19. set_facts / registered vars
20. role (and include_role) params
21. include params
22. extra vars (for example, -e "user=my_user")(always win precedence)

In general, Ansible gives precedence to variables that were defined more recently, more actively, and with more explicit scope. Variables in the defaults folder inside a role are easily overridden. Anything in the vars directory of the role overrides previous versions of that variable in the namespace. Host and/or inventory variables override role defaults, but explicit includes such as the vars directory or an include_vars task override inventory variables.

Ansible merges different variables set in inventory so that more specific settings override more generic settings. For example, ansible_ssh_user specified as a group_var is overridden by ansible_user specified as a host_var. For details about the precedence of variables set in inventory

#### Playbook Execution

```bash
ansible-playbook -v -i ./ansible/inventory/all.yaml -i ./ansible/inventory/all_cn-tj.yaml -i ansible/inventory/staging_cn-tj.yaml --vault-id staging@**** ./ansible/playbooks/site.yml -e ansible_ssh_private_key_file=/xxx.rsa -e app_version=xxx -e pypi_repository=pypi-internal-snapshot
```
