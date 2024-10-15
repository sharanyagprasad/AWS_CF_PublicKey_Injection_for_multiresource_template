
# How to Inject a Public Key into EC2 using CloudFormation on Windows: A Step-by-Step Guide

## 1. Introduction:

When launching an EC2 instance, secure SSH access is crucial. While AWS provides multiple ways to manage SSH keys, using CloudFormation to inject a public key into your instance is a scalable solution. In this guide, we’ll walk you through how to generate SSH keys on Windows and use a CloudFormation template to inject the public key into an EC2 instance. We’ll also explore the benefits and limitations of this approach to help you make an informed decision.


## 2. Prerequisites:
List out what is needed before starting this implementation:

- AWS Account with appropriate permissions (for CloudFormation, EC2, and VPC).
- AWS CLI installed and configured.
- Basic knowledge of YAML, CloudFormation templates, and SSH.
- Access to PowerShell (for Windows users).
- Access to VSCode to create the code.

## 3. Drawbacks of Creating Key Pairs via AWS CloudFormation:
- CloudFormation allows users to create EC2 key pairs automatically, but these key pairs cannot be downloaded after creation.
- Once a private key is lost, there's no way to retrieve it, which can lock users out of their instances.
- In contrast, injecting a pre-generated public key allows for better control and security as you retain the private key on your local machine.

## 4. Why Inject a Public Key Using CloudFormation?

- **Automation and Scalability**: Injecting a public key into an EC2 instance via a CloudFormation template enables you to automate instance creation while maintaining secure SSH access.
- **Security Best Practices**: Public key cryptography ensures that only someone with the corresponding private key can access the instance.
- **Consistency**: Automating key-pair injection in CloudFormation ensures uniformity across multiple instances, especially useful in autoscaling groups or in environments with rapidly changing infrastructure.

## 5. Pros and Cons of Injecting Public Keys via CloudFormation:
We’ll analyze the benefits and potential pitfalls of this approach so readers can understand when to use this method.

### Pros:
- **Enhanced Security**: You control the private key, ensuring that no third-party can access it.
Automated Deployments: CloudFormation templates allow for automated deployments, reducing manual errors.
- **Flexibility**: Works seamlessly in multi-region or multi-account environments.
### Cons:
- **Requires Pre-Generated Keys**: You need to generate and manage the SSH keys manually, as CloudFormation cannot generate private keys for you.
- **No Private Key Recovery**: If the private key is lost, there's no fallback, and you will be locked out of the instance.

## 6. Architecture Overview:
This setup uses AWS CloudFormation to securely launch and access EC2 instances with SSH keys. Here's a breakdown:

- **VPC (Virtual Private Cloud)**: Defines an isolated network with a CIDR block (10.0.0.0/16), allowing for flexible subnet creation.
- **Subnet**: Houses the EC2 instance within the VPC, using a smaller CIDR block (10.0.1.0/24), ensuring network segmentation.
- **Security Group**: Acts as a firewall, allowing SSH access (port 22) from any IP. This can be further restricted for security.
- **EC2 Instance**: A t2.micro instance, cost-effective for small applications, created within the defined subnet and security group.
- **Key Pair**: Injects a pre-generated SSH public key into the instance, providing full control over the private key on the local machine.

## 7. Real-World Use Case:


- **Infrastructure-as-Code (IaC) Environments**: Automating infrastructure provisioning using CloudFormation ensures consistency and repeatability across multiple AWS accounts and regions. Every deployment of the EC2 instance is guaranteed to follow the same configuration, which is crucial in environments with strict compliance requirements.

- **DevOps Pipelines**: In continuous integration/continuous deployment (CI/CD) pipelines, developers often need automated EC2 instance creation. Injecting SSH keys into instances via CloudFormation helps developers easily access instances during debugging or testing phases.

- **Enterprise Use Cases**: Large enterprises often manage multiple AWS accounts and regions. Injecting public keys into instances via CloudFormation simplifies the process of securely accessing instances without the risk of losing private keys, ensuring a more streamlined DevOps workflow.

- **Temporary or Auto-scaling Instances**: In scenarios where instances need to scale up or down automatically (e.g., auto-scaling groups), this approach ensures that all instances are provisioned with secure access, allowing admins to SSH into any instance in the auto-scaling group as needed.







## Step 1: Generate SSH Key Pair on Windows

1. Open PowerShell.
2. Change directory to the `.ssh` folder by running:

    ```powershell
    cd ~/.ssh
    ```
    This command checks for an existing .ssh folder.

3. Check the contents of the `.ssh` directory:

    ```powershell
    dir ~/.ssh
    ```

    If you get an error or no folder exists, follow the steps below to create it.

    ### If the `.ssh` Folder Doesn't Exist:
    Run the following to create the `.ssh` folder (skip if already exists):

    ```powershell
    mkdir C:\Users\<YourUsername>\.ssh
    ```

    Replace `<YourUsername>` with your actual username. Then navigate to it:

    ```powershell
    cd C:\Users\<YourUsername>\.ssh
    ```

4. Generate SSH Key Pair

    Generate the SSH key pair (private and public key) by running:

    ```powershell
    ssh-keygen -t rsa -b 2048 -f C:\Users\<YourUsername>\.ssh\my-key-pair
    ```

- You will be prompted for a passphrase. Press **Enter** to skip (or enter one for security).
- This will create two files:
  - `my-key-pair` (Private key)
  - `my-key-pair.pub` (Public key)

## Step 2: Verify the Key Files and  Copy the Public Key

List the contents of the `.ssh` folder to ensure both files have been created:

```powershell
dir C:\Users\<YourUsername>\.ssh
```

You should see:

```
my-key-pair
my-key-pair.pub
```

- **View and Copy the Public Key**

Open the public key file to view and copy its contents:

```powershell
notepad C:\Users\<YourUsername>\.ssh\my-key-pair.pub
```

This file will contain a long string that starts with `ssh-rsa`. Copy this string as you will need it for the next step.

## Step 3: Inject the Public Key into CloudFormation Template

 CloudFormation template (publickeyInjection_Ec2.yaml) will include the public key you just generated. This template creates the necessary AWS resources (VPC, Subnet, Security Group, EC2 instance) and injects the SSH public key into the instance.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Create EC2 instance with SSH access using public key

Resources:
  MyVPC:
    Type: "AWS::EC2::VPC"
    Properties:
      CidrBlock: "10.0.0.0/16"
      Tags:
        - Key: "Name"
          Value: "MyVPC"

  MySubnet:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: "10.0.1.0/24"
      AvailabilityZone: "eu-central-1a"
      Tags:
        - Key: "Name"
          Value: "MySubnet"

  MyEC2SecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: "Allow SSH"
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: "tcp"
          FromPort: "22"
          ToPort: "22"
          CidrIp: "0.0.0.0/0"

  MyEC2Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      InstanceType: "t2.micro"
      KeyName: !Ref MyKeyPair
      SubnetId: !Ref MySubnet
      SecurityGroupIds:
        - !Ref MyEC2SecurityGroup
      ImageId: "ami-0c55b159cbfafe1f0" # specific to the availability zone

  MyKeyPair:
    Type: "AWS::EC2::KeyPair"
    Properties:
      KeyName: "MyKeyPair"
      PublicKeyMaterial: "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEArH... (Your Public Key Here)"
```

### Step-by-Step Code Explanation:

- **MyVPC**: Creates a VPC with a CIDR block of `10.0.0.0/16`.
- **MySubnet**: A subnet within the VPC with a smaller CIDR block of `10.0.1.0/24` located in `eu-central-1a` availability zone.
- **MyEC2SecurityGroup**: A security group allowing SSH traffic on port 22.
- **MyEC2Instance**: Creates a `t2.micro` instance with the security group, key pair, and subnet.
- **MyKeyPair**: Injects your public key (generated earlier) into the EC2 instance, enabling SSH access.

## Step 4: Deploy the CloudFormation Stack

Deploy the CloudFormation stack using either the AWS Console or CLI.

- **Via AWS CLI**:

    ```bash
    aws cloudformation create-stack --stack-name MyEC2Stack --template-body file://publickeyInjection_Ec2.yaml
    ```

## Step 5: SSH into the EC2 Instance

Once the stack is deployed, find the public IP address of the EC2 instance and connect via SSH using the private key.

```bash
ssh -i C:\Users\<YourUsername>\.ssh\my-key-pair ec2-user@<Public-IP>
```

Replace `<Public-IP>` with the actual public IP address of your EC2 instance.

---

Now you’ve successfully set up a CloudFormation stack, injected your public key into an EC2 instance, and connected to it via SSH. This method ensures full control over key management, crucial for secure cloud deployments.

Visit my Github repo [AWS_CF_PublicKey_Injection_for_multiresource_template](https://github.com/sharanyagprasad/AWS_CF_PublicKey_Injection_for_multiresource_template) for the current implementation.

Feel free to explore these:
- [An Introduction to AWS CloudFormation: Laying the Foundation for Cloud Infrastructure](https://aws-iac-cloudformation-introduction.hashnode.dev/an-introduction-to-aws-cloudformation-laying-the-foundation-for-cloud-infrastructure) for Infrastructure as Code and AWS CF Basics.
- [AWS CloudFormation MultiResource Stack](https://aws-iac-cloudformation-introduction.hashnode.dev/aws-cloudformation-a-guide-to-multi-resource-deployment-ec2-s3-iam-vpc) for more advanced use cases.

