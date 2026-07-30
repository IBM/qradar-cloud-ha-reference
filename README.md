# QRadar High Availability Reference Architecture - CloudFormation Template

This CloudFormation template deploys the complete VPC infrastructure and EC2 instances for a highly available QRadar SIEM deployment on AWS. See [Network Archotecture](NETWORK_ARCHITECTURE.md) for details of the infrastructure design.

## Architecture Overview

The template creates:

- **VPC**: `10.0.0.0/16` named "qradar-ha-reference"
- **2 Public Subnets**: `10.0.1.0/24` and `10.0.2.0/24` (one per AZ)
- **2 Private Subnets**: `10.0.11.0/24` and `10.0.12.0/24` (one per AZ)
- **Internet Gateway**: For public subnet internet access
- **2 NAT Gateways**: One per AZ for private subnet internet access
- **Route Tables**: Properly configured for public and private subnets
- **IAM Role & Instance Profile**: For AWS Systems Manager (SSM) access to EC2 instances
- **Security Groups**: For Console, Event Processor, Data Nodes, NLBs, and internal QRadar communication
- **2 Network Load Balancers**:
  - Console NLB (HTTPS/443)
  - Event Processor NLB (Syslog TCP/UDP 514, TLS 6514)
- **8 EC2 Instances** (4 primary in AZ1, 4 secondary in AZ2):
  - Console Primary & Secondary
  - Event Processor Primary & Secondary
  - Data Node 1 Primary & Secondary
  - Data Node 2 Primary & Secondary
  - All instances configured with SSM access for secure management

## Prerequisites

Before deploying this template, you need:

1. **AWS Account** with appropriate permissions
2. **EC2 Key Pair** in the target region for SSH access
3. **QRadar AMI** that supports HA (7.6.0+) or active QRadar subscription
4. **AWS CLI** or access to AWS Console

## Parameters

The template accepts the following parameters:

| Parameter | Description | Default | Required |
|-----------|-------------|---------|----------|
| `LatestAmiId` | AMI ID for EC2 instances | Latest QRadar | No |
| `KeyPairName` | EC2 Key Pair name for SSH access | - | Yes |
| `ConsoleInstanceType` | Instance type for Console instances | m5.2xlarge | No |
| `EPInstanceType` | Instance type for Event Processor instances | m5.2xlarge | No |
| `DNInstanceType` | Instance type for Data Node instances | m5.2xlarge | No |
| `DataVolumeSize` | Size of additional EBS data volume in GB (gp3, encrypted) | 300 | No |

## Deployment Instructions

### Using AWS Console

1. Navigate to **CloudFormation** in the AWS Console
2. Click **Create Stack** → **With new resources**
3. Choose **Upload a template file**
4. Upload `qradar-ha-infrastructure.yaml`
5. Click **Next**
6. Enter a **Stack name** (e.g., `qradar-ha-stack`)
7. Fill in the required parameters:
   - Select your **KeyPairName**
   - Optionally adjust instance types
   - Optionally specify a QRadar AMI ID
8. Click **Next** through the remaining screens
9. Check the acknowledgment box for IAM resources
10. Click **Create Stack**

### Using AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name qradar-ha-stack \
  --template-body file://qradar-ha-infrastructure.yaml \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue=your-key-pair-name \
    ParameterKey=ConsoleInstanceType,ParameterValue=m5.2xlarge \
    ParameterKey=EPInstanceType,ParameterValue=m5.2xlarge \
    ParameterKey=DNInstanceType,ParameterValue=m5.2xlarge \
    ParameterKey=DataVolumeSize,ParameterValue=300 \
  --capabilities CAPABILITY_NAMED_IAM
```

**Note**: The `--capabilities CAPABILITY_NAMED_IAM` flag is required because the template creates IAM roles for SSM access.

### Monitoring Deployment

Monitor the stack creation progress:

```bash
aws cloudformation describe-stacks \
  --stack-name qradar-ha-stack \
  --query 'Stacks[0].StackStatus'
```

Or watch events in real-time:

```bash
aws cloudformation describe-stack-events \
  --stack-name qradar-ha-stack \
  --max-items 10
```

## Stack Outputs

After successful deployment, the stack provides the following outputs:

### Network Resources
- `VPCId`: VPC identifier
- `PublicSubnet1Id`, `PublicSubnet2Id`: Public subnet IDs
- `PrivateSubnet1Id`, `PrivateSubnet2Id`: Private subnet IDs

### Load Balancers
- `ConsoleNLBDNSName`: DNS name for Console access
- `EPNLBDNSName`: DNS name for Event Processor (log ingestion)
- `ConsoleAccessURL`: Full HTTPS URL for Console access
- `SyslogEndpointTCP`: Syslog TCP endpoint
- `SyslogEndpointUDP`: Syslog UDP endpoint
- `SyslogEndpointTLS`: Syslog TLS endpoint

### EC2 Instances
- Instance IDs for all 8 QRadar components
- Private IP addresses for Console and Event Processor instances

### Retrieving Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name qradar-ha-stack \
  --query 'Stacks[0].Outputs'
```

## Accessing EC2 Instances

### AWS Systems Manager (SSM) Session Manager (Recommended)

All EC2 instances are configured with AWS Systems Manager access, allowing secure shell access without SSH keys or bastion hosts:

```bash
# List all instances
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=qradar-*" \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value|[0],State.Name]' \
  --output table

# Connect to an instance using SSM Session Manager
aws ssm start-session --target i-1234567890abcdef0

# Or use the AWS Console:
# EC2 → Instances → Select instance → Connect → Session Manager
```

**Benefits of SSM Session Manager**:
- No need for SSH keys or bastion hosts
- No need to open SSH ports in security groups
- All session activity is logged to CloudWatch
- Supports port forwarding and tunneling
- Works with instances in private subnets

### Traditional SSH Access

If you prefer SSH access:

1. Deploy a bastion host in a public subnet
2. Configure security groups to allow SSH from the bastion
3. Use SSH key pairs to connect

```bash
# SSH via bastion host
ssh -i your-key.pem -J ec2-user@bastion-ip ec2-user@private-instance-ip
```

## Post-Deployment Configuration

After the infrastructure is deployed, you need to:

1. **Initialize Overlay Network and Console** (as root):
   - initialize the overlay network (`192.168.60.0/23`) on the console primary instance (`/opt/vrouter/bin/vrouter/router_management.sh -i 192.168.60.0/23`)
   - get the private IP of the local eth0 interface(`export PRIVATEIP=$(ip -br addr show eth0 | awk '{print $3}' | cut -d/ -f1)`)
   - add the console primary to the overlay network (`/opt/vrouter/bin/vrouter/router_management.sh -a $PRIVATEIP -n 2`)
   - after a minute there will be a loss of connectivity, just wait for it to return before proceeding
   - setup the QRadar console as normal (`/root/setup 3199`)

2. **Set Up Overlay Network on Other Instances** (as root):
   - On each instance, run:
     - `sed -i -e 's/^PasswordAuthentication.*/PasswordAuthentication yes/g' /etc/ssh/sshd_config`
     - `systemctl restart sshd`
     - `passwd` a root password is needed during setup
     - `ip -br addr show eth0 | awk '{print $3}' | cut -d/ -f1`
   - Return to the console and run: `/opt/vrouter/bin/vrouter/router_management.sh -a PRIVATEIP -n 2` where PRIVATEIP is copied from the last command above
   - Use the following order to visit hosts to make addressing simpler:
     - Console secondary (will be 192.168.60.4)
     - EP primary (will be 192.168.60.6)
     - EP secondary (will be 192.168.60.8)
     - DN1 primary  (will be 192.168.60.10)
     - DN1 secondary (will be 192.168.60.12)
     - DN2 primary (will be 192.168.60.14)
     - DN2 secondary  (will be 192.168.60.16)

