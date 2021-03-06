# VPC Builder

Builds out a "fully" featured VPC summarising the complexity associated with a VPC such as Internet & Customer Gateways, Subnets, Routetables and NATGateways.

It also adds in VPC Flowlogs with an IAM role and supports full dynamic allocation of IPv6 with the VPC and to each subnet.

The IPv6 handles Egress Internet Gateway and default route against ::/0

## Deployment
Notice that in [`vpc.yaml`](./vpc.yaml) the resources is not and AWS supported type.  Its specification is defined in [`transform.yaml`](./transform.yaml).  In order to submit cloudformation templates using this `Type` you must have the transform deployed first.

### Deploy the Transform

First make an S3 bucket and add it to the `makefile` at `BUCKET_NAME`.  In order for the deployment to work you must also set `USER_PROFILE` (use `default` if not required).

Deploy the transform:

```bash
make deployTransform
```

### Use the transform to build a VPC

#### Configure your Template

Open `vpc.yaml` and replace the account number with your own so that the template knows which transform to apply:

```yaml
Transform: "<ACCT#_goes_here>::VPC"
```

#### Virtual Private Gateway

Ideally you should never spin up a VPGW in Cloudformation. If you ever plan to attach it to a Direct Connect Virtual Interface you wont be able to tear up & down the VPC without destroying the VIF attachment. Either by hand in the console (shudder) or ideally via the CLI/SDK call with the following

```bash
aws ec2 create-vpn-gateway --type ipsec.1 --amazon-side-asn <AWS BGP ASN>
```

You can omit the AWS BOP ASN if you're not sure what you would like to make it and can happily utilise the standard ASN provided by AWS.

```bash
aws ec2 create-vpn-gateway --type ipsec.1
```

**Make a note of the "VpnGatewayId" that is returned, you will need it in the final step**

#### Build the VPC

Now that the transform is deployed, you have created a VPN gateway and you have enabled the `vpc.yaml` to use it you are ready to build the VPC.

```bash
aws cloudformation deploy \
        --capabilities CAPABILITY_IAM \
        --template-file vpc.yaml \
        --stack-name 'VPC' \
        --parameter-overrides VGW=vgw-05811955cb8607072   # Replace with your VpnGatewayId
```

## To Do

Add Outputs with Exports for critical resources
VPC Endpoints for all AWS Services
Add a little better handling of custom pieces (e.g. different route gateways)
Adding proper IPv6 regex and handling with NetworkACLs

## Basic Usage

Utilise the yaml structure below as a template, changing the Account ID in the transformation definition.
It will support the removal of Subnets, RouteTables, NATGateways and NetworkACLs.

## Network ACL Breakdown

```yaml
RuleName: "rule_number,protocol_number,[allow|deny],egress[true|false],cidr[0-255.0-255.0-255.0-255/0-32],from_port,to_port"
```

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Private VPC Template
Parameters:
  VGW: {Description: VPC Gateway, Type: String, Default: vgw-012345678}
