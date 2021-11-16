## Netscaler Automated Certificate Renewal with Ansible and Let's Encrypt

This set of Ansible playbooks are designed to automatically acquire a Let's Encrypt certificate and update an existing certificate pair on a Netscaler. By updating a Netscaler certificate it means any services using the certificate are automatically updated too.

This Let's Encrypt module (le_renew-cert.yml) largely uses the script by Jamie Scaife from Digital Ocean, https://www.digitalocean.com/community/tutorials/how-to-acquire-a-let-s-encrypt-certificate-using-ansible-on-ubuntu-18-04. I made the following changes.

* Added steps at the start to start Apache and enable HTTP via firewalld. This enables me to leave NAT open on a firewall but the service isn't exposed unless the playbook is running.
* Changed the code to use variables for every modifiable item.
* Changed the OpenSSL configuration file path to /etc/pki/tls/openssl.cnf, to suit my RHEL server. The default value was /etc/ssl/openssl.cnf, for Ubuntu.
* Changed the 'Implement http-01 challenge files' step to loop over 'challenge_data', rather than 'domain_names'. This is important because if Let's Encrypt has cached your request, it won't send back a response for every domain specified. See https://github.com/ansible/ansible/issues/67949 for more information.
* Added steps at the end to stop Apache and remove the HTTP firewalld rule. These two steps are set to run under the always block, to ensure the web service doesn't remain accessible in case some of the Let's Encrypt steps fail.

## Environment

* Ansible version: 2.9.25
* Ansible host: CentOS Linux 7 (Core)
* Web Server: Apache 2.4.6
* Netscaler version: 13.0 (76.31)
* nzs-ansible-1: This is the Ansible host and also runs the web service to host Let's Encrypt challenge files
* 192.168.126.98: This is the Netscaler management IP

## Playbook Tasks

### le_renew-cert.yml

* The Let's Encrypt playbook uses the acme_certificate module, https://docs.ansible.com/ansible/latest/collections/community/crypto/acme_certificate_module.html#examples.
* With the configuration as it is, the playbook is designed to issue a single certificate with 3 SANs. To adjust this you will need to modify the 'san_#' variables in 'nzs-ansible-1' and line #39 in le_renew-cert.yml.
* To test the script, execute it against the Let's Encrypt's staging URL, https://acme-staging-v02.api.letsencrypt.org/directory. You can do this by updating the 'acme_directory' variable.

1. Start Apache service
2. Add firewalld rule to allow HTTP inbound
3. Create required directories in /opt/letsencrpt (where all the certificate files will be stored)
4. Generate a Let's Encrypt account key
5. Generate Let's Encrypt private key
6. Generate Let's Encrypt CSR
7. Begin Let's Encrypt challenges
8. Create challenge directories
9. Implement challenge files
10. Complete Let's Encrypt challenges
11. Remove firewalld rule allowing HTTP
12. Stop Apache service

### ns_update-cert.yml

* The Netscaler playbook uses the default Ansible URI module to call Netscaler's API, https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html

1. Delete old certificate file from the Netscaler
2. Delete old private key file from the Netscaler
3. Upload new certificate file to the Netscaler
4. Upload new private key file to the Netscaler
5. Update Netscaler certificate
6. Save Netscaler configuration

### ux_archive-cert.yml

1. Create archive directory
2. Archive private key
3. Archive certificate files

### renew-and-update-cert.yml

* This is the main playbook and it calls the above playbooks in the correct order

## Execution

### Requirements

* Ansible host (I used a CentOS 7 server)
* A web server which can host the challenge files (I used the same server as above)
* Access to a Netscaler which already has a certificate & key pair installed (a key pair which you will be updating)

### Update Variables

1. Update at least the following variables for the host running the web server (nzs-ansible-1). With the configuration as it is, the playbook is designed to issue a single certificate with 3 SANs. To adjust this you will need to modify the 'san_#' variables in 'nzs-ansible-1' and line #39 in le_renew-cert.yml.
* ns_user - Your Netscaler access username
* ns_pass - Your Netscaler access password (should use Ansible Vault)
* acme_email - Notification email address for certificate renewals
* domain_name - Primary domain name
* san_1 - SAN 1
* san_2 - SAN 2
* http_root_directory - The root directory of your web server where challenge files will be hosted

2. Update the following variables for the Netscaler host (192.168.126.98).
* certtoupdate - Name of the certificate & key pair on the Netscaler
* certfilename - Name of the certificate file which was created by the Let's Encrypt playbook (le_renew-cert.yml). By default this is {{ domain_name }}.pem.
* keyfilename - Name of the private key file which was created by the Let's Encrypt playbook (le_renew-cert.yml). By default this is {{ domain_name }}.key.
* certfilepath - Path to the certificate file created by the Let's Encrypt playbook (le_renew-cert.yml). By default this is "/opt/letsencrypt/certs/{{ domain_name }}.pem".
* keyfilepath - Path to the private key file created by the Let's Encrypt playbook (le_renew-cert.yml). By default this is "/opt/letsencrypt/keys/{{ domain_name }}.key".

### Execute Playbook

```
ansible-playbook ~/ansible-playbooks/renew-and-update-cert.yml --vault-password-file=/root/vault_key
```

### Scheduled Execution (via a Cron job)

Run the playbook at 1.15pm on every 2nd of the month.

```
15 13 2 * * ansible-playbook /root/ansible-playbooks/renew-and-update-cert.yml --vault-password-file=/root/vault_key
```