2. **Configure QRadar Software on Other instances** (as root):
   - Console secondary: `/root/setup 500 CONSOLE`
   - EP primary: `/root/setup 1899`
   - EP secondary: `/root/setup 500 NONCONSOLE`
   - DataNode primaries: `/root/setup 1400`
   - DataNode secondaries: `/root/setup 500 DATANODE`
   - wait for setups to complete

3. **Configure QRadar HA**:
   - Use the HA Wizard in the QRadar Admin UI to add secondaries
   - Assuming you followed the recommended order above, the primary and secondary IPs will be:
     - Console: `192.168.60.3` (primary) and `192.168.60.4` (secondary)
     - EP: `192.168.60.7` (primary) and `192.168.60.8` (secondary)
     - DataNodes: `192.168.60.11` (primary) and `192.168.60.12` (secondary), `192.168.60.15` (primary) and `192.168.60.16` (secondary)
   - If the above order was not followed, adjust the IP addresses accordingly in the HA Wizard. As a rule, the new primary IP will be the orginal +1

## Security Considerations

### Security Groups

The template creates the following security groups:

- **Console Security Group**: Allows HTTPS (`443`) from Console NLB, SSH from VPC
- **EP Security Group**: Allows Syslog (`514, 6514`) from EP NLB, SSH from VPC
- **Data Node Security Group**: Allows SSH from VPC
- **QRadar Internal Security Group**: Allows all traffic between QRadar instances (for overlay network and IPSec)
- **Console NLB Security Group**: Allows HTTPS (`443`) from internet
- **EP NLB Security Group**: Allows Syslog (`514, 6514`) from VPC

### Recommended Enhancements

Since this deployment is meant as a proof of concept or demonstration only, consider these security enhancements before any long-term usage:

1. **Restrict NLB Access**: Update NLB security groups to allow only specific source IPs
2. **VPN/Direct Connect**: Set up VPN or Direct Connect for secure management access
3. **Network ACLs**: Add subnet-level network ACLs for defense in depth
4. **VPC Flow Logs**: Enable VPC Flow Logs for network traffic analysis
5. **AWS WAF**: Deploy AWS WAF in front of Console NLB for web application protection
6. **SSM Session Logging**: Configure CloudWatch Logs for SSM session activity
7. **IAM Policies**: Restrict SSM access to specific users or roles

## Cost Considerations

This deployment includes the following AWS resources that incur costs:

- **EC2 Instances**: 8 instances (default m5.2xlarge = ~$0.384/hour each)
- **EBS Volumes**: 8 additional gp3 volumes (default 300GB each = ~$24/month per volume)
- **NAT Gateways**: 2 NAT Gateways (~$0.045/hour each + data transfer)
- **Network Load Balancers**: 2 NLBs (~$0.0225/hour each + LCU charges)
- **Elastic IPs**: 2 EIPs for NAT Gateways (free when attached)
- **Data Transfer**: Inter-AZ and internet data transfer charges

**Estimated Monthly Cost**: ~$2,700-3,200 (varies by region and usage)

### Cost Optimization Tips

1. Use Reserved Instances or Savings Plans for EC2 instances
2. Right-size instance types and EBS volumes based on actual QRadar requirements
3. Consider using VPC endpoints to reduce NAT Gateway data transfer
4. Monitor and optimize data transfer between AZs
5. Adjust EBS volume size based on actual data retention needs

## Troubleshooting

### Stack Creation Fails

1. **Check CloudFormation Events**: Review error messages in the Events tab
2. **Verify Prerequisites**: Ensure Key Pair exists in the target region
3. **Check Service Limits**: Verify you haven't exceeded EC2 or VPC limits
4. **IAM Permissions**: Ensure you have permissions to create all resources

### Cannot Access Instances via SSM

