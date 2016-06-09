#!/bin/sh
$script = <<SCRIPT
export DEBIAN_FRONTEND=noninteractive
echo "Hostname: $(hostname -f)"
cat /etc/hosts*
echo "Update packages"
sudo apt-get -yqq update > /dev/null 2>&1

#http://openstack.prov12n.com/quiet-or-unattended-installing-openldap-on-ubuntu-14-04/
echo "Install OpenLDAP, password is: password"
echo -e " \
slapd slapd/root_password password password
slapd slapd/root_password_again password password
slapd slapd/internal/adminpw password password
slapd slapd/internal/generated_adminpw password password
slapd slapd/password2 password password
slapd slapd/password1 password password
slapd slapd/allow_ldap_v2 boolean false
slapd slapd/no_configuration boolean false
" | sudo debconf-set-selections
sudo apt-get install -y slapd ldap-utils  > /dev/null 2>&1


cat > /tmp/base.ldif << "EOF"
dn: ou=people,dc=example,dc=com
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=example,dc=com
objectClass: organizationalUnit
ou: groups
EOF

ldapadd -x -D cn=admin,dc=example,dc=com -w password -f /tmp/base.ldif

cat > /tmp/users.ldif << "EOF"
dn: cn=Michael Jackson,ou=people,dc=example,dc=com
objectclass: inetOrgPerson
cn: Michael Jackson
sn: jackson
uid: jacksonm
userpassword: jacksonm
mail: jacksonm@example.com
description: cn=dummyempty,ou=groups,dc=example,dc=com

dn: cn=Deanna Troi,ou=people,dc=example,dc=com
objectclass: inetOrgPerson
cn: Deanna Troi
sn: Troi
uid: troid
userpassword: troid
mail: troid@example.com

dn: cn=William Riker,ou=people,dc=example,dc=com
objectclass: inetOrgPerson
cn: William Riker
sn: Riker
uid: rikerw
userpassword: rikerw
mail: rikerw@example.com

dn: cn=ldaprole,ou=groups,dc=example,dc=com
objectClass: groupOfUniqueNames
cn: ldaprole
uniqueMember: cn=Michael Jackson,ou=people,dc=example,dc=com
uniqueMember: cn=Deanna Troi,ou=people,dc=example,dc=com
uniqueMember: cn=William Riker,ou=people,dc=example,dc=com
EOF

ldapadd -x -D cn=admin,dc=example,dc=com -w password -f /tmp/users.ldif

sudo slapcat

echo "Install OpenSSL and tools"
sudo apt-get -yqq install ntp ntpdate haveged openssl wget git > /dev/null 2>&1
#entropy generator
haveged -w 1024 > /dev/null 2>&1
echo "Install MIT Kerberos"
#https://github.com/sukharevd/hadoop-install/blob/master/bin/install-kerberos.sh
sudo debconf-set-selections <<< 'krb5-admin-server krb5-admin-server/kadmind boolean true'
sudo debconf-set-selections <<< 'krb5-admin-server krb5-admin-server/newrealm note'

mkdir -p /var/log/kerberos > /dev/null 2>&1
touch /var/log/kerberos/krb5libs.log > /dev/null 2>&1
touch /var/log/kerberos/krb5kdc.log > /dev/null 2>&1
touch /var/log/kerberos/kadmind.log > /dev/null 2>&1

DNS_ZONE="example.com"
REALM=$(echo "$DNS_ZONE" | tr '[:lower:]' '[:upper:]')
KERBEROS_FQDN="krbldap.example.com"

cat > /etc/krb5.conf << "EOF"
[realms]
    ${REALM} = {
        kdc = ${KERBEROS_FQDN}:8888
        kdc = localhost:8888
        kdc = 127.0.0.1:8888
        kdc = ${KERBEROS_FQDN}:88
        kdc = localhost:88
        kdc = 127.0.0.1:88
        #admin_server = ${KERBEROS_FQDN}:8749
        default_domain = ${DNS_ZONE}
    }

[domain_realm]
    .${DNS_ZONE} = ${REALM}
    ${DNS_ZONE} = ${REALM}

