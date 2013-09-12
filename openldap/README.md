

This playbook deploys a openldap server which can be access via ssl as well as normal access.

It also installs phpldapadmin which can be used to graphically manage the ldap
the ui can be access via "http://<ip_of_ldap_server>/ldapadmin

user: cn=Manager,dc=ben,dc=com

passwd: passme


Note: The certificates used are precreated and the server certificate has the hostname gives "awx12", so incase you are accessing
the ldap via ssl use the URI as "ldaps://awx12" and make an entry for awx12 in /etc/hosts.