1. **Check IAM Role**: Verify the EC2 instance has the correct IAM instance profile attached
2. **SSM Agent**: Ensure SSM agent is running on the instance (pre-installed on QRadar AMIs)
3. **Internet Connectivity**: Verify instances can reach SSM endpoints via NAT Gateway
4. **VPC Endpoints**: Consider adding VPC endpoints for SSM to avoid NAT Gateway costs:
   - com.amazonaws.region.ssm
   - com.amazonaws.region.ssmmessages
   - com.amazonaws.region.ec2messages

### Cannot Access Instances via SSH

1. **Check Security Groups**: Verify security group rules allow SSH traffic
2. **Verify Route Tables**: Ensure route tables are correctly configured
3. **NAT Gateway Status**: Confirm NAT Gateways are in "available" state
4. **Network ACLs**: Check for restrictive Network ACL rules
5. **Key Pair**: Ensure you're using the correct SSH key pair
6. **Serial Port**: If SSH is still not accessible, use serial console, very vrouter configuration

### Verify VRouter Configuration ###

1. **Local Interfaces**: Verify that there are `peth0` and `veth0` interfaces. 
   - `peth0` should have the IP address of the QRadar host on the overlay network (e.g: from `192.168.60.0/23`)
   - `veth0` should be connected to `peth0` as a bridge member.
2. **vrouter Interfaces**: Within the `vrouter` network namespace, verify that there are `eth0`, `bridge0`, and `veth1` interfaces. 
   - `bridge0` should have the overlay gateway address (e.g: `192.168.60.1`)
   - `veth1` should be connected to `bridge0` as a bridge member. 
   - `eth0` should have the AWS assigned private IP address for the EC2 instance (e.g: from `10.0.11.0/24` or `10.0.12.0/24`).
3. **vrouter Firewall**: within the `vrouter` network namespace, verify that there are iptables NAT rules that forward ports 22 and 443 to the QRadar host IP address on the overlay network (there may be other ports forwarded as well).
4. **vrouter Tunnels**: within the `vrouter` network namespace, verify that there are:
   - an `overlay0` interface with the remote address of the HA peer
   - `overlayXX` interfaces where XX is a QRadar internal host id (`51`, `52`, `53`, ...) and the remote IP corresponds to that host. 
   - The exact number of and names of the `overlayXX` interfaces depends on the number of QRadar hosts in the HA cluster and the order in which they were added.
   - This is an advanced troubleshooting step which is detailed in XXX.

### Load Balancer Health Checks Failing

1. **Instance Status**: Verify EC2 instances are running
2. **Security Groups**: Ensure instances allow traffic from NLB security groups
3. **QRadar Services**: Confirm QRadar services are running and listening on correct ports
4. **Target Group Configuration**: Verify target group health check settings

## Cleanup

To delete all resources created by this stack:

```bash
aws cloudformation delete-stack --stack-name qradar-ha-stack
```

**Warning**: This will permanently delete all resources, including EC2 instances and their data. Ensure you have backups before deleting.

## Architecture Diagram

See `NETWORK_ARCHITECTURE.md` for detailed architecture documentation and diagrams.

## Support and Documentation

- [AWS CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)
- [IBM QRadar Documentation](https://www.ibm.com/docs/en/qradar-common)
- [QRadar High Availability Guide](https://www.ibm.com/docs/en/qradar-common?topic=qradar-high-availability)

## License

This reference architecture is provided as-is for educational and reference purposes. Copyright © 2023 IBM Corporation.

## Notes

- The overlay network and IPSec tunnels mentioned in the architecture documentation cannot be automated with CloudFormation and must be configured manually after deployment
- The default AMI is QRadar 7.6 for testing purposes; replace with more recent QRadar AMI if applicable
- Instance types can be adjusted based on QRadar sizing requirements
- Consider implementing additional AWS services like AWS Backup, AWS Systems Manager, and Amazon CloudWatch for enhanced operations