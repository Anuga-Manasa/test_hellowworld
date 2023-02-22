Static website url:https://hello-world-2693142890.s3.amazonaws.com/index.html

first I went to I am >> users and added a user by specifying the name and attach existing policy directly and I have given adminstrator access 
after creating a user In the security credentials I have added access key to allow programatic access 
next in terminal I have configured aws user using "aws config" command
I was asked to enter access key , secret access key and region

then I created python file like below and executed it using python hello-world.py
It displayed me with static website url and I had and vpc connection, ec2 instance and a bucket which can be used to host a static website


import json
import boto3
import paramiko

#creating an e2c client
aws_management_console = boto3.session.Session(profile_name="default")
ec2_client = aws_management_console.client(service_name="ec2")

# creating a vpc

vpc_id = ec2_client.create_vpc(CidrBlock='192.168.10.0/24')['Vpc']['VpcId']

#creating a subnet
subnet_id = ec2_client.create_subnet(CidrBlock='192.168.10.0/28', VpcId=vpc_id)['Subnet']['SubnetId']

# create an internet gateway and attach it to the VPC
gateway_response = ec2_client.create_internet_gateway()
gateway_id = gateway_response['InternetGateway']['InternetGatewayId']
ec2_client.attach_internet_gateway(InternetGatewayId=gateway_id, VpcId=vpc_id)

# create a route table and associate it with the subnet
route_table_response = ec2_client.create_route_table(VpcId=vpc_id)
route_table_id = route_table_response['RouteTable']['RouteTableId']
ec2_client.create_route(RouteTableId=route_table_id, DestinationCidrBlock='0.0.0.0/0', GatewayId=gateway_id)
ec2_client.associate_route_table(RouteTableId=route_table_id, SubnetId=subnet_id)



# create a security group that allows SSH access and HTTP access from anywhere
security_group_response = ec2_client.create_security_group(GroupName='ssh-http-access', Description='Security group for SSH and HTTP access', VpcId=vpc_id)
security_group_id = security_group_response['GroupId']
ec2_client.authorize_security_group_ingress(GroupId=security_group_id, IpProtocol='tcp', FromPort=22, ToPort=22, CidrIp='0.0.0.0/0')
ec2_client.authorize_security_group_ingress(GroupId=security_group_id, IpProtocol='tcp', FromPort=80, ToPort=80, CidrIp='0.0.0.0/0')

# creating a ec2 instance
instance = ec2_client.run_instances(
        ImageId="ami-0557a15b87f6559cf",
        MinCount=1,
        MaxCount=1,
        InstanceType="t2.micro",
        KeyName="Manu",
        NetworkInterfaces=[
        {
            'SubnetId': subnet_id,
            'DeviceIndex': 0,
            'AssociatePublicIpAddress': True,
            #'Groups': ['sg-0123456789abcdef']
        }]
    )
instance_id = instance['Instances'][0]['InstanceId']

ec2_client.get_waiter('instance_running').wait(InstanceIds=[instance_id])

# Get information about the instance
response = ec2_client.describe_instances(InstanceIds=[instance_id])

# Get the private DNS name of the instance
private_dns_name = response['Reservations'][0]['Instances'][0]['PrivateDnsName']

# Get the private IP address of the instance
private_ip_address = response['Reservations'][0]['Instances'][0]['PrivateIpAddress']

# create an S3 bucket and upload the index.html file
s3 = boto3.client('s3')
session = boto3.Session()
region = session.region_name
bucket_name = 'hello-world-2693142890'
s3.create_bucket(Bucket=bucket_name)

# set the website configuration for the bucket
website_configuration = {
    'IndexDocument': {'Suffix': 'index.html'}
}

s3.put_bucket_website(Bucket=bucket_name, WebsiteConfiguration=website_configuration)

# upload the index.html file to the bucket
index_file_content = '<html><head><title>Hello world!</title></head><body><h1>Hello world!</h1></body></html>'
s3.put_object(Body=index_file_content, Bucket=bucket_name, Key='index.html', ContentType='text/html')

# view the bucket website URL
print(f"S3 bucket URL: http://{bucket_name}.s3-website-{s3.meta.region_name}.amazonaws.com")



# install Nginx on the EC2 instance
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect(hostname=private_dns_name, username='ubuntu', key_filename='/new.ppk')
stdin, stdout, stderr = ssh.exec_command('sudo apt update && sudo apt install nginx')
stdout.channel.recv_exit_status()
ssh.close()





