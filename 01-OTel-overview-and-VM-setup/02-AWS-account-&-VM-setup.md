# **1. AWS Account Creation**

## **Step-by-Step AWS Account Creation**

### **Step 1: Visit the AWS Sign-Up Page**
- Go to [AWS Sign-Up](https://aws.amazon.com/)
- Click on **Create an AWS Account**

### **Step 2: Enter Your Email & Account Details**
- Provide a valid email address
- Choose a **Root user** name (preferably your full name or business name)
- Set a strong password
- Click **Continue**

### **Step 3: Contact Information**
- Choose **Personal** or **Business** account type
- Fill in your **full name, phone number, and address**
- Click **Continue**

### **Step 4: Payment Information**
- Enter a valid credit or debit card (AWS requires this for identity verification)
- AWS may place a small temporary charge for verification
- Click **Continue**

### **Step 5: Identity Verification**
- Enter the **OTP** received via SMS or a phone call
- Complete the CAPTCHA verification
- Click **Verify**

### **Step 6: Choose a Support Plan**
- Select **Basic (Free)** unless you need premium support
- Click **Continue**

### **Step 7: Access Your AWS Console**
- Once verified, go to the [AWS Console](https://aws.amazon.com/console/)
- Log in using your email and password
- Explore AWS services!

## **Common Challenges & How to Overcome Them**

### **1. Credit/Debit Card Issues**
- Ensure your card supports international transactions
- Some virtual or prepaid cards may not work
- Contact your bank if the payment is declined

### **2. OTP or Phone Verification Problems**
- Use a phone number that can receive international SMS
- Try an alternate number if OTP does not arrive
- Check spam/junk SMS folders

### **3. Billing & Unexpected Charges**
- AWS Free Tier is available but has usage limits
- Set up **Billing Alerts** in AWS to track spending
- Enable **IAM Users** for better security (you will learn in next lectures)

### **4. Account Activation Delays**
- Sometimes, AWS takes a few hours to activate your account
- If delayed, check your email for verification requests
- Contact AWS Support if activation exceeds 24 hours
  

# **2. AWS IAM User Creation**

## **Step-by-Step IAM User Creation**

### **Step 1: Log in to AWS Console**
- Go to [AWS Management Console](https://aws.amazon.com/console/)
- Sign in with your **Root Account**
- Navigate to **IAM Dashboard**

### **Step 2: Open IAM Users Section**
- Click on **Users** in the left navigation pane
- Click **Add User**

### **Step 3: Enter User Details**
- Provide a **User Name** (e.g., `devops-user`)
- Select **AWS Credential Type**:
  - **Access Key – Programmatic Access** (for CLI, SDKs, API access)
  - **Password – AWS Management Console Access** (for GUI access)
- Click **Next**

### **Step 4: Assign Permissions**
- Choose how to set permissions:
  - **Attach Existing Policies Directly** (e.g., `AdministratorAccess`, `PowerUserAccess`, `ReadOnlyAccess`)
  - **Add to a Group** (recommended for better management)
  - **Copy from Another User**
- Click **Next**

### **Step 5: Add Tags (Optional)**
- Assign metadata like `Department: DevOps`, `Project: E-Commerce`
- Click **Next**

### **Step 6: Review and Create User**
- Review all details carefully
- Click **Create User**

### **Step 7: Download Credentials**
- Download the **Access Key ID & Secret Access Key** (if programmatic access is enabled)
- Save these credentials securely (they won’t be shown again)
- Click **Close**

# **3. AWS EC2 Instance Setup**

## **Step-by-Step EC2 Instance Setup**

### **Step 1: Log in to AWS Console**
- Go to [AWS Management Console](https://aws.amazon.com/console/)
- Sign in with your AWS account credentials
- Navigate to **EC2 Dashboard**

### **Step 2: Launch a New EC2 Instance**
- Click **Launch Instance**
- Enter an instance name

### **Step 3: Choose an Amazon Machine Image (AMI)**
- Select an OS
- Click **Select**

### **Step 4: Choose an Instance Type**
- Select a suitable instance type:
  - **t3.xlarge+** (For heavier workloads)
- Click **Next**

### **Step 5: Configure Instance Details**
- Keep the default **VPC & Subnet** (unless customizing networking)
- Enable **Auto-assign Public IP** (for internet access)
- Click **Next**

### **Step 6: Add Storage**
- Default storage: 8GB but you can change that to 30 GB as we will download a lot of container images related to the project.
- Click **Next**

### **Step 7: Configure Security Group**
- Create a new security group or use an existing one
- Allow **SSH (port 22)** for Linux instances

### **Step 8: Generate or Select a Key Pair**
- Select **Create a new key pair**
- Choose **RSA** format and download the `.pem` file
- **Store the key securely** (You will need it to access your instance)
- Click **Launch Instance**

### **Step 9: Connect to Your EC2 Instance**
- Navigate to **EC2 Dashboard** → **Instances**
- Select the instance and click **Connect**

#### **For Linux Instances:**
- Use SSH from terminal:
  ```bash
  ssh -i /path/to/your-key.pem ec2-user@your-instance-public-ip
  ```

## **Common Challenges & How to Overcome Them**

### **1. Key Pair Issues**
- If you lose your `.pem` file, you **cannot recover it**
- Create a new key pair and manually update the instance’s key (requires console access)

### **2. SSH Connection Problems**
- Ensure **Port 22** is open in the security group
- Use `chmod 400 your-key.pem` to set the correct permissions for the key file
- Verify the correct **public IP address** is used

### **3. EC2 Instance Not Accessible via Internet**
- Ensure **Auto-assign Public IP** was enabled
- Check **Security Group & NACL rules**
- Consider using **Elastic IP** for a static IP address

### **4. High Costs Due to Running Instances**
- Stop or terminate unused instances
- Use **AWS Billing Dashboard** to monitor usage
- Set up **AWS Budgets & Alerts**