[libdefaults]
    default_realm = ${REALM}
    dns_lookup_realm = false
    dns_lookup_kdc = false
    forwardable=true
    dns_canonicalize_hostname = false
    rdns = false
    ignore_acceptor_hostname = true
    allow_weak_crypto = true

#[kdc]
#    profile = /etc/krb5kdc/kdc.conf

#[logging]
#    default = FILE:/var/log/kerberos/krb5libs.log
#    kdc = FILE:/var/log/kerberos/krb5kdc.log
#    admin_server = FILE:/var/log/kerberos/kadmind.log
EOF
sed -i -e 's/${KERBEROS_FQDN}/'$KERBEROS_FQDN'/g' /etc/krb5.conf
sed -i -e 's/${DNS_ZONE}/'$DNS_ZONE'/g' /etc/krb5.conf
sed -i -e 's/${REALM}/'$REALM'/g' /etc/krb5.conf

cat /etc/krb5.conf
cp /etc/krb5.conf /vagrant/krb5.conf

mkdir -p /etc/krb5kdc
mkdir -p /var/lib/krb5kdc

cat > /etc/krb5kdc/kdc.conf << "EOF"
[libdefaults]
        debug = true
        
[logging]
    kdc = FILE:/var/log/krb5kdc.log

[kdcdefaults]
    kdc_ports = 749,88
    kdc_tcp_ports = 88

[realms]
    ${REALM} = {
        database_name = /var/lib/krb5kdc/principal
        admin_keytab = FILE:/etc/krb5kdc/kadm5.keytab
        acl_file = /etc/krb5kdc/kadm5.acl
        key_stash_file = /etc/krb5kdc/stash
        
        kdc_ports = 749,88
        max_life = 10h 0m 0s
        max_renewable_life = 7d 0h 0m 0s
        master_key_type = des3-hmac-sha1
        #supported_enctypes = aes256-cts:normal arcfour-hmac:normal des3-hmac-sha1:normal des-cbc-crc:normal des:normal des:v4 des:norealm des:onlyrealm des:afs3
        default_principal_flags = +preauth
    }
EOF
sed -i -e 's/${REALM}/'$REALM'/g' /etc/krb5kdc/kdc.conf

cat /etc/krb5kdc/kdc.conf

sudo apt-get -qqy install krb5-{admin-server,kdc}
sudo kdb5_util create -s -P aaaBBBccc123
sudo /etc/init.d/krb5-kdc restart
sudo /etc/init.d/krb5-admin-server restart
sudo /etc/init.d/slapd restart

sudo /usr/sbin/kadmin.local -q 'addprinc -randkey HTTP/localhost'
sudo /usr/sbin/kadmin.local -q "ktadd -k /tmp/http_srv.keytab  HTTP/localhost"

sudo /usr/sbin/kadmin.local -q 'addprinc -randkey testuser1'
sudo /usr/sbin/kadmin.local -q "ktadd -k /tmp/testuser1.keytab testuser1"

sudo /usr/sbin/kadmin.local -q 'addprinc -randkey adm'
sudo /usr/sbin/kadmin.local -q "ktadd -k /tmp/adm.keytab adm"

sudo /usr/sbin/kadmin.local -q 'addprinc -randkey rikerw'
sudo /usr/sbin/kadmin.local -q "ktadd -k /tmp/rikerw.keytab rikerw"

cp /tmp/*.keytab /vagrant/

sudo kinit -V -kt /tmp/testuser1.keytab testuser1

echo "export KRB5_CONFIG=krb5.conf"
echo "kinit -t testuser1.keytab testuser1"

SCRIPT
# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.hostname = "krbldap.example.com"
  config.vm.network :forwarded_port, guest: 88, host: 8888, protocol: 'udp'
  config.vm.network :forwarded_port, guest: 88, host: 8888, protocol: 'tcp'
  config.vm.network :forwarded_port, guest: 389, host: 8389
  config.vm.network :forwarded_port, guest: 636, host: 8636

  config.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--cpus", "1", "--memory", "1024"]
  end
  config.vm.provision "shell", inline: $script
end