```yml
---
- name: DNS playbook
  hosts: dns
  vars:
    a_csv_users: "{{ lookup('file', 'sources/ldap.csv') | community.general.from_csv }}"
    b_csv_uids: "{{ a_csv_users | map(attribute='uid') }}"
  tasks:
    - name: existing
      community.general.ldap_search:
        bind_dn: cn=admin,dc=flychina,dc=cn
        bind_pw: Passw0rd!
        dn: "ou=Lab,dc=flychina,dc=cn"
        scope: "onelevel"
        attrs:
          - uid
      register: existing_users

    - name: remove
      community.general.ldap_entry:
        bind_dn: cn=admin,dc=flychina,dc=cn
        bind_pw: Passw0rd!
        dn: "{{ item.dn }}"
        state: absent
      loop: "{{ existing_users.results }}"
      when: item.uid not in b_csv_uids
      loop_control:
        label: "{{ item.uid }}"

    - name: add
      community.general.ldap_entry:
        bind_dn: cn=admin,dc=flychina,dc=cn
        bind_pw: Passw0rd!
        dn: "uid={{item.uid}},ou=Lab,dc=flychina,dc=cn"
        objectClass:
          - inetOrgPerson
          - posixAccount
          - shadowAccount
        attributes:
          uid: "{{ item.uid }}"
          cn: "{{ item.cn }}"
          givenName: "{{ item.givenName }}"
          sn: "{{ item.sn }}"
          uidNumber: "{{ item.uidNumber }}"
          gidNumber: "{{ item.gidNumber }}"
          mail: "{{ item.mail }}"
          homeDirectory: "{{ item.homeDirectory }}"
          loginShell: "{{ item.loginShell }}"
        state: present
      loop: "{{ a_csv_users }}"
      loop_control:
        label: "{{ item.uid }}"
      notify: setpw
  handlers:
    - name: setpw
      community.general.ldap_passwd:
        bind_dn: cn=admin,dc=flychina,dc=cn
        bind_pw: Passw0rd!
        dn: "uid={{item.uid}},ou=Lab,dc=flychina,dc=cn"
        passwd: "{{ item.password }}"
      loop: "{{ a_csv_users }}"
      loop_control:
        label: "{{ item.uid }}"
```
