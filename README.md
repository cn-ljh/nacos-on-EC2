# nacos-on-EC2

Deployment Guide Nacos cluster on Amazon EC2

## Run the Cloudformation Template
This template will create:
1. an Aurora MySQL cluster
2. 3 EC2 instances
3. 2 NLB(one for internet access, one for internal access) with the target group to Nacos EC2 cluster
4. IAM roles for EC2 to access SSM

Output info: 
1. External/Internal NLB DNS name
2. EC2 instances IP address
3. AuroraDB Cluster Endpoint

## Config and Start EC2 Nacos Service
1. create System manager parameter store
   Example:
   name: nacos-cluster
   value:
'''text
   #ip: port
   <EC2Instance1PrivateIP>:8848
   <EC2Instance2PrivateIP>:8848
   <EC2Instance2PrivateIP>:8848
3. Run Command using AWS-RunShellScript document:
'''shell
   # Get cluster config from Parameter Store
   # Remember to replace the parameter name and region code that match your deployment
  config_content=$(aws ssm get-parameter --name nacos-cluster.conf --query Parameter.Value --output text --region cn-northwest-1)
  
  
  # Create backup of existing config
  cd /opt/nacos/conf
  if [[ -f cluster.conf ]]; then
      sudo cp cluster.conf cluster.conf.backup || true
  else
      touch cluster.conf
  fi
  
  # Write new config
  echo "$config_content" | sudo tee /opt/nacos/conf/cluster.conf
  
  # Set proper ownership and permissions
  sudo chmod 644 /opt/nacos/conf/cluster.conf
  
  cat << 'EOF' > /etc/systemd/system/nacos.service
  [Unit]
  Description=Nacos Service
  After=network.target
  
  [Service]
  Type=forking
  ExecStart=/opt/nacos/bin/startup.sh
  ExecStop=/opt/nacos/bin/shutdown.sh
  RemainAfterExit=yes
  
  [Install]
  WantedBy=multi-user.target
  EOF
  
  # Set proper permissions for the service file
  chmod 644 /etc/systemd/system/nacos.service
  
  # Reload systemd to recognize the new service
  systemctl daemon-reload
  
  # Enable the service to start on boot
  systemctl enable nacos
  
  # Start the service
  systemctl start nacos
'''
## Start Using nacos service
Access via internet with public ELB DNS Name
Access via internal with private ELB DNS Name
