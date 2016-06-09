# kerberos_ldap_environment
Vagrant box which provides a simple LDAP and Kerberos infrastructure for testing purposes

## Usage
* ``vagrant up``
* After the virtual machine is started you find some new files in your directory
 * krb5.conf: That defines how to read the Kerberos server and various other Kerberos related options
 * http_srv.keytab: Contains a ``HTTP/localhost``principal. This is the keytab you can use on the elasticsearch serverside together with Search Guard
 * All other *.keytab files: They can be used to logon via curl

## Configure Search Guard
* Copy http_srv.keytab into config/ folder of all elasticsearch nodes
* Add there lines to elasticsearch.yml on all nodes:
 * ``searchguard.kerberos.krb5_filepath: /path/to/kerberos_ldap_environment/krb5.conf``
 * ``searchguard.kerberos.acceptor_keytab_filepath: http_srv.keytab``
* Configure something like this in sg_config.yml:
 
 ```
       kerberos_auth_domain: 
        enabled: true
        order: 4
        http_authenticator:
          type: kerberos # NOT FREE FOR COMMERCIAL USE
          challenge: true
          config:
            # If true a lot of kerberos/security related debugging output will be logged to standard out
            krb_debug: false
            # Acceptor (Server) Principal name, must be present in acceptor_keytab_path file
            acceptor_principal: 'HTTP/localhost'
            # If true then the realm will be stripped from the user name
            strip_realm_from_principal: true
        authentication_backend:
          type: noop
 ```

## Login via curl
* ``export KRB5_CONFIG=krb5.conf``
* ``kinit -t rikerw.keytab rikerw``
* ``curl --negotiate -k -u: https://localhost:9200/_searchguard/authinfo``
