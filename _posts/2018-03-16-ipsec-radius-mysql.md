---
published: false
---
## IPsec VPN server with RADIUS and mysql on Debian 9

### System configuration
`echo 1 > /proc/sys/net/ipv4/ip_forward`  
`iptables`

### Install packages
We will need following packages:
1. strongswan - IPsec server
2. freeradius - RADIUS server
3. freeradius-mysql - modul for freeradius to integrate with mysql
4. libcharon-extra-plugins - modul for strongswan to integrate witg RADIUS server
5. mysql-server - not exactly mysql but mariadb

When we know what to install, lets do it:  
`$ sudo apt install strongswan freeradius freeradius-mysql libcharon-extra-plugins`


### Basic configuration
First we configure simple IKEv1 Main Mode with PSK + Xauth, where users will be authenticated by RADIUS and stored in mysql.

#### IPsec configuration 

```
# /etc/ipsec.conf - strongSwan IPsec configuration file

config setup
	charondebug="dmn 2, mgr 2, ike 2, chd 2, job 1, cfg 2, knl 2, net 2, enc 1, lib 2"

conn %default
	left=1.1.1.1				# server IP
    leftid=@vpn.ipsec.net
	right=%any					# jakie IP klienta moze sie do nas laczyc
	ikelifetime=60m
	keylife=20m			# synonym for lifetime
	rekeymargin=3m
	keyingtries=1
	keyexchange=ikev1
	dpddelay=30s
	dpdtimeout=150s
	ike=aes128-sha256-modp2048
	esp=aes128-sha256-modp2048
	auto=add					# samo zaladowanie tej definicji vpn'a

conn rw
	leftsubnet=172.26.0.0/16	# adresacja serwera, np secondary IP na gl. interfejsie
	leftauth=psk				# uwierzytelnienie serwera w fazie 1
	rightsourceip=%radius		# jaki dac IP klientowi (virtual IP; daj z radiusa)
	rightauth=psk				# jak klient ma sie uwierzytelnic w fazie 1
	rightauth2=xauth-eap		# jakie ma byc uwierzytelnienie w fazie 2
```

```
# /etc/ipsec.secrets - strongSwan IPsec secrets file

@vpn.ipsec.net : PSK randompass
1.1.1.1 : PSK randompass
```

#### IPsec integration with RADIUS
```
# strongswan.conf - strongSwan configuration file


charon {
	load_modular = yes
	plugins {
		include strongswan.d/charon/*.conf
		eap-radius {
			accounting = yes
			servers {
				radius_srv {
					secret = testing123
					address = 127.0.0.1
					auth_port = 1812		
					acct_port = 1813
				}			
			}
		}
	}
	dns1 = 172.26.0.1
	dns2 = 9.9.9.9
}

include strongswan.d/*.conf
```

#### RADIUS configuration
```
/etc/freeradius/3.0/clients.conf
client localhost {
	(...)
	secret = testing123
    (...)
}
```

#### mysql configuration
```
$ sudo mysql_secure_installation
Enter current password for root (enter for none):
Set root password? [Y/n]
Remove anonymous users? [Y/n]
Disallow root login remotely? [Y/n]
Remove test database and access to it? [Y/n]
Reload privilege tables now? [Y/n]

sudo mysql -u root -p
mysql> use mysql;
mysql> update user set plugin='mysql_native_password' where user='root';
mysql> flush privileges; 
mysql> quit;

mysql -u root -p
mysql> create database radius;
mysql> grant all privileges on radius.* to radius@localhost identified by "radius_db_pass";
mysql> flush privileges;
mysql> use radius;
mysql> source /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
# add username|password :
mysql> INSERT INTO radcheck VALUES (null, 'user1','Cleartext-Password',':=','pass1');
mysql> INSERT INTO radcheck VALUES (null, 'user2','Cleartext-Password',':=','pass2');
mysql> INSERT INTO radreply VALUES (null, 'user1','Framed-IP-Address',':=','172.26.2.10');
mysql> INSERT INTO radreply VALUES (null, 'user2','Framed-IP-Address',':=','172.26.2.20');
mysql> quit;
#systemctl start/enable mysql?
```

#### RADIUS integration with mysql

