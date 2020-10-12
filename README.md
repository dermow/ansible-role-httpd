# ansible-role-httpd

## Requirements
* httpd >= 2.4.39

## Supported operating systems
* Debian based (tested on Debian 9)
* RedHat based (tested on Centos 7)

## Role variables
| parameter | description | default (RedHat) | default (Debian) |
|---|---|---|---|
| auto_generate_dhparams | generate Diffie Hellman parameters if not existing | False | False |
| dh_params_path | Path to Diffie Hellman parameters | {{ httpd_base_dir }}/dhparam.pem | {{ httpd_base_dir }}/dhparam.pem |
| dh_params_size | Size for Diffie Hellman parameters | 2048 | 2048 |
| httpd_base_config | basic config file | /etc/httpd/conf/httpd.conf | /etc/apche2/apache2.conf |
| httpd_base_dir | base dir for httpd | /etc/httpd | /etc/apache2 |
| httpd_conf_dir | base dir for other configurations | /etc/httpd/conf.d | /etc/apache2/conf-enabled |
| httpd_log_dir | log directory | /var/log/httpd | /var/log/apache2 |
| httpd_packages | needed packages to be installed | ['httpd', 'mod_ssl'] | ["apache2", "apache2-utils"] |
| httpd_service_name | service name | httpd | apache2 |
| httpd_service_path | service path | /usr/sbin | /usr/sbin |
| httpd_vhosts_dir | base dir for vhost configurations | /etc/httpd/conf.d | /etc/apache2/sites-enabled |
| **httpd_vhosts** | dictionary with all the vhost definitions | [] | []|
| SSLStaplingResponderTimeout | apache config 'SSLStaplingResponderTimeout' | 5 | 5 |
| SSLStaplingReturnResponderErrors | apache config 'SSLStaplingReturnResponderErrors' | Off | Off |
| SSLStaplingCache | apache config 'SSLStaplingCacheSSLStaplingCache' | shmcb:/var/run/ocsp(128000) | shmcb:/var/run/ocsp(128000) |



## vhost config
Each vhost entry in 'httpd_vhosts' supports the following parameters

| parameter | description | possible values | mandatory | default |
|---|---|---|---|---|
| extra_blocks | list of extra blocks to add to the config | text blocks | no | [] |
| name | vhost name | string | yes | none |
| http_port | vhost http_port | 1-65565 | no | 80 |
| https_port | vhost https_port | 1-65565 | no | 443 |
| ProxyPreserveHost | vhost config 'ProxyPreserveHost' | 'on', 'off' | omit |
| ProxyRequests | vhost config 'ProxyRequests' | 'on', 'off' | 'off' |
| proxy_rules | list of proxy rules | valid proxy rules | no | omit |
| redirect_http | if true an vhost for redirecting http to https will be created | true, false | no | false |
| ServerAdmin | vhost config 'ServerAdmin' | valid e-mail address | no | omit |
| ServerName | vhost config 'ServerName' | valid server name | no | vhost name |
| SSLCertificateFile | Path to SSL certificate | valid path | when type is SSL | none |
| SSLCertificateKeyFile | Path to SSL certificate key | valid path | when type is SSL | none |
| SSLCipherSuite | list of SSL ciphers | valid cipher list | no | see 'Ciphers (Default)' |
| SSLComporession | vhost config 'SSLCompression' | 'on', 'off' | no | 'off' |
| SSLHonorCipherOrder | vhost config 'SSLHonorCipherOrder' | 'on', 'off' | no | 'off' |
| SSLOpenSSLConfCmd  | list of SSLOpenSSLConfCmd lines | SSLOpenSSLConfCmdsd | no | see 'OpenSSLConfCmd (default)' |
| SSLOptions | vhost config 'SSLOptions' | valid apache SSLOptions | no | '+StrictRequire' |
| SSLProtocol | vhost config 'SSLProtocol' | valid SSLProtocol config | no | '-all + TLSv1.2' |
| SSLSessionTickets | vhost config 'SSLSessionTickets' | 'on', 'off' | no | 'off' |
| SSLUseStapling | vhost config 'SSLUseStapling' | 'on', 'off' | no | 'off' |
| type | vhost type | 'http', 'https' | yes | none |

### Ciphers (Default)
* TLS-CHACHA20-POLY1305-SHA256
* TLS-AES-256-GCM-SHA384
* ECDHE-RSA-AES256-GCM-SHA512
* DHE-RSA-AES256-GCM-SHA512
* ECDHE-RSA-AES256-GCM-SHA384
* DHE-RSA-AES256-GCM-SHA384

### OpenSSLConfCmd (default)
```` yml
- "Curves X448:secp521r1:secp384r1:prime256v1"
- "ECDHParameters secp384r1"
````

### proxy rules
````yaml
...
  proxy_rules:
    - target: "/"
      proxy: "http://127.0.0.1:8080/"
      reverse: "http://127.0.0.1:8080/"
...
````
### Example vhosts definition
````yaml
httpd_vhosts:
  - name: cloud.example.com
    type: https
    port: 443
    redirect_http: True
    ProxyRequests: "Off"
    ProxyPreserveHost: "On"
    proxy_rules:
      - target: "/"
        proxy: "http://127.0.0.1:8081/"
        reverse: "http://127.0.0.1:8081/"
    ServerAdmin: "you@example.com"
    LogFormatCombined: '%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\" vhost_combined'
    LogFormatCommon:  '%v %h %l %u %t \"%r\" %>s %b\" vhost_common'
    ErrorLog: "{{ httpd_log_dir }}/error.log"
    CustomLog: "{{ httpd_log_dir }}/access.log combined"
    SSLOptions: "+StrictRequire"
    SSLCertificateFile: "/etc/letsencrypt/live/cloud.example.com/fullchain.pem"
    SSLCertificateKeyFile: "/etc/letsencrypt/live/cloud.example.com/privkey.pem"
    extra_blocks:
      - name: docroot
        block: |
           <Directory /var/www/html/>
            Options +FollowSymlinks
            AllowOverride All
            <IfModule mod_dav.c>
                Dav off
            </IfModule>
            SetEnv HOME /var/www/html/nextcloud
            SetEnv HTTP_HOME /var/www/html/nextcloud
           </Directory>
      - name: headers
        block: |
           <IfModule mod_headers.c>
            Header always set Strict-Transport-Security "max-age=15768000; preload"
            Header set Referrer-Policy "strict-origin-when-cross-origin"
            Header set X-Content-Type-Options "nosniff"
            Header always set X-Frame-Options "SAMEORIGIN"
           </IfModule>
      - name: android_fix
        block: |
          RewriteEngine On
          RewriteCond %{HTTP:Authorization} ^(.*)
          RewriteRule .* - [e=HTTP_AUTHORIZATION:%1]

````
