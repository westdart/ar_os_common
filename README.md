# ar_os_common
Ansible Role that holds various task files that implement a specific 
function on Openshift.

Each of these functions could in turn be refactored into a specific 
ansible role, but currently it suffices to leave them here.

## Task files
As this role contains several functions based on task files, each is 
listed with its own requirements

### main.yml
Copies list of templates stored in 'files' to known location

#### Variables
| Variable               | Description                         | Default          |
| --------               | -----------                         | -------          |
| ar_os_common_selection | List of required templates required | all are selected |
| ar_os_common_dest      | Directory in which to place files   | ''               |

------------------------------------------------------------------------

### login.yml
Login to OpenShift. Ensures forced login when needed by comparing the
currently logged in cluster (if any) with the required one.

## Requirements
- oc command line is available

#### Dependencies
- casl-ansible/roles/openshift-login

#### Parameters
| Variable                           | Description                                                                                                 | Default        |
| --------                           | -----------                                                                                                 | -------        |
| ar_os_common_openshift_login_url   | URL endpoint to send login credentials (used in 'oc login'                                                  | null (invalid) |
| ar_os_common_openshift_username    | Username of user to login                                                                                   | undefined      |
| ar_os_common_openshift_token       | Token for user to enable login (username is strictly not required with token, but is enforced in this role) | '' (invalid)   |
| ar_os_common_openshift_password    | Password for user to enable login                                                                           | '' (invalid)   |
| ar_os_common_openshift_login_force | Whether to force a login                                                                                    | false          |

#### Defaults

| Variable                                     | Description                                                             | Default                                                                                                          |
| --------                                     | -----------                                                             | -------                                                                                                          |
| ar_os_common_openshift_login_credentials     | Openshift login credentials provided as a map/dict                      | omitted unless 'openshift_login_credentials' is defined                                                          |
| ar_os_common_login_basic_assertions          | List of assertions made before execution                                |                                                                                                                  |
| ar_os_common_login_secret_map_assertions     | Assertions made if map/dict of credentials used                         | Ensures that the login URL is a key in the map and that entry contains a username and either a password or token |
| ar_os_common_login_secret_map_assertions_msg | Message given on an assertion failure when checking credential map/dict | See defaults.yml                                                                                                 |
| ar_os_common_login_credential_assertions     | Assertions made on supplied credentials                                 | Ensures that either a password or a token are provided alongside a username                                      |
| ar_os_common_login_credential_assertions_msg | Message given on an assertion failure when checking credentials         | See defaults.yml                                                                                                 |

#### Secrets
The following variables should be provided through an encrypted source:
- ar_os_common_openshift_login_credentials
- ar_os_common_openshift_token
- ar_os_common_openshift_password

#### External variables
The following external variables are used within the role:

| Variable                    | Description                                                           | Default |
| --------                    | -----------                                                           | ------- |
| openshift_login_credentials | Map/dict of Openshift login URLS to usernames and passwords or tokens | N/A     |

For ar_os_common_openshift_login_credentials the structure is:
```
  '<openshift login url>':
    openshift_user: '<Username attributed to login>'
    openshift_password: '<Password for openshift_user to login>'
    openshift_token: '<Token to enable Openshift login>'    
```
Where either 'openshift_password' or 'openshift_token' should be 
specified. Which one wins out if both are specified is deferred to 
casl-ansible/roles/openshift-login

#### Example Playbook

In this example, a vault named, and accessible by, 'openshift-credentials.vault'
is used to supply a map/dict of Openshift login URLs to credentials.
This specific vault must have a top level entry named 'https://master.openshift.local:8443'
and that entry must have a 'openshift_user' entry and either a
'openshift_password' or 'openshift_token' entry.

```
- hosts: localhost
  vars_files:
    - openshift-credentials.vault
    
  tasks:
    - name: Login to OpenShift
      include_role:
        name: ar_os_common
        tasks_from: login
      vars:
        ar_os_common_openshift_login_url: "https://master.openshift.local:8443"
        
```

------------------------------------------------------------------------

### get-cert.yml
Obtain a certificate chain from a server (calls validate-cert.yml)

#### Requirements
- openssl is installed

#### Parameters
| Variable               | Description                                           | Default        |
| --------               | -----------                                           | -------        |
| ar_os_common_cert_host | The host from which to obtain the certificate         | null (invalid) |
| ar_os_common_cert_port | The port on which the TLS enabled server is listening | null (invalid) |
| ar_os_common_cert_path | The path where the certifiacte should be stored       | null (invalid) |

#### Defaults

| Variable                     | Description                              | Default                                                                                        |
| --------                     | -----------                              | -------                                                                                        |
| ar_os_common_cert_assertions | List of assertions made before execution | Ensures ar_os_common_cert_host, ar_os_common_cert_port and ar_os_common_cert_path are provided |

#### Example Playbook

```
- hosts: localhost
    
  tasks:
    - name: Include common role to get certificate
      include_role:
        name: ar_os_common
        tasks_from: get-cert
      vars:
        ar_os_common_cert_host: "master.openshift.local"
        ar_os_common_cert_port: "8443"
        ar_os_common_cert_path: "/tmp/master.openshift.local.crt"
        
```

------------------------------------------------------------------------

### place-docker-certs.yml
Only used on Docker version < 1.13 - adds the certs to the docker 
configuration. 

Takes a certificate file located at 'ar_os_common_cert_path' and updates
docker to make the cert trusted.

## Requirements
- Docker installed on target hosts

#### Parameters
See get-cert.yml

#### Defaults
See get-cert.yml

#### Example Playbook

```
- hosts: docker_hosts
    
  tasks:
    - name: Include common role to get certificate
      include_role:
        name: ar_os_common
        tasks_from: place-docker-certs
      vars:
        ar_os_common_cert_host: "master.openshift.local"
        ar_os_common_cert_port: "8443"
        ar_os_common_cert_path: "/tmp/master.openshift.local.crt"
```

------------------------------------------------------------------------

### place-registry-certs.yml
Copies registry certs to required location on target hosts and triggers 
update-ca-trust and docker restarts to make the certificate trusted on
the target machine.

#### Parameters
See get-cert.yml

#### Defaults
See get-cert.yml

#### Example Playbook

```
- hosts: target_hosts
    
  tasks:
    - name: Include common role to get certificate
      include_role:
        name: ar_os_common
        tasks_from: place-registry-cert
      vars:
        ar_os_common_cert_host: "master.openshift.local"
        ar_os_common_cert_port: "8443"
        ar_os_common_cert_path: "/tmp/master.openshift.local.crt"
```

------------------------------------------------------------------------

### registry-list.yml
Outputs list of upstream registries to '/etc/containers/registries.update(!conf)' 
(untested)

#### Parameters
| Variable                   | Description               | Default   |
| --------                   | -----------               | -------   |
| ar_os_common_registry_list | List of docker registries | undefined |

#### Example Playbook

```
- hosts: target_hosts    
  tasks:
    - name: Include common role to get certificate
      include_role:
        name: ar_os_common
        tasks_from: registry-list
      vars:
        ar_os_common_registry_list:
          - docker1.local
          - docker2.local
        
```

------------------------------------------------------------------------

### validate-cert.yml
Validate a certificate for expiry and hostname applicability.
This fails the play if the cert is not validated.

#### Requirements
- openssl is installed

#### Parameters
See get-cert.yml

#### Defaults
See get-cert.yml

#### Example Playbook

```
- hosts: target_hosts
    
  tasks:
    - name: Include common role to get certificate
      include_role:
        name: ar_os_common
        tasks_from: validate-cert
      vars:
        ar_os_common_cert_host: "master.openshift.local"
        ar_os_common_cert_port: "8443"
        ar_os_common_cert_path: "/tmp/master.openshift.local.crt"
```

------------------------------------------------------------------------


## License

MIT / BSD

## Author Information

This role was created in 2020 by David Stewart (dstewart@redhat.com)