```
/etc/freeradius/3.0/sites-enabled/default
# delete lines with 'files' and 'unix', uncomment those with 'sql'
server default {
	listen {
		type = auth
		ipv4addr = 127.0.0.1
		port = 0
		limit {
		      max_connections = 16
		      lifetime = 0
		      idle_timeout = 30
		}
	}
	listen {
		type = acct
		ipv4addr = 127.0.0.1
		port = 0
		limit {
		}
	}
    
	authorize {
		filter_username
		preprocess
		chap
		mschap
		digest
		suffix
		eap {
			ok = return
		}
		files
		-sql
		-ldap
		expiration
		logintime
		pap
	}
    
	authenticate {
		Auth-Type PAP {
			pap
		}
        
		Auth-Type CHAP {
			chap
		}
        
		Auth-Type MS-CHAP {
			mschap
		}
        
		mschap
		digest
		eap
	}
    
	preacct {
		preprocess
		acct_unique
		suffix
		files
	}
    
	accounting {
		detail
		unix
		-sql
		exec
		attr_filter.accounting_response
	}
    
	session {
	}
    
	post-auth {
		update {
			&reply: += &session-state:
		}
        
		-sql
		exec
		remove_reply_message_if_eap
		Post-Auth-Type REJECT {
			-sql
			attr_filter.access_reject
			eap
			remove_reply_message_if_eap
		}
	}
    
	pre-proxy {
	}
    
	post-proxy {
		eap
	}
}
```

```
/etc/freeradius/3.0/sites-enabled/inner-tunnel
# delete lines with 'files' and 'unix', uncomment those with 'sql'
server inner-tunnel {
	listen {
	       ipaddr = 127.0.0.1
	       port = 18120
	       type = auth
	}
    
	authorize {
		filter_username
		chap
		mschap
		suffix
		update control {
			&Proxy-To-Realm := LOCAL
		}
        
		eap {
			ok = return
		}
        
		files
		-sql
		-ldap
		expiration
		logintime
		pap
	}
    
	authenticate {
		Auth-Type PAP {
			pap
		}
        
		Auth-Type CHAP {
			chap
		}
        
		Auth-Type MS-CHAP {
			mschap
		}
        
		mschap
		eap
	}
    
	session {
		radutmp
		sql
	}
    
	post-auth {
		-sql
		Post-Auth-Type REJECT {
			-sql
			attr_filter.access_reject
			update outer.session-state {
				&Module-Failure-Message := &request:Module-Failure-Message
			}
		}
	}
    
	pre-proxy {
	}
    
	post-proxy {
		eap
	}
} # inner-tunnel server block
```

```
/etc/freeradius/3.0/mods-enabled# ln -s ../mods-available/sql sql
```

```
/etc/freeradius/3.0/mods-enabled/sql

sql {
	driver = "rlm_sql_mysql"
	dialect = "mysql"
	server = "localhost"
	port = 3306
	login = "radius"
	password = "radius_db_pass"
	radius_db = "radius"
	acct_table1 = "radacct"
	acct_table2 = "radacct"
	postauth_table = "radpostauth"
	authcheck_table = "radcheck"
	groupcheck_table = "radgroupcheck"
	authreply_table = "radreply"
	groupreply_table = "radgroupreply"
	usergroup_table = "radusergroup"
	delete_stale_sessions = yes
	pool {
		start = ${thread[pool].start_servers}
		min = ${thread[pool].min_spare_servers}
		max = ${thread[pool].max_servers}
		spare = ${thread[pool].max_spare_servers}
		uses = 0
		retry_delay = 30
		lifetime = 0
		idle_timeout = 60
	}
	client_table = "nas"
	group_attribute = "SQL-Group"
	$INCLUDE ${modconfdir}/${.:name}/main/${dialect}/queries.conf
}
```

### IPsec authentication with certificates
#### Certificates [not in this palce!]
```
ipsec pki --gen --outform pem > ca.pem
ipsec pki --self --in ca.pem --dn "C=CN, O=Alipay, CN=singapo.kiritostudio.com" --ca --outform pem >ca.cert.pem
ipsec pki --gen --outform pem > server.pem
ipsec pki --pub --in server.pem | ipsec pki --issue --cacert ca.cert.pem --cakey ca.pem --dn "C=CN, O=Alipay, CN=singapo.kiritostudio.com" --san="singapo.kiritostudio.com" --flag serverAuth --flag ikeIntermediate --outform pem > server.cert.pem
ipsec pki --gen --outform pem > client.pem
ipsec pki --pub --in client.pem | ipsec pki --issue --cacert ca.cert.pem --cakey ca.pem --dn "C=CN, O=Alipay, CN=singapo.kiritostudio.com" --outform pem > client.cert.pem
openssl pkcs12 -export -inkey client.pem -in client.cert.pem -name "client" -certfile ca.cert.pem -caname "singapo.kiritostudio.com"  -out client.cert.p12
cp -r ca.cert.pem /usr/local/etc/ipsec.d/cacerts/
cp -r server.cert.pem /usr/local/etc/ipsec.d/certs/
cp -r server.pem /usr/local/etc/ipsec.d/private/
cp -r client.cert.pem /usr/local/etc/ipsec.d/certs/
cp -r client.pem  /usr/local/etc/ipsec.d/private/
```

### IPsec configuriation for different clients








