ssh node01 then ssh node02
mkdir /etc/node_exporter/
touch /etc/node_exporter/config.yml
chmod 700 /etc/node_exporter
chmod 600 /etc/node_exporter/config.yml
chown -R nodeusr:nodeusr /etc/node_exporter

# edit node_exporter service
vi /etc/systemd/system/node_exporter.service

# change:
ExecStart=/usr/local/bin/node_exporter 
# to
ExecStart=/usr/local/bin/node_exporter --web.config=/etc/node_exporter/config.yml

# reload daemon and node_exporter service
systemctl daemon-reload
systemctl restart node_exporter
exit

# Configure Prometheus and Node servers to use authentication to communicate
ssh node01 then ssh node02

apt update
apt install apache2-utils -y

# Generate password hash
htpasswd -nBC 10 "" | tr -d ':\n'; echo
# It will ask for the password twice 
New password: 
Re-type new password: 

# Edit
vi /etc/node_exporter/config.yml
# Add below lines in it:
basic_auth_users:
  prometheus: hashed-password
# Restart node_exporter service then exit
systemctl restart node_exporter
exit
# You can verify the changes using "curl"
curl http://node01:9100/metrics

# return output should be Unauthorized

# Try to access the metrics using correct credentials now:
curl -u prometheus:secret-password http://node01:9100/metrics
curl -u prometheus:secret-password http://node02:9100/metrics

# Now, let's configure the Prometheus server to use authentication when scraping metrics from node servers.
vi /etc/prometheus/prometheus.yml

# Under - job_name: "nodes" add below lines:
basic_auth:
  username: prometheus
  password: "Your hashed password"
# Restart prometheus service
systemctl restart prometheus

# Configure node exporter to use TLS on both nodes i.e node01 and node02. You can generate a certificate and key using the below command:
ssh node01 the ssh node02

# Generate the certificate and the key
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout node_exporter.key -out node_exporter.crt -subj "/C=US/ST=California/L=Oakland/O=MyOrg/CN=localhost" -addext "subjectAltName = DNS:localhost"

mv node_exporter.crt node_exporter.key /etc/node_exporter/
# Change ownership
chown nodeusr.nodeusr /etc/node_exporter/node_exporter.key
chown nodeusr.nodeusr /etc/node_exporter/node_exporter.crt

vi /etc/node_exporter/config.yml
# Add below lines in this file:
tls_server_config:
  cert_file: node_exporter.crt
  key_file: node_exporter.key
systemctl restart node_exporter
exit
# You can verify the changes using "curl" command:
curl -u prometheus:secret-password -k https://node01:9100/metrics
# Let's configure Prometheus server to use HTTPS for scraping the node_exporter

# Copy the certificate from node01 to Prometheus server
scp root@node01:/etc/node_exporter/node_exporter.crt /etc/prometheus/node_exporter.crt
# Change certificate file ownership
chown prometheus.prometheus /etc/prometheus/node_exporter.crt
vi /etc/prometheus/prometheus.yml 
# Add below given lines under - job_name: "nodes"
  scheme: https
    tls_config:
      ca_file: /etc/prometheus/node_exporter.crt
      insecure_skip_verify: true
# Restart prometheus service
systemctl restart prometheus

# # Access the Prometheus UI using Prometheus button on the top bar
# Under Status -> Targets you will see that both nodes are getting scrapped over https endpoints