Mappings: {}
Resources:

    ELENDELBUILDVPC:
        Type: Elendel::Network::VPC
        Properties:
            CIDR: 172.16.0.0/20
            Details: {VPCName: PRIVATEEGRESSVPC, VPCDesc: Private Egress VPC, Region: ap-southeast-2, IPv6: True}
            Tags: {Name: PRIVATE-EGRESS-VPC, Template: VPC for private endpoints egress only}
            DHCP: {Name: DhcpOptions, DNSServers: 172.16.0.2, NTPServers: 169.254.169.123, NTBType: 2}
            Subnets:
                ReservedMgmt1: {CIDR: 172.16.0.0/26, AZ: 0, NetACL: InternalSubnetAcl, RouteTable: InternalRT1 }
                ReservedMgmt2: {CIDR: 172.16.1.0/26, AZ: 1, NetACL: InternalSubnetAcl, RouteTable: InternalRT2 }
                ReservedMgmt3: {CIDR: 172.16.2.0/26, AZ: 2, NetACL: InternalSubnetAcl, RouteTable: InternalRT3 }
                ReservedNet1: {CIDR: 172.16.0.192/26, AZ: 0, NetACL: RestrictedSubnetAcl, RouteTable: PublicRT }
                ReservedNet2: {CIDR: 172.16.1.192/26, AZ: 1, NetACL: RestrictedSubnetAcl, RouteTable: PublicRT }
                ReservedNet3: {CIDR: 172.16.2.192/26, AZ: 2, NetACL: RestrictedSubnetAcl, RouteTable: PublicRT }
                Internal1: {CIDR: 172.16.3.0/24, AZ: 0, NetACL: InternalSubnetAcl, RouteTable: InternalRT1 }
                Internal2: {CIDR: 172.16.4.0/24, AZ: 1, NetACL: InternalSubnetAcl, RouteTable: InternalRT2 }
                Internal3: {CIDR: 172.16.5.0/24, AZ: 2, NetACL: InternalSubnetAcl, RouteTable: InternalRT3 }
                PerimeterInternal1: {CIDR: 172.16.6.0/24, AZ: 0, NetACL: InternalSubnetAcl, RouteTable: InternalRT1 }
                PerimeterInternal2: {CIDR: 172.16.7.0/24, AZ: 1, NetACL: InternalSubnetAcl, RouteTable: InternalRT2 }
                PerimeterInternal3: {CIDR: 172.16.8.0/24, AZ: 2, NetACL: InternalSubnetAcl, RouteTable: InternalRT3 }
            RouteTables:
                PublicRT:
                  - RouteName: PublicRoute
                    RouteCIDR: 0.0.0.0/0
                    RouteGW: InternetGateway
                  - RouteName: PublicRouteIPv6
                    RouteCIDR: ::/0
                    RouteGW: InternetGateway
                InternalRT1:
                InternalRT2:
                InternalRT3:
            NATGateways:
                NATGW1:
                    {Subnet: ReservedNet1, Routetable: InternalRT1}
                NATGW2:
                    {Subnet: ReservedNet2, Routetable: InternalRT2}
                NATGW3:
                    {Subnet: ReservedNet3, Routetable: InternalRT3}
            SecurityGroups:
                VPCEndpoint:
                    GroupDescription: VPC Endpoint Interface Firewall Rules
                    SecurityGroupIngress:
                    - [icmp,-1,-1,172.16.0.0/20, All ICMP Traffic]
                    - [tcp,0,65535,172.16.0.0/20, All TCP Traffic]
                    - [udp,0,65535,172.16.0.0/20, All UDP Traffic]
                    SecurityGroupEgress:
                    - [icmp,-1,-1,172.16.0.0/20, All ICMP Traffic]
                    - [tcp,0,65535,172.16.0.0/20, All TCP Traffic]
                    - [udp,0,65535,172.16.0.0/20, All UDP Traffic]
                    Tags:
                      Name: VPCEndpoint
            Endpoints:
                cloudformation:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                cloudtrail:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                codebuild:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                config:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                dynamodb:
                    Type: Gateway
                    RouteTableIds:
                      - PublicRT
                      - InternalRT1
                      - InternalRT2
                      - InternalRT3
                    PolicyDocument: |
                        {
                            "Version":"2012-10-17",
                            "Statement":[
                                {
                                    "Effect":"Allow",
                                    "Principal": "*",
                                    "Action":["s3:*"],
                                    "Resource":["*"]
                                }
                            ]
                        }
                ec2:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                ec2messages:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                elasticloadbalancing:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                events:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                execute-api:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                kinesis-streams:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                kms:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                logs:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                monitoring:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                sagemaker.api:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                sagemaker.runtime:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                s3:
                    Type: Gateway
                    RouteTableIds:
                      - PublicRT
                      - InternalRT1
                      - InternalRT2
                      - InternalRT3
                    PolicyDocument: |
                        {
                            "Version":"2012-10-17",
                            "Statement":[
                                {
                                    "Effect":"Allow",
                                    "Principal": "*",
                                    "Action":["s3:*"],
                                    "Resource":["*"]
                                }
                            ]
                        }
                secretsmanager:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                servicecatalog:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                sns:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                ssm:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
                ssmmessages:
                    Type: Interface
                    SubnetIds:
                      - ReservedMgmt1
                      - ReservedMgmt2
                      - ReservedMgmt3
                    SecurityGroupIds:
                      - VPCEndpoint
            NetworkACLs:
                RestrictedSubnetAcl:
                    RestrictedSubnetAclEntryInTCPUnReserved: "90,6,allow,false,0.0.0.0/0,1024,65535"
                    RestrictedSubnetAclEntryInUDPUnReserved: "91,17,allow,false,0.0.0.0/0,1024,65535"
                    RestrictedSubnetAclEntryInTCPUnReservedIPv6: "92,6,allow,false,::/0,1024,65535"
                    RestrictedSubnetAclEntryInUDPUnReservedIPv6: "93,17,allow,false,::/0,1024,65535"
                    RestrictedSubnetAclEntryOutTCPUnReserved: "90,6,allow,true,0.0.0.0/0,1024,65535"
                    RestrictedSubnetAclEntryOutUDPUnReserved: "91,17,allow,true,0.0.0.0/0,1024,65535"
                    RestrictedSubnetAclEntryOutTCPUnReservedIPv6: "92,6,allow,true,::/0,1024,65535"
                    RestrictedSubnetAclEntryOutUDPUnReservedIPv6: "93,17,allow,true,::/0,1024,65535"
                    RestrictedSubnetAclEntryOutPuppet: "94,6,allow,true,172.16.0.0/16,8140,8140"
                    RestrictedSubnetAclEntryOutHTTP: "101,6,allow,true,0.0.0.0/0,80,80"
                    RestrictedSubnetAclEntryOutHTTPS: "102,6,allow,true,0.0.0.0/0,443,443"
                    RestrictedSubnetAclEntryOutSSH: "103,6,allow,true,0.0.0.0/0,22,22"
                    RestrictedSubnetAclEntryOutHTTPIPv6: "104,6,allow,true,::/0,80,80"
                    RestrictedSubnetAclEntryOutHTTPSIPv6: "105,6,allow,true,::/0,443,443"
                    RestrictedSubnetAclEntryOutSSHIPv6: "106,6,allow,true,::/0,22,22"
                    RestrictedSubnetAclEntryInHTTP: "101,6,allow,false,0.0.0.0/0,80,80"
                    RestrictedSubnetAclEntryInHTTPS: "102,6,allow,false,0.0.0.0/0,443,443"
                    RestrictedSubnetAclEntryInHTTPIPv6: "103,6,allow,false,::/0,80,80"
                    RestrictedSubnetAclEntryInHTTPSIPv6: "104,6,allow,false,::/0,443,443"
                    RestrictedSubnetAclEntryIn: "110,-1,allow,false,172.16.0.0/16,1,65535"
                    RestrictedSubnetAclEntryOut: "110,-1,allow,true,172.16.0.0/16,1,65535"
                    RestrictedSubnetAclEntryNTP: "120,6,allow,true,0.0.0.0/0,123,123"
                    RestrictedSubnetAclEntryInSquid2: "140,6,allow,false,172.16.0.0/16,3128,3128"
                    RestrictedSubnetAclEntryInDNSTCP: "150,6,allow,false,172.16.0.0/16,53,53"
                    RestrictedSubnetAclEntryOutDNSTCP: "150,6,allow,true,0.0.0.0/0,53,53"
                    RestrictedSubnetAclEntryOutDNSTCPIPv6: "151,6,allow,true,::/0,53,53"
                    RestrictedSubnetAclEntryInDNSUDP: "160,17,allow,false,172.16.0.0/16,53,53"
                    RestrictedSubnetAclEntryOutDNSUDP: "160,17,allow,true,0.0.0.0/0,53,53"
                    RestrictedSubnetAclEntryOutDNSUDPIPv6: "161,17,allow,true,::/0,53,53"
                    RestrictedSubnetAclEntryInNetBios: "170,6,allow,false,172.16.0.0/16,389,389"
                    RestrictedSubnetAclEntryOutNetBios: "170,6,allow,true,172.16.0.0/16,389,389"
                    RestrictedSubnetAclEntryInNetBios1: "80,6,allow,false,172.16.0.0/16,137,139"
                    RestrictedSubnetAclEntryOutNetBios1: "180,6,allow,true,172.16.0.0/16,137,139"
                InternalSubnetAcl:
                    InternalSubnetAclEntryIn: "100,-1,allow,false,172.16.0.0/16,1,65535"
                    InternalSubnetAclEntryOut: "100,-1,allow,true,172.16.0.0/16,1,65535"
                    InternalSubnetAclEntryInTCPUnreserved: "102,6,allow,false,0.0.0.0/0,1024,65535"
                    InternalSubnetAclEntryInUDPUnreserved: "103,17,allow,false,0.0.0.0/0,1024,65535"
                    InternalSubnetAclEntryInTCPUnreservedIPv6: "104,6,allow,false,::/0,1024,65535"
                    InternalSubnetAclEntryInUDPUnreservedIPv6: "105,17,allow,false,::/0,1024,65535"
                    InternalSubnetAclEntryOutHTTP: "102,6,allow,true,0.0.0.0/0,80,80"
                    InternalSubnetAclEntryOutHTTPS: "103,6,allow,true,0.0.0.0/0,443,443"
                    InternalSubnetAclEntryOutHTTPIPv6: "104,6,allow,true,::/0,80,80"
                    InternalSubnetAclEntryOutHTTPSIPv6: "105,6,allow,true,::/0,443,443"
                    InternalSubnetAclEntryOutTCPUnreserved: "106,6,allow,true,172.16.0.0/16,1024,65535"
                    InternalSubnetAclEntryOutUDPUnreserved: "107,6,allow,true,172.16.0.0/16,1024,65535"
                    InternalSubnetAclEntryOutTCPDNS: "110,6,allow,true,0.0.0.0/0,53,53"
                    InternalSubnetAclEntryOutUDPDNS: "111,17,allow,true,0.0.0.0/0,53,53"
                    InternalSubnetAclEntryOutTCPDNSIPv6: "112,6,allow,true,::/0,53,53"
                    InternalSubnetAclEntryOutUDPDNSIPv6: "113,17,allow,true,::/0,53,53"
                    InternalSubnetAclEntryOutSSH: "150,6,allow,true,0.0.0.0/0,22,22"
Transform: "012345678901::VPC"
```